---
search:
  boost: 2
---

# 代理架构 (Agent architectures)

许多 LLM 应用程序在 LLM 调用之前和/或之后实现特定的步骤控制流。例如，[RAG](https://github.com/langchain-ai/rag-from-scratch) 执行与用户问题相关的文档检索，并将这些文档传递给 LLM，以便根据提供的文档上下文进行模型响应。

有时我们不想硬编码固定的控制流，而是希望 LLM 系统能够选择自己的控制流来解决更复杂的问题！这是 [代理 (agent)](https://blog.langchain.dev/what-is-an-agent/) 的一个定义：*代理是使用 LLM 来决定应用程序控制流的系统。* LLM 可以通过多种方式控制应用程序：

- LLM 可以在两条潜在路径之间进行路由
- LLM 可以决定在许多工具中调用哪一个
- LLM 可以决定生成的答案是否足够，或者是否需要更多工作

结果，存在许多不同类型的 [代理架构](https://blog.langchain.dev/what-is-a-cognitive-architecture/)，这使得 LLM 具有不同级别的控制权。

![Agent Types](img/agent_types.png)

## 路由器 (Router)

路由器允许 LLM 从一组指定的选项中选择单个步骤。这是一种代理架构，表现出相对有限的控制级别，因为 LLM 通常专注于做出单个决策，并从有限的一组预定义选项中产生特定输出。路由器通常采用几个不同的概念来实现这一点。

### 结构化输出 (Structured Output)

使用 LLM 的结构化输出通过提供特定的格式或架构来工作，LLM 应在其响应中遵循这些格式或架构。这类似于工具调用，但更通用。虽然工具调用通常涉及选择和使用预定义函数，但结构化输出可用于任何类型的格式化响应。实现结构化输出的常用方法包括：

1. 提示工程：通过系统提示指示 LLM 以特定格式响应。
2. 输出解析器：使用后处理从 LLM 响应中提取结构化数据。
3. 工具调用：利用某些 LLM 的内置工具调用功能来生成结构化输出。

结构化输出对于路由至关重要，因为它们确保 LLM 的决策可以被系统可靠地解释和执行。在 [此操作指南中了解有关结构化输出的更多信息](https://python.langchain.com/docs/how_to/structured_output/)。

## 工具调用代理 (Tool-calling agent)

虽然路由器允许 LLM 做出单个决策，但更复杂的代理架构通过两种主要方式扩展了 LLM 的控制：

1. 多步决策：LLM 可以一个接一个地做出的一系列决策，而不仅仅是一个。
2. 工具访问：LLM 可以从各种工具中进行选择并使用它们来完成任务。

[ReAct](https://arxiv.org/abs/2210.03629) 是一种流行的通用代理架构，它结合了这些扩展，集成了三个核心概念。

1. [工具调用 (Tool calling)](#tool-calling): 允许 LLM 根据需要选择和使用各种工具。
2. [记忆 (Memory)](#memory): 使代理能够保留和使用先前步骤中的信息。
3. [规划 (Planning)](#planning): 使 LLM 能够创建并遵循多步计划来实现目标。

这种架构允许更复杂和更灵活的代理行为，超越简单的路由，实现具有多步动态问题解决。与原始 [论文](https://arxiv.org/abs/2210.03629) 不同，今天的代理依赖于 LLM 的 [工具调用](#tool-calling) 功能，并对 [消息](./low_level.md#why-use-messages) 列表进行操作。

在 LangGraph 中，您可以使用预构建的 [代理](../agents/agents.md#2-create-an-agent) 开始使用工具调用代理。

### 工具调用 (Tool calling)

每当您希望代理与外部系统交互时，工具都非常有用。外部系统（例如 API）通常需要特定的输入模式或有效负载，而不是自然语言。例如，当我们绑定 API 作为工具时，我们会让模型了解所需的输入模式。模型将选择基于用户的自然语言输入来调用工具，并且它将返回符合工具所需架构的输出。

[许多 LLM 提供商支持工具调用](https://python.langchain.com/docs/integrations/chat/)，并且 LangChain 中的 [工具调用接口](https://blog.langchain.dev/improving-core-tool-interfaces-and-docs-in-langchain/) 很简单：您只需将任何 Python `函数` 传递给 `ChatModel.bind_tools(function)`。

![Tools](img/tool_call.png)

### 记忆 (Memory)

[记忆](../how-tos/memory/add-memory.md) 对于代理至关重要，使它们能够在解决问题的多个步骤中保留和利用信息。它在不同的规模上运作：

1. [短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory): 允许代理访问在序列中较早步骤获取的信息。
2. [长期记忆](../how-tos/memory/add-memory.md#add-long-term-memory): 使代理能够回顾以前交互中的信息，例如对话中的过去消息。

LangGraph 提供对记忆实现的完全控制：

- [`State`](./low_level.md#state): 用户定义的架构，指定要保留的记忆的确切结构。
- [`Checkpointer`](./persistence.md#checkpoints): 在会话中的不同交互之间每一步存储状态的机制。
- [`Store`](./persistence.md#memory-store): 跨会话存储用户特定或应用程序级数据的机制。

这种灵活的方法允许您根据特定的代理架构需求定制记忆系统。有效的记忆管理增强了代理维持上下文、从过去的经验中学习以及随着时间的推移做出更明智决策的能力。有关添加和管理记忆的实用指南，请参阅 [记忆](../how-tos/memory/add-memory.md)。

### 规划 (Planning)

在工具调用 [代理](../agents/overview.md#what-is-an-agent) 中，LLM 在 while 循环中被重复调用。在每一步，代理决定要调用哪些工具，以及这些工具的输入应该是什么。然后执行这些工具，并将输出作为观察结果反馈给 LLM。当代理决定它有足够的信息来解决用户请求并且不值得调用更多工具时，while 循环终止。

## 自定义代理架构 (Custom agent architectures)

虽然路由器和工具调用代理（如 ReAct）很常见，但 [自定义代理架构](https://blog.langchain.dev/why-you-should-outsource-your-agentic-infrastructure-but-own-your-cognitive-architecture/) 通常会为特定任务带来更好的性能。LangGraph 提供了几个强大的功能来构建定制的代理系统：

### 人机回圈 (Human-in-the-loop)

人工参与可以显着提高代理的可靠性，特别是对于敏感任务。这可能涉及：

- 批准特定操作
- 提供反馈以更新代理的状态
- 在复杂的决策过程中提供指导

当完全自动化不可行或不理想时，人机回圈模式至关重要。要在我们的 [人机回圈指南](./human_in_the_loop.md) 中了解更多信息。

### 并行化 (Parallelization)

并行处理对于高效的多代理系统和复杂任务至关重要。LangGraph 通过其 [Send](./low_level.md#send) API 支持并行化，从而实现：

- 多个状态的并发处理
- map-reduce 类操作的实现
- 独立子任务的高效处理

有关实际实现，请参阅我们的 [map-reduce 教程](../how-tos/graph-api.md#map-reduce-and-the-send-api)

### 子图 (Subgraphs)

[子图](./subgraphs.md) 对于管理复杂的代理架构至关重要，尤其是在 [多代理系统](./multi_agent.md) 中。它们允许：

- 单个代理的隔离状态管理
- 代理团队的层级组织
- 代理与主系统之间的受控通信

子图通过状态架构中的重叠键与父图通信。这实现了灵活的模块化代理设计。有关实现细节，请参阅我们的 [子图操作指南](../how-tos/subgraph.md)。

### 反思 (Reflection)

反思机制可以通过以下方式显着提高代理的可靠性：

1. 评估任务的完成情况和正确性
2. 提供反馈以进行迭代改进
3. 启用自我纠正和学习

虽然通常基于 LLM，但反思也可以使用确定性方法。例如，在编码任务中，编译错误可以作为反馈。在 [此视频中使用 LangGraph 进行自我纠正代码生成](https://www.youtube.com/watch?v=MvNdgmM7uyc) 中演示了这种方法。

通过利用这些功能，LangGraph 能够创建复杂的、特定于任务的代理架构，这些架构可以处理复杂的工作流、有效协作并不断提高其性能。
