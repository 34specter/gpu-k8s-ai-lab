# 基于Ubuntu的Kubernetes节点基础环境准备

> 本文是 Kubernetes 集群部署系列的第 01 篇，面向需要搭建或维护 Kubernetes 节点的工程师，说明如何完成节点操作系统、Linux 内核、containerd 以及 Kubernetes 基础工具的统一准备。
>
> 文中的域名、IP、路径和主机名均为脱敏后的示例，不能直接用于生产环境。实际部署时应替换为本组织批准的版本、软件源和网络配置。

---

## 1. 背景：为什么要做节点基础环境准备

Kubernetes 并不是只安装一个 `kubeadm` 命令就可以运行。一个可加入集群的节点至少需要具备以下基础能力：

1. Linux 内核能够承载容器网络和转发流量；
2. 节点不会因为 swap 影响 kubelet 的资源管理；
3. 存在符合 Kubernetes 要求的容器运行时；
4. `kubelet` 能够作为节点代理运行；
5. `kubeadm` 能够执行集群初始化或节点加入；
6. `kubectl` 能够作为集群管理客户端使用；
7. 所有节点的运行时、工具和关键配置保持一致。

本篇对应项目中的三个脚本：

```text
01_1_prepare_node.sh       # Linux/OS 前置配置
01_2_install_containerd.sh # 安装并配置 containerd
01_3_install_kube_tools.sh # 安装 kubelet、kubeadm、kubectl
```

它们解决的是“节点还没有达到 Kubernetes 安装基线”的问题，为后续 `kubeadm init`、`kubeadm join` 和 CNI 部署提供统一环境。

### 本篇完成的工作

- 关闭当前运行中的 swap，并禁止 `/etc/fstab` 中的 swap 在重启后恢复；
- 加载 `overlay` 和 `br_netfilter` 内核模块；
- 开启桥接流量进入 iptables/ip6tables 的处理路径；
- 开启 IPv4 转发；
- 安装 containerd，生成默认配置，并使用 systemd cgroup；
- 安装 `kubelet`、`kubeadm` 和 `kubectl`；
- 暂停三个 Kubernetes 软件包的自动升级，以保持版本可控；
- 通过版本和服务状态命令完成基础验收。

### 本篇不负责的工作

本篇不会完成：

- `kubeadm init` 控制面初始化；
- Worker 加入集群；
- Calico、Cilium 等 CNI 部署；
- Harbor CA 信任配置；
- Pod 网段和 Service 网段的最终规划；
- 应用部署和 Ingress 配置。

---

## 2. 组件关系：这些软件分别负责什么

### 2.1 containerd

containerd 是容器运行时，负责：

- 拉取和管理镜像；
- 创建、启动和停止容器；
- 管理容器的文件系统和生命周期；
- 通过 CRI 为 kubelet 提供 Kubernetes 所需的运行时接口。

Kubernetes 通常不直接操作 containerd 的底层实现，而是通过 CRI 调用容器运行时。调用链可以简化为：

```text
kubelet
   |
   | CRI
   v
containerd
   |
   +-- 镜像管理
   +-- 容器生命周期
   +-- sandbox/pause 容器
```

### 2.2 kubelet

`kubelet` 是每个 Kubernetes 节点上的代理进程，负责：

- 向 API Server 注册节点；
- 根据 PodSpec 创建和维护 Pod；
- 通过 CRI 调用 containerd；
- 汇报节点和 Pod 状态；
- 执行探针、挂载和部分节点级管理任务。

安装 `kubelet` 并不等于节点已经加入集群。它需要在后续 `kubeadm init` 或 `kubeadm join` 后，获得集群配置才能正常工作。

### 2.3 kubeadm

`kubeadm` 是集群生命周期工具，常用于：

- 初始化控制面；
- 生成证书和 kubeconfig；
- 生成 Worker 加入命令；
- 执行节点加入和部分升级操作。

### 2.4 kubectl

`kubectl` 是 Kubernetes API 客户端。它不负责启动容器，而是通过 kubeconfig 访问 API Server，执行资源查询和变更。

因此，三者的关系是：

