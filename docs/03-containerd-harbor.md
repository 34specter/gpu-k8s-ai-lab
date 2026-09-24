# 基于Containerd与自签名CA的Harbor镜像仓库接入

> 本文是 Kubernetes 集群部署系列的第 02 篇，介绍如何让 containerd 安全地访问使用自签名证书的内网 Harbor，并将 Kubernetes 所需的基础镜像从外部仓库切换到内网镜像仓库。
>
> 文中的域名、IP、主机名、证书路径和镜像名称均为脱敏后的示例，不能直接用于生产环境。实际部署时应替换为本组织的网络、证书和仓库配置。

---

## 1. 背景：为什么需要配置 Harbor 信任

Kubernetes 节点启动 Pod 时，kubelet 会通过 CRI 调用 containerd 拉取镜像。镜像通常来自 Docker Hub、`registry.k8s.io` 或企业内部的 Harbor。

在企业内网中，Harbor 常常使用内部 PKI 或自签名 CA 签发 HTTPS 证书。这样可以保护镜像传输，但节点默认并不认识这套 CA。containerd 访问 Harbor 时可能出现：

```text
x509: certificate signed by unknown authority
```

这不是“镜像不存在”，而是 TLS 证书链验证失败：

```text
containerd 不信任 Harbor 服务器证书的签发者
```

本篇解决的问题是：

1. 让所有需要拉取镜像的节点获得 Harbor CA；
2. 让 containerd 将该 CA 用于指定 Harbor 域名的 TLS 验证；
3. 确保 Harbor 域名能够解析到正确的内网地址；
4. 将 pause/sandbox 镜像切换为内网仓库，减少对外部仓库和公网连接的依赖；
5. 为后续 kubeadm、Calico、Ingress 和业务 Pod 从 Harbor 拉取镜像做好准备。

### 本篇实际完成的工作

项目中这一步由两个脚本完成：

```text
02_1_distribute_harbor_ca.sh       # 分发 CA 并配置 Harbor 地址映射
02_2_config_containerd_harbor.sh   # 配置 containerd 的 registry 信任和镜像路径
```

第一个脚本负责把证书放到每个节点；第二个脚本负责让 containerd 使用证书。两者顺序不能颠倒。

### 本篇不负责的工作

本篇不会自动完成：

- Harbor 服务本身的安装；
- Harbor 项目、用户和机器人账户创建；
- 镜像同步、镜像重命名或推送；
- Harbor 服务器证书签发；
- Kubernetes 集群初始化；
- 通过 `crictl pull` 验证某个具体业务镜像。

脚本能够完成的是节点侧的信任配置，Harbor 端的镜像和认证仍需要单独准备。

---

## 2. 端到端调用链

镜像拉取链路可以抽象为：

```text
kubelet
   |
   | CRI 请求
   v
containerd
   |
   | 解析 registry 域名、读取 hosts.toml 和 ca.crt
   v
HTTPS/TLS
   |
   | TCP 443
   v
registry.example.internal
   |
   v
Harbor Registry API / 镜像仓库
```

每一层可能产生不同类型的错误：

| 层次 | 典型问题 | 常见现象 |
|---|---|---|
| DNS/hosts | 域名无法解析或指向错误地址 | `no such host`、连接到错误服务 |
| TCP/防火墙 | 443 端口不可达 | timeout、connection refused |
| TLS | CA 不受信、SAN 不匹配 | `x509` 错误 |
| Harbor 认证 | 用户或 token 无权限 | `401 Unauthorized`、`403 Forbidden` |
| 镜像路径 | 项目或 tag 不存在 | `manifest unknown`、`not found` |
| containerd 配置 | 未加载 `hosts.toml` | 仍然使用默认 registry 行为 |

因此，安装 `ca.crt` 只是解决 TLS 信任链问题，不能代替 DNS、网络、Harbor 认证和镜像内容检查。

---

## 3. TLS 与 CA 信任原理

### 3.1 HTTPS 证书验证过程

containerd 使用 HTTPS 访问 Harbor 时，大致会执行以下验证：

1. 根据镜像引用中的域名建立 TCP 连接；
2. 发起 TLS ClientHello，并带上目标域名；
3. Harbor 返回服务器证书链；
4. containerd 检查证书是否由信任的 CA 签发；
5. 检查证书有效期；
6. 检查证书的 SAN 是否包含访问使用的域名；
7. 验证通过后，才开始 Registry API 请求。

必须使用证书中的域名访问 Harbor。例如，证书签发给：

