# 基于BIND9：部署高效内网DNS服务器

> 本文是 Kubernetes 集群部署系列的第 00 篇，介绍如何使用 BIND9 在集群 Master 节点上部署内网 DNS，并让 Master、Worker 节点通过统一的域名访问集群主机。
>
> 适用场景：实验室、裸机 Kubernetes 集群、私有云或企业内部网络。本文以项目中的实际脚本为依据，重点讲清楚“做了什么、为什么这样做、如何验证以及出现问题时如何排查”。

---

## 1. 为什么 Kubernetes 集群需要内网 DNS

### 1.1 解决的问题

一个 Kubernetes 集群通常包含 Master、Worker、镜像仓库、存储服务和监控服务等多个节点。节点之间如果只依赖 IP 地址通信，会产生几个问题：

- 配置文件中到处都是 IP，难以阅读和维护；
- 节点 IP 发生变化时，需要修改大量配置；
- Harbor、API Server 等服务不容易使用统一的服务名称访问；
- TLS 证书通常签发给域名，直接使用 IP 可能导致证书名称不匹配；
- `kubeadm`、Containerd 和后续应用部署都需要稳定的网络连通性。

因此，本项目为集群规划了一个内部域名：

```text
cluster.example.internal
```

并为节点分配完整域名（FQDN）：

```text
k8s-master.cluster.example.internal
k8s-worker-1.cluster.example.internal
k8s-worker-2.cluster.example.internal
internal-registry.cluster.example.internal
```

DNS 负责把这些名称解析为节点的内网 IP。

### 1.2 本文完成的工作

本篇实际完成以下内容：

1. 在 Master 节点安装并运行 BIND9；
2. 创建 `cluster.example.internal` 权威 DNS Zone；
3. 建立 Master、Worker 和 Harbor 的 A 记录；
4. 将 Master、Worker 的系统 DNS 指向 `10.20.30.160`；
5. 使用 `dig`、`getent` 和 `resolvectl` 分层验证解析结果；
6. 为后续 `kubeadm` 初始化、Containerd 拉取 Harbor 镜像提供稳定的名称解析基础。

本文**不负责**安装 Kubernetes、配置 Containerd、部署 Calico，也不负责 Kubernetes 集群内部的 CoreDNS。它解决的是 Kubernetes 节点所在基础网络中的主机名解析问题。

---

## 2. 整体架构与网络地址规划

### 2.1 项目中的地址表

项目当前使用两类地址：

| 节点 | 集群内网 IP | 管理/SSH 地址 | DNS 名称 |
|---|---:|---:|---|
| Master | `10.20.30.160` | `192.168.50.160` | `k8s-master.cluster.example.internal` |
| Worker-1 | `10.20.30.161` | `192.168.50.161` | `k8s-worker-1.cluster.example.internal` |
| Worker-2 | `10.20.30.162` | `192.168.50.162` | `k8s-worker-2.cluster.example.internal` |
| Harbor | `10.20.30.169` | 未配置 | `internal-registry.cluster.example.internal` |

项目的地址规划可以概括为：

```text
10.20.30.0/24       集群节点内网
192.168.50.0/24     管理/SSH 网络（实际是否可用取决于环境分配）
```

其中：

- `10.20.30.x` 用于集群节点之间的内网通信和 DNS 解析结果；
- `192.168.50.x` 仅用于脚本从 Master SSH/SCP 到 Worker；
- BIND9 监听的 DNS 地址是 Master 的 `10.20.30.160`；
- DNS 记录返回的是 `10.20.30.x`，而不是用于 SSH 的管理地址。

这些地址是项目环境中**预先规划的静态地址**，不是 BIND9 或 Kubernetes 自动生成的。部署到其他环境时，必须根据真实的网络规划修改脚本，不能直接照搬。

### 2.2 私有地址的边界

`10.0.0.0/8` 属于 RFC 1918 私有地址范围，适合在企业内网、实验网络或云 VPC 内部使用。私有地址可以在彼此隔离的网络中重复，但在同一个二层网络、可路由网络、VPN、专线或 VPC 互联范围内必须唯一。

因此，下面两件事要区分：

- 其他完全隔离的公司网络也使用 `10.20.30.160`：通常没有问题；
- 当前集群所在网络中已经有另一台机器使用 `10.20.30.160`：会造成 IP 冲突。

