# NCP-AIN 双节点 GPU 集群与 AI 训练/推理部署平台

面向 AI 服务器部署场景，在双节点 GPU 实验环境中实践从 Linux 节点初始化、Kubernetes 集群搭建、GPU 容器运行时配置，到 RoCE/InfiniBand 网络验证、NCCL 集合通信、PyTorch DDP 分布式训练和 KubeRay + vLLM 多卡推理服务的完整链路。

> **实验边界**：本项目面向双节点实验环境，不将其包装为生产级大规模集群交付经验。仓库中的配置模板使用占位符；只有经过实际验证并附有证据的内容，才应标记为已完成。

## 项目目标

打通以下链路：

```text
主机初始化 → 容器运行时 → Kubernetes/GPU → RoCE/IB/RDMA → NCCL
    → PyTorch DDP/LoRA 训练 → KubeRay + vLLM 多卡推理 API
```

当前工作聚焦 S5000 卡测试，同时沉淀 GPU 性能测试、问题复现、数据采集和故障定位方法，并将这些能力延伸到双节点服务器部署、GPU 集群运维、高速网络和 AI 服务上线。

## 技术栈

- **服务器与容器**：Linux、containerd 1.7.x、NVIDIA Container Toolkit、Harbor 私有镜像仓库
- **Kubernetes**：Kubernetes v1.28.2、kubeadm、kubelet、kubectl、Calico v3.26.1、Ingress-Nginx、Nginx、TLS
- **GPU 与高速网络**：NVIDIA RTX 2000 Ada 16GB × 2、Mellanox ConnectX、RoCE v2 25GbE、InfiniBand 100G、RDMA、PFC/ECN/DCB
- **分布式通信与训练**：CUDA、NCCL 2.19.x、nccl-tests、MPI、PyTorch 2.2、torchrun、DDP、LoRA
- **推理与存储**：KubeRay、Ray 2.53、vLLM 0.6.x、Tensor Parallelism、NVMe-oF over IB、OpenAI-compatible API

## 项目成果记录

以下内容是项目目标和成果记录的入口，具体完成状态应以 `docs/08-acceptance-checklist.md` 及脱敏证据为准：

- 双节点 Kubernetes GPU 实验集群：节点前置初始化、containerd、Harbor CA、kubeadm、Worker、Calico、NVIDIA Device Plugin、Ingress 和宿主机 Nginx。
- RoCE/NCCL 跨节点通信：使用 `ib_send_bw`、NCCL AllReduce、NCCL 日志、RDMA 计数器和 GPU 监控分层验证；实验记录中的 Golden Config 为 `QP=1、Channel=1`，曾测得 RDMA 带宽 **23.17 Gbps**、约 25GbE 线速的 **92.7%**，NCCL Bus BW 约 **2.30 GB/s**。
- 双节点 PyTorch DDP 训练：使用 `torchrun + NCCL` 实践 Qwen2.5-1.5B LoRA 微调，并保留常驻 Pod 调试模式与 Kubernetes Indexed Job 自动化模式。
- KubeRay + vLLM 多卡推理：针对约 28GB 的 Qwen2.5-14B，实践双节点 TP=2、NVMe-oF over IB 共享模型存储、Service、Ingress、TLS 和 OpenAI 兼容 API。
- 问题闭环：覆盖镜像拉取、NCCL 初始化、GID 不一致、Ray GPU 资源、模型路径和外部 HTTPS 等问题的分层排查与验证。

简历或对外材料应根据实际参与程度使用“参与完成”“独立复现”或“实践验证”等准确表述。

## 目录导航

| 目录 | 内容 |
| --- | --- |
| [`docs/`](docs/) | 分阶段部署文档、验收清单、故障排查和证据规范 |
| [`inventory/`](inventory/) | 双节点硬件、软件、网络拓扑模板 |
| [`configs/`](configs/) | 版本、NCCL、训练和推理参数模板 |
| [`deploy/`](deploy/) | containerd、Kubernetes、网络、训练和推理部署材料 |
| [`scripts/`](scripts/) | 按主机、集群、网络、训练、推理和验证分域的自动化脚本 |
| [`tests/`](tests/) | Smoke、NCCL、训练和推理验收入口 |
| [`examples/`](examples/) | 不含真实地址和凭据的命令示例 |
| [`artifacts/`](artifacts/) | 本地生成的日志和报告目录，内容默认不提交 |

## 推荐实施顺序

1. 填写 `inventory/` 和 `configs/` 中的本地副本，确认节点、接口、GPU、版本和通信路径。
2. 按 `docs/01`–`docs/03` 完成主机、containerd、Kubernetes 和 GPU 运行时前置检查。
3. 按 `docs/04`–`docs/05` 分层验证 RoCE/IB、RDMA、NCCL 和 Golden Config。
4. 按 `docs/06` 部署并验证 PyTorch DDP/LoRA 训练。
5. 按 `docs/07` 部署并验证 KubeRay + vLLM 推理服务。
6. 使用 `docs/08-acceptance-checklist.md` 汇总命令输出、指标、日志和问题闭环。

## 安全与提交约定

- 不提交真实 IP、主机名、序列号、访问令牌、密码、TLS 私钥、模型权重或未脱敏日志。
- 以 `.example`、`<node-a-ip>` 等占位符维护可分享模板；真实配置放在本地并由 `.gitignore` 保护。
- 不直接把“计划部署”写成“已完成”；部署材料应关联实际验证命令和证据位置。
- 生成的报告、日志和大文件放在 `artifacts/`，只提交必要的索引或结论。
