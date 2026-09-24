# GPU 容器运行时

覆盖 containerd、NVIDIA Container Toolkit、Harbor 私有仓库 CA 信任和 NVIDIA Device Plugin。

验收重点：

- containerd 使用预期 runtime，并能拉取实验镜像。
- 节点主机和容器内 GPU 查询结果一致。
- Device Plugin 在两节点发布预期 GPU 资源。
- Harbor CA 配置不通过跳过 TLS 校验解决问题。
- 记录镜像、runtime、插件版本和 `kubectl describe node` 摘要。

证书私钥、registry 凭据和完整 secret 内容只保留在受控环境。