`192.168.50.x` 是本文脱敏后的管理网络示例。实际环境中应使用由网络管理员分配的管理网地址，不能把示例地址直接用于生产。如果实际环境的管理网不同，应替换 `00-2_config_all_nodes_dns.sh` 中的 Worker 地址。

### 2.3 与 Kubernetes Pod、Service 网段分离

节点内网、Pod 网络和 Service 网络是三种不同的网络：

| 网络类型 | 示例 | 分配对象 |
|---|---|---|
| 节点内网 | `10.20.30.0/24` | Master、Worker、Harbor 的物理/虚拟网卡 |
| Pod 网段 | `192.168.0.0/16` 或 `10.244.0.0/16` | Calico 等 CNI 分配给 Pod |
| Service 网段 | 常见为 `10.96.0.0/12` | Kubernetes Service 的 ClusterIP |

三者不能重叠，否则可能出现路由选择错误。本文只配置节点内网 DNS，不负责决定 Pod CIDR 和 Service CIDR；后续 kubeadm、Calico 的规划必须避开 `10.20.30.0/24`。

---

## 3. DNS 和 BIND9 的核心原理

### 3.1 DNS 查询路径

本项目的查询路径如下：

```text
应用/命令
   |
   | getaddrinfo、getent、Containerd 等系统调用
   v
本机 resolver 配置（Netplan / systemd-resolved / resolv.conf）
   |
   | DNS 查询：UDP 53，必要时使用 TCP 53
   v
10.20.30.160:53（Master 上的 BIND9）
   |
   +-- cluster.example.internal：由本地权威 Zone 直接回答
   |
   +-- 其他外部域名：转发给 114.114.114.114 或 8.8.8.8
```

例如查询：

```bash
dig @10.20.30.160 k8s-worker-1.cluster.example.internal +short
```

BIND9 会在本地 `/etc/bind/db.cluster.example.internal` 中查找记录并返回：

```text
10.20.30.161
```

查询外部域名时，如果本地没有对应权威 Zone，BIND9 会按照 `forwarders` 配置把请求转发给上游 DNS。

### 3.2 权威解析与递归/转发解析

BIND9 在本文中同时承担两类角色：

1. **权威 DNS 服务器**：
   - 对 `cluster.example.internal` Zone 负责；
   - 直接回答本项目配置的节点记录；
   - 数据来源是本机 Zone 文件。

2. **递归/转发 DNS 服务器**：
   - 对不属于本地 Zone 的域名进行查询；
   - 将请求转发给 `114.114.114.114` 和 `8.8.8.8`；
   - 把结果缓存后返回给客户端。

本项目不是在公网注册 `cluster.example.internal`，而是在集群内部创建了一个同名的内部 Zone。只有把 DNS 查询指向 `10.20.30.160` 的节点，才能解析这些内部记录。

### 3.3 Zone、SOA、NS 和 A 记录

Zone 是 DNS 对某个域名命名空间的管理范围。本文的 Zone 是：

```text
cluster.example.internal
```

Zone 文件中几个关键记录的含义如下：

```bind
@       IN      SOA     k8s-master.cluster.example.internal. admin.cluster.example.internal. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      k8s-master.cluster.example.internal.

k8s-master           IN      A       10.20.30.160
k8s-worker-1         IN      A       10.20.30.161
k8s-worker-2         IN      A       10.20.30.162
internal-registry           IN      A       10.20.30.169
```

- `SOA`（Start of Authority）：声明 Zone 的权威起点和计时参数；
- `Serial`：Zone 版本号，修改记录后应递增；
- `NS`（Name Server）：声明该 Zone 的权威 DNS 服务器；
- `A`：将主机名映射到 IPv4 地址；
- `TTL`：记录可被缓存的时间，本文设置为 `604800` 秒，即 7 天；
- 末尾的点表示完整域名，避免 BIND9 将域名再次拼接 Zone 后缀。

### 3.4 `dig`、`getent` 和 `resolvectl` 的区别

| 命令 | 验证对象 | 是否绕过系统 resolver |
|---|---|---|
| `dig @10.20.30.160 name +short` | 指定 DNS 服务器是否能回答 | 是，直接查询指定服务器 |
| `getent hosts name` | 当前系统应用实际能否解析 | 否，使用系统名称解析配置 |
| `resolvectl status` | 当前系统配置的 DNS Server 和搜索域 | 查看配置状态 |

这三者必须分开理解：

- `dig` 成功，只能说明 BIND9 能回答；
- `getent` 成功，才说明当前节点的应用解析链路可用；
- `resolvectl` 可以确认节点是否真的把 `10.20.30.160` 配置成 DNS。

