# 指南 (Guides)

本节中的页面提供有关以下主题的概念概述和操作方法：

## 代理开发 (Agent development)

- [概述 (Overview)](../agents/overview.md)：使用预构建组件构建代理。
- [运行代理 (Run an agent)](../agents/run_agents.md)：通过提供输入、解释输出、启用流式传输和控制执行限制来运行代理。

## LangGraph API

- [Graph API](../concepts/low_level.md)：使用 Graph API 通过图范式定义工作流。
- [Functional API](../concepts/functional_api.md)：使用 Functional API 通过函数式范式构建工作流，而无需考虑图结构。
- [运行时 (Runtime)](../concepts/pregel.md)：Pregel 实现 LangGraph 的运行时，管理 LangGraph 应用程序的执行。

## 核心功能 (Core capabilities)

这些功能在 LangGraph OSS 和 LangGraph Platform 中均可用。

- [流式传输 (Streaming)](../concepts/streaming.md)：从 LangGraph 图中流式传输输出。
- [持久性 (Persistence)](../concepts/persistence.md)：持久化 LangGraph 图的状态。
- [持久执行 (Durable execution)](../concepts/durable_execution.md)：在图执行的关键点保存进度。
- [记忆 (Memory)](../concepts/memory.md)：记住有关先前交互的信息。
- [上下文 (Context)](../agents/context.md)：将外部数据传递给 LangGraph 图，为图执行提供上下文。
- [模型 (Models)](../agents/models.md)：将各种 LLM 集成到您的 LangGraph 应用程序中。
- [工具 (Tools)](../concepts/tools.md)：直接与外部系统交互。
- [人机交互 (Human-in-the-loop)](../concepts/human_in_the_loop.md)：在工作流的任何点暂停图并等待人工输入。
- [时间旅行 (Time travel)](../concepts/time-travel.md)：回到 LangGraph 图执行中的特定点。
- [子图 (Subgraphs)](../concepts/subgraphs.md)：构建模块化图。
- [多代理 (Multi-agent)](../concepts/multi_agent.md)：将复杂的工作流分解为多个代理。
- [MCP](../concepts/mcp.md)：在 LangGraph 图中使用 MCP 服务器。
- [评估 (Evaluation)](../agents/evals.md)：使用 LangSmith 评估图的性能。
