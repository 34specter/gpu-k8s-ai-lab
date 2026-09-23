# PyTorch DDP 分布式训练

目标是使用 `torchrun + NCCL` 在双节点上实践 Qwen2.5-1.5B LoRA 微调，同时保留常驻 Pod 调试模式和 Kubernetes Indexed Job 自动化模式。

部署前确认：

- 两节点镜像、模型路径和依赖版本一致。
- `MASTER_ADDR`、端口、rank/world size 和 GPU 资源正确。
- NCCL/RDMA 环境变量来自已验证的通信配置，而非盲目复制。
- 输出目录、日志和 checkpoint 有明确的生命周期。

验收应包含启动、跨节点通信、训练步数/损失、正常退出和失败重试行为。
