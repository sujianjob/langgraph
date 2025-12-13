# 如何流式传输 (How to stream)

流式传输对于使基于 LLM 的应用程序对最终用户感到即时响应至关重要。

LangGraph 提供了几个流式传输模式，针对不同的用例进行了优化：

- [`values`](#stream-values): 流式传输状态的完整值，这允许您在图中的每一步后获取最新的状态。
- [`updates`](#stream-updates): 流式传输状态的更新，这允许您在图中的每一步后获取该步骤对状态所做的更改。
- [`custom`](#stream-custom-data): 流式传输自定义数据（例如消息块），这允许您流式传输 LLM 令牌、进度更新/状态或来自节点内部的任何其他数据。
- [`messages`](#stream-messages): 流式传输 LLM 消息块（和元数据），这允许您轻松地从图内的任何位置访问 LLM 令牌。
- [`debug`](#stream-debug-events): 流式传输调试事件，这允许您查看图中发生的且在其他模式下不可见的详细事件。

| 模式 | 用例 |
| :--- | :--- |
| `values` | 用于在图中的每一步后流式传输 **完整状态** |
| `updates` | 用于在图中的每一步后流式传输 **状态更新** |
| `custom` | 用于从节点内部流式传输 **自定义数据** (例如, 令牌) |
| `messages` | 用于从图内流式传输 **LLM 令牌** 和 **元数据** |
| `debug` | 用于流式传输 **调试事件** |

您可以将这些模式中的任何一个传递给 [`.stream()`](../reference/graphs.md#langgraph.graph.graph.CompiledGraph.stream)（或 [`.astream`](../reference/graphs.md#langgraph.graph.graph.CompiledGraph.astream)）方法。

!!! tip "Multiple streaming modes"
    您可以同时使用多种流式传输模式！只需将模式列表传递给 `stream_mode`，例如 `stream_mode=["values", "messages"]`。

## 设置 (Setup)

首先，让我们安装所需的包并设置我们的 API 密钥

```python
%%capture --no-stderr
%pip install -U langgraph langchain-openai langchain-community
```

```python
import getpass
import os

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

<details>
<summary>设置 TypeScript 环境</summary>

```typescript
import { z } from "zod";

// Define the state schema
const State = z.object({
  messages: z.array(z.any()),
  topic: z.string(),
});
```

</details>

## 定义图 (Define the graph)

我们将使用一个简单的 ReAct 代理作为本指南的示例。

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Literal, TypedDict
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import create_react_agent

@tool
def get_weather(city: Literal["nyc", "sf"]):
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")

tools = [get_weather]
model = ChatOpenAI(model_name="gpt-4o-mini", temperature=0)

graph = create_react_agent(model, tools)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const getWeather = tool((input) => {
  if (input.city === "nyc") {
    return "It might be cloudy in nyc";
  } else if (input.city === "sf") {
    return "It's always sunny in sf";
  } else {
    throw new Error("Unknown city");
  }
}, {
  name: "get_weather",
  description: "Use this to get weather information.",
  schema: z.object({
    city: z.enum(["nyc", "sf"]),
  }),
});

const tools = [getWeather];
const model = new ChatOpenAI({ model: "gpt-4o-mini", temperature: 0 });

const graph = createReactAgent({ llm: model, tools });
```
</details>

## 支持的流模式 (Supported stream modes)

### Stream values

`values` 模式流式传输图在每一步后的完整状态。如果您的图有多个路径（例如，扇出），它将为每个路径流式传输一个值。

这是默认的流式传输模式。

<details>
<summary>查看 Python 代码</summary>

```python
inputs = {"messages": [("human", "what's the weather in sf")]}

for chunk in graph.stream(inputs, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const inputs = { messages: [{ role: "user", content: "what's the weather in sf" }] };

for await (const chunk of await graph.stream(inputs, { streamMode: "values" })) {
  console.log(chunk.messages[chunk.messages.length - 1]);
}
```
</details>

```
================================ Human Message =================================

what's the weather in sf
================================== Ai Message ==================================
Tool Calls:
  get_weather (call_Abc123)
 Call ID: call_Abc123
  Args:
    city: sf
================================= Tool Message =================================
Name: get_weather

It's always sunny in sf
================================== Ai Message ==================================

The weather in San Francisco is currently sunny.
```

### Stream updates

`updates` 模式流式传输图在每一步后的状态更新。如果您的图有多个路径，它将为每个路径流式传输一个更新。

<details>
<summary>查看 Python 代码</summary>

```python
inputs = {"messages": [("human", "what's the weather in sf")]}

for chunk in graph.stream(inputs, stream_mode="updates"):
    for node, values in chunk.items():
        print(f"Receiving update from node: '{node}'")
        print(values)
        print("\n\n")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const inputs = { messages: [{ role: "user", content: "what's the weather in sf" }] };

for await (const chunk of await graph.stream(inputs, { streamMode: "updates" })) {
  for (const [node, values] of Object.entries(chunk)) {
    console.log(`Receiving update from node: '${node}'`);
    console.log(values);
    console.log("\n\n");
  }
}
```
</details>

```
Receiving update from node: 'agent'
{'messages': [AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_Def456', 'function': {'arguments': '{"city":"sf"}', 'name': 'get_weather'}, 'type': 'function'}]}, response_metadata={'token_usage': {'completion_tokens': 15, 'prompt_tokens': 57, 'total_tokens': 72}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_3bc1b5746c', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-789', tool_calls=[{'name': 'get_weather', 'args': {'city': 'sf'}, 'id': 'call_Def456'}])]}



Receiving update from node: 'tools'
{'messages': [ToolMessage(content="It's always sunny in sf", name='get_weather', tool_call_id='call_Def456')]}



Receiving update from node: 'agent'
{'messages': [AIMessage(content='The weather in San Francisco is currently sunny.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 9, 'prompt_tokens': 84, 'total_tokens': 93}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_3bc1b5746c', 'finish_reason': 'stop', 'logprobs': None}, id='run-101')]}
```

### Stream debug events

`debug` 模式流式传输图中发生的所有详细事件。这对于调试非常有用，因为它包含有关图执行的详细信息，例如节点何时开始、结束、LLM 何时被调用等。

<details>
<summary>查看 Python 代码</summary>

```python
inputs = {"messages": [("human", "what's the weather in sf")]}

for chunk in graph.stream(inputs, stream_mode="debug"):
    print(chunk)
    print("\n\n")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const inputs = { messages: [{ role: "user", content: "what's the weather in sf" }] };

for await (const chunk of await graph.stream(inputs, { streamMode: "debug" })) {
  console.log(chunk);
  console.log("\n\n");
}
```
</details>

### Stream messages

`messages` 模式允许您流式传输来自图内部调用的任何 LangChain 聊天模型的 LLM 令牌。

流式输出是 `(message_chunk, metadata)` 元组，其中：

- `message_chunk`: 来自 LLM 的令牌或消息段。
- `metadata`: 包含有关图节点和 LLM 调用的详细信息的字典。

> 如果您的 LLM 不可用作 LangChain 集成，您可以使用 `custom` 模式代为流式传输其输出。有关详细信息，请参见 [use with any LLM](#use-with-any-llm)。

!!! warning "Manual config required for async in Python < 3.11"

    当在 Python < 3.11 中使用异步代码时，您必须显式地将 `RunnableConfig` 传递给 `ainvoke()` 以启用正确的流式传输。有关详细信息，请参阅 [Async with Python < 3.11](#async) 或升级到 Python 3.11+。

<details>
<summary>查看 Python 代码</summary>

```python
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START


@dataclass
class MyState:
    topic: str
    joke: str = ""


llm = init_chat_model(model="openai:gpt-4o-mini")

def call_model(state: MyState):
    """Call the LLM to generate a joke about a topic"""
    # highlight-next-line
    llm_response = llm.invoke( # (1)!
        [
            {"role": "user", "content": f"Generate a joke about {state.topic}"}
        ]
    )
    return {"joke": llm_response.content}

graph = (
    StateGraph(MyState)
    .add_node(call_model)
    .add_edge(START, "call_model")
    .compile()
)

for message_chunk, metadata in graph.stream( # (2)!
    {"topic": "ice cream"},
    # highlight-next-line
    stream_mode="messages",
):
    if message_chunk.content:
        print(message_chunk.content, end="|", flush=True)
```

1. 请注意，即使使用 `.invoke` 而不是 `.stream` 运行 LLM，也会发出消息事件。
2. "messages" 流模式返回 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的令牌，`metadata` 是一个包含有关调用 LLM 的图节点的信息和其他信息的字典。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

const MyState = z.object({
  topic: z.string(),
  joke: z.string().default(""),
});

const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

const callModel = async (state: z.infer<typeof MyState>) => {
  // Call the LLM to generate a joke about a topic
  const llmResponse = await llm.invoke([
    { role: "user", content: `Generate a joke about ${state.topic}` },
  ]); // (1)!
  return { joke: llmResponse.content };
};

const graph = new StateGraph(MyState)
  .addNode("callModel", callModel)
  .addEdge(START, "callModel")
  .compile();

for await (const [messageChunk, metadata] of await graph.stream(
  // (2)!
  { topic: "ice cream" },
  { streamMode: "messages" }
)) {
  if (messageChunk.content) {
    console.log(messageChunk.content + "|");
  }
}
```

1. 请注意，即使使用 `.invoke` 而不是 `.stream` 运行 LLM，也会发出消息事件。
2. "messages" 流模式返回 `[messageChunk, metadata]` 元组的迭代器，其中 `messageChunk` 是 LLM 流式传输的令牌，`metadata` 是一个包含有关调用 LLM 的图节点的信息和其他信息的字典。
</details>

#### 按 LLM 调用筛选 (Filter by LLM invocation)

您可以将 `tags` 与 LLM 调用相关联，以按 LLM 调用筛选流式传输的令牌。

<details>
<summary>查看 Python 代码</summary>

```python
from langchain.chat_models import init_chat_model

llm_1 = init_chat_model(model="openai:gpt-4o-mini", tags=['joke']) # (1)!
llm_2 = init_chat_model(model="openai:gpt-4o-mini", tags=['poem']) # (2)!

graph = ... # define a graph that uses these LLMs

async for msg, metadata in graph.astream(  # (3)!
    {"topic": "cats"},
    # highlight-next-line
    stream_mode="messages",
):
    if metadata["tags"] == ["joke"]: # (4)!
        print(msg.content, end="|", flush=True)
```

1. llm_1 标记为 "joke"。
2. llm_2 标记为 "poem"。
3. `stream_mode` 设置为 "messages" 以流式传输 LLM 令牌。`metadata` 包含有关 LLM 调用的信息，包括标签。
4. 按元数据中的 `tags` 字段筛选流式传输的令牌，以仅包括具有 "joke" 标签的 LLM 调用的令牌。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { ChatOpenAI } from "@langchain/openai";

const llm1 = new ChatOpenAI({
  model: "gpt-4o-mini",
  tags: ['joke'] // (1)!
});
const llm2 = new ChatOpenAI({
  model: "gpt-4o-mini",
  tags: ['poem'] // (2)!
});

const graph = // ... define a graph that uses these LLMs

for await (const [msg, metadata] of await graph.stream( // (3)!
  { topic: "cats" },
  { streamMode: "messages" }
)) {
  if (metadata.tags?.includes("joke")) { // (4)!
    console.log(msg.content + "|");
  }
}
```

1. llm1 标记为 "joke"。
2. llm2 标记为 "poem"。
3. `streamMode` 设置为 "messages" 以流式传输 LLM 令牌。`metadata` 包含有关 LLM 调用的信息，包括标签。
4. 按元数据中的 `tags` 字段筛选流式传输的令牌，以仅包括具有 "joke" 标签的 LLM 调用的令牌。
</details>

<details>
<summary>扩展示例：按标签筛选</summary>

Python 和 TypeScript 的完整示例，展示了如何在具有不同标签的多个 LLM 调用中进行筛选。
</details>

#### 按节点筛选 (Filter by node)

要仅从特定节点流式传输令牌，请使用 `stream_mode="messages"` 并按流式传输元数据中的 `langgraph_node` 字段筛选输出：

<details>
<summary>查看 Python 代码</summary>

```python
for msg, metadata in graph.stream( # (1)!
    inputs,
    # highlight-next-line
    stream_mode="messages",
):
    # highlight-next-line
    if msg.content and metadata["langgraph_node"] == "some_node_name": # (2)!
        ...
```

1. "messages" 流模式返回 `(message_chunk, metadata)` 元组。
2. 按元数据中的 `langgraph_node` 字段筛选流式传输的令牌。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
for await (const [msg, metadata] of await graph.stream(
  // (1)!
  inputs,
  { streamMode: "messages" }
)) {
  if (msg.content && metadata.langgraph_node === "some_node_name") {
    // (2)!
    // ...
  }
}
```

1. "messages" 流模式返回 `[messageChunk, metadata]` 元组。
2. 按元数据中的 `langgraph_node` 字段筛选流式传输的令牌。
</details>

<details>
<summary>扩展示例：从特定节点流式传输 LLM 令牌</summary>

Python 和 TypeScript 的完整示例，展示了如何按节点筛选流式传输的令牌。
</details>

### Stream custom data

:::python
要从 LangGraph 节点或工具内部发送 **自定义用户定义数据**，请按照下列步骤操作：

1. 使用 `get_stream_writer()` 访问流写入器并发出自定义数据。
2. 调用 `.stream()` 或 `.astream()` 时设置 `stream_mode="custom"` 以在流中获取自定义数据。您可以组合多种模式（例如，`["updates", "custom"]`），但至少必须有一个是 `"custom"`。

!!! warning "No `get_stream_writer()` in async for Python < 3.11"

    在 Python < 3.11 上运行的异步代码中，`get_stream_writer()` 将不起作用。
    相反，向您的节点或工具添加 `writer` 参数并手动传递它。
    有关使用示例，请参见 [Async with Python < 3.11](#async)。
:::

:::js
要从 LangGraph 节点或工具内部发送 **自定义用户定义数据**，请按照下列步骤操作：

1. 使用 `LangGraphRunnableConfig` 中的 `writer` 参数发出自定义数据。
2. 调用 `.stream()` 时设置 `streamMode: "custom"` 以在流中获取自定义数据。您可以组合多种模式（例如，`["updates", "custom"]`），但至少必须有一个是 `"custom"`。
:::

#### 节点 (Node)

<details>
<summary>查看 Python 代码</summary>

```python
from typing import TypedDict
from langgraph.config import get_stream_writer
from langgraph.graph import StateGraph, START

class State(TypedDict):
    query: str
    answer: str

def node(state: State):
    writer = get_stream_writer()  # (1)!
    writer({"custom_key": "Generating custom data inside node"}) # (2)!
    return {"answer": "some data"}

graph = (
    StateGraph(State)
    .add_node(node)
    .add_edge(START, "node")
    .compile()
)

inputs = {"query": "example"}

# Usage
for chunk in graph.stream(inputs, stream_mode="custom"):  # (3)!
    print(chunk)
```

1. 获取流写入器以发送自定义数据。
2. 发出自定义键值对（例如，进度更新）。
3. 设置 `stream_mode="custom"` 以在流中接收自定义数据。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START, LangGraphRunnableConfig } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  query: z.string(),
  answer: z.string(),
});

const graph = new StateGraph(State)
  .addNode("node", async (state, config) => {
    config.writer({ custom_key: "Generating custom data inside node" }); // (1)!
    return { answer: "some data" };
  })
  .addEdge(START, "node")
  .compile();

const inputs = { query: "example" };

// Usage
for await (const chunk of await graph.stream(inputs, { streamMode: "custom" })) { // (2)!
  console.log(chunk);
}
```

1. 使用写入器发出自定义键值对。
2. 设置 `streamMode: "custom"` 以在流中接收自定义数据。
</details>

#### 工具 (Tool)

<details>
<summary>查看 Python 代码</summary>

```python
from langchain_core.tools import tool
from langgraph.config import get_stream_writer

@tool
def query_database(query: str) -> str:
    """Query the database."""
    writer = get_stream_writer() # (1)!
    # highlight-next-line
    writer({"data": "Retrieved 0/100 records", "type": "progress"}) # (2)!
    # perform query
    # highlight-next-line
    writer({"data": "Retrieved 100/100 records", "type": "progress"}) # (3)!
    return "some-answer"


graph = ... # define a graph that uses this tool

for chunk in graph.stream(inputs, stream_mode="custom"): # (4)!
    print(chunk)
```

1. 访问流写入器以发送自定义数据。
2. 发出自定义键值对。
3. 发出另一个自定义键值对。
4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import { LangGraphRunnableConfig } from "@langchain/langgraph";
import { z } from "zod";

const queryDatabase = tool(
  async (input, config: LangGraphRunnableConfig) => {
    config.writer({ data: "Retrieved 0/100 records", type: "progress" }); // (1)!
    // perform query
    config.writer({ data: "Retrieved 100/100 records", type: "progress" }); // (2)!
    return "some-answer";
  },
  {
    name: "query_database",
    description: "Query the database.",
    schema: z.object({
      query: z.string().describe("The query to execute."),
    }),
  }
);

const graph = // ... define a graph that uses this tool

for await (const chunk of await graph.stream(inputs, { streamMode: "custom" })) { // (3)!
  console.log(chunk);
}
```

1. 使用写入器发出自定义键值对。
2. 发出另一个自定义键值对。
3. 设置 `streamMode: "custom"` 以在流中接收自定义数据。
</details>

### Use with any LLM

您可以使用 `stream_mode="custom"` 来流式传输来自 **任何 LLM API** 的数据 —— 即使该 API **没有** 实现 LangChain 聊天模型接口。

这使您可以集成原始 LLM 客户端或提供自己的流式传输接口的外部服务，使 LangGraph 对于自定义设置非常灵活。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.config import get_stream_writer

def call_arbitrary_model(state):
    """Example node that calls an arbitrary model and streams the output"""
    # highlight-next-line
    writer = get_stream_writer() # (1)!
    # Assume you have a streaming client that yields chunks
    for chunk in your_custom_streaming_client(state["topic"]): # (2)!
        # highlight-next-line
        writer({"custom_llm_chunk": chunk}) # (3)!
    return {"result": "completed"}

graph = (
    StateGraph(State)
    .add_node(call_arbitrary_model)
    # Add other nodes and edges as needed
    .compile()
)

for chunk in graph.stream(
    {"topic": "cats"},
    # highlight-next-line
    stream_mode="custom", # (4)!
):
    # The chunk will contain the custom data streamed from the llm
    print(chunk)
```

1. 获取流写入器以发送自定义数据。
2. 使用您的自定义流式传输客户端生成 LLM 令牌。
3. 使用写入器将自定义数据发送到流。
4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { LangGraphRunnableConfig } from "@langchain/langgraph";

const callArbitraryModel = async (
  state: any,
  config: LangGraphRunnableConfig
) => {
  // Example node that calls an arbitrary model and streams the output
  // Assume you have a streaming client that yields chunks
  for await (const chunk of yourCustomStreamingClient(state.topic)) {
    // (1)!
    config.writer({ custom_llm_chunk: chunk }); // (2)!
  }
  return { result: "completed" };
};

const graph = new StateGraph(State)
  .addNode("callArbitraryModel", callArbitraryModel)
  // Add other nodes and edges as needed
  .compile();

for await (const chunk of await graph.stream(
  { topic: "cats" },
  { streamMode: "custom" } // (3)!
)) {
  // The chunk will contain the custom data streamed from the llm
  console.log(chunk);
}
```

1. 使用您的自定义流式传输客户端生成 LLM 令牌。
2. 使用写入器将自定义数据发送到流。
3. 设置 `streamMode: "custom"` 以在流中接收自定义数据。
</details>

<details>
<summary>扩展示例：流式传输任意聊天模型</summary>

Python 和 TypeScript 的完整示例，展示了如何在不使用 LangChain 聊天模型接口的情况下流式传输 LLM 令牌。
</details>

### Disable streaming for specific chat models

如果您的应用程序混合了支持流式传输的模型和不支持流式传输的模型，您可能需要为不支持它的模型显式禁用流式传输。

<details>
<summary>查看 Python 代码</summary>

在初始化模型时设置 `disable_streaming=True`。

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    # highlight-next-line
    disable_streaming=True # (1)!
)
```

1. 设置 `disable_streaming=True` 以禁用聊天模型的流式传输。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

在初始化模型时设置 `streaming: false`。

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  model: "o1-preview",
  streaming: false, // (1)!
});
```

1. 设置 `streaming: false` 以禁用聊天模型的流式传输。
</details>

### Async with Python < 3.11 { #async }

在 Python 版本 < 3.11 中，[asyncio tasks](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) 不支持 `context` 参数。
这限制了 LangGraph 自动传播上下文的能力，并以两个主要方式影响 LangGraph 的流式传输机制：

1. 您 **必须** 将 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 显式传递给异步 LLM 调用（例如，`ainvoke()`），因为回调不会自动传播。
2. 您 **不能** 在异步节点或工具中使用 `get_stream_writer()` —— 您必须直接传递 `writer` 参数。