```text
registry.example.internal
```

但客户端使用裸 IP 访问：

```text
https://192.168.50.169
```

即使 CA 已经安装，也可能因为证书 SAN 不包含该 IP 而失败。因此本文要求：

- 镜像引用使用证书对应的 Harbor 域名；
- DNS 或 `/etc/hosts` 将该域名解析到正确的地址；
- containerd 的 registry 配置目录名称与访问域名完全一致。

### 3.2 CA、服务器证书和私钥的区别

| 文件/对象 | 作用 | 是否应分发到节点 |
|---|---|---|
| CA 证书 | 验证服务器证书的签发者 | 是，分发公开证书部分 |
| Harbor 服务器证书 | Harbor 对外证明身份 | 通常保存在 Harbor 服务端 |
| Harbor 私钥 | Harbor 证明自己身份 | 否，绝不能分发到节点 |
| 客户端证书/私钥 | 双向 TLS 客户端认证 | 只有启用 mTLS 且有明确需求时才分发 |

本篇只分发 CA 证书。CA 证书本身可以公开给需要验证 Harbor 的客户端，但必须保护 CA 私钥。脚本不应把私钥文件误当成 CA 文件。

### 3.3 为什么不应该关闭 TLS 校验

有些临时方案会配置：

```text
insecure_skip_verify = true
```

这会让客户端不再验证服务器身份，攻击者可能通过中间人方式替换镜像内容。正确做法是：

```text
保留 HTTPS
    + 安装正确的 CA
    + 使用证书匹配的域名
    + 验证证书链和有效期
```

这样既保留加密，也保留身份认证。

---

## 4. Containerd 的 registry 配置模型

### 4.1 registry-specific 目录

本文使用 containerd 的按仓库配置目录：

```text
/etc/containerd/certs.d/registry.example.internal/
├── ca.crt
└── hosts.toml
```

目录名必须与镜像引用中的 registry host 一致。比如镜像引用为：

```text
registry.example.internal/platform/pause:3.9
```

containerd 就会查找：

```text
/etc/containerd/certs.d/registry.example.internal/
```

### 4.2 `ca.crt`

`ca.crt` 是签发 Harbor 服务器证书的 CA 公钥证书。containerd 使用它验证服务器证书链。

它不是：

- Harbor 的服务器私钥；
- 用户登录密码；
- Harbor 项目 token；
- 任意一张客户端证书。

CA 文件应为 PEM 编码，通常内容类似：

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

可以使用 OpenSSL 检查内容：

```bash
openssl x509 -in /etc/containerd/certs.d/registry.example.internal/ca.crt \
  -noout -subject -issuer -dates -fingerprint -sha256
```

### 4.3 `hosts.toml`

脚本生成的逻辑配置如下：

```toml
server = "https://registry.example.internal"

[host."https://registry.example.internal"]
  capabilities = ["pull", "resolve", "push"]
  ca = "/etc/containerd/certs.d/registry.example.internal/ca.crt"
```

字段含义：

- `server`：该 registry 的默认服务地址；
- `[host."..."]`：定义实际访问的 registry endpoint；
- `capabilities`：声明该 endpoint 可以执行的操作；
- `pull`：拉取镜像；
- `resolve`：解析镜像 tag 或 digest；
- `push`：推送镜像；
- `ca`：指定该 endpoint 使用的 CA 文件。

当前脚本同时授予 `pull`、`resolve` 和 `push`。生产环境应根据节点职责收敛权限。例如只允许节点拉取时，应评估是否可以去掉 `push`，避免节点被用于向仓库写入镜像。

### 4.4 `config_path`

仅创建 `hosts.toml` 并不一定会让 containerd 读取它。containerd 主配置需要指定 registry 配置目录：

```toml
config_path = "/etc/containerd/certs.d"
```

脚本通过替换 `/etc/containerd/config.toml` 中的空配置来设置这一项。配置更新后必须重启 containerd：

```bash
systemctl restart containerd
```

### 4.5 pause/sandbox 镜像

Kubernetes Pod 通常包含一个 pause（sandbox）容器，用于持有 Pod 的网络命名空间。业务容器共享这个 Pod sandbox 的网络命名空间。

可以把它理解为：

```text
Pod sandbox / pause 容器
        |
        +-- 持有 Pod 网络命名空间
        |
        +-- 业务容器加入同一网络命名空间
```

脚本将 containerd 配置中的外部 pause 镜像替换为内网仓库镜像：

```text
registry.example.internal/platform/pause:3.9
```

