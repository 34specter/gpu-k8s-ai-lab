# 双节点实验验收清单

## 主机与运行时

- [ ] 两节点版本、时间同步、GPU 和接口信息已记录。
- [ ] containerd、Harbor CA 和 GPU runtime 已验证。

## Kubernetes/GPU

- [ ] 控制平面和 Worker 为 Ready。
- [ ] Calico 网络正常，GPU Device Plugin 发布资源。
- [ ] 测试 Pod 能调度并访问 GPU。

## RoCE/IB/NCCL

- [ ] RoCE/IB 链路、GID、MTU、PFC/ECN/DCB 已确认。
- [ ] `ib_send_bw` 或等价 RDMA 测试有脱敏输出。
- [ ] NCCL 集合通信成功，日志和带宽指标可追溯。

## 训练与推理

- [ ] 双节点 DDP/LoRA 任务成功启动、运行并退出。
- [ ] Ray GPU 资源、模型路径和 vLLM TP=2 已验证。
- [ ] OpenAI-compatible API 通过内部测试请求。

## 证据与问题闭环

每项记录 `status`（planned/verified/blocked）、日期、环境版本、命令摘要、证据位置和问题编号。对故障使用“现象—分层排查—修复—指标验证—结论”的格式。
