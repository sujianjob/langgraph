# 无效的聊天记录 (INVALID_CHAT_HISTORY)

:::python
当 `call_model` 图节点接收到格式错误的消息列表时，预构建的 @[create_react_agent][create_react_agent] 会引发此错误。具体来说，当存在带有 `tool_calls`（LLM 请求调用工具）的 `AIMessages`，但没有相应的 `ToolMessage`（工具调用结果返回给 LLM）时，即为格式错误。
:::

:::js
当 `callModel` 图节点接收到格式错误的消息列表时，预构建的 @[createReactAgent][create_react_agent] 会引发此错误。具体来说，当存在带有 `tool_calls`（LLM 请求调用工具）的 `AIMessage`，但没有相应的 `ToolMessage`（工具调用结果返回给 LLM）时，即为格式错误。
:::

出现此错误的原因可能有以下几种：

:::python

1. 您在调用图时手动传递了格式错误的消息列表，例如 `graph.invoke({'messages': [AIMessage(..., tool_calls=[...])]})`
2. 图在收到来自 `tools` 节点的更新（即 ToolMessages 列表）之前被中断，
   并且您使用非 None 或非 ToolMessage 的输入调用了它，
   例如 `graph.invoke({'messages': [HumanMessage(...)]}, config)`。

   此中断可能是由以下方式之一触发的：

   - 您在 `create_react_agent` 中手动设置了 `interrupt_before = ['tools']`
   - 其中一个工具引发了未由 @[ToolNode][ToolNode] (`"tools"`) 处理的错误

:::

:::js

1. 您在调用图时手动传递了格式错误的消息列表，例如 `graph.invoke({messages: [new AIMessage({..., tool_calls: [...]})]})`
2. 图在收到来自 `tools` 节点的更新（即 ToolMessages 列表）之前被中断，
   并且您使用非 null 或非 ToolMessage 的输入调用了它，
   例如 `graph.invoke({messages: [new HumanMessage(...)]}, config)`。

   此中断可能是由以下方式之一触发的：

   - 您在 `createReactAgent` 中手动设置了 `interruptBefore: ['tools']`
   - 其中一个工具引发了未由 @[ToolNode][ToolNode] (`"tools"`) 处理的错误

:::

## 故障排除 (Troubleshooting)

要解决此问题，您可以执行以下操作之一：

1. 不要使用格式错误的消息列表调用图
2. 如果发生中断（手动或由于错误），您可以：

:::python

- 提供与现有工具调用匹配的 ToolMessages 并调用 `graph.invoke({'messages': [ToolMessage(...)]})`。
  **注意**：这会将消息附加到历史记录中，并从 START 节点运行图。

  - 手动更新状态并从中断处恢复图：

          1. 使用 `graph.get_state(config)` 获取图状态中的最新消息列表
          2. 修改消息列表以从 AIMessages 中删除未回答的工具调用

或者添加带有与未回答的工具调用匹配的 tool_call_ids 的 ToolMessages 3. 使用修改后的消息列表调用 `graph.update_state(config, {'messages': ...})` 4. 恢复图，例如调用 `graph.invoke(None, config)`
:::

:::js

- 提供与现有工具调用匹配的 ToolMessages 并调用 `graph.invoke({messages: [new ToolMessage(...)]})`。
  **注意**：这会将消息附加到历史记录中，并从 START 节点运行图。

  - 手动更新状态并从中断处恢复图：

          1. 使用 `graph.getState(config)` 获取图状态中的最新消息列表
          2. 修改消息列表以从 AIMessages 中删除未回答的工具调用

或者添加带有与未回答的工具调用匹配的 `toolCallId` 的 ToolMessages 3. 使用修改后的消息列表调用 `graph.updateState(config, {messages: ...})` 4. 恢复图，例如调用 `graph.invoke(null, config)`
:::