---

## 4. 实现组成与脚本职责

本篇由三个脚本组成：

```text
scripts/
├── 00-1_setup_dns_server.sh       # 在 Master 安装并配置 BIND9
├── 00-2_config_all_nodes_dns.sh   # 配置 Master 和 Worker 的系统 DNS
└── 00-3_verify_dns.sh              # 从多个层次验证 DNS
```

推荐始终按 `00-1 → 00-2 → 00-3` 的顺序执行。

> 文档中的 `/opt/k8s-dns/scripts/...` 是脱敏后的示例路径。实际执行时请替换为脚本真实位置。

---

## 5. 执行前置条件

### 5.1 网络条件

在执行脚本前，需要确认：

- Master 已配置 `10.20.30.160`；
- Worker-1 已配置 `10.20.30.161`；
- Worker-2 已配置 `10.20.30.162`；
- Master 到两个 Worker 的管理地址可达；
- 每台节点存在 `/etc/netplan/*.yaml`；
- Netplan 中至少有一个接口配置了 `10.20.30.x` 地址；
- 当前静态地址没有和 DHCP 地址池或其他服务器冲突。

检查命令：

```bash
ip -4 addr
ip route
ls -l /etc/netplan/
```

Worker 上应能看到对应的 `10.20.30.x` 地址。脚本并不会自动创建节点的内网网卡、交换机 VLAN、路由或网关。

### 5.2 软件和权限条件

脚本需要 root 权限。Master 初次安装 BIND9 前，还必须通过现有 DNS 访问 APT 软件源：

```bash
sudo -i
apt-get update
getent hosts archive.ubuntu.com
```

Master 和 Worker 都需要 Python、PyYAML 和 Netplan。第二个脚本会在 Worker 上运行临时 Python 配置器，其中包含：

```python
import yaml
```

因此需要提前安装：

```bash
apt-get update
apt-get install -y python3 python3-yaml netplan.io
```

三个脚本使用的常见命令包括 `bash`、`ssh`、`scp`、`python3`、`netplan`、`dig`、`getent`、`awk` 和 `resolvectl`。部分工具可能需要额外安装，例如：

```bash
apt-get install -y dnsutils openssh-client
```

### 5.3 SSH 条件

第二、第三个脚本会从 Master SSH 到：

```text
root@192.168.50.161
root@192.168.50.162
```

建议执行脚本前先验证：

```bash
ssh root@192.168.50.161 'hostname; id'
ssh root@192.168.50.162 'hostname; id'
```

需要满足：

- Worker 的 SSH 服务已启动；
- Master 到 Worker 的 TCP 22 端口可达；
- root 登录认证可用；
- SSH 密钥或其他认证方式已配置；
- 使用脚本时不会停在密码输入阶段。

脚本设置了 `StrictHostKeyChecking=no`，只会跳过主机指纹确认，不会绕过密码、公钥或权限认证。

### 5.4 配置备份

第二个脚本会修改 Netplan 文件，并执行 `netplan apply`。执行前建议备份：

```bash
cp -a /etc/netplan /etc/netplan.backup.$(date +%F-%H%M%S)
cp -a /etc/bind /etc/bind.backup.$(date +%F-%H%M%S) 2>/dev/null || true
```

`netplan apply` 可能造成网络短暂重配置，最好通过带外控制台或可靠的本地终端执行，避免 SSH 断开后无法恢复。

---

## 6. 第一步：在 Master 部署 BIND9

### 6.1 执行脚本

在 Master 上以 root 执行：

```bash
bash /opt/k8s-dns/scripts/00-1_setup_dns_server.sh
```

脚本对应文件为：

```text
scripts/00-1_setup_dns_server.sh
```

### 6.2 脚本做了什么

#### 安装软件包

```bash
apt-get update -y
apt-get install -y bind9 bind9-utils
```

`bind9` 提供 DNS 服务进程，`bind9-utils` 提供配置检查和诊断工具。

#### 配置全局选项

脚本生成 `/etc/bind/named.conf.options`：

```bind
options {
    directory "/var/cache/bind";
    allow-query { any; };
    forwarders { 114.114.114.114; 8.8.8.8; };
    listen-on { 127.0.0.1; 10.20.30.160; };
    dnssec-validation no;
    listen-on-v6 { any; };
};
```

各配置项的作用：