```text
kubeadm 负责搭建/加入集群
kubelet 负责节点上的 Pod 执行
kubectl 负责用户与 API Server 交互
containerd 负责实际容器运行
```

---

## 3. 执行顺序与整体流程

推荐在每一台 Master/Worker 节点上按以下顺序执行：

```text
操作系统前置配置
        |
        v
containerd 安装与 cgroup 配置
        |
        v
kubelet / kubeadm / kubectl 安装
        |
        v
后续 kubeadm 初始化或加入集群
```

顺序不能随意颠倒：

- 先配置内核和 swap，避免 kubeadm 预检失败；
- 再安装并验证 containerd，确保 kubelet 后续有可用运行时；
- 最后安装 Kubernetes 工具，避免工具已安装但底层运行时不满足要求。

脚本应在每台节点本地执行。示例路径经过脱敏：

```bash
bash /opt/k8s-platform/scripts/01_1_prepare_node.sh
bash /opt/k8s-platform/scripts/01_2_install_containerd.sh
bash /opt/k8s-platform/scripts/01_3_install_kube_tools.sh
```

---

## 4. 执行前置条件

### 4.1 操作系统与权限

执行前需要确认：

- 使用受支持的 Ubuntu/Debian 系统；
- 当前节点具有 root 权限，或可以通过 `sudo` 提权；
- systemd 正常运行；
- 节点能够访问经过组织批准的 APT 软件源；
- 节点时间与其他集群节点基本同步；
- 节点主机名、DNS 和网络配置已经完成；
- 节点之间能够访问后续 Kubernetes 所需端口。

检查命令：

```bash
id
cat /etc/os-release
uname -r
systemctl is-system-running --wait 2>/dev/null || true
date -u
```

### 4.2 软件源和版本策略

`01_3_install_kube_tools.sh` 会：

1. 创建 APT keyring 目录；
2. 下载 Kubernetes 软件源公钥并转换为 keyring；
3. 写入 Kubernetes APT 软件源；
4. 安装 `kubelet`、`kubeadm`、`kubectl`；
5. 使用 `apt-mark hold` 暂停三个包的自动升级。

示例脚本中的软件源只是项目环境配置。生产环境应使用：

- 组织批准的内部镜像源；
- 可校验来源和签名的仓库；
- 明确的 Kubernetes 版本矩阵；
- 经过验证的升级和回滚流程。

不要因为安装方便，就把未经审核的软件源或网络上的任意公钥加入系统信任范围。

### 4.3 配置备份

脚本会修改以下持久化配置：

```text
/etc/fstab
/etc/modules-load.d/k8s.conf
/etc/sysctl.d/k8s.conf
/etc/containerd/config.toml
/etc/apt/keyrings/
/etc/apt/sources.list.d/
```

在生产节点上执行前，应使用配置管理系统或至少进行备份：

```bash
cp -a /etc/fstab /etc/fstab.backup.$(date +%F-%H%M%S)
cp -a /etc/containerd /etc/containerd.backup.$(date +%F-%H%M%S) 2>/dev/null || true
```

---

## 5. Linux 前置配置原理

### 5.1 为什么 Kubernetes 通常要求关闭 swap

swap 是内存不足时将部分内存页换出到磁盘的机制。它可以提高普通服务器在内存压力下的存活概率，但会引入不可预测的 I/O 延迟。

Kubernetes 的调度和驱逐逻辑需要较准确地感知节点内存状态。如果节点允许任意使用 swap，可能出现：

- 容器内存性能突然下降；
- kubelet 的资源判断与实际执行延迟不一致；
- Pod 发生不可预测的磁盘换入换出；
- `kubeadm` 预检直接提示 swap 未关闭。

脚本执行：

```bash
swapoff -a
```

这只会立即关闭当前启用的 swap。为了防止重启后恢复，脚本还会将 `/etc/fstab` 中包含 `swap` 的行注释掉：

```bash
sed -i '/swap/s/^/#/g' /etc/fstab
```

两者的区别是：

| 操作 | 作用范围 |
|---|---|
| `swapoff -a` | 当前运行周期立即关闭 |
| 注释 `/etc/fstab` | 防止系统重启后重新挂载 |

