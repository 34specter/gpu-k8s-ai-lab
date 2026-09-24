# KubeRay + vLLM 多卡推理

目标是实践 KubeRay + vLLM 的双节点 TP=2 推理服务，为约 28GB 的 Qwen2.5-14B 提供 OpenAI-compatible API。

需分层验证：

1. Ray 节点状态和 GPU 资源总量/分配。
2. 两个节点对 NVMe-oF over IB 模型路径的可见性与读取性能。
3. vLLM 启动参数、显存占用、Tensor Parallel 分片和健康状态。
4. Service、Ingress、TLS 和 API 鉴权/访问路径。
5. 使用非敏感测试请求验证响应，并记录延迟、错误和日志。

模型权重、TLS 私钥、API token 和外部域名不提交；部署清单只有在实际环境验证后再加入。
