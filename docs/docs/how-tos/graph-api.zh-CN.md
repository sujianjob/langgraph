# 如何使用 Graph API (How to use the graph API)

本指南演示了 LangGraph 的 Graph API 的基础知识。它涵盖了 [状态 (state)](#define-and-update-state)，以及如何组合常见的图结构，如 [序列 (sequences)](#create-a-sequence-of-steps)、[分支 (branches)](#create-branches) 和 [循环 (loops)](#create-and-control-loops)。它还涵盖了 LangGraph 的控制功能，包括用于 map-reduce 工作流的 [Send API](#map-reduce-and-the-send-api) 和用于结合状态更新与节点间 "跳转" 的 [Command API](#combine-control-flow-and-state-updates-with-command)。

## 设置 (Setup)

<details>
<summary>查看 Python 设置</summary>

安装 `langgraph`:

```bash
pip install -U langgraph
```
</details>

<details>
<summary>查看 TypeScript 设置</summary>

安装 `langgraph`:

```bash
npm install @langchain/langgraph
```
</details>

!!! tip "Set up LangSmith for better debugging"

    注册 [LangSmith](https://smith.langchain.com) 以快速发现问题并改善您的 LangGraph 项目的性能。LangSmith 允许您使用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用程序 — 详情请阅读 [文档](https://docs.smith.langchain.com)。

## 定义和更新状态 (Define and update state)

这里我们展示如何在 LangGraph 中定义和更新 [状态](../concepts/low_level.md#state)。我们将演示：

1. 如何使用状态定义图的 [架构 (schema)](../concepts/low_level.md#schema)
2. 如何使用 [归约器 (reducers)](../concepts/low_level.md#reducers) 来控制状态更新的处理方式。

### 定义状态 (Define state)

<details>
<summary>查看 Python 代码</summary>

LangGraph 中的 [状态](../concepts/low_level.md#state) 可以是 `TypedDict`、`Pydantic` 模型或 dataclass。下面我们将使用 `TypedDict`。有关使用 Pydantic 的详细信息，请参阅 [此部分](#use-pydantic-models-for-graph-state)。

```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

此状态跟踪 [消息](https://python.langchain.com/docs/concepts/messages/) 对象的列表，以及一个额外的整数字段。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

LangGraph 中的 [状态](../concepts/low_level.md#state) 可以使用 Zod 架构定义。下面我们将使用 Zod。有关使用替代方法的详细信息，请参阅 [此部分](#alternative-state-definitions)。

```typescript
import { BaseMessage } from "@langchain/core/messages";
import { z } from "zod";

const State = z.object({
  messages: z.array(z.custom<BaseMessage>()),
  extraField: z.number(),
});
```

此状态跟踪 [消息](https://js.langchain.com/docs/concepts/messages/) 对象的列表，以及一个额外的整数字段。
</details>

默认情况下，图将具有相同的输入和输出架构，并且状态决定了该架构。有关如何定义不同的输入和输出架构，请参阅 [此部分](#define-input-and-output-schemas)。

### 更新状态 (Update state)

<details>
<summary>查看 Python 代码</summary>

让我们构建一个包含单个节点的示例图。我们的 [节点](../concepts/low_level.md#nodes) 只是一个 Python 函数，它读取图的状态并对其进行更新。此函数的第一个参数始终是状态：

```python
from langchain_core.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

此节点只是将一条消息附加到我们的消息列表，并填充一个额外的字段。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

让我们构建一个包含单个节点的示例图。我们的 [节点](../concepts/low_level.md#nodes) 只是一个 TypeScript 函数，它读取图的状态并对其进行更新。此函数的第一个参数始终是状态：

```typescript
import { AIMessage } from "@langchain/core/messages";

const node = (state: z.infer<typeof State>) => {
  const messages = state.messages;
  const newMessage = new AIMessage("Hello!");
  return { messages: messages.concat([newMessage]), extraField: 10 };
};
```

此节点只是将一条消息附加到我们的消息列表，并填充一个额外的字段。
</details>

!!! important
    节点应直接返回对状态的更新，而不是就地改变 (mutating) 状态。

<details>
<summary>查看 Python 代码</summary>

接下来让我们定义一个包含此节点的简单图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义对此状态进行操作的图。然后我们使用 `add_node` 来填充我们的图。

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```

在本例中，我们的图只执行单个节点。让我们继续进行简单的调用：

```python
from langchain_core.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```

```python
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

接下来让我们定义一个包含此节点的简单图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义对此状态进行操作的图。然后我们使用 `addNode` 来填充我们的图。

```typescript
import { StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", node)
  .addEdge("__start__", "node")
  .compile();
  
import { HumanMessage } from "@langchain/core/messages";

const result = await graph.invoke({ messages: [new HumanMessage("Hi")], extraField: 0 });
console.log(result);
```

```typescript
{ messages: [HumanMessage { content: 'Hi' }, AIMessage { content: 'Hello!' }], extraField: 10 }
```
</details>

### 使用归约器处理状态更新 (Process state updates with reducers)

状态中的每个键都可以有自己的独立 [归约器 (reducer)](../concepts/low_level.md#reducers) 函数，它控制如何应用来自节点的更新。如果没有显式指定归约器函数，则假定对键的所有更新都应覆盖它。

<details>
<summary>查看 Python 代码</summary>

对于 `TypedDict` 状态架构，我们可以通过使用归约器函数注释状态的相应字段来定义归约器。

```python
from typing_extensions import Annotated
import operator

def add(left, right):
    """Can also import `add` from the `operator` built-in."""
    return left + right

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    # highlight-next-line
    return {"messages": [new_message], "extra_field": 10}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

对于 Zod 状态架构，我们可以使用架构字段上的特殊 `.langgraph.reducer()` 方法来定义归约器。

```typescript
import "@langchain/langgraph/zod";

const State = z.object({
  // highlight-next-line
  messages: z.array(z.custom<BaseMessage>()).langgraph.reducer((x, y) => x.concat(y)),
  extraField: z.number(),
});

const node = (state: z.infer<typeof State>) => {
  const newMessage = new AIMessage("Hello!");
  // highlight-next-line
  return { messages: [newMessage], extraField: 10 };
};
```
</details>

#### MessagesState

在实践中，更新消息列表还有其他的考虑因素：

- 我们可能希望更新状态中的现有消息。
- 我们可能希望接受 [消息格式](../concepts/low_level.md#using-messages-in-your-graph) 的简写，例如 [OpenAI 格式](https://python.langchain.com/docs/concepts/messages/#openai-format)。

LangGraph 包含一个预构建的 `MessagesState` (Python) / `MessagesZodState` (JS)，它为了方便处理了这些考虑因素。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { MessagesZodState } from "@langchain/langgraph";

const State = MessagesZodState.extend({
  extraField: z.number(),
});
```
</details>

### 定义输入和输出架构 (Define input and output schemas)

默认情况下，`StateGraph` 使用单一架构运行，并且预计所有节点都使用该架构进行通信。但是，也可以为图定义不同的输入和输出架构。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# Define the schema for the input
class InputState(TypedDict):
    question: str

# Define the schema for the output
class OutputState(TypedDict):
    answer: str

# Define the overall schema, combining both input and output
class OverallState(InputState, OutputState):
    pass

# ... (rest of the implementation)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
// Define the schema for the input
const InputState = z.object({
  question: z.string(),
});

// Define the schema for the output
const OutputState = z.object({
  answer: z.string(),
});

// Define the overall schema, combining both input and output
const OverallState = InputState.merge(OutputState);

// ... (rest of the implementation)
```
</details>

### 在节点之间传递私有状态 (Pass private state between nodes)

在某些情况下，您可能希望节点交换对中间逻辑至关重要但不需要成为图的主要架构一部分的信息。

<details>
<summary>查看 Python 代码</summary>

```python
# The overall state of the graph (public)
class OverallState(TypedDict):
    a: str

# Output from node_1 contains private data
class Node1Output(TypedDict):
    private_data: str

# The private data is only shared between node_1 and node_2
def node_1(state: OverallState) -> Node1Output:
    return {"private_data": "set by node_1"}

class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    return {"a": "set by node_2"}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const OverallState = z.object({ a: z.string() });
const Node1Output = z.object({ privateData: z.string() });

const node1 = (state: z.infer<typeof OverallState>): z.infer<typeof Node1Output> => {
  return { privateData: "set by node1" };
};

const Node2Input = z.object({ privateData: z.string() });

const node2 = (state: z.infer<typeof Node2Input>): z.infer<typeof OverallState> => {
  return { a: "set by node2" };
};
```
</details>

### 使用 Pydantic 模型作为图状态 (Use Pydantic models for graph state)

`StateGraph` 在初始化时接受 `state_schema` 参数，该参数指定图中节点可以访问和更新的状态的 "形状"。Pydantic 模型可用于 `state_schema` 以在 **输入** 上添加运行时验证。

<details>
<summary>查看 Python 代码</summary>

```python
from pydantic import BaseModel

class OverallState(BaseModel):
    a: str
```
</details>

## 添加运行时配置 (Add runtime configuration)

有时您希望能够在调用图时对其进行配置。例如，您可能希望能够在运行时指定要使用的 LLM 或系统提示，而 *不使用这些参数污染图状态*。

1. 指定配置的架构
2. 将配置添加到节点或条件边的函数签名中
3. 将配置传入图

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.runtime import Runtime

# 1. Specify config schema
class ContextSchema(TypedDict):
    my_runtime_value: str

# 2. Define a graph that accesses the config in a node
def node(state: State, runtime: Runtime[ContextSchema]):
    if runtime.context["my_runtime_value"] == "a":
        return {"my_state_value": 1}
    # ...

builder = StateGraph(State, context_schema=ContextSchema)
# ...
graph = builder.compile()

# 3. Pass in configuration at runtime:
graph.invoke({}, context={"my_runtime_value": "a"})
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
// 1. Specify config schema
const ConfigurableSchema = z.object({
  myRuntimeValue: z.string(),
});

// 2. Define a graph that accesses the config in a node
const graph = new StateGraph(State)
  .addNode("node", (state, config) => {
    if (config?.configurable?.myRuntimeValue === "a") {
      return { myStateValue: 1 };
    }
    // ...
  })
  // ...
  .compile();

// 3. Pass in configuration at runtime:
await graph.invoke({}, { configurable: { myRuntimeValue: "a" } });
```
</details>

## 添加重试策略 (Add retry policies)

LangGraph 允许您向节点添加重试策略。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const graph = new StateGraph(State)
  .addNode("nodeName", nodeFunction, { retryPolicy: {} })
  .compile();
```
</details>

## 添加节点缓存 (Add node caching)

节点缓存对于避免重复操作非常有用，例如执行昂贵的操作时。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)

graph = builder.compile(cache=InMemoryCache())
```
</details>

## 创建步骤序列 (Create a sequence of steps)

这里我们演示如何构建一个简单的步骤序列。

<details>
<summary>查看 Python 代码</summary>

```python
# Add nodes
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# Add edges
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")

# Or use shorthand
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const builder = new StateGraph(State)
  .addNode("step1", step1)
  .addNode("step2", step2)
  .addNode("step3", step3)
  .addEdge(START, "step1")
  .addEdge("step1", "step2")
  .addEdge("step2", "step3");
```
</details>

## 创建分支 (Create branches)

LangGraph 通过扇出 (fan-out) 和扇入 (fan-in) 机制提供对并行执行节点的原生支持。

### 并行运行图节点 (Run graph nodes in parallel)

<details>
<summary>查看 Python 代码</summary>

```python
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addNode("c", nodeC)
  .addNode("d", nodeD)
  .addEdge(START, "a")
  .addEdge("a", "b")
  .addEdge("a", "c")
  .addEdge("b", "d")
  .addEdge("c", "d")
  .addEdge("d", END)
  .compile();
```
</details>

### 推迟节点执行 (Defer node execution)

当您希望延迟节点的执行直到完成所有其他挂起的任务时，推迟节点执行非常有用。

<details>
<summary>查看 Python 代码</summary>

```python
# highlight-next-line
builder.add_node(d, defer=True)
```
</details>

### 条件分支 (Conditional branching)

如果您的扇出应根据状态在运行时变化，可以使用 `add_conditional_edges`。

<details>
<summary>查看 Python 代码</summary>

```python
def conditional_edge(state: State) -> Literal["b", "c"]:
    return state["which"]

builder.add_conditional_edges("a", conditional_edge)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const conditionalEdge = (state: z.infer<typeof State>): "b" | "c" => {
  return state.which as "b" | "c";
};

const graph = new StateGraph(State)
  // ...
  .addConditionalEdges("a", conditionalEdge)
  .compile();
```
</details>

## Map-Reduce 和 Send API (Map-Reduce and the Send API)

LangGraph 使用 Send API 支持 map-reduce 和其他高级分支模式。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.types import Send

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { Send } from "@langchain/langgraph";

const continueToJokes = (state: z.infer<typeof OverallState>) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
};

const graph = new StateGraph(OverallState)
  // ...
  .addConditionalEdges("generateTopics", continueToJokes)
  .compile();
```
</details>

## 创建和控制循环 (Create and control loops)

在创建带有循环的图时，我们需要一种终止执行的机制。这通常通过添加一个 [条件边 (conditional edge)](../concepts/low_level.md#conditional-edges) 来完成，该边在满足终止条件时路由到 [END](../concepts/low_level.md#end-node) 节点。

<details>
<summary>查看 Python 代码</summary>

```python
def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const route = (state: z.infer<typeof State>): "b" | typeof END => {
  if (terminationCondition(state)) {
    return END;
  } else {
    return "b";
  }
};

const graph = new StateGraph(State)
  // ...
  .addConditionalEdges("a", route)
  .addEdge("b", "a")
  .compile();
```
</details>

### 施加递归限制 (Impose a recursion limit)

LangGraph 具有内置的递归限制，以防止无限循环。

<details>
<summary>查看 Python 代码</summary>

```python
try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("Recursion Error")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
try {
  await graph.invoke(inputs, { recursionLimit: 3 });
} catch (error) {
  if (error instanceof GraphRecursionError) {
    console.log("Recursion Error");
  }
}
```
</details>

## 使用 Command 结合控制流和状态更新 (Combine control flow and state updates with Command)

我们也可以在节点内结合控制流（边）和状态更新。这对于想要在条件逻辑旁边更新状态的节点（如路由节点）很有用。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.types import Command

def node_a(state: State) -> Command[Literal["node_b"]]:
    return Command(
        update={"foo": "bar"},
        goto="node_b"
    )
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { Command } from "@langchain/langgraph";

const nodeA = (state: State): Command => {
  return new Command({
    update: { foo: "bar" },
    goto: "node_b",
  });
};
```
</details>

## 可视化您的图 (Visualize your graph)

<details>
<summary>查看 Python 代码</summary>

```python
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
```
</details>