- `directory`：BIND9 工作目录；
- `allow-query`：允许哪些客户端查询，当前为任意来源；
- `forwarders`：本地没有权威答案时使用的上游 DNS；
- `listen-on`：BIND9 监听本机回环地址和 `10.20.30.160`；
- `dnssec-validation no`：关闭 DNSSEC 验证；这是当前脚本的明确取值，适合受控实验环境，但生产环境应按网络条件评估；
- `listen-on-v6`：允许监听 IPv6 地址。

#### 注册 Zone

脚本生成 `/etc/bind/named.conf.local`：

```bind
zone "cluster.example.internal" {
    type master;
    file "/etc/bind/db.cluster.example.internal";
};
```

这里的 `master` 表示本机是该 Zone 的主权威服务器，记录数据来自指定的 Zone 文件。

#### 生成 Zone 文件

脚本生成：

```text
/etc/bind/db.cluster.example.internal
```

并写入 Master、Worker 和 Harbor 的 A 记录。随后执行：

```bash
chown bind:bind /etc/bind/db.cluster.example.internal
systemctl restart named
systemctl enable named
```

其中 `restart` 让新配置立即生效，`enable` 确保服务随系统启动。

### 6.3 单独验证 BIND9

```bash
systemctl status named --no-pager
dig @127.0.0.1 k8s-master.cluster.example.internal +short
dig @127.0.0.1 k8s-worker-1.cluster.example.internal +short
```

预期分别得到：

```text
10.20.30.160
10.20.30.161
```

---

## 7. 第二步：配置所有节点使用 Master DNS

### 7.1 执行脚本

仍然在 Master 上执行：

```bash
bash /opt/k8s-dns/scripts/00-2_config_all_nodes_dns.sh
```

### 7.2 脚本的执行流程

脚本先定义：

```bash
DNS_SERVER='10.20.30.160'
SEARCH_DOMAIN='cluster.example.internal'
```

然后按照以下顺序操作：

1. 生成临时 Python 配置脚本；
2. 通过 `scp` 复制到 Worker-1；
3. 通过 `ssh` 在 Worker-1 上执行；
4. 对 Worker-2 执行同样操作；
5. 最后在 Master 本机执行；
6. 删除临时文件。

远程脚本会扫描 `/etc/netplan/*.yaml`，优先选择不包含 `cloud-init` 的文件，然后查找配置了 `10.20.30.x` 地址的以太网接口：

```python
if addr.startswith("10.20.30."):
    conf["nameservers"] = {
        "addresses": [DNS_SERVER],
        "search": [SEARCH_DOMAIN]
    }
```

最终写入的逻辑配置类似：

```yaml
nameservers:
  addresses:
    - 10.20.30.160
  search:
    - cluster.example.internal
```

然后执行：

```bash
netplan apply
```

### 7.3 Search Domain 的作用

设置：

```text
search cluster.example.internal
```

后，系统在解析短名称时可以尝试拼接域名。例如：

```bash
getent hosts k8s-worker-1
```

系统可能按以下名称查询：

```text
k8s-worker-1.cluster.example.internal
```

实际生产配置中，建议应用和集群配置尽量使用完整域名，避免短名称在不同搜索域环境中产生歧义。

### 7.4 脚本的假设和限制

当前脚本针对项目实验环境编写，有以下假设：

- Worker SSH 地址写死为 `192.168.50.161` 和 `192.168.50.162`；
- DNS Server 写死为 `10.20.30.160`；
- 通过 `10.20.30.x` 判断集群内网接口；
- 依赖远程节点预先安装 `python3-yaml`；
- 选择第一个符合条件的 Netplan 文件；
- 会重写 YAML 的格式；
- `set -euo pipefail` 下，远程 SSH/SCP 失败会导致脚本中断。

如果真实环境使用不同的网卡、地址或配置管理方式，需要先修改脚本，不能仅修改 DNS Zone 文件。

---

## 8. 第三步：验证 DNS

### 8.1 执行验证脚本

```bash
bash /opt/k8s-dns/scripts/00-3_verify_dns.sh
```

脚本分三层验证。

### 8.2 第一层：直接查询 DNS 服务器

```bash
dig @10.20.30.160 k8s-master.cluster.example.internal +short
dig @10.20.30.160 k8s-worker-1.cluster.example.internal +short
dig @10.20.30.160 k8s-worker-2.cluster.example.internal +short
```

这一步验证 BIND9 自身是否能通过 `10.20.30.160:53` 返回正确结果。

### 8.3 第二层：验证各节点的系统解析