这样做的好处是：

- 集群初始化不必依赖公网 registry；
- 镜像下载路径统一；
- 可在 Harbor 内对基础镜像进行审计和缓存；
- 受限网络环境下更容易复现部署。

注意：修改配置只改变 containerd 期望使用的镜像引用，Harbor 中必须提前存在完全匹配的 repository 和 tag。

### 4.6 `/etc/crictl.yaml`

脚本还会生成：

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
```

`crictl` 是 CRI 调试客户端：

- `runtime-endpoint`：查询 Pod、容器和运行时状态；
- `image-endpoint`：执行镜像查询和拉取；
- Unix socket：containerd 暴露 CRI 服务的本地入口；
- `timeout`：命令等待运行时响应的时间。

这不会创建 Harbor 账号，也不会自动完成 registry 登录。Harbor 启用认证时，仍需要按 containerd/CRI 支持方式提供凭据。

---

## 5. 前置条件

### 5.1 Harbor 端

执行节点配置前，Harbor 应满足：

- 服务已经运行；
- Harbor 证书由待分发的 CA 签发；
- 证书的 SAN 包含客户端实际使用的 registry 域名；
- 目标项目和镜像已经创建或准备好；
- 如果 Harbor 开启认证，已有可用的 robot account 或其他凭据；
- Harbor 的 443 端口可以从所有 Kubernetes 节点访问。

### 5.2 节点端

所有实际拉取镜像的节点都需要：

- 安装 containerd；
- 具有 `/etc/containerd`；
- 可以解析 `registry.example.internal`，或通过 `/etc/hosts` 映射；
- 可以访问 Harbor 的 TCP 443；
- 具有 root 权限；
- Master 可以通过 SSH/SCP 访问 Worker（若使用脚本自动分发）；
- 节点时间基本同步；
- CA 文件来源可信且未过期。

### 5.3 证书来源

脱敏后的 CA 来源示例：

```text
/path/to/harbor-ca.crt
```

脚本会先检查该文件存在。如果文件不存在，分发脚本直接退出。实际执行前还应检查证书格式、签发者、有效期和指纹，而不仅是文件是否存在。

### 5.4 地址示例

本文统一使用以下示例值：

| 角色 | 示例值 |
|---|---|
| Registry 域名 | `registry.example.internal` |
| Registry 内网地址 | `10.20.30.169` |
| Worker-1 管理地址 | `192.168.50.161` |
| Worker-2 管理地址 | `192.168.50.162` |
| 配置目录 | `/etc/containerd/certs.d/registry.example.internal/` |
| 脚本目录 | `/opt/k8s-platform/scripts/` |

这些只是文档示例，实际部署必须替换。

---

## 6. 第一步：分发 Harbor CA

### 6.1 执行入口

在控制节点执行：

```bash
bash /opt/k8s-platform/scripts/02_1_distribute_harbor_ca.sh
```

### 6.2 脚本做了什么

#### 检查 CA 源文件

脚本首先检查：

```bash
if [ ! -f "$CA_SRC" ]; then
    echo "ERROR: CA 证书不存在"
    exit 1
fi
```

这能避免把一个不存在的文件复制到节点，但不能证明文件内容是正确 CA。

#### 配置 hosts 映射

脚本在本机和 Worker 的 `/etc/hosts` 中添加类似记录：

```text
10.20.30.169 registry.example.internal
```

这样即使内部 DNS 尚未准备好，节点也能解析该 registry 名称。脚本使用 `grep -q` 判断是否已有包含该主机名的行，避免重复添加。

需要注意：`/etc/hosts` 是本地静态解析配置，不能替代正式 DNS 管理。地址变更时，所有节点的 hosts 文件都需要同步修改。

#### 配置 Master 本地证书目录

脚本创建：

```text
/etc/containerd/certs.d/registry.example.internal/
```

然后将 CA 源文件复制为：

```text
/etc/containerd/certs.d/registry.example.internal/ca.crt
```

#### 通过 SSH/SCP 配置 Worker

脚本使用 SSH 在 Worker 上创建目录，再使用 SCP 复制证书：

```text
控制节点 --SSH/SCP--> Worker-1
控制节点 --SSH/SCP--> Worker-2
```

它依赖：

- Worker 的 SSH 服务正常；
- 控制节点可以 root 登录；
- 管理地址正确；
- SSH 认证不会在执行过程中等待人工输入。

### 6.3 脚本没有做什么

`02_1_distribute_harbor_ca.sh` 当前不会：

- 检查 CA 是否为 PEM 证书；
- 检查 CA 是否在有效期内；
- 校验 CA 指纹是否符合预期；
- 连接 Harbor 验证服务器证书；
- 重启 containerd；
- 测试镜像拉取；
- 配置 Harbor 用户认证。

因此分发成功只表示文件复制动作完成，不能单独作为 Harbor 接入成功的证明。

### 6.4 分发后检查

在每台节点执行：

```bash
ls -l /etc/containerd/certs.d/registry.example.internal/ca.crt
openssl x509 -in /etc/containerd/certs.d/registry.example.internal/ca.crt \
  -noout -subject -issuer -dates
