---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# LangGraph 快速入门 (LangGraph quickstart)

本指南向您展示如何设置和使用 LangGraph 的 **预构建**、**可重用** 组件，这些组件旨在帮助您快速可靠地构建代理系统。

## 先决条件 (Prerequisites)

在开始本教程之前，请确保您拥有以下内容：

- 一个 [Anthropic](https://console.anthropic.com/settings/keys) API 密钥

## 1. 安装依赖项 (Install dependencies)

如果您还没有安装，请安装 LangGraph 和 LangChain：

:::python

```
pip install -U langgraph "langchain[anthropic]"
```

!!! info

    安装 `langchain[anthropic]` 是为了让代理可以调用 [模型](https://python.langchain.com/docs/integrations/chat/)。

:::

:::js

```bash
npm install @langchain/langgraph @langchain/core @langchain/anthropic
```

!!! info

    安装 `@langchain/core` 和 `@langchain/anthropic` 是为了让代理可以调用 [模型](https://js.langchain.com/docs/integrations/chat/)。

:::

## 2. 创建代理 (Create an agent)

:::python
要创建代理，请使用 @[`create_react_agent`][create_react_agent]：

```python
from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:  # (1)!
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",  # (2)!
    tools=[get_weather],  # (3)!
    prompt="You are a helpful assistant"  # (4)!
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

1. 定义代理要使用的工具。工具可以定义为普通的 Python 函数。有关更高级的工具用法和自定义，请查看 [工具 (tools)](../how-tos/tool-calling.md) 页面。
2. 为代理提供一个语言模型。要了解有关为代理配置语言模型的更多信息，请查看 [模型 (models)](./models.md) 页面。
3. 提供供模型使用的工具列表。
4. 为代理使用的语言模型提供系统提示（指令）。
   :::

:::js
要创建代理，请使用 [`createReactAgent`](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.createReactAgent.html)：

```typescript
import { ChatAnthropic } from "@langchain/anthropic";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(
  // (1)!
  async ({ city }) => {
    return `It's always sunny in ${city}!`;
  },
  {
    name: "get_weather",
    description: "Get weather for a given city.",
    schema: z.object({
      city: z.string().describe("The city to get weather for"),
    }),
  }
);

const agent = createReactAgent({
  llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }), // (2)!
  tools: [getWeather], // (3)!
  stateModifier: "You are a helpful assistant", // (4)!
});

// Run the agent
await agent.invoke({
  messages: [{ role: "user", content: "what is the weather in sf" }],
});
```

1. 定义代理要使用的工具。工具可以使用 `tool` 函数定义。有关更高级的工具用法和自定义，请查看 [工具 (tools)](./tools.md) 页面。
2. 为代理提供一个语言模型。要了解有关为代理配置语言模型的更多信息，请查看 [模型 (models)](./models.md) 页面。
3. 提供供模型使用的工具列表。
4. 为代理使用的语言模型提供系统提示（指令）。
   :::

## 3. 配置 LLM (Configure an LLM)

:::python
要使用特定参数（如温度）配置 LLM，请使用 [init_chat_model](https://python.langchain.com/api_reference/langchain/chat_models/langchain.chat_models.base.init_chat_model.html)：

```python
from langchain.chat_models import init_chat_model
from langgraph.prebuilt import create_react_agent

# highlight-next-line
model = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    # highlight-next-line
    temperature=0
)

agent = create_react_agent(
    # highlight-next-line
    model=model,
    tools=[get_weather],
)
```

:::

:::js
要使用特定参数（如温度）配置 LLM，请使用模型实例：

```typescript
import { ChatAnthropic } from "@langchain/anthropic";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

// highlight-next-line
const model = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
  // highlight-next-line
  temperature: 0,
});

