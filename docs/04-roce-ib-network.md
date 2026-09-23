# RoCE/InfiniBand 网络验证

分层验证管理网、25GbE RoCE v2、100G InfiniBand、RDMA 和模型存储链路。

建议顺序：

1. 确认接口、链路、MTU、路由、PFC/ECN/DCB 和 GID。
2. 使用 `ib_send_bw` 或等价工具验证点到点吞吐和稳定性。
3. 对照 RDMA 计数器和 GPU 监控，排除只看单一指标。
4. 如启用 NVMe-oF over IB，分别验证发现、挂载、读写和两个节点可见性。
5. 保存环境矩阵、命令、关键指标和异常时间线。

真实接口名和地址填写到本地 inventory，不提交到公共仓库。