getent hosts registry.example.internal
grep -n 'registry.example.internal' /etc/hosts
```

---

## 7. 第二步：配置 containerd 信任 Harbor

### 7.1 执行入口

在每台需要拉取 Harbor 镜像的节点执行：

```bash
bash /opt/k8s-platform/scripts/02_2_config_containerd_harbor.sh
```

脚本要求第一步已经完成。如果没有 `ca.crt`，它会退出：

```text
ERROR: 缺少 registry CA，请先执行证书分发步骤
```

### 7.2 创建 `hosts.toml`

脚本写入：

```toml
server = "https://registry.example.internal"

[host."https://registry.example.internal"]
  capabilities = ["pull", "resolve", "push"]
  ca = "/etc/containerd/certs.d/registry.example.internal/ca.crt"
```

这里通过 registry 域名关联 CA，而不是把 CA 配成所有 HTTPS 站点的全局信任。按仓库隔离信任范围更安全。

### 7.3 设置 registry 配置路径

脚本修改：

```text
config_path = "/etc/containerd/certs.d"
```

如果当前 containerd 配置没有预期的 `config_path = ""` 字符串，脚本中的 `sed` 可能无法修改它。执行后应主动检查：

```bash
grep -n 'config_path' /etc/containerd/config.toml
```

### 7.4 替换 sandbox 镜像

脚本将默认 pause 引用替换为内网仓库引用。由于脚本只替换预设的两个 tag，实际 containerd 配置中的 tag 如果不同，替换可能不会发生。

检查：

```bash
grep -Rni 'sandboxImage\|pause:' /etc/containerd/config.toml
```

并确认 Harbor 中存在相同路径：

```text
registry.example.internal/platform/pause:<matching-tag>
```

### 7.5 写入 crictl 配置并重启

脚本生成 `/etc/crictl.yaml`，然后执行：

```bash
systemctl restart containerd
```

重启会中断当前节点上的容器运行时连接。已加入生产集群的节点应按维护窗口和驱逐策略操作，不能在不了解影响的情况下直接重启。

验证：

```bash
systemctl is-active containerd
systemctl is-enabled containerd
grep -n 'config_path' /etc/containerd/config.toml
cat /etc/crictl.yaml
```

---

## 8. 端到端验收

### 8.1 文件和配置验收

```bash
test -s /etc/containerd/certs.d/registry.example.internal/ca.crt
openssl x509 -in /etc/containerd/certs.d/registry.example.internal/ca.crt -noout -dates
cat /etc/containerd/certs.d/registry.example.internal/hosts.toml
grep -n 'config_path' /etc/containerd/config.toml
systemctl status containerd --no-pager
```

### 8.2 TLS 验收

使用 CA 直接验证 Harbor 的服务器证书：

```bash
openssl s_client \
  -connect registry.example.internal:443 \
  -servername registry.example.internal \
  -CAfile /etc/containerd/certs.d/registry.example.internal/ca.crt \
  </dev/null 2>/dev/null | grep -E 'Verify return code|subject=|issuer='
```

也可以用 curl 验证 HTTPS 握手和 Registry API：

```bash
curl --cacert /etc/containerd/certs.d/registry.example.internal/ca.crt \
  -i https://registry.example.internal/v2/
