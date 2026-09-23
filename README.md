# NCP-AIN GPU 集群实验平台

一个面向双节点 GPU 实验环境的部署、验证与问题排查仓库。

本项目以可复现的实验流程为主线，覆盖从主机初始化到 AI 工作负载运行的完整链路：

```text
Linux 主机
  → containerd / GPU 容器运行时
  → Kubernetes / GPU 资源
  → RoCE / InfiniBand / RDMA
  → NCCL 集合通信
  → PyTorch DDP 分布式训练
  → KubeRay + vLLM 推理服务
```

## 项目范围

本仓库用于整理以下内容：

- 双节点 GPU 实验环境的主机、容器和 Kubernetes 配置；
- RoCE v2、InfiniBand、RDMA 和 NCCL 的分层验证方法；
- PyTorch `torchrun` / DDP 训练任务和 LoRA 实验；
- KubeRay、vLLM、Tensor Parallelism 和 OpenAI 兼容 API；
- 模型存储、Ingress、TLS、监控指标和常见故障排查；
- 部署命令、验证结果和脱敏实验记录。

> **实验环境说明**：本项目针对两节点实验集群设计，不代表生产级高可用集群或大规模集群交付方案。配置模板中的地址、域名、凭据和版本参数需要结合实际环境确认。

## 实验环境与技术栈

| 类别 | 组件 |
| --- | --- |
| 主机与容器 | Linux、containerd 1.7.x、NVIDIA Container Toolkit、Harbor |
| Kubernetes | Kubernetes v1.28.2、kubeadm、kubelet、kubectl、Calico v3.26.1、Ingress-Nginx |
| GPU 与网络 | NVIDIA GPU、Mellanox ConnectX、RoCE v2 25GbE、InfiniBand 100G、RDMA |
| 通信与训练 | CUDA、NCCL 2.19.x、nccl-tests、MPI、PyTorch 2.2、torchrun、DDP、LoRA |
| 推理与存储 | KubeRay、Ray 2.53、vLLM 0.6.x、Tensor Parallelism、NVMe-oF over IB |

具体版本以 [`configs/versions.example.env`](configs/versions.example.env) 为准；版本升级后应重新执行对应验证流程。

## 仓库结构

| 路径 | 说明 |
| --- | --- |
| [`docs/`](docs/) | 分阶段部署文档、网络说明、验收清单和故障排查规范 |
| [`inventory/`](inventory/) | 节点、硬件、软件和网络拓扑模板 |
| [`configs/`](configs/) | 版本、NCCL、训练和推理参数模板 |
| [`deploy/`](deploy/) | containerd、Kubernetes、GPU、网络、训练和推理部署入口 |
| [`scripts/`](scripts/) | 按主机、集群、网络、训练、推理和验证划分的脚本入口 |
| [`tests/`](tests/) | 平台 Smoke、NCCL、训练和推理验证入口 |
| [`examples/`](examples/) | 不包含真实地址和凭据的命令示例 |
| [`artifacts/`](artifacts/) | 本地生成日志和报告的目录，内容默认不提交 |

## 推荐阅读与执行顺序

1. 阅读 [`docs/00-project-scope.md`](docs/00-project-scope.md)，确认项目边界和记录规则。
2. 根据 [`inventory/README.md`](inventory/README.md) 和配置模板整理本地实验环境信息。
3. 按主机前置条件、containerd、Kubernetes 和 GPU runtime 顺序完成基础环境准备。
4. 按 [`docs/04-roce-ib-network.md`](docs/04-roce-ib-network.md) 验证网络、RDMA 和存储链路。
5. 按 [`docs/05-nccl-validation.md`](docs/05-nccl-validation.md) 验证跨节点集合通信。
6. 按 [`docs/06-distributed-training.md`](docs/06-distributed-training.md) 验证 DDP/LoRA 训练任务。
7. 按 [`docs/07-kuberay-vllm-serving.md`](docs/07-kuberay-vllm-serving.md) 验证多卡推理服务。
8. 使用 [`docs/08-acceptance-checklist.md`](docs/08-acceptance-checklist.md) 汇总每一层的结果和证据。

## 配置方式

示例配置只用于说明字段和参数位置。使用时复制到本地文件，再填写真实环境值：

```bash
cp inventory/cluster.example.yaml inventory/cluster.yaml
cp inventory/topology.example.yaml inventory/topology.yaml
cp configs/nccl-golden.env.example configs/nccl-golden.env
cp configs/training.env.example configs/training.env
cp configs/serving.env.example configs/serving.env
```

本地配置文件已由 [`.gitignore`](.gitignore) 排除。请勿将以下内容提交到仓库：

- 节点真实 IP、主机名、序列号和内部域名；
- SSH 密钥、TLS 私钥、Registry 凭据和 API Token；
- 模型权重、checkpoint、运行日志和未脱敏的故障转储；
- 包含 Secret 内容或真实 `kubeconfig` 的文件。

## 验证记录

每次实验建议记录以下信息：

- 实验日期、节点和软件版本；
- 执行命令及关键参数；
- 预期结果与实际结果；
- 日志、指标或截图的脱敏路径；
- 异常现象、排查过程、修复内容和最终结论。

只有具备可追溯验证证据的内容，才标记为 `verified`。规划中的内容使用 `planned`，暂时无法运行的内容使用 `blocked`，避免将模板误认为已经完成的部署。

## 当前状态

仓库当前提供项目文档、目录骨架和安全的示例模板。实际部署脚本、Kubernetes 清单和测试结果应随着实验验证逐步补充，不直接复制未经确认的生产配置。

## 免责声明

本仓库中的命令和配置面向受控实验环境。执行涉及网络、DNS、GPU 驱动、containerd、Kubernetes、RDMA 或存储的变更前，请先确认环境、备份配置，并准备带外恢复方式。
