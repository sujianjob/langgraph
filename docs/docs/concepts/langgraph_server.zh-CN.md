---
search:
  boost: 2
---

# LangGraph Server

**LangGraph Server** 提供了一个用于创建和管理基于代理的应用程序的 API。它建立在 [助手 (assistants)](assistants.md) 的概念之上，助手是为特定任务配置的代理，并包括内置的 [持久化](persistence.md#memory-store) 和 **任务队列**。这种通用的 API 支持广泛的代理应用程序用例，从后台处理到实时交互。

使用 LangGraph Server 创建和管理 [助手](assistants.md)、[线程](./persistence.md#threads)、[运行](./assistants.md#execution)、[cron 作业](../cloud/concepts/cron_jobs.md)、[webhooks](../cloud/concepts/webhooks.md) 等。

!!! tip "API reference"
  
    有关 API 端点和数据模型的详细信息，请参阅 [LangGraph Platform API 参考文档](../cloud/reference/api/api_ref.html)。

## 应用程序结构 (Application structure)

要部署 LangGraph Server 应用程序，您需要指定要部署的图，以及任何相关的配置设置，例如依赖项和环境变量。

阅读 [应用程序结构](./application_structure.md) 指南以了解如何构建 LangGraph 应用程序以进行部署。

## 部署的组成部分 (Parts of a deployment)

当您部署 LangGraph Server 时，您正在部署一个或多个 [图 (graphs)](#graphs)、一个用于 [持久化](persistence.md) 的数据库和一个任务队列。

### 图 (Graphs)

当您使用 LangGraph Server 部署图时，您正在部署 [助手 (Assistant)](assistants.md) 的“蓝图”。

[助手](assistants.md) 是与特定配置设置配对的图。您可以为每个图创建多个助手，每个助手都具有独特的设置，以适应可以由同一图服务的不同用例。

部署后，LangGraph Server 将使用图的默认配置设置自动为每个图创建一个默认助手。

!!! note

    我们通常认为图实现了 [代理 (agent)](agentic_concepts.md)，但图不一定需要实现代理。例如，图可以实现一个仅支持来回对话的简单聊天机器人，而无需影响任何应用程序控制流的能力。实际上，随着应用程序变得越来越复杂，图通常会实现更复杂的流程，这可能会使用协同工作的 [多个代理](./multi_agent.md)。

### 持久化和任务队列 (Persistence and task queue)

LangGraph Server 利用数据库进行 [持久化](persistence.md) 和任务队列。

目前，仅支持 [Postgres](https://www.postgresql.org/) 作为 LangGraph Server 的数据库，[Redis](https://redis.io/) 作为任务队列。

如果您使用 [LangGraph Platform](./langgraph_cloud.md) 进行部署，这些组件将为您管理。如果您在自己的基础设施上部署 LangGraph Server，则需要自行设置和管理这些组件。

请查看 [部署选项](./deployment_options.md) 指南以获取有关如何设置和管理这些组件的更多信息。

## 了解更多 (Learn more)

* LangGraph [应用程序结构](./application_structure.md) 指南解释了如何构建 LangGraph 应用程序以进行部署。
* [LangGraph Platform API 参考](../cloud/reference/api/api_ref.html) 提供了有关 API 端点和数据模型的详细信息。