生产环境不应只依赖字符串匹配修改 `/etc/fstab`，应先确认被修改的是预期的 swap 行，并记录变更。

验证：

```bash
swapon --show
# 没有输出表示当前没有启用 swap
```

### 5.2 `overlay` 模块的作用

容器镜像通常由多个只读层组成，容器启动时再叠加一个可写层。OverlayFS 通过联合挂载把这些层呈现为一个统一目录：

```text
只读镜像层 1
只读镜像层 2
只读镜像层 3
     +
可写容器层
     |
     v
容器看到的统一文件系统
```

`overlay` 内核模块为这种存储驱动提供支持。脚本将其写入：

```text
/etc/modules-load.d/k8s.conf
```

并立即加载：

```bash
modprobe overlay
```

写入 modules-load 配置是为了让模块在后续系统启动时自动加载。

### 5.3 `br_netfilter` 与桥接流量

Kubernetes 网络插件会创建 Linux bridge、虚拟网卡和网络命名空间。默认情况下，经过 Linux bridge 的数据包不一定会进入 iptables/ip6tables 的过滤路径。

加载 `br_netfilter` 后，桥接的 IPv4/IPv6 流量可以交给 netfilter 处理，使 Kubernetes 的：

- Service 规则；
- 网络策略；
- SNAT/DNAT；
- 节点防火墙规则；
- CNI 转发逻辑

能够按预期生效。

脚本执行：

```bash
modprobe br_netfilter
```

并在 `/etc/sysctl.d/k8s.conf` 中写入：

```text
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

### 5.4 IPv4 转发

节点作为 Pod 网络和外部网络之间的转发节点时，需要允许 Linux 转发 IPv4 数据包：

```text
net.ipv4.ip_forward = 1
```

这并不等于自动放行所有流量。它只是打开内核的转发能力，实际是否允许通过仍受路由、iptables/nftables、防火墙和 CNI 规则影响。

### 5.5 `sysctl --system` 的作用

脚本创建 `/etc/sysctl.d/k8s.conf` 后执行：

```bash
sysctl --system
```

该命令会读取系统的 sysctl 配置目录并将参数加载到当前运行内核中。因此配置具有两个效果：

- 立即修改当前运行内核；
- 保存在文件中，便于下次启动重新加载。

验证：

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

---

## 6. 安装和配置 containerd

### 6.1 脚本实际执行的步骤

`01_2_install_containerd.sh` 的主要逻辑是：

```bash
apt-get update -y
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

### 6.2 为什么要生成默认配置

containerd 可以使用默认行为运行，但 Kubernetes 节点需要明确的配置，便于：

- 统一 cgroup 驱动；
- 后续接入私有镜像仓库；
- 设置 sandbox 镜像；
- 配置 CRI 和 registry；
- 进行审计和排障。

命令：

```bash
containerd config default > /etc/containerd/config.toml
```

会将当前 containerd 版本对应的默认配置写入文件。不同 containerd 版本生成的配置结构可能不同，因此不要盲目把某个版本的配置文件复制到另一个版本。

### 6.3 Systemd cgroup 的原理

Linux cgroup 用于限制和统计进程组的 CPU、内存、进程数等资源。containerd 和 kubelet 都需要选择 cgroup 管理方式。

当前脚本将 containerd 配置为：

```toml
SystemdCgroup = true
```

这表示使用 systemd 管理 cgroup 层级。Kubernetes 官方实践通常要求 kubelet 和容器运行时使用一致的 cgroup 驱动。

如果一个组件使用 systemd，另一个组件使用 cgroupfs，可能造成：

- 节点注册异常；
- Pod 资源限制行为不一致；
- 节点压力驱逐判断异常；
- 升级或重启后出现难以定位的问题。

后续 kubeadm 初始化时也应确认 kubelet 的 cgroup 配置与 containerd 一致。

### 6.4 systemd restart 与 enable

```bash
systemctl restart containerd
systemctl enable containerd
```

- `restart`：立即重启服务，使新配置生效；
- `enable`：创建开机启动关系，确保重启后自动启动。

二者并不互相替代。只 `restart` 不能保证机器重启后服务自动启动，只 `enable` 也不会让当前运行中的服务读取新配置。

