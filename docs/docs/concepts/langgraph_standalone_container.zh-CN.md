---
search:
  boost: 2
---

# 独立容器 (Standalone Container)

要部署 [LangGraph Server](../concepts/langgraph_server.md)，请遵循 [如何部署独立容器](../cloud/deployment/standalone_container.md) 的操作指南。

## 概述 (Overview)

独立容器部署选项是限制最少的部署模型。没有 [控制平面](./langgraph_control_plane.md)。[数据平面](./langgraph_data_plane.md) 基础设施由您管理。

|                   | [控制平面](../concepts/langgraph_control_plane.md) | [数据平面](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **它是什么？** | 不适用 | <ul><li>LangGraph Servers</li><li>Postgres, Redis, 等</li></ul> |
| **它托管在哪里？** | 不适用 | 您的云环境 |
| **谁配置和管理它？** | 不适用 | 您 |

!!! warning

      LangGraph Platform 不应部署在无服务器环境中。归零扩展可能会导致任务丢失，并且扩展将无法可靠地工作。

## 架构 (Architecture)

![Standalone Container](./img/langgraph_platform_deployment_architecture.png)

## 计算平台 (Compute Platforms)

### Kubernetes

独立容器部署选项支持将数据平面基础设施部署到 Kubernetes 集群。

### Docker

独立容器部署选项支持将数据平面基础设施部署到任何支持 Docker 的计算平台。
