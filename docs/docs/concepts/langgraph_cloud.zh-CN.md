---
search:
  boost: 2
---

# Cloud SaaS

要部署 [LangGraph Server](../concepts/langgraph_server.md)，请遵循 [如何部署到 Cloud SaaS](../cloud/deployment/cloud.md) 的操作指南。

## 概述 (Overview)

Cloud SaaS 部署选项是一种完全托管的部署模式，我们在我们的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。

|                                    | [控制平面 (Control plane)](../concepts/langgraph_control_plane.md)                                                                                     | [数据平面 (Data plane)](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **它是什么？(What is it?)**                    | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul> | <ul><li>用于将部署与控制平面状态协调的数据平面“侦听器”</li><li>LangGraph Server</li><li>Postgres, Redis, 等</li></ul> |
| **它托管在哪里？(Where is it hosted?)**            | LangChain 的云 (LangChain's cloud)                                                                                                                           | LangChain 的云 (LangChain's cloud)                                                                                                                                   |
| **谁提供和管理它？(Who provisions and manages it?)** | LangChain                                                                                                                                   | LangChain                                                                                                                                           |

## 架构 (Architecture)

![Cloud SaaS](./img/self_hosted_control_plane_architecture.png)
