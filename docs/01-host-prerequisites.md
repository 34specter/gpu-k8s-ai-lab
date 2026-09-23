# 主机前置条件

部署前逐节点确认并记录：

- Linux 发行版、内核、主机名、时间同步和 DNS/hosts。
- GPU 型号、驱动、CUDA、`nvidia-smi` 和容器内 GPU 可见性。
- 管理网、RoCE/IB 接口、MTU、链路状态和路由。
- containerd 1.7.x、CNI 依赖、镜像仓库访问和 Harbor CA 信任。
- 防火墙、端口、内核参数、ulimit、HugePages（如工作负载需要）。

实际命令和结果放入 `artifacts/` 的本地目录，并在验收清单中引用脱敏结论。不要在本文件写入真实地址或凭据。
