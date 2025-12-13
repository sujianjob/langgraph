---
search:
boost: 2
---

# 流式传输 (Streaming)

LangGraph 实现了一个流式传输系统来展示实时更新，从而实现响应迅速且透明的用户体验。

LangGraph 的流式传输系统允许您将图运行的实时反馈呈现给您的应用程序。
您可以流式传输三种主要类型的数据：

1. **工作流进度 (Workflow progress)** — 在每个图节点执行后获取状态更新。
2. **LLM 令牌 (LLM tokens)** — 在生成时流式传输语言模型令牌。
3. **自定义更新 (Custom updates)** — 发出用户定义的信号（例如，“已获取 10/100 条记录”）。

## LangGraph 流式传输能做什么 (What’s possible with LangGraph streaming)

- [**流式传输 LLM 令牌 (Stream LLM tokens)**](../how-tos/streaming.md#messages) — 从任何地方捕获令牌流：节点内部、子图或工具。
- [**从工具发出进度通知 (Emit progress notifications from tools)**](../how-tos/streaming.md#stream-custom-data) — 直接从工具函数发送自定义更新或进度信号。
- [**从子图流式传输 (Stream from subgraphs)**](../how-tos/streaming.md#stream-subgraph-outputs) — 包括来自父图和任何嵌套子图的输出。
- [**使用任何 LLM (Use any LLM)**](../how-tos/streaming.md#use-with-any-llm) — 从任何 LLM 流式传输令牌，即使它不是使用 `custom` 流模式的 LangChain 模型。
- [**使用多种流模式 (Use multiple streaming modes)**](../how-tos/streaming.md#stream-multiple-modes) — 从 `values`（完整状态）、`updates`（状态增量）、`messages`（LLM 令牌 + 元数据）、`custom`（任意用户数据）或 `debug`（详细跟踪）中进行选择。
