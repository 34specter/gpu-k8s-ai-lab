# Kubernetes 集群初始化

目标组件：Kubernetes v1.28.2、kubeadm、kubelet、kubectl、Calico v3.26.1。

建议顺序：

1. 完成两节点前置检查并确定 Pod/Service 网段。
2. 在控制平面节点执行 kubeadm 初始化并安全保存 join 信息。
3. 安装 CNI，确认节点从 NotReady 变为 Ready。
4. 让 Worker 加入集群，检查节点标签、资源和事件。
5. 记录版本、配置摘要和验收输出。

本目录暂不提交包含真实 token 的 join 命令或未经验证的完整 YAML。