```bash
getent hosts k8s-master.cluster.example.internal
getent hosts k8s-worker-1.cluster.example.internal
getent hosts k8s-worker-2.cluster.example.internal
```

脚本会在 Master 本机和两个 Worker 上执行这类查询。这一步比单独使用 `dig` 更接近 Kubernetes、Containerd 和普通应用的真实使用方式。

### 8.4 第三层：检查 resolver 配置

```bash
resolvectl status
```

如果系统没有 `resolvectl`，脚本会退回查看：

```bash
cat /etc/resolv.conf
```

验收时应确认各节点 DNS Server 为：

```text
10.20.30.160
```

快速验收：

```bash
getent hosts k8s-master.cluster.example.internal
getent hosts k8s-worker-1.cluster.example.internal
```

### 8.5 推荐验收标准

只有同时满足以下条件，才可以认为 DNS 前置工作完成：

- `named` 服务处于 active/running；
- `dig @10.20.30.160` 能返回正确地址；
- Master 上 `getent` 能解析；
- 两个 Worker 上 `getent` 能解析；
- 各节点 resolver 配置使用 `10.20.30.160`；
- 外部域名在允许联网时也能通过 forwarder 解析。

---

## 9. 配置文件与数据流总结

```text
00-1_setup_dns_server.sh
    |
    +-- /etc/bind/named.conf.options
    |      全局监听、查询权限、forwarders、DNSSEC 选项
    |
    +-- /etc/bind/named.conf.local
    |      注册 cluster.example.internal Zone
    |
    +-- /etc/bind/db.cluster.example.internal
           权威 DNS 记录

00-2_config_all_nodes_dns.sh
    |
    +-- Worker 的 /etc/netplan/*.yaml
    +-- Master 的 /etc/netplan/*.yaml
           nameservers: 10.20.30.160

00-3_verify_dns.sh
    |
    +-- dig       验证 DNS 服务
    +-- getent    验证系统解析链路
    +-- resolvectl 验证节点 resolver 配置
```

一个完整的请求过程是：

```text
Containerd / kubeadm / 应用
        |
        v
系统 resolver
        |
        v
10.20.30.160:53
        |
        +-- internal-registry.cluster.example.internal
        |       -> 10.20.30.169
        |
        +-- archive.ubuntu.com
                -> forwarder 查询并缓存
```

---

## 10. 常见故障排查

### 10.1 BIND9 无法启动

先查看服务状态和日志：

```bash
systemctl status named --no-pager
journalctl -u named -n 100 --no-pager
```

检查全局配置和 Zone 配置：

```bash
named-checkconf
named-checkzone cluster.example.internal /etc/bind/db.cluster.example.internal
```

常见原因：

- Zone 文件语法错误；
- `named.conf.local` 的文件路径错误；
- IP 地址或网卡配置与 `listen-on` 不匹配；
- 文件权限不允许 `bind` 用户读取；
- 53 端口已被其他 DNS 服务占用。

### 10.2 53 端口没有监听

```bash
ss -lntup | grep ':53'
```

如果没有监听，检查 `named` 日志。如果只有 `127.0.0.1:53` 而没有 `10.20.30.160:53`，重点检查：

```bash
ip -4 addr
ip route
```

以及 `/etc/bind/named.conf.options` 中的 `listen-on`。

### 10.3 `dig @10.20.30.160` 超时

按顺序检查：

```bash
ping -c 3 10.20.30.160
ss -lunpt | grep ':53'
```

然后检查防火墙是否放行 UDP/TCP 53。DNS 查询通常使用 UDP，响应过大或特定场景可能回退到 TCP，因此两种协议都应考虑。

### 10.4 `dig` 成功但 `getent` 失败

这说明 BIND9 本身正常，但当前节点没有正确使用它。检查：

```bash
resolvectl status
cat /etc/resolv.conf
getent hosts k8s-worker-1.cluster.example.internal
```

重点确认：

- DNS Server 是否是 `10.20.30.160`；
- Netplan 是否确实应用成功；
- `/etc/resolv.conf` 是否被其他服务覆盖；
- 节点到 `10.20.30.160` 的 53 端口是否可达。

### 10.5 `No module named yaml`

在报错节点安装 PyYAML：

```bash
apt-get update
apt-get install -y python3-yaml
python3 -c 'import yaml; print("PyYAML OK")'
```

### 10.6 找不到正确的 Netplan 接口

脚本只会给包含 `10.20.30.x` 地址的以太网接口写入 DNS。检查：

