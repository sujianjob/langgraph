---
search:
  boost: 2
---

# 自托管数据平面 (Self-Hosted Data Plane)

自托管部署有两个版本：[自托管数据平面](./deployment_options.md#self-hosted-data-plane) 和 [自托管控制平面](./deployment_options.md#self-hosted-control-plane)。

!!! info "Important"

    自托管数据平面部署选项需要 [Enterprise](plans.md) 计划。

## 要求 (Requirements)

- 您使用 `langgraph-cli` 和/或 [LangGraph Studio](./langgraph_studio.md) 应用程序在本地测试图。
- 您使用 `langgraph build` 命令构建映像。

## 自托管数据平面 (Self-Hosted Data Plane)

[自托管数据平面](../cloud/deployment/self_hosted_data_plane.md) 部署选项是一种“混合”部署模式，我们在我们的云中管理 [控制平面](./langgraph_control_plane.md)，而您在您的云中管理 [数据平面](./langgraph_data_plane.md)。此选项提供一种安全管理数据平面基础设施的方法，同时将控制平面管理交给我们。使用自托管数据平面版本时，您使用 [LangSmith](https://smith.langchain.com/) API 密钥进行身份验证。

|                                    | [控制平面 (Control plane)](../concepts/langgraph_control_plane.md)                                                                                     | [数据平面 (Data plane)](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **它是什么？(What is it?)**                    | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul> | <ul><li>用于将部署与控制平面状态协调的数据平面“侦听器”</li><li>LangGraph Server</li><li>Postgres, Redis, 等</li></ul> |
| **它托管在哪里？(Where is it hosted?)**            | LangChain 的云 (LangChain's cloud)                                                                                                                           | 您的云 (Your cloud)                                                                                                                                          |
| **谁提供和管理它？(Who provisions and manages it?)** | LangChain                                                                                                                                   | 您 (You)                                                                                                                                                 |

有关如何将 [LangGraph Server](../concepts/langgraph_server.md) 部署到自托管数据平面的信息，请参阅 [部署到自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)

### 架构 (Architecture)

![Self-Hosted Data Plane Architecture](./img/self_hosted_data_plane_architecture.png)

### 计算平台 (Compute Platforms)

- **Kubernetes**: 自托管数据平面部署选项支持将数据平面基础设施部署到任何 Kubernetes 集群。
- **Amazon ECS**: 即将推出！

!!! tip
如果您想部署到 Kubernetes，可以遵循 [自托管数据平面部署指南](../cloud/deployment/self_hosted_data_plane.md)。
