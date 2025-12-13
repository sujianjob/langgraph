---
search:
  boost: 2
---

# 图 API 概念 (Graph API concepts)

## 图 (Graphs)

LangGraph 的核心是用图来建模代理工作流。您使用三个关键组件来定义代理的行为：

1. [`State` (状态)](#state): 表示应用程序当前快照的共享数据结构。它可以是任何数据类型，但通常使用共享状态模式定义。

2. [`Nodes` (节点)](#nodes): 编码代理逻辑的函数。它们接收当前状态作为输入，执行一些计算或副作用，并返回更新的状态。

3. [`Edges` (边)](#edges): 确定基于当前状态下一个执行哪个 `Node` 的函数。它们可以是条件分支或固定转换。

通过组合 `Nodes` 和 `Edges`，您可以创建复杂的、循环的工作流，并随着时间的推移演变状态。不过，真正的力量来自于 LangGraph 管理该状态的方式。需要强调的是：`Nodes` 和 `Edges` 只不过是函数——它们可以包含 LLM 或仅仅是普通的旧代码。

简而言之：_节点做工作，边告诉下一步做什么_。

LangGraph 的底层图算法使用 [消息传递](https://en.wikipedia.org/wiki/Message_passing) 来定义通用程序。当节点完成其操作时，它会沿着一条或多条边向其他节点发送消息。然后，这些接收节点执行其函数，将生成的消息传递给下一组节点，依此类推。受 Google 的 [Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) 系统启发，程序以离散的“超级步 (super-steps)”进行。

一个超级步可以被视为对图节点的一次迭代。并行运行的节点属于同一个超级步，而顺序运行的节点属于不同的超级步。在图执行开始时，所有节点都处于 `inactive` 状态。当节点在其任何传入边（或“通道”）上接收到新消息（状态）时，它将变为 `active`。然后，活动节点运行其函数并做出响应更新。在每个超级步结束时，没有传入消息的节点通过将自己标记为 `inactive` 来投票 `halt`。当所有节点都 `inactive` 且没有消息在传输中时，图执行终止。

### StateGraph

`StateGraph` 类是要使用的主要图类。这是由用户定义的 `State` 对象参数化的。

### 编译图 (Compiling your graph)

要构建图，您首先定义 [状态 (state)](#state)，然后添加 [节点 (nodes)](#nodes) 和 [边 (edges)](#edges)，然后编译它。究竟什么是编译图，为什么需要它？

编译是一个非常简单的步骤。它对图的结构提供了一些基本检查（没有孤立节点等）。这也是您可以指定运行时参数（如 [检查点 (checkpointers)](./persistence.md) 和断点）的地方。您只需调用 `.compile` 方法即可编译图：

:::python

```python
graph = graph_builder.compile(...)
```

:::

:::js

```typescript
const graph = new StateGraph(StateAnnotation)
  .addNode("nodeA", nodeA)
  .addEdge(START, "nodeA")
  .addEdge("nodeA", END)
  .compile();
```

:::

在您可以使用图之前，您 **必须** 编译它。

## 状态 (State)

:::python
定义图时做的第一件事是定义图的 `State`。`State` 由 [图的模式](#schema) 以及指定如何将更新应用于状态的 [`reducer` 函数](#reducers) 组成。`State` 的模式将是图中所有 `Nodes` 和 `Edges` 的输入模式，可以是 `TypedDict` 或 `Pydantic` 模型。所有 `Nodes` 将发出对 `State` 的更新，然后使用指定的 `reducer` 函数应用这些更新。
:::

:::js
定义图时做的第一件事是定义图的 `State`。`State` 由 [图的模式](#schema) 以及指定如何将更新应用于状态的 [`reducer` 函数](#reducers) 组成。`State` 的模式将是图中所有 `Nodes` 和 `Edges` 的输入模式，可以是 Zod 模式或使用 `Annotation.Root` 构建的模式。所有 `Nodes` 将发出对 `State` 的更新，然后使用指定的 `reducer` 函数应用这些更新。
:::

### 模式 (Schema)

:::python
指定图模式的主要文档化方法是使用 [`TypedDict`](https://docs.python.org/3/library/typing.html#typing.TypedDict)。如果您想在状态中提供默认值，请使用 [`dataclass`](https://docs.python.org/3/library/dataclasses.html)。如果您想要递归数据验证，我们也支持使用 Pydantic [BaseModel](../how-tos/graph-api.md#use-pydantic-models-for-graph-state) 作为您的图状态（尽管请注意，pydantic 的性能不如 `TypedDict` 或 `dataclass`）。

默认情况下，图将具有相同的输入和输出模式。如果您想更改此设置，您还可以直接指定显式的输入和输出模式。当您有很多键，并且有些显式用于输入而其他用于输出时，这很有用。请参阅 [此处指南](../how-tos/graph-api.md#define-input-and-output-schemas) 了解如何使用。
:::

:::js
指定图模式的主要文档化方法是使用 Zod 模式。但是，我们也支持使用 `Annotation` API 来定义图的模式。

默认情况下，图将具有相同的输入和输出模式。如果您想更改此设置，您还可以直接指定显式的输入和输出模式。当您有很多键，并且有些显式用于输入而其他用于输出时，这很有用。
:::

#### 多个模式 (Multiple schemas)

通常，所有图节点都使用单个模式进行通信。这意味着它们将读取和写入相同的状态通道。但是，在某些情况下，我们需要对此进行更多控制：

- 内部节点可以传递图的输入/输出中不需要的信息。
- 我们可能还希望为图使用不同的输入/输出模式。例如，输出可能仅包含单个相关的输出键。

可以让节点写入图内部的私有状态通道以进行内部节点通信。我们可以简单地定义一个私有模式，`PrivateState`。

也可以为图定义显式的输入和输出模式。在这些情况下，我们定义一个“内部”模式，其中包含与图操作相关的 _所有_ 键。但是，如果你也定义了 `input` 和 `output` 模式，它们是“内部”模式的子集，以约束图的输入和输出。有关更多详细信息，请参阅 [本指南](../how-tos/graph-api.md#define-input-and-output-schemas)。

让我们看一个例子：

:::python

```python
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # Write to OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # Read from OverallState, write to PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # Read from PrivateState, write to OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
# {'graph_output': 'My name is Lance'}
```

:::

:::js

```typescript
const InputState = z.object({
  userInput: z.string(),
});

const OutputState = z.object({
  graphOutput: z.string(),
});

const OverallState = z.object({
  foo: z.string(),
  userInput: z.string(),
  graphOutput: z.string(),
});

const PrivateState = z.object({
  bar: z.string(),
});

const graph = new StateGraph({
  state: OverallState,
  input: InputState,
  output: OutputState,
})
  .addNode("node1", (state) => {
    // Write to OverallState
    return { foo: state.userInput + " name" };
  })
  .addNode("node2", (state) => {
    // Read from OverallState, write to PrivateState
    return { bar: state.foo + " is" };
  })
  .addNode(
    "node3",
    (state) => {
      // Read from PrivateState, write to OutputState
      return { graphOutput: state.bar + " Lance" };
    },
    { input: PrivateState }
  )
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", "node3")
  .addEdge("node3", END)
  .compile();

await graph.invoke({ userInput: "My" });
// { graphOutput: 'My name is Lance' }
```

:::

这里有两点微妙且重要的地方需要注意：

:::python

1. 我们将 `state: InputState` 作为输入模式传递给 `node_1`。但是，我们写入 `foo`，这是 `OverallState` 中的一个通道。我们如何写入未包含在输入模式中的状态通道？这是因为节点 _可以写入图状态中的任何状态通道。_ 图状态是初始化时定义的状态通道的并集，其中包括 `OverallState` 以及过滤器 `InputState` 和 `OutputState`。

2. 我们使用 `StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)` 初始化图。那么，我们如何在 `node_2` 中写入 `PrivateState`？如果未在 `StateGraph` 初始化中传递该模式，图如何获得对此模式的访问权限？我们可以这样做，因为 _节点也可以声明额外的状态通道_，只要存在状态模式定义即可。在这种情况下，定义了 `PrivateState` 模式，因此我们可以将 `bar` 添加为图中的新状态通道并对其进行写入。
   :::

:::js

1. 我们将 `state` 作为输入模式传递给 `node1`。但是，我们写入 `foo`，这是 `OverallState` 中的一个通道。我们如何写入未包含在输入模式中的状态通道？这是因为节点 _可以写入图状态中的任何状态通道。_ 图状态是初始化时定义的状态通道的并集，其中包括 `OverallState` 以及过滤器 `InputState` 和 `OutputState`。

2. 我们使用 `StateGraph({ state: OverallState, input: InputState, output: OutputState })` 初始化图。那么，我们如何在 `node2` 中写入 `PrivateState`？如果未在 `StateGraph` 初始化中传递该模式，图如何获得对此模式的访问权限？我们可以这样做，因为 _节点也可以声明额外的状态通道_，只要存在状态模式定义即可。在这种情况下，定义了 `PrivateState` 模式，因此我们可以将 `bar` 添加为图中的新状态通道并对其进行写入。
   :::

### Reducer (Reducers)

Reducer 是理解如何将节点的更新应用于 `State` 的关键。`State` 中的每个键都有其自己独立的 reducer 函数。如果没有显式指定 reducer 函数，则假定对该键的所有更新都应覆盖它。有几种不同类型的 reducer，从默认类型的 reducer 开始：

#### 默认 Reducer (Default Reducer)

这两个示例展示了如何使用默认 reducer：

**示例 A:**

:::python

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

:::

:::js

```typescript
const State = z.object({
  foo: z.number(),
  bar: z.array(z.string()),
});
```

:::

在此示例中，未为任何键指定 reducer 函数。假设图的输入是：

:::python
`{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` 模式 - 只需一个更新。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，则 `State` 将变为 `{"foo": 2, "bar": ["bye"]}`。
:::

:::js
`{ foo: 1, bar: ["hi"] }`。然后假设第一个 `Node` 返回 `{ foo: 2 }`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` 模式 - 只需一个更新。应用此更新后，`State` 将变为 `{ foo: 2, bar: ["hi"] }`。如果第二个节点返回 `{ bar: ["bye"] }`，则 `State` 将变为 `{ foo: 2, bar: ["bye"] }`。
:::

**示例 B:**

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

在此示例中，我们使用 `Annotated` 类型为第二个键 (`bar`) 指定了一个 reducer 函数 (`operator.add`)。请注意，第一个键保持不变。假设图的输入是 `{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` 模式 - 只需一个更新。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，则 `State` 将变为 `{"foo": 2, "bar": ["hi", "bye"]}`。请注意，这里的 `bar` 键是通过将两个列表相加来更新的。
:::

:::js

```typescript
import { z } from "zod";
import { withLangGraph } from "@langchain/langgraph/zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
  }),
});
```

在此示例中，我们使用 `withLangGraph` 函数为第二个键 (`bar`) 指定了一个 reducer 函数。请注意，第一个键保持不变。假设图的输入是 `{ foo: 1, bar: ["hi"] }`。然后假设第一个 `Node` 返回 `{ foo: 2 }`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` 模式 - 只需一个更新。应用此更新后，`State` 将变为 `{ foo: 2, bar: ["hi"] }`。如果第二个节点返回 `{ bar: ["bye"] }`，则 `State` 将变为 `{ foo: 2, bar: ["hi", "bye"] }`。请注意，这里的 `bar` 键是通过将两个数组相加来更新的。
:::

### 在图状态中使用消息 (Working with Messages in Graph State)

#### 为什么要使用消息？ (Why use messages?)

:::python
大多数现代 LLM 提供者都有一个聊天模型接口，该接口接受消息列表作为输入。LangChain 的 [`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models) 特别接受 `Message` 对象列表作为输入。这些消息有多种形式，例如 `HumanMessage`（用户输入）或 `AIMessage`（LLM 响应）。要了解有关消息对象是什么的更多信息，请参阅 [此](https://python.langchain.com/docs/concepts/#messages) 概念指南。
:::

:::js
大多数现代 LLM 提供者都有一个聊天模型接口，该接口接受消息列表作为输入。LangChain 的 [`ChatModel`](https://js.langchain.com/docs/concepts/#chat-models) 特别接受 `Message` 对象列表作为输入。这些消息有多种形式，例如 `HumanMessage`（用户输入）或 `AIMessage`（LLM 响应）。要了解有关消息对象是什么的更多信息，请参阅 [此](https://js.langchain.com/docs/concepts/#messages) 概念指南。
:::

#### 在图中使用消息 (Using Messages in your Graph)

:::python
在许多情况下，将之前的对话历史记录作为消息列表存储在图状态中很有帮助。为此，我们可以向图状态添加一个键（通道），用于存储 `Message` 对象列表，并使用 reducer 函数对其进行注释（参见下面示例中的 `messages` 键）。reducer 函数对于告诉图如何在每次状态更新时更新状态中的 `Message` 对象列表至关重要（例如，当节点发送更新时）。如果您不指定 reducer，则每次状态更新都会用最近提供的值覆盖消息列表。如果您想简单地将消息追加到现有列表，可以使用 `operator.add` 作为 reducer。

但是，您可能还希望手动更新图状态中的消息（例如，人在回路）。如果您使用 `operator.add`，您发送到图的手动状态更新将追加到现有消息列表，而不是更新现有消息。为了避免这种情况，您需要一个能够跟踪消息 ID 并在更新时覆盖现有消息的 reducer。为实现此目的，您可以使用预构建的 `add_messages` 函数。对于全新的消息，它会简单地追加到现有列表中，但它也会正确处理现有消息的更新。
:::

:::js
在许多情况下，将之前的对话历史记录作为消息列表存储在图状态中很有帮助。为此，我们可以向图状态添加一个键（通道），用于存储 `Message` 对象列表，并使用 reducer 函数对其进行注释（参见下面示例中的 `messages` 键）。reducer 函数对于告诉图如何在每次状态更新时更新状态中的 `Message` 对象列表至关重要（例如，当节点发送更新时）。如果您不指定 reducer，则每次状态更新都会用最近提供的值覆盖消息列表。如果您想简单地将消息追加到现有列表，可以使用连接数组的函数作为 reducer。

但是，您可能还希望手动更新图状态中的消息（例如，人在回路）。如果您使用简单的连接函数，您发送到图的手动状态更新将追加到现有消息列表，而不是更新现有消息。为了避免这种情况，您需要一个能够跟踪消息 ID 并在更新时覆盖现有消息的 reducer。为实现此目的，您可以使用预构建的 `MessagesZodState` 模式。对于全新的消息，它会简单地追加到现有列表中，但它也会正确处理现有消息的更新。
:::

#### 序列化 (Serialization)

:::python
除了跟踪消息 ID 之外，即使在 `messages` 通道上收到状态更新，`add_messages` 函数也会尝试将消息反序列化为 LangChain `Message` 对象。查看有关 LangChain 序列化/反序列化的更多信息 [这里](https://python.langchain.com/docs/how_to/serialization/)。这允许以以下格式发送图输入/状态更新：

```python
# this is supported
{"messages": [HumanMessage(content="message")]}

# and this is also supported
{"messages": [{"type": "human", "content": "message"}]}
```

由于使用 `add_messages` 时状态更新总是反序列化为 LangChain `Messages`，因此您应该使用点表示法访问消息属性，如 `state["messages"][-1].content`。下面是一个使用 `add_messages` 作为其 reducer 函数的图扩展示例。

```python
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

:::

:::js
除了跟踪消息 ID 之外，即使在 `messages` 通道上收到状态更新，`MessagesZodState` 也会尝试将消息反序列化为 LangChain `Message` 对象。这允许以以下格式发送图输入/状态更新：

```typescript
// this is supported
{
  messages: [new HumanMessage("message")];
}

// and this is also supported
{
  messages: [{ role: "human", content: "message" }];
}
```

由于使用 `MessagesZodState` 时状态更新总是反序列化为 LangChain `Messages`，因此您应该使用点表示法访问消息属性，如 `state.messages[state.messages.length - 1].content`。下面是一个使用 `MessagesZodState` 的图扩展示例：

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";

const graph = new StateGraph(MessagesZodState)
  ...
```

`MessagesZodState` 使用单个 `messages` 键定义，该键是 `BaseMessage` 对象列表，并使用适当的 reducer。通常，除了消息之外，还有更多状态需要跟踪，因此我们看到人们扩展此状态并添加更多字段，如下所示：

```typescript
const State = z.object({
  messages: MessagesZodState.shape.messages,
  documents: z.array(z.string()),
});
```

:::

:::python

#### MessagesState

由于在状态中拥有消息列表非常常见，因此存在一个名为 `MessagesState` 的预构建状态，可以轻松使用消息。`MessagesState` 使用单个 `messages` 键定义，该键是 `AnyMessage` 对象列表，并使用 `add_messages` reducer。通常，除了消息之外，还有更多状态需要跟踪，因此我们看到人们对该状态进行子类化并添加更多字段，如下所示：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

:::

## 节点 (Nodes)

:::python

在 LangGraph 中，节点是 Python 函数（同步或异步），它们接受以下参数：

1. `state`: 图的 [state](#state)
2. `config`: 一个 `RunnableConfig` 对象，包含配置信息（如 `thread_id`）和跟踪信息（如 `tags`）
3. `runtime`: 一个 `Runtime` 对象，包含 [运行时 `context`](#runtime-context) 和其他信息（如 `store` 和 `stream_writer`）

类似于 `NetworkX`，您使用 @[add_node][add_node] 方法将这些节点添加到图中：

```python
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_config(state: State, config: RunnableConfig):
    print("In node with thread_id: ", config["configurable"]["thread_id"])
    return {"results": f"Hello, {state['input']}!"}


builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_config", node_with_config)
...
```

:::

:::js

在 LangGraph 中，节点通常是函数（同步或异步），它们接受以下参数：

1. `state`: 图的 [state](#state)
2. `config`: 一个 `RunnableConfig` 对象，包含配置信息（如 `thread_id`）和跟踪信息（如 `tags`）

您可以使用 `addNode` 方法将节点添加到图中。

```typescript
import { StateGraph } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

const State = z.object({
  input: z.string(),
  results: z.string(),
});

const builder = new StateGraph(State);
  .addNode("myNode", (state, config) => {
    console.log("In node: ", config?.configurable?.user_id);
    return { results: `Hello, ${state.input}!` };
  })
  addNode("otherNode", (state) => {
    return state;
  })
  ...
```

:::

在幕后，函数被转换为 [RunnableLambda](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.RunnableLambda.html)s，这也为您的函数添加了批处理和异步支持，以及原生跟踪和调试。

如果您在不指定名称的情况下向图中添加节点，它将被赋予一个等效于函数名称的默认名称。

:::python

```python
builder.add_node(my_node)
# You can then create edges to/from this node by referencing it as `"my_node"`
```

:::

:::js

```typescript
builder.addNode(myNode);
// You can then create edges to/from this node by referencing it as `"myNode"`
```

:::

### `START` 节点 (`START` Node)

`START` 节点是一个特殊节点，表示将用户输入发送到图的节点。引用此节点的主要目的是确定应首先调用哪些节点。

:::python

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### `END` 节点 (`END` Node)

`END` 节点是一个特殊节点，表示终止节点。当您想表示哪些边在完成后没有操作时，引用此节点。

:::python

```python
from langgraph.graph import END

graph.add_edge("node_a", END)
```

:::

:::js

```typescript
import { END } from "@langchain/langgraph";

graph.addEdge("nodeA", END);
```

:::

### 节点缓存 (Node Caching)

:::python
LangGraph 支持基于节点输入的任务/节点缓存。要使用缓存：

- 编译图时指定缓存（或指定入口点）
- 为节点指定缓存策略 （cache policy）。每个缓存策略支持：
  - `key_func` 用于根据节点的输入生成缓存键，默认为使用 pickle 对输入进行 `hash`。
  - `ttl`，缓存的生存时间（秒）。如果未指定，缓存将永不过期。

例如：

```python
import time
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy


class State(TypedDict):
    x: int
    result: int


builder = StateGraph(State)


def expensive_node(state: State) -> dict[str, int]:
    # expensive computation
    time.sleep(2)
    return {"result": state["x"] * 2}


builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))  # (1)!
[{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))  # (2)!
[{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

1. 第一次运行需要两秒钟才能运行（由于模拟的昂贵计算）。
2. 第二次运行利用缓存并迅速返回。
   :::

:::js
LangGraph 支持基于节点输入的任务/节点缓存。要使用缓存：

- 编译图时指定缓存（或指定入口点）
- 为节点指定缓存策略 （cache policy）。每个缓存策略支持：
  - `keyFunc`，用于根据节点的输入生成缓存键。
  - `ttl`，缓存的生存时间（秒）。如果未指定，缓存将永不过期。

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";
import { InMemoryCache } from "@langchain/langgraph-checkpoint";

const graph = new StateGraph(MessagesZodState)
  .addNode(
    "expensive_node",
    async () => {
      // Simulate an expensive operation
      await new Promise((resolve) => setTimeout(resolve, 3000));
      return { result: 10 };
    },
    { cachePolicy: { ttl: 3 } }
  )
  .addEdge(START, "expensive_node")
  .compile({ cache: new InMemoryCache() });

await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (1)!
// [{"expensive_node": {"result": 10}}]
await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (2)!
// [{"expensive_node": {"result": 10}, "__metadata__": {"cached": true}}]
```

:::

## 边 (Edges)

边定义了逻辑的路由方式以及图如何决定停止。这是代理如何工作以及不同节点如何相互通信的重要组成部分。有几种关键类型的边：

- 普通边 (Normal Edges): 直接从一个节点转到下一个节点。
- 条件边 (Conditional Edges): 调用一个函数来确定下一个去哪个节点。
- 入口点 (Entry Point): 当用户输入到达时首先调用哪个节点。
- 条件入口点 (Conditional Entry Point): 调用一个函数来确定当用户输入到达时首先调用哪个节点。

一个节点可以有多个传出边。如果一个节点有多个传出边，**所有** 这些目标节点将在下一个超级步中并行执行。

### 普通边 (Normal Edges)

:::python
如果您 **总是** 想从节点 A 转到节点 B，您可以直接使用 @[add_edge][add_edge] 方法。

```python
graph.add_edge("node_a", "node_b")
```

:::

:::js
如果您 **总是** 想从节点 A 转到节点 B，您可以直接使用 @[`addEdge`][add_edge] 方法。

```typescript
graph.addEdge("nodeA", "nodeB");
```

:::

### 条件边 (Conditional Edges)

:::python
如果您想 **可选地** 路由到 1 个或多个边（或可选地终止），可以使用 @[add_conditional_edges][add_conditional_edges] 方法。此方法接受节点的名称和一个在该节点执行后调用的“路由函数”：

```python
graph.add_conditional_edges("node_a", routing_function)
```

类似于节点，`routing_function` 接受图的当前 `state` 并返回一个值。

默认情况下，返回值 `routing_function` 用作下一个要发送状态的节点（或节点列表）的名称。所有这些节点将在下一个超级步中并行运行。

您可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点的名称。

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
如果您想 **可选地** 路由到 1 个或多个边（或可选地终止），可以使用 @[`addConditionalEdges`][add_conditional_edges] 方法。此方法接受节点的名称和一个在该节点执行后调用的“路由函数”：

```typescript
graph.addConditionalEdges("nodeA", routingFunction);
```

类似于节点，`routingFunction` 接受图的当前 `state` 并返回一个值。

默认情况下，返回值 `routingFunction` 用作下一个要发送状态的节点（或节点列表）的名称。所有这些节点将在下一个超级步中并行运行。

您可以选择提供一个对象，将 `routingFunction` 的输出映射到下一个节点的名称。

```typescript
graph.addConditionalEdges("nodeA", routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

!!! tip

    如果您想在一个函数中结合状态更新和路由，请使用 [`Command`](#command) 而不是条件边。

### 入口点 (Entry Point)

:::python
入口点是图启动时运行的第一个节点。您可以使用虚拟 @[`START`][START] 节点到要执行的第一个节点的 @[`add_edge`][add_edge] 方法来指定进入图的位置。

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js
入口点是图启动时运行的第一个节点。您可以使用虚拟 @[`START`][START] 节点到要执行的第一个节点的 @[`addEdge`][add_edge] 方法来指定进入图的位置。

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### 条件入口点 (Conditional Entry Point)

:::python
条件入口点允许您根据自定义逻辑从不同的节点开始。您可以使用虚拟 @[`START`][START] 节点的 @[`add_conditional_edges`][add_conditional_edges] 来完成此操作。

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

您可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点的名称。

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
条件入口点允许您根据自定义逻辑从不同的节点开始。您可以使用虚拟 @[`START`][START] 节点的 @[`addConditionalEdges`][add_conditional_edges] 来完成此操作。

```typescript
import { START } from "@langchain/langgraph";

graph.addConditionalEdges(START, routingFunction);
```

您可以选择提供一个对象，将 `routingFunction` 的输出映射到下一个节点的名称。

```typescript
graph.addConditionalEdges(START, routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

## `Send`

:::python
默认情况下，`Nodes` 和 `Edges` 是提前定义的，并对同一个共享状态进行操作。但是，在某些情况下，确切的边可能无法提前知道，和/或您可能希望 `State` 的不同版本同时存在。这种情况的常见示例是 [map-reduce](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 设计模式。在此设计模式中，第一个节点可能会生成一个对象列表，您可能希望将其他某个节点应用于所有这些对象。对象的数量可能无法提前知道（意味着边的数量可能无法知道），并且下游 `Node` 的输入 `State` 应该是不同的（每个生成的对象一个）。

为了支持这种设计模式，LangGraph 支持从条件边返回 @[`Send`][Send] 对象。`Send` 接受两个参数：第一个是节点的名称，第二个是要传递给该节点的状态。

```python
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

:::

:::js
默认情况下，`Nodes` 和 `Edges` 是提前定义的，并对同一个共享状态进行操作。但是，在某些情况下，确切的边可能无法提前知道，和/或您可能希望 `State` 的不同版本同时存在。这种情况的常见示例是 map-reduce 设计模式。在此设计模式中，第一个节点可能会生成一个对象列表，您可能希望将其他某个节点应用于所有这些对象。对象的数量可能无法提前知道（意味着边的数量可能无法知道），并且下游 `Node` 的输入 `State` 应该是不同的（每个生成的对象一个）。

为了支持这种设计模式，LangGraph 支持从条件边返回 @[`Send`][Send] 对象。`Send` 接受两个参数：第一个是节点的名称，第二个是要传递给该节点的状态。

```typescript
import { Send } from "@langchain/langgraph";

graph.addConditionalEdges("nodeA", (state) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
});
```

:::

## `Command`

:::python
将控制流（边）和状态更新（节点）结合起来可能很有用。例如，您可能希望 **同时** 执行状态更新 **并** 在 **同一个** 节点中决定下一个去哪个节点。LangGraph 提供了一种通过从节点函数返回 @[`Command`][Command] 对象来实现此目的的方法：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

使用 `Command`，您还可以实现动态控制流行为（与 [条件边](#conditional-edges) 相同）：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

:::

:::js
将控制流（边）和状态更新（节点）结合起来可能很有用。例如，您可能希望 **同时** 执行状态更新 **并** 在 **同一个** 节点中决定下一个去哪个节点。LangGraph 提供了一种通过从节点函数返回 `Command` 对象来实现此目的的方法：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "myOtherNode",
  });
});
```

使用 `Command`，您还可以实现动态控制流行为（与 [条件边](#conditional-edges) 相同）：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  if (state.foo === "bar") {
    return new Command({
      update: { foo: "baz" },
      goto: "myOtherNode",
    });
  }
});
```

在节点函数中使用 `Command` 时，添加节点时必须添加 `ends` 参数以指定它可以路由到哪些节点：

```typescript
builder.addNode("myNode", myNode, {
  ends: ["myOtherNode", END],
});
```

:::

!!! important

    在节点函数中返回 `Command` 时，必须添加返回类型注释，列出节点路由到的节点名称列表，例如 `Command[Literal["my_other_node"]]`。这是图渲染所必需的，并告诉 LangGraph `my_node` 可以导航到 `my_other_node`。

查看此 [操作指南](../how-tos/graph-api.md#combine-control-flow-and-state-updates-with-command)，了解有关如何使用 `Command` 的端到端示例。

### 我什么时候应该使用 Command 而不是条件边？ (When should I use Command instead of conditional edges?)

- 当您需要 **同时** 更新图状态 **和** 路由到不同的节点时，请使用 `Command`。例如，在实施 [多代理切换](./multi_agent.md#handoffs) 时，路由到不同的代理并将一些信息传递给该代理很重要。
- 使用 [条件边](#conditional-edges) 在不更新状态的情况下有条件地在节点之间路由。

### 导航到父图中的节点 (Navigating to a node in a parent graph)

:::python
如果您正在使用 [子图](./subgraphs.md)，您可能希望从子图中的节点导航到不同的子图（即父图中的不同节点）。为此，您可以在 `Command` 中指定 `graph=Command.PARENT`：

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # where `other_subgraph` is a node in the parent graph
        graph=Command.PARENT
    )
```

!!! note

    将 `graph` 设置为 `Command.PARENT` 将导航到最近的父图。

!!! important "State updates with `Command.PARENT`"

    当您将更新从子图节点发送到父图节点，且该键由父图和子图 [状态模式](#schema) 共享时，您 **必须** 为要在父图状态中更新的键定义一个 [reducer](#reducers)。请参阅此 [示例](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph)。

:::

:::js
如果您正在使用 [子图](./subgraphs.md)，您可能希望从子图中的节点导航到不同的子图（即父图中的不同节点）。为此，您可以在 `Command` 中指定 `graph: Command.PARENT`：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph", // where `otherSubgraph` is a node in the parent graph
    graph: Command.PARENT,
  });
});
```

!!! note

    将 `graph` 设置为 `Command.PARENT` 将导航到最近的父图。

!!! important "State updates with `Command.PARENT`"

    当您将更新从子图节点发送到父图节点，且该键由父图和子图 [状态模式](#schema) 共享时，您 **必须** 为要在父图状态中更新的键定义一个 [reducer](#reducers)。

:::

这在实施 [多代理切换](./multi_agent.md#handoffs) 时特别有用。

查看 [本指南](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph) 了解详情。

### 在工具内部使用 (Using inside tools)

一个常见的用例是从工具内部更新图状态。例如，在客户支持应用程序中，您可能希望在对话开始时根据帐号或 ID 查找客户信息。

有关详细信息，请参阅 [本指南](../how-tos/graph-api.md#use-inside-tools)。

### 人在回路 (Human-in-the-loop)

:::python
`Command` 是人在回路工作流的重要组成部分：当使用 `interrupt()` 收集用户输入时，然后使用 `Command` 提供输入并通过 `Command(resume="User input")` 恢复执行。查看 [此概念指南](./human_in_the_loop.md) 以获取更多信息。
:::

:::js
`Command` 是人在回路工作流的重要组成部分：当使用 `interrupt()` 收集用户输入时，然后使用 `Command` 提供输入并通过 `new Command({ resume: "User input" })` 恢复执行。查看 [人在回路概念指南](./human_in_the_loop.md) 以获取更多信息。
:::

## 图迁移 (Graph Migrations)

LangGraph 可以轻松处理图定义（节点、边和状态）的迁移，即使在使用检查点跟踪状态时也是如此。

- 对于图末尾的线程（即未中断），您可以更改图的整个拓扑结构（即所有节点和边，删除、添加、重命名等）
- 对于当前中断的线程，我们支持除重命名/删除节点之外的所有拓扑更改（因为该线程现在可能即将进入不再存在的节点）——如果这是一个阻碍因素，请联系我们，我们可以优先考虑解决方案。
- 对于修改状态，我们完全向后和向前兼容添加和删除键
- 重命名的状态键会丢失现有线程中保存的状态
- 类型以不兼容方式更改的状态键目前可能会导致线程在更改之前具有状态的问题——如果这是一个阻碍因素，请联系我们，我们可以优先考虑解决方案。

:::python

## 运行时上下文 (Runtime Context)

创建图时，您可以指定传递给节点的运行时上下文的 `context_schema`。这对于传递
不属于图状态的信息给节点很有用。例如，您可能希望传递依赖项，例如模型名称或数据库连接。

```python
@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)
```

:::

:::js

创建图时，您还可以标记图的某些部分是可配置的。这通常用于轻松切换模型或系统提示。这允许您创建一个单一的“认知架构”（图），但拥有它的多个不同实例。

您可以选择在创建图时指定配置模式。

```typescript
import { z } from "zod";

const ConfigSchema = z.object({
  llm: z.string(),
});

const graph = new StateGraph(State, ConfigSchema);
```

:::

:::python
通过使用 `invoke` 方法的 `context` 参数，您可以将此上下文传递到图中。

```python
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

:::

:::js
您可以使用 `configurable` 配置字段将此配置传递到图中。

```typescript
const config = { configurable: { llm: "anthropic" } };

await graph.invoke(inputs, config);
```

:::

然后，您可以在节点或条件边内访问和使用此上下文：

```python
from langgraph.runtime import Runtime

def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
    ...
```

有关配置的完整细分，请参阅 [本指南](../how-tos/graph-api.md#add-runtime-configuration)。
:::

:::js

```typescript
graph.addNode("myNode", (state, config) => {
  const llmType = config?.configurable?.llm || "openai";
  const llm = getLlm(llmType);
  return { results: `Hello, ${state.input}!` };
});
```

:::

### 递归限制 (Recursion Limit)

:::python
递归限制设置了图在单次执行期间可以执行的最大 [超级步](#graphs) 数。一旦达到限制，LangGraph 将引发 `GraphRecursionError`。默认情况下，此值设置为 25 步。递归限制可以在运行时在任何图上设置，并通过配置字典传递给 `.invoke`/`.stream`。重要的是，`recursion_limit` 是一个独立的 `config` 键，不应像所有其他用户定义的配置那样在 `configurable` 键内传递。请参阅下面的示例：

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

阅读 [此操作指南](https://langchain-ai.github.io/langgraph/how-tos/recursion-limit/) 了解有关递归限制如何工作的更多信息。
:::

:::js
递归限制设置了图在单次执行期间可以执行的最大 [超级步](#graphs) 数。一旦达到限制，LangGraph 将引发 `GraphRecursionError`。默认情况下，此值设置为 25 步。递归限制可以在运行时在任何图上设置，并通过配置对象传递给 `.invoke`/`.stream`。重要的是，`recursionLimit` 是一个独立的 `config` 键，不应像所有其他用户定义的配置那样在 `configurable` 键内传递。请参阅下面的示例：

```typescript
await graph.invoke(inputs, {
  recursionLimit: 5,
  configurable: { llm: "anthropic" },
});
```

:::

## 可视化 (Visualization)

能够可视化图通常很棒，尤其是当它们变得越来越复杂时。LangGraph 附带了几种内置的可视化图的方法。有关更多信息，请参阅 [此操作指南](../how-tos/graph-api.md#visualize-your-graph)。
