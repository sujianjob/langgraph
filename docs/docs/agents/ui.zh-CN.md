---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# UI

您可以通过 [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 使用预构建的聊天 UI 与任何 LangGraph 代理进行交互。使用 [已部署版本](https://agentchat.vercel.app) 是最快的入门方式，并允许您与本地和已部署的图进行交互。

## 在 UI 中运行代理

首先，[在本地](../tutorials/langgraph-platform/local-server.md) 设置 LangGraph API 服务器，或在 [LangGraph Platform](https://langchain-ai.github.io/langgraph/cloud/quick_start/) 上部署您的代理。

然后，导航到 [Agent Chat UI](https://agentchat.vercel.app)，或克隆存储库并 [在本地运行开发服务器](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#setup)：

<video controls src="../assets/base-chat-ui.mp4" type="video/mp4"></video>

!!! Tip

    UI 对呈现工具调用和工具结果消息提供了开箱即用的支持。要自定义显示的消息，请参阅 Agent Chat UI 文档中的 [在聊天中隐藏消息](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat) 部分。

## 添加人机交互 (Add human-in-the-loop)

Agent Chat UI 完全支持 [人机交互 (human-in-the-loop)](../concepts/human_in_the_loop.md) 工作流。要尝试它，请将 `src/agent/graph.py` 中的代理代码（来自 [部署](../tutorials/langgraph-platform/local-server.md) 指南）替换为此 [代理实现](../how-tos/human_in_the_loop/add-human-in-the-loop.md#add-interrupts-to-any-tool)：

<video controls src="../assets/interrupt-chat-ui.mp4" type="video/mp4"></video>

!!! Important

    如果您的 LangGraph 代理使用 @[`HumanInterrupt` schema][HumanInterrupt] 进行中断，Agent Chat UI 的效果最佳。如果您不使用该架构，Agent Chat UI 将能够呈现传递给 `interrupt` 函数的输入，但它将不完全支持恢复您的图。

## 生成式 UI (Generative UI)

您还可以在 Agent Chat UI 中使用生成式 UI。

生成式 UI 允许您定义 [React](https://react.dev/) 组件，并将它们从 LangGraph 服务器推送到 UI。有关构建生成式 UI LangGraph 代理的更多文档，请阅读 [这些文档](https://langchain-ai.github.io/langgraph/cloud/how-tos/generative_ui_react/)。
