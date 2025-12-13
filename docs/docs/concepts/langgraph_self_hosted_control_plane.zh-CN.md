# 自托管控制平面 (Self-Hosted Control Plane)

自托管部署有两个版本：[自托管数据平面](./deployment_options.md#self-hosted-data-plane) 和 [自托管控制平面](./deployment_options.md#self-hosted-control-plane)。

!!! info "Important"

    自托管控制平面部署选项需要 [Enterprise](plans.md) 计划。

## 要求 (Requirements)

- 您使用 [LangGraph CLI](./langgraph_cli.md) 和/或 [LangGraph Studio](./langgraph_studio.md) 应用程序在本地测试图。
- 您使用 `langgraph build` 命令构建映像。
- 您已部署自托管 LangSmith 实例。
- 您正在为 LangSmith 实例使用 Ingress。所有代理都将在该 ingress 后面部署为 Kubernetes 服务。

## 自托管控制平面 (Self-Hosted Control Plane)

[自托管控制平面](./langgraph_self_hosted_control_plane.md) 部署选项是一种完全自托管的部署模式，您在您的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项使您可以完全控制和负责控制平面和数据平面基础设施。

|                                    | [控制平面 (Control plane)](../concepts/langgraph_control_plane.md)                                                                                     | [数据平面 (Data plane)](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **它是什么？ (What is it?)**                    | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul> | <ul><li>用于将部署与控制平面状态协调的数据平面“侦听器”</li><li>LangGraph Server</li><li>Postgres, Redis, 等</li></ul> |
| **它托管在哪里？ (Where is it hosted?)**            | 您的云 (Your cloud)                                                                                                                                  | 您的云 (Your cloud)                                                                                                                                          |
| **谁提供和管理它？ (Who provisions and manages it?)** | 您 (You)                                                                                                                                         | 您 (You)                                                                                                                                                 |

### 架构 (Architecture)

![Self-Hosted Control Plane Architecture](./img/self_hosted_control_plane_architecture.png)

### 计算平台 (Compute Platforms)

- **Kubernetes**: 自托管控制平面部署选项支持将控制平面和数据平面基础设施部署到任何 Kubernetes 集群。

!!! tip
如果您想在您的 LangSmith 实例上启用此功能，请遵循 [自托管控制平面部署指南](../cloud/deployment/self_hosted_control_plane.md)。