验证：

```bash
systemctl is-enabled containerd
systemctl is-active containerd
containerd --version
```

---

## 7. 安装 kubelet、kubeadm 和 kubectl

### 7.1 APT keyring 的作用

脚本执行：

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://<approved-kubernetes-mirror>/apt/doc/apt-key.gpg \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg --yes
```

APT 使用仓库签名校验机制确认下载的索引和软件包来自可信的签名密钥。将 key 保存到独立 keyring，并在软件源中通过 `signed-by` 指定，可以缩小该密钥的信任范围。

示例：

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] \
https://<approved-kubernetes-mirror>/apt/ kubernetes-xenial main
```

文档中的镜像地址是脱敏占位符。生产环境应使用组织批准的源，并在导入公钥前通过官方渠道、校验和或带外方式确认密钥指纹。

### 7.2 三个工具的安装

```bash
apt-get update -y
apt-get install -y kubelet kubeadm kubectl
```

安装后：

- `kubelet` 可作为节点服务运行；
- `kubeadm` 可用于初始化或加入集群；
- `kubectl` 可在配置 kubeconfig 后访问 API Server。

脚本没有在本篇执行 `kubeadm init` 或 `kubeadm join`，因此此时节点可能还没有加入任何集群。

### 7.3 为什么使用 apt-mark hold

脚本执行：

```bash
apt-mark hold kubelet kubeadm kubectl
```

Kubernetes 组件版本之间存在兼容关系。自动升级可能导致：

- kubelet 与控制面版本不匹配；
- 集群升级顺序被打乱；
- 运行时行为改变；
- 出现未经验证的版本组合。

`hold` 不是永久禁止升级，而是把升级纳入人工或自动化变更流程。升级时需要先解除 hold，完成验证后再按策略设置。

查看状态：

```bash
apt-mark showhold
```

### 7.4 版本验证

```bash
kubeadm version
kubectl version --client
```

应重点确认：

- 三台节点的主版本和次版本符合集群版本矩阵；
- `kubeadm`、`kubelet`、`kubectl` 没有混装不兼容版本；
- 输出来自期望的二进制路径，而不是旧版本残留。

---

## 8. 按脚本实施

### 8.1 执行 OS 前置脚本

在每台节点执行：

```bash
bash /opt/k8s-platform/scripts/01_1_prepare_node.sh
```

预期会看到：

```text
OK: node prereq done
```

完成后检查：

```bash
swapon --show
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward
```

### 8.2 安装 containerd

```bash
bash /opt/k8s-platform/scripts/01_2_install_containerd.sh
```

预期核心结果：

```text
containerd containerd.io 1.x.x
OK: containerd installed
```

检查配置：

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
systemctl status containerd --no-pager
```

预期 `SystemdCgroup` 为 `true`，服务状态为 active/running。

### 8.3 安装 Kubernetes 工具

```bash
bash /opt/k8s-platform/scripts/01_3_install_kube_tools.sh
```

检查：

```bash
kubeadm version
kubectl version --client
apt-mark showhold
```

### 8.4 多节点一致性检查

在所有节点执行相同检查，确保结果一致：

```bash
hostname
uname -r
containerd --version
kubeadm version
systemctl is-active containerd
swapon --show
```

节点基础环境的核心不是“某一台机器命令执行成功”，而是所有即将加入集群的节点具备相同的能力和兼容版本。

---

## 9. 常见故障排查

### 9.1 重启后 swap 又出现

检查：

```bash
swapon --show
grep -n 'swap' /etc/fstab
```

如果 `/etc/fstab` 中仍有有效 swap 行，说明持久化配置未被正确修改。不要直接删除未知挂载项，先确认设备或文件确实是 swap。

### 9.2 `br_netfilter` 或 `overlay` 加载失败

检查内核是否提供模块：

```bash
modprobe br_netfilter
modprobe overlay
lsmod | grep -E 'overlay|br_netfilter'
```

如果失败，可能原因包括：

- 当前内核未安装匹配的 modules 包；
- 内核裁剪掉了模块；
- 模块名称或版本不匹配；
- 节点尚未重启到期望内核。

查看日志：

```bash
dmesg | tail -n 50
```

### 9.3 sysctl 参数没有生效

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
cat /etc/sysctl.d/k8s.conf
```

