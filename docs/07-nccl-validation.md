# NCCL 集合通信验证

通过 nccl-tests、NCCL 日志和 RDMA 工具验证跨节点 AllReduce 等集合通信。

实验记录中的参考配置为 `QP=1、Channel=1`，曾记录 RDMA 带宽 23.17 Gbps（约 25GbE 线速的 92.7%）和 NCCL Bus BW 约 2.30 GB/s；这些数值必须附带硬件、消息大小、进程布局和命令，不能脱离环境直接承诺。

每次验证至少记录：

- GPU、驱动、CUDA、NCCL、nccl-tests 版本。
- 节点数、GPU 数、网卡/HCA、GID index、接口和环境变量。
- 测试命令、消息范围、平均/峰值带宽和错误信息。
- NCCL 日志、RDMA 计数器和证据路径。

常见问题包括 GID 不一致、接口选择错误、镜像缺少 RDMA 能力和端口/防火墙限制。