const agent = createReactAgent({
  // highlight-next-line
  llm: model,
  tools: [getWeather],
});
```

:::

有关如何配置 LLM 的更多信息，请参阅 [模型 (Models)](./models.md)。

## 4. 添加自定义提示 (Add a custom prompt)

提示指导 LLM 如何表现。添加以下类型的提示之一：

- **静态 (Static)**：字符串被解释为 **系统消息**。
- **动态 (Dynamic)**：在 **运行时** 根据输入或配置生成的消息列表。

=== "Static prompt"

    定义固定的提示字符串或消息列表：

    :::python
    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # A static prompt that never changes
        # highlight-next-line
        prompt="Never answer questions about the weather."
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
    )
    ```
    :::

    :::js
    ```typescript
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatAnthropic } from "@langchain/anthropic";

    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }),
      tools: [getWeather],
      // A static prompt that never changes
      // highlight-next-line
      stateModifier: "Never answer questions about the weather."
    });

    await agent.invoke({
      messages: [{ role: "user", content: "what is the weather in sf" }]
    });
    ```
    :::

=== "Dynamic prompt"

    :::python
    定义一个根据代理的状态和配置返回消息列表的函数：

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:  # (1)!
        user_name = config["configurable"].get("user_name")
        system_msg = f"You are a helpful assistant. Address the user as {user_name}."
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # highlight-next-line
        prompt=prompt
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        config={"configurable": {"user_name": "John Smith"}}
    )
    ```

    1. 动态提示允许在构建 LLM 输入时包含非消息 [上下文 (context)](./context.md)，例如：

        - 在运行时传递的信息，如 `user_id` 或 API 凭据（使用 `config`）。
        - 在多步推理过程中更新的内部代理状态（使用 `state`）。

        动态提示可以定义为接受 `state` 和 `config` 并返回要发送给 LLM 的消息列表的函数。
    :::

    :::js
    定义一个根据代理的状态和配置返回消息的函数：

    ```typescript
    import { type BaseMessageLike } from "@langchain/core/messages";
    import { type RunnableConfig } from "@langchain/core/runnables";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    // highlight-next-line
    const dynamicPrompt = (state: { messages: BaseMessageLike[] }, config: RunnableConfig): BaseMessageLike[] => {  // (1)!
      const userName = config.configurable?.user_name;
      const systemMsg = `You are a helpful assistant. Address the user as ${userName}.`;
      return [{ role: "system", content: systemMsg }, ...state.messages];
    };

    const agent = createReactAgent({
      llm: "anthropic:claude-3-5-sonnet-latest",
      tools: [getWeather],
      // highlight-next-line
      stateModifier: dynamicPrompt
    });

    await agent.invoke(
      { messages: [{ role: "user", content: "what is the weather in sf" }] },
      // highlight-next-line
      { configurable: { user_name: "John Smith" } }
    );
    ```

    1. 动态提示允许在构建 LLM 输入时包含非消息 [上下文 (context)](./context.md)，例如：

        - 在运行时传递的信息，如 `user_id` 或 API 凭据（使用 `config`）。
        - 在多步推理过程中更新的内部代理状态（使用 `state`）。

        动态提示可以定义为接受 `state` 和 `config` 并返回要发送给 LLM 的消息列表的函数。
    :::

有关更多信息，请参阅 [上下文 (Context)](./context.md)。

## 5. 添加记忆 (Add memory)

要允许与代理进行多轮对话，您需要通过在创建代理时提供检查点器 (checkpointer) 来启用 [持久性 (persistence)](../concepts/persistence.md)。在运行时，您需要提供包含 `thread_id`（对话/会话的唯一标识符）的配置：

:::python

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

# highlight-next-line
checkpointer = InMemorySaver()

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    checkpointer=checkpointer  # (1)!
)

# Run the agent
# highlight-next-line
config = {"configurable": {"thread_id": "1"}}
sf_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
    # highlight-next-line
    config  # (2)!
)
ny_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what about new york?"}]},
    # highlight-next-line
    config
)
```