```

返回 `401 Unauthorized` 不一定说明 TLS 失败。它通常表示：

- TLS 已经成功；
- Harbor 正常响应；
- 但请求还需要认证。

如果看到 `x509`，才优先排查 CA、证书 SAN、有效期和访问域名。

### 8.3 containerd/CRI 验收

```bash
crictl info
crictl images
ctr plugins ls | grep -E 'io.containerd.grpc.v1.cri|cri'
```

然后使用实际存在且允许访问的镜像进行测试：

```bash
crictl pull registry.example.internal/platform/<image>:<tag>
```

如果 Harbor 需要认证，应先按组织规定配置凭据。不要把密码直接写在共享文档、命令历史或脚本中。

### 8.4 pause 镜像验收

确认配置中的 pause 镜像使用内网仓库：

```bash
grep -Rni 'sandboxImage\|pause:' /etc/containerd/config.toml
```

在可控测试节点上，可以通过创建测试 Pod 或执行受控的 CRI 拉取，结合 containerd 日志确认实际请求路径：

```bash
journalctl -u containerd -n 100 --no-pager
```

文章不能把“配置文件已修改”当作“镜像已经成功拉取”。后者必须通过实际拉取或创建 Pod 验证。

---

## 9. 常见故障排查

### 9.1 `x509: certificate signed by unknown authority`

排查顺序：

```bash
ls -l /etc/containerd/certs.d/registry.example.internal/ca.crt
openssl x509 -in /etc/containerd/certs.d/registry.example.internal/ca.crt -noout -issuer -subject
cat /etc/containerd/certs.d/registry.example.internal/hosts.toml
grep -n 'config_path' /etc/containerd/config.toml
systemctl restart containerd
```

重点检查：

- CA 是否是签发 Harbor 证书的那一张；
- CA 文件是否完整、没有被截断；
- `hosts.toml` 中的路径是否正确；
- registry 目录名称是否与镜像中的域名完全一致；
- containerd 是否在配置修改后重启；
- 实际执行拉取的节点是否也安装了 CA。

### 9.2 证书 SAN 与访问域名不匹配

查看 Harbor 服务器证书：

```bash
openssl s_client -connect registry.example.internal:443 \
  -servername registry.example.internal </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A2 'Subject Alternative Name'
```

如果 SAN 中没有 `registry.example.internal`，应重新签发正确证书，不能仅通过修改 hosts 文件规避 TLS 身份校验。

### 9.3 `no such host` 或连接超时

```bash
getent hosts registry.example.internal
cat /etc/hosts
ip route
nc -vz registry.example.internal 443
```

这类问题优先属于 DNS、hosts、路由、防火墙或端口问题，不是 CA 问题。

### 9.4 `hosts.toml` 似乎没有生效

```bash
grep -n 'config_path' /etc/containerd/config.toml
cat /etc/containerd/certs.d/registry.example.internal/hosts.toml
journalctl -u containerd -n 100 --no-pager
```

确认：

- containerd 版本支持当前配置模型；
- `config_path` 位于当前版本实际使用的 registry 配置段；
- 文件权限允许 root/containerd 读取；
- 重启后日志没有配置解析错误；
- 镜像引用的 registry host 与目录名一致。

必要时可以查看当前运行时配置：

```bash
containerd config dump | grep -n -A3 -B3 'config_path'
```

### 9.5 `401 Unauthorized` 或 `403 Forbidden`

这通常表示 TLS 已通过，接下来失败的是 Harbor 认证或授权。检查：

- 用户名和密码/robot token 是否正确；
- 目标项目是否允许该账户拉取；
- 镜像路径是否属于该项目；
- 凭据是否配置在实际调用 containerd 的节点；
- 凭据是否已过期。

不要把这个问题误判为“CA 未安装”。

### 9.6 `manifest unknown` 或 `not found`

检查镜像完整名称和 tag：

```bash
crictl pull registry.example.internal/platform/<image>:<tag>
```

需要确认 Harbor 中存在完全一致的：

```text
registry / project / repository / tag
```

镜像在上游仓库存在，不代表它已经存在于内网 Harbor。需要先完成同步或推送。

### 9.7 pause 镜像仍然访问外部仓库

检查：

```bash
grep -Rni 'sandboxImage\|pause:' /etc/containerd/config.toml
```

可能原因：

- 当前配置使用的 pause tag 不在脚本替换范围内；
- containerd 配置文件路径不同；
- 修改后没有重启 containerd；
- Harbor 中的镜像路径或 tag 不存在，运行时回退或拉取失败；
- kubeadm 使用了显式指定的其他 pause 镜像。

应以 kubeadm、containerd 当前版本和实际配置为准，不要只依赖脚本的 `sed` 替换。

### 9.8 只有部分节点能拉取

在每个节点执行同一组检查：

```bash
hostname
getent hosts registry.example.internal
test -s /etc/containerd/certs.d/registry.example.internal/ca.crt
systemctl is-active containerd
cat /etc/containerd/certs.d/registry.example.internal/hosts.toml
```

常见原因是：

- CA 只分发到了控制节点；
- Worker 的 `/etc/hosts` 未更新；
- 某个节点 containerd 未重启；
- 节点使用了不同的 containerd 版本或配置路径；
- 防火墙策略只允许部分节点访问 443。

---

## 10. 安全与生产化建议

### 10.1 使用正确的信任链

生产环境优先使用组织 PKI 或公认受信任 CA 签发 Harbor 证书。如果必须使用内部 CA：

- 保存并保护 CA 私钥；
- 只分发 CA 公钥证书；
- 记录 CA 指纹和有效期；
- 监控证书到期时间；
- 设计证书轮换和回滚流程。

### 10.2 不要关闭证书校验

不要使用以下方式“解决” x509 错误：

```text
跳过 TLS 验证
使用 HTTP 替代 HTTPS
把服务器私钥分发到所有节点
```

这些方式会使镜像传输或服务器身份失去可靠保护。

### 10.3 收敛 registry 能力

当前示例授予：

```toml
capabilities = ["pull", "resolve", "push"]
```

如果节点只需要拉取镜像，可以评估改为：

```toml
capabilities = ["pull", "resolve"]
```

推送镜像应在受控的构建机或发布流水线上完成，而不是让所有 Kubernetes 节点都具备推送权限。

### 10.4 凭据管理

CA 信任和 Harbor 用户认证是两件事。生产环境应：

- 使用 Harbor robot account；
- 按项目和操作类型授予最小权限；
- 使用 Secret 管理系统或受控的 credential store；
- 避免将密码写进 shell 历史、公开脚本和知识库；
- 定期轮换 token。

### 10.5 节点变更窗口

重启 containerd 会影响当前节点的容器运行时连接。已加入生产集群的节点应：

1. 先确认 Pod 副本和业务容错能力；
2. 按维护流程 cordon/drain；
3. 修改并验证配置；
4. 重启 containerd；
5. 执行实际镜像拉取测试；
6. 恢复节点调度。

实验环境可以直接执行脚本，但不能把实验操作方式原样用于在线业务节点。

### 10.6 建立分发审计

建议记录：

- CA 指纹；
- 分发到哪些节点；
- 分发时间和执行人；
- containerd 版本；
- Harbor 证书有效期；
- 拉取测试结果；
- 配置变更前后差异。

---

## 11. 推荐实施清单

### 第一步：确认 CA 和 Harbor

```bash
openssl x509 -in /path/to/harbor-ca.crt \
  -noout -subject -issuer -dates -fingerprint -sha256
