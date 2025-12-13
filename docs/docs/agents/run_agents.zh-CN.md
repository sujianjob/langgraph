---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 运行代理 (Running agents)

代理支持同步和异步执行，既可以使用 `.invoke()` / `await .ainvoke()` 获得完整响应，也可以使用 `.stream()` / `.astream()` 获得 [流式](../how-tos/streaming.md) 的 **增量** 输出。本节解释了如何提供输入、解释输出、启用流式传输以及控制执行限制。

## 基本用法

代理可以在两种主要模式下执行：

:::python

- **同步 (Synchronous)** 使用 `.invoke()` 或 `.stream()`
- **异步 (Asynchronous)** 使用 `await .ainvoke()` 或带有 `.astream()` 的 `async for`
  :::

:::js

- **同步 (Synchronous)** 使用 `.invoke()` 或 `.stream()`
- **异步 (Asynchronous)** 使用 `await .invoke()` 或带有 `.stream()` 的 `for await`
  :::

:::python
=== "同步调用"

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)

    # highlight-next-line
    response = agent.invoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

=== "异步调用"

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)
    # highlight-next-line
    response = await agent.ainvoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

:::

:::js

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const agent = createReactAgent(...);
// highlight-next-line
const response = await agent.invoke({
    "messages": [
        { "role": "user", "content": "what is the weather in sf" }
    ]
});
```

:::

## 输入和输出

代理使用的语言模型期望一个 `messages` 列表作为输入。因此，代理的输入和输出作为一个 `messages` 列表存储在代理 [状态 (state)](../concepts/low_level.md#working-with-messages-in-graph-state) 的 `messages` 键下。

## 输入格式

代理输入必须是具有 `messages` 键的字典。支持的格式有：

:::python
| 格式 | 示例 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串 (String) | `{"messages": "Hello"}` — 被解释为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典 | `{"messages": {"role": "user", "content": "Hello"}}` |
| 消息列表 | `{"messages": [{"role": "user", "content": "Hello"}]}` |
| 带有自定义状态 | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用自定义 `state_schema` |
:::

:::js
| 格式 | 示例 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串 (String) | `{"messages": "Hello"}` — 被解释为 [HumanMessage](https://js.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典 | `{"messages": {"role": "user", "content": "Hello"}}` |
| 消息列表 | `{"messages": [{"role": "user", "content": "Hello"}]}` |
| 带有自定义状态 | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用自定义状态定义 |
:::

:::python
消息会自动转换为 LangChain 的内部消息格式。您可以阅读 LangChain 文档中有关 [LangChain 消息](https://python.langchain.com/docs/concepts/messages/#langchain-messages) 的更多信息。
:::

:::js
消息会自动转换为 LangChain 的内部消息格式。您可以阅读 LangChain 文档中有关 [LangChain 消息](https://js.langchain.com/docs/concepts/messages/#langchain-messages) 的更多信息。
:::

!!! tip "使用自定义代理状态"

    :::python
    您可以直接在输入字典中提供代理状态模式中定义的其他字段。这允许基于运行时数据或先前工具输出的动态行为。
    有关完整详细信息，请参阅[上下文指南](./context.md)。
    :::

    :::js
    您可以直接在状态定义中提供代理状态中定义的其他字段。这允许基于运行时数据或先前工具输出的动态行为。
    有关完整详细信息，请参阅[上下文指南](./context.md)。
    :::

!!! note

    :::python
    `messages` 的字符串输入将转换为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage)。此行为不同于 `create_react_agent` 中的 `prompt` 参数，后者作为字符串传递时被解释为 [SystemMessage](https://python.langchain.com/docs/concepts/messages/#systemmessage)。
    :::

    :::js
    `messages` 的字符串输入将转换为 [HumanMessage](https://js.langchain.com/docs/concepts/messages/#humanmessage)。此行为不同于 `createReactAgent` 中的 `prompt` 参数，后者作为字符串传递时被解释为 [SystemMessage](https://js.langchain.com/docs/concepts/messages/#systemmessage)。
    :::

## 输出格式

:::python
代理输出是一个包含以下内容的字典：

- `messages`: 执行期间交换的所有消息的列表（用户输入、助手回复、工具调用）。
- 可选的 `structured_response`，如果配置了 [结构化输出](./agents.md#6-configure-structured-output)。
- 如果使用自定义 `state_schema`，输出中也可能包含与您定义的字段相对应的其他键。这些可以保存来自工具执行或提示逻辑的更新状态值。
:::

:::js
代理输出是一个包含以下内容的字典：

- `messages`: 执行期间交换的所有消息的列表（用户输入、助手回复、工具调用）。
- 可选的 `structuredResponse`，如果配置了 [结构化输出](./agents.md#6-configure-structured-output)。
- 如果使用自定义状态定义，输出中也可能包含与您定义的字段相对应的其他键。这些可以保存来自工具执行或提示逻辑的更新状态值。
:::

有关使用自定义状态模式和访问上下文的更多详细信息，请参阅[上下文指南](./context.md)。

## 流式输出 (Streaming output)

代理支持流式响应以实现响应更快的应用程序。这包括：

- 每一步之后的 **进度更新**
- 生成时的 **LLM token**
- 执行期间的 **自定义工具消息**

流式传输在同步和异步模式下均可用：

:::python
=== "同步流式"

    ```python
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