1. `checkpointer` 允许代理在工具调用循环的每一步存储其状态。这启用了 [短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory) 和 [人机交互 (human-in-the-loop)](../concepts/human_in_the_loop.md) 功能。
2. 传递带有 `thread_id` 的配置，以便能够在未来的代理调用中恢复相同的对话。
   :::

:::js

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { MemorySaver } from "@langchain/langgraph";

# highlight-next-line
const checkpointer = new MemorySaver();

const agent = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  tools: [getWeather],
  // highlight-next-line
  checkpointSaver: checkpointer, // (1)!
});

// Run the agent
// highlight-next-line
const config = { configurable: { thread_id: "1" } };
const sfResponse = await agent.invoke(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  // highlight-next-line
  config // (2)!
);
const nyResponse = await agent.invoke(
  { messages: [{ role: "user", content: "what about new york?" }] },
  // highlight-next-line
  config
);
```

1. `checkpointSaver` 允许代理在工具调用循环的每一步存储其状态。这启用了 [短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory) 和 [人机交互 (human-in-the-loop)](../concepts/human_in_the_loop.md) 功能。
2. 传递带有 `thread_id` 的配置，以便能够在未来的代理调用中恢复相同的对话。
   :::

:::python
当您启用检查点器时，它会将代理状态存储在提供的检查点器数据库（如果使用 `InMemorySaver`，则存储在内存）中的每一步。
:::

:::js
当您启用检查点器时，它会将代理状态存储在提供的检查点器数据库（如果使用 `MemorySaver`，则存储在内存）中的每一步。
:::

请注意，在上面的示例中，当第二次使用相同的 `thread_id` 调用代理时，第一次对话的原始消息历史记录将自动包含在内，同时包含新的用户输入。

有关更多信息，请参阅 [记忆 (Memory)](../how-tos/memory/add-memory.md)。

## 6. 配置结构化输出 (Configure structured output)

:::python
要生成符合架构的结构化响应，请使用 `response_format` 参数。架构可以使用 `Pydantic` 模型或 `TypedDict` 定义。结果可以通过 `structured_response` 字段访问。

```python
from pydantic import BaseModel
from langgraph.prebuilt import create_react_agent

class WeatherResponse(BaseModel):
    conditions: str

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    response_format=WeatherResponse  # (1)!
)

response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)

# highlight-next-line
response["structured_response"]
```

1.  当提供 `response_format` 时，将在代理循环的末尾添加一个单独的步骤：代理消息历史记录被传递给具有结构化输出的 LLM，以生成结构化响应。

        要向此 LLM 提供系统提示，请使用元组 `(prompt, schema)`，例如 `response_format=(prompt, WeatherResponse)`。

    :::

:::js
要生成符合架构的结构化响应，请使用 `responseFormat` 参数。架构可以使用 `Zod` 架构定义。结果可以通过 `structuredResponse` 字段访问。

```typescript
import { z } from "zod";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const WeatherResponse = z.object({
  conditions: z.string(),
});

const agent = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  tools: [getWeather],
  // highlight-next-line
  responseFormat: WeatherResponse, // (1)!
});

const response = await agent.invoke({
  messages: [{ role: "user", content: "what is the weather in sf" }],
});

// highlight-next-line
response.structuredResponse;
```

1.  当提供 `responseFormat` 时，将在代理循环的末尾添加一个单独的步骤：代理消息历史记录被传递给具有结构化输出的 LLM，以生成结构化响应。

        要向此 LLM 提供系统提示，请使用对象 `{ prompt, schema }`，例如 `responseFormat: { prompt, schema: WeatherResponse }`。

    :::

!!! Note "LLM 后处理 (LLM post-processing)"

    结构化输出需要额外调用 LLM 才能根据架构格式化响应。

## 下一步 (Next steps)

- [在本地部署您的代理](../tutorials/langgraph-platform/local-server.md)
- [了解有关预构建代理的更多信息](../agents/overview.md)
- [LangGraph 平台快速入门](../cloud/quick_start.md)
