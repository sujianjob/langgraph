---
search:
  boost: 2
---

# 部署选项 (Deployment Options)

## 免费部署 (Free deployment)

[本地 (Local)](../tutorials/langgraph-platform/local-server.md): 部署用于本地测试和开发。

## 生产部署 (Production deployment)

使用 [LangGraph 平台 (LangGraph Platform)](langgraph_platform.md) 部署主要有 4 种选项：

1. [Cloud SaaS](#cloud-saas)

1. [自托管数据平面 (Self-Hosted Data Plane)](#self-hosted-data-plane)

1. [自托管控制平面 (Self-Hosted Control Plane)](#self-hosted-control-plane)

1. [独立容器 (Standalone Container)](#standalone-container)


快速比较：

|                      | **Cloud SaaS** | **Self-Hosted Data Plane** | **Self-Hosted Control Plane** | **Standalone Container** |
|----------------------|----------------|----------------------------|-------------------------------|--------------------------|
| **[控制平面 UI/API (Control plane UI/API)](../concepts/langgraph_control_plane.md)** | 是 (Yes) | 是 (Yes) | 是 (Yes) | 否 (No) |
| **CI/CD** | 由平台内部管理 (Managed internally by platform) | 由您外部管理 (Managed externally by you) | 由您外部管理 (Managed externally by you) | 由您外部管理 (Managed externally by you) |
| **数据/计算驻留 (Data/compute residency)** | LangChain 的云 (LangChain's cloud) | 您的云 (Your cloud) | 您的云 (Your cloud) | 您的云 (Your cloud) |
| **LangSmith 兼容性 (LangSmith compatibility)** | 跟踪到 LangSmith SaaS (Trace to LangSmith SaaS) | 跟踪到 LangSmith SaaS (Trace to LangSmith SaaS) | 跟踪到自托管 LangSmith (Trace to Self-Hosted LangSmith) | 可选跟踪 (Optional tracing) |
| **[定价 (Pricing)](https://www.langchain.com/pricing-langgraph-platform)** | Plus | Enterprise | Enterprise | Enterprise |

## Cloud SaaS

[Cloud SaaS](./langgraph_cloud.md) 部署选项是一种完全托管的部署模式，我们在我们的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项提供了部署和管理 LangGraph Server 的简便方法。

将您的 GitHub 存储库连接到平台，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。构建过程（即 CI/CD）由平台内部管理。

有关更多信息，请参阅：

* [Cloud SaaS 概念指南](./langgraph_cloud.md)
* [如何部署到 Cloud SaaS](../cloud/deployment/cloud.md)

## 自托管数据平面 (Self-Hosted Data Plane)

!!! info "Important"
    自托管数据平面部署选项需要 [Enterprise](../concepts/plans.md) 计划。

[自托管数据平面](./langgraph_self_hosted_data_plane.md) 部署选项是一种“混合”部署模式，我们在我们的云中管理 [控制平面](./langgraph_control_plane.md)，而您在您的云中管理 [数据平面](./langgraph_data_plane.md)。此选项提供了一种安全管理数据平面基础设施的方法，同时将控制平面管理交给我们。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 映像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。

支持的计算平台：[Kubernetes](https://kubernetes.io/)，[Amazon ECS](https://aws.amazon.com/ecs/) (即将推出!)

有关更多信息，请参阅：

* [自托管数据平面概念指南](./langgraph_self_hosted_data_plane.md)
* [如何部署自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)

## 自托管控制平面 (Self-Hosted Control Plane)

!!! info "Important"
    自托管控制平面部署选项需要 [Enterprise](../concepts/plans.md) 计划。

[自托管控制平面](./langgraph_self_hosted_control_plane.md) 部署选项是一种完全自托管的部署模式，您在您的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项使您可以完全控制和负责控制平面和数据平面基础设施。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 映像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。

支持的计算平台：[Kubernetes](https://kubernetes.io/)

有关更多信息，请参阅：

* [自托管控制平面概念指南](./langgraph_self_hosted_control_plane.md)
* [如何部署自托管控制平面](../cloud/deployment/self_hosted_control_plane.md)

## 独立容器 (Standalone Container)

[独立容器](./langgraph_standalone_container.md) 部署选项是限制最少的部署模式。使用任何 [可用](./plans.md) 许可证选项，在您的云中部署 LangGraph Server 的独立实例。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 映像，并使用您选择的容器部署工具部署 LangGraph Server。映像可以部署到任何计算平台。

有关更多信息，请参阅：

* [独立容器概念指南](./langgraph_standalone_container.md)
* [如何部署独立容器](../cloud/deployment/standalone_container.md)

## 相关内容 (Related)

有关更多信息，请参阅：

* [LangGraph 平台计划](./plans.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