getent hosts registry.example.internal
```

### 第二步：分发 CA

```bash
bash /opt/k8s-platform/scripts/02_1_distribute_harbor_ca.sh
```

### 第三步：在所有节点配置 containerd

```bash
bash /opt/k8s-platform/scripts/02_2_config_containerd_harbor.sh
```

### 第四步：检查配置

```bash
ls -l /etc/containerd/certs.d/registry.example.internal/ca.crt
cat /etc/containerd/certs.d/registry.example.internal/hosts.toml
grep -n 'config_path\|sandboxImage\|pause:' /etc/containerd/config.toml
systemctl is-active containerd
```

### 第五步：验证 TLS 和镜像拉取

```bash
curl --cacert /etc/containerd/certs.d/registry.example.internal/ca.crt \
  -i https://registry.example.internal/v2/

crictl info
crictl pull registry.example.internal/platform/<image>:<tag>
```

其中 `<image>:<tag>` 必须替换为 Harbor 中真实存在且当前节点有权限访问的镜像。

---

## 12. 小结

这一步不是简单地“复制一个证书”，而是建立了一条完整的镜像供应链信任路径：

```text
Harbor 服务器证书
        |
        | 由内部 CA 签发
        v
节点保存 CA 公钥证书
        |
        v
containerd 的 hosts.toml 指定该 CA
        |
        v
containerd 验证 HTTPS 服务器身份
        |
        v
通过 CRI 安全拉取 Harbor 镜像
```

同时，pause/sandbox 镜像被切换到内网仓库，使 Kubernetes 的基础 Pod 网络组件减少对外部 registry 的依赖。

最终验收不能只看：

```text
ca.crt 文件存在
```

而应至少确认：

```text
域名解析正确
+ 443 端口可达
+ TLS 证书验证通过
+ containerd 读取了 registry 配置
+ Harbor 认证和授权成功
+ 实际镜像可以拉取
```

只有这条链路全部打通，才算真正完成了 Containerd 对内网 Harbor 的接入。