```bash
grep -RIn '10\.0\.0\.' /etc/netplan/
ip -4 addr
```

如果真实内网网段不是 `10.20.30.0/24`，需要修改脚本中的判断逻辑和 DNS 地址。

### 10.7 新增或修改 DNS 记录后不生效

编辑 Zone 文件：

```bash
vim /etc/bind/db.cluster.example.internal
```

每次修改都应递增 SOA Serial，例如从 `1` 改成 `2`，然后检查并重新加载：

```bash
named-checkzone cluster.example.internal /etc/bind/db.cluster.example.internal
systemctl reload named
```

当前安装脚本使用 `systemctl restart named`，也可以使用 restart，但生产环境通常优先使用 reload，减少服务中断。

如果客户端或 BIND9 仍返回旧结果，需要考虑 TTL 缓存。本文 TTL 为 7 天，测试期间修改记录后可能需要等待缓存过期，或清理相关缓存。

---

## 11. 安全、可用性和生产化建议

当前脚本适合受控的实验网络，但有几个配置不应未经评估直接用于生产：

### 11.1 限制查询来源

当前配置为：

```bind
allow-query { any; };
```

这意味着所有能够访问 DNS 端口的来源都可以查询。生产环境应限制为实际内网网段，例如：

```bind
allow-query { 10.20.30.0/24; 127.0.0.1; };
```

具体网段必须按真实网络修改。

### 11.2 限制监听地址

只在需要提供服务的内网接口监听 DNS，避免不必要地暴露到其他网络。IPv6 也应根据实际是否使用来决定是否监听：

```bind
listen-on { 127.0.0.1; 10.20.30.160; };
```

### 11.3 DNSSEC 验证

当前脚本明确设置：

```bind
dnssec-validation no;
```

这会关闭 BIND9 对递归查询结果的 DNSSEC 验证。实验环境中可能为了减少依赖而这样设置，但生产环境应根据上游 DNS、网络策略和系统版本评估是否启用验证。

### 11.4 避免单点故障

当前只有 Master 提供 DNS：

```text
所有节点 -> 10.20.30.160
```

如果 Master 宕机，节点可能无法解析 Harbor 和其他内部域名。生产环境可以增加 Secondary DNS，或让另一台基础设施节点提供备用解析服务，并在 Netplan 中配置多个 DNS 地址。

### 11.5 变更管理与监控

建议为 Zone 文件和 BIND9 配置建立：

- 配置备份；
- 版本管理；
- `named-checkconf` 和 `named-checkzone` 的变更前检查；
- 服务状态监控；
- DNS 查询失败、响应延迟和 53 端口可用性监控；
- 明确的 SOA Serial 递增规则。

---

## 12. 后续扩展方向

完成本文后，可以在此基础上继续扩展：

1. 增加反向解析 Zone（PTR 记录）；
2. 为 Harbor、Ingress 和应用域名增加更多 A/CNAME 记录；
3. 配置 BIND9 主从架构；
4. 为不同内网客户端配置不同视图（views）；
5. 启用日志、统计和审计；
6. 使用配置管理工具替代硬编码 IP 和临时 Python 脚本；
7. 将节点地址、管理网段、集群网段抽取为统一变量；
8. 将 DNS 健康检查纳入集群部署流水线。

---

## 13. 一键执行清单

确认前置条件满足后，在 Master 上执行：

```bash
# 1. 安装并配置 BIND9
bash /opt/k8s-dns/scripts/00-1_setup_dns_server.sh

# 2. 配置 Master 和 Worker 使用 Master DNS
bash /opt/k8s-dns/scripts/00-2_config_all_nodes_dns.sh

# 3. 验证 DNS
bash /opt/k8s-dns/scripts/00-3_verify_dns.sh
```

快速验收：

```bash
dig @10.20.30.160 k8s-master.cluster.example.internal +short
getent hosts k8s-worker-1.cluster.example.internal
getent hosts internal-registry.cluster.example.internal
```

预期核心结果：

```text
k8s-master.cluster.example.internal    -> 10.20.30.160
k8s-worker-1.cluster.example.internal  -> 10.20.30.161
k8s-worker-2.cluster.example.internal  -> 10.20.30.162
internal-registry.cluster.example.internal    -> 10.20.30.169
```

当以上结果全部正确后，说明 Kubernetes 集群的基础节点 DNS 已经准备完成，可以继续执行后续操作系统、Containerd 和 kubeadm 部署步骤。