=== "异步流式"

    ```python
    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

:::

:::js

```typescript
for await (const chunk of agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: "updates" }
)) {
  console.log(chunk);
}
```

:::

!!! tip

    有关完整详细信息，请参阅[流式传输指南](../how-tos/streaming.md)。

## 最大迭代次数 (Max iterations)

:::python
为了控制代理执行并避免无限循环，请设置递归限制。这定义了代理在引发 `GraphRecursionError` 之前可以采取的最大步骤数。您可以在运行时或在通过 `.with_config()` 定义代理时配置 `recursion_limit`：
:::

:::js
为了控制代理执行并避免无限循环，请设置递归限制。这定义了代理在引发 `GraphRecursionError` 之前可以采取的最大步骤数。您可以在运行时或在通过 `.withConfig()` 定义代理时配置 `recursionLimit`：
:::

:::python
=== "运行时 (Runtime)"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )

    try:
        response = agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            # highlight-next-line
            {"recursion_limit": recursion_limit},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

=== "`.with_config()`"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )
    # highlight-next-line
    agent_with_recursion_limit = agent.with_config(recursion_limit=recursion_limit)

    try:
        response = agent_with_recursion_limit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

:::

:::js
=== "运行时 (Runtime)"

    ```typescript
    import { GraphRecursionError } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/langgraph/prebuilt";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    const maxIterations = 3;
    // highlight-next-line
    const recursionLimit = 2 * maxIterations + 1;
    const agent = createReactAgent({
        llm: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
        tools: [getWeather]
    });

    try {
        const response = await agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            // highlight-next-line
            { recursionLimit }
        );
    } catch (error) {
        if (error instanceof GraphRecursionError) {
            console.log("Agent stopped due to max iterations.");
        }
    }
    ```

=== "`.withConfig()`"

    ```typescript
    import { GraphRecursionError } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/langgraph/prebuilt";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    const maxIterations = 3;
    // highlight-next-line
    const recursionLimit = 2 * maxIterations + 1;
    const agent = createReactAgent({
        llm: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
        tools: [getWeather]
    });
    // highlight-next-line
    const agentWithRecursionLimit = agent.withConfig({ recursionLimit });

    try {
        const response = await agentWithRecursionLimit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        );
    } catch (error) {
        if (error instanceof GraphRecursionError) {
            console.log("Agent stopped due to max iterations.");
        }
    }
    ```

:::

:::python

## 其他资源

- [LangChain 中的异步编程](https://python.langchain.com/docs/concepts/async)
  :::