确认文件格式正确，并重新加载：

```bash
sysctl --system
```

### 9.4 containerd 安装成功但服务未运行

```bash
systemctl status containerd --no-pager
journalctl -u containerd -n 100 --no-pager
containerd config dump | grep -n 'SystemdCgroup'
```

常见原因：

- 配置文件语法或字段与当前版本不兼容；
- cgroup 或内核能力不足；
- 53/其他端口问题通常不是 containerd 启动失败的直接原因，应查看日志确认；
- 旧配置残留导致生成的新配置未按预期生效。

### 9.5 APT 更新或 Kubernetes 工具安装失败

```bash
apt-get update
apt-cache policy kubeadm kubelet kubectl
cat /etc/apt/sources.list.d/kubernetes.list
ls -l /etc/apt/keyrings/
```

重点区分：

- DNS 解析失败；
- HTTPS/代理失败；
- 仓库签名失败；
- 软件包版本不存在；
- 系统版本与仓库不匹配。

### 9.6 kubeadm 预检提示 cgroup 或 swap 问题

```bash
swapon --show
grep -n 'SystemdCgroup' /etc/containerd/config.toml
systemctl is-active containerd
```

然后确认 kubelet 的实际配置和后续 kubeadm 使用的运行时端点。不要只看到 containerd 版本正确，就认为 cgroup 配置一定正确。

---

## 10. 安全与生产化建议

### 10.1 软件源与密钥

- 只使用经过审核的软件源；
- 不要将未知来源的 APT 公钥加入全局信任；
- 记录导入的密钥指纹；
- 使用版本仓库或内部镜像保证可复现；
- 对升级过程设置测试、灰度和回滚步骤。

### 10.2 不要无条件忽略错误

示例脚本中存在：

```bash
swapoff -a || true
modprobe overlay || true
modprobe br_netfilter || true
```

这种写法可以提高实验环境脚本的连续执行能力，但也可能掩盖真实错误。生产脚本应在忽略错误前：

- 判断命令失败是否可接受；
- 输出明确告警；
- 对关键能力进行后置验证；
- 必要时终止执行。

### 10.3 版本和变更管理

- 统一记录 OS、内核、containerd、kubelet、kubeadm、kubectl 版本；
- 不依赖自动升级改变节点状态；
- 升级前确认 Kubernetes 版本偏差策略；
- 保存 containerd 和 sysctl 配置；
- 变更后在一台非关键节点进行验证。

### 10.4 资源和时间同步

Kubernetes 节点还应具备足够的 CPU、内存、磁盘和 inode，并保持时间同步。证书校验、Lease、日志时间线和控制器逻辑都依赖合理的系统时间。

---

## 11. 验收清单

在继续执行 kubeadm 相关步骤前，逐项确认：

```bash
# 1. swap 已关闭
swapon --show

# 2. 内核模块存在
lsmod | grep -E 'overlay|br_netfilter'

# 3. 内核参数正确
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward

# 4. containerd 正常运行
containerd --version
systemctl is-enabled containerd
systemctl is-active containerd

grep -n 'SystemdCgroup' /etc/containerd/config.toml

# 5. Kubernetes 工具已安装并版本一致
kubeadm version
kubectl version --client
apt-mark showhold
```

当所有节点均通过以上检查后，才进入下一阶段的 kubeadm 初始化、Worker 加入或 Harbor 信任配置。

---

## 12. 小结

这三个脚本建立的是 Kubernetes 节点的“操作系统与运行时基线”：

```text
关闭 swap
  + 加载容器网络所需内核能力
  + 开启转发和桥接过滤
  + 安装并统一配置 containerd
  + 安装并锁定 Kubernetes 工具版本
  = 具备进入 kubeadm 部署阶段的节点
```

理解这些步骤的原理，比只记住几条安装命令更重要。后续出现节点 NotReady、Pod 网络异常、容器运行时故障或 kubeadm 预检失败时，排查入口通常就在本篇建立的这些基础配置中。
