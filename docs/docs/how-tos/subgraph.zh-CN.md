# 如何使用子图 (How to use subgraphs)

[子图](../concepts/subgraphs.md) 允许您创建和管理状态的不同部分。这对于构建 [多智能体系统](../concepts/multi_agent.md) 和执行复杂任务非常有用。

本指南提供了有关如何：

- 添加 [同一状态模式](#state-schema) 的子图
- 添加 [不同状态模式](#transform-state-schema) 的子图
- 在 [父图中](#compiled-graph) 添加子图

的示例。

## 设置 (Setup)

首先，让我们安装所需的包

```python
%%capture --no-stderr
%pip install -U langgraph
```

## 简单示例 (Simple Example)

最简单的方法是将已编译的图在添加节点时直接作为节点函数。

!!! note "子图作为编译后的图 (Subgraph as a compiled graph)"
    重要的是，子图必须是 **已编译的图**（即 `subgraph = graph_builder.compile()`）。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import START, StateGraph
from typing import TypedDict

# 1. Define subgraph
class SubgraphState(TypedDict):
    foo: str

def subgraph_node_1(state: SubgraphState):
    return {"foo": "bar"}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 2. Define parent graph
class ParentState(TypedDict):
    foo: str

def node_1(state: ParentState):
    return {"foo": "hi!"}

builder = StateGraph(ParentState)
builder.add_node("node_1", node_1)
# Note that we are adding the compiled subgraph as a node
builder.add_node("node_2", subgraph)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
graph = builder.compile()
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

// 1. Define subgraph
const SubgraphState = z.object({
  foo: z.string().optional(),
});

const subgraphBuilder = new StateGraph(SubgraphState)
  .addNode("subgraphNode1", (state) => ({ foo: "bar" }))
  .addEdge(START, "subgraphNode1");

const subgraph = subgraphBuilder.compile();

// 2. Define parent graph
const ParentState = z.object({
  foo: z.string().optional(),
});

const builder = new StateGraph(ParentState)
  .addNode("node1", (state) => ({ foo: "hi!" }))
  // Note that we are adding the compiled subgraph as a node
  .addNode("node2", subgraph)
  .addEdge(START, "node1")
  .addEdge("node1", "node2");

const graph = builder.compile();
```
</details>

现在，让我们调用图来查看输出。

```python
for chunk in graph.stream({"foo": "foo"}):
    print(chunk)
```

```python
{'node_1': {'foo': 'hi!'}}
{'node_2': {'foo': 'bar'}}
```

## 状态模式 (State Schema)

当使用子图时，您必须定义子图状态与父图状态之间的交互方式。

### 共享状态键 (Shared state keys)

父图和子图之间共享的任何键都将自动传递给子图，并且任何更新都将自动传回给父图。

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Optional, Annotated
from langgraph.graph import StateGraph, START, END
# Note: we are using `add` reducer to allow redundant updates from the subgraph
from operator import add 

# Define message type for redundancy
class Message(TypedDict):
    content: str

# 1. Define subgraph
class SubgraphState(TypedDict):
    # Shared keys
    foo: str
    bar: str
    # Subgraph-only key 
    # (optional, but good for clarity)
    baz: Optional[str]

def subgraph_node(state: SubgraphState):
    # Subgraph can access and update shared keys
    return {
        "bar": " world",
        "baz": " private state" 
    }

subgraph = StateGraph(SubgraphState)
subgraph.add_node(subgraph_node)
subgraph.add_edge(START, "subgraph_node")
subgraph_app = subgraph.compile()

# 2. Define parent graph 
class ParentState(TypedDict):
    foo: str
    bar: str

def parent_node(state: ParentState):
    return {"foo": "Hello"}

graph = StateGraph(ParentState)
graph.add_node("parent_node", parent_node)
graph.add_node("subgraph", subgraph_app)

graph.add_edge(START, "parent_node")
graph.add_edge("parent_node", "subgraph")
graph.add_edge("subgraph", END)

app = graph.compile()

# Run
print(app.invoke({"foo": "", "bar": ""}))
# Output: {'foo': 'Hello', 'bar': ' world'}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// 1. Define subgraph
const SubgraphState = z.object({
  // Shared keys
  foo: z.string(),
  bar: z.string(),
  // Subgraph-only key
  baz: z.string().optional(),
});

const subgraphBuilder = new StateGraph(SubgraphState)
  .addNode("subgraphNode", (state) => ({
    bar: " world",
    baz: " private state",
  }))
  .addEdge(START, "subgraphNode");
const subgraphApp = subgraphBuilder.compile();

// 2. Define parent graph
const ParentState = z.object({
  foo: z.string(),
  bar: z.string(),
});

const builder = new StateGraph(ParentState)
  .addNode("parentNode", (state) => ({ foo: "Hello" }))
  .addNode("subgraph", subgraphApp)
  .addEdge(START, "parentNode")
  .addEdge("parentNode", "subgraph")
  .addEdge("subgraph", END);

const app = builder.compile();

// Run
console.log(await app.invoke({ foo: "", bar: "" }));
// Output: { foo: 'Hello', bar: ' world' }
```
</details>

在上面的示例中：
* `foo` 和 `bar` 是共享键。
* `subgraph_node` 可以读取和更新 `foo` 和 `bar`。
* `baz` 是子图私有的（未在 `ParentState` 中定义），因此不会泄露给父图。

### 转换状态模式 (Transform state schema)

有时您可能希望将父状态转换为不同的子图状态，或者防止父状态和子图状态之间的键冲突。

您可以通过使用传递给 `.add_node` 的 `input` 和 `output` 参数来实现这一点，该函数接受一个状态并返回另一个状态。

<details>
<summary>查看 Python 代码</summary>

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# 1. Define subgraph
class SubgraphState(TypedDict):
    # Subgraph expects "inner_val"
    inner_val: str
    output_val: str

def subgraph_node(state: SubgraphState):
    return {"output_val": state["inner_val"] + " processed"}

subgraph = StateGraph(SubgraphState)
subgraph.add_node(subgraph_node)
subgraph.add_edge(START, "subgraph_node")
subgraph_app = subgraph.compile()

# 2. Define parent graph
class ParentState(TypedDict):
    # Parent has "outer_val"
    outer_val: str
    final_result: str

def parent_node(state: ParentState):
    return {"outer_val": "data"}

# Input transformation: ParentState -> SubgraphState
def input_transform(state: ParentState) -> SubgraphState:
    return {
        "inner_val": state["outer_val"],
        # Initialize other keys if needed
        "output_val": ""
    }

# Output transformation: SubgraphState -> ParentState
def output_transform(state: SubgraphState) -> dict:
    return {
        "final_result": state["output_val"]
    }

graph = StateGraph(ParentState)
graph.add_node("parent_node", parent_node)

# Add subgraph with transformations
graph.add_node(
    "subgraph", 
    subgraph_app
).with_config(
    input_mapper=input_transform,
    output_mapper=output_transform
)

graph.add_edge(START, "parent_node")
graph.add_edge("parent_node", "subgraph")
graph.add_edge("subgraph", END)

app = graph.compile()

# Run
print(app.invoke({"outer_val": "", "final_result": ""}))
# Output: {'outer_val': 'data', 'final_result': 'data processed'}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// 1. Define subgraph
const SubgraphState = z.object({
  innerVal: z.string(),
  outputVal: z.string(),
});

const subgraphBuilder = new StateGraph(SubgraphState)
  .addNode("subgraphNode", (state) => ({
    outputVal: state.innerVal + " processed",
  }))
  .addEdge(START, "subgraphNode");
const subgraphApp = subgraphBuilder.compile();

// 2. Define parent graph
const ParentState = z.object({
  // Parent has "outer_val"
  outerVal: z.string(),
  finalResult: z.string(),
});

const builder = new StateGraph(ParentState)
  .addNode("parentNode", (state) => ({ outerVal: "data" }))
  // Add subgraph with transformations
  .addNode("subgraph", subgraphApp, {
    // Input transformation: ParentState -> SubgraphState
    inputMapper: (state) => ({
      innerVal: state.outerVal,
      outputVal: "",
    }),
    // Output transformation: SubgraphState -> ParentState
    outputMapper: (state) => ({
      finalResult: state.outputVal,
    }),
  })
  .addEdge(START, "parentNode")
  .addEdge("parentNode", "subgraph")
  .addEdge("subgraph", END);

const app = builder.compile();

// Run
console.log(await app.invoke({ outerVal: "", finalResult: "" }));
// Output: { outerVal: 'data', finalResult: 'data processed' }
```
</details>

## 编译后的图 (Compiled Graph)

将编译后的图（子图）作为节点添加到另一个图（父图）时，父图将像对待其他节点一样对待它。

这意味着：
1. **状态传递**：父节点将当前状态传递给子图节点。
2. **执行**：子图节点（编译后的图）运行直到完成。
3. **输出**：子图的最终状态将作为该节点的输出返回，并用于更新父图的状态。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import START, StateGraph
from typing import TypedDict

# 1. Define subgraph
class SubgraphState(TypedDict):
    foo: str # note that this key is shared with the parent graph state
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "bar"}

def subgraph_node_2(state: SubgraphState):
    # note that this node is using a state key ('bar') that is only available in the subgraph
    # and is sending update on the shared state key ('foo')
    return {"foo": state["foo"] + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_node(subgraph_node_2)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
subgraph = subgraph_builder.compile()

# 2. Define parent graph
class ParentState(TypedDict):
    foo: str

def node_1(state: ParentState):
    return {"foo": "hi! " + state["foo"]}

builder = StateGraph(ParentState)
builder.add_node("node_1", node_1)
# Note that we are adding the compiled subgraph as a node
builder.add_node("node_2", subgraph)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
graph = builder.compile()
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

// 1. Define subgraph
const SubgraphState = z.object({
  foo: z.string(), // note that this key is shared with the parent graph state
  bar: z.string(),
});

const subgraphBuilder = new StateGraph(SubgraphState)
  .addNode("subgraphNode1", (state) => ({ bar: "bar" }))
  .addNode("subgraphNode2", (state) => ({
    // note that this node is using a state key ('bar') that is only available in the subgraph
    // and is sending update on the shared state key ('foo')
    foo: state.foo + state.bar,
  }))
  .addEdge(START, "subgraphNode1")
  .addEdge("subgraphNode1", "subgraphNode2");
const subgraph = subgraphBuilder.compile();

// 2. Define parent graph
const ParentState = z.object({
  foo: z.string(),
});

const builder = new StateGraph(ParentState)
  .addNode("node1", (state) => ({ foo: "hi! " + state.foo }))
  // Note that we are adding the compiled subgraph as a node
  .addNode("node2", subgraph)
  .addEdge(START, "node1")
  .addEdge("node1", "node2");

const graph = builder.compile();
```
</details>

现在，让我们调用图来查看输出。

```python
for chunk in graph.stream({"foo": "foo"}):
    print(chunk)
```

```python
{'node_1': {'foo': 'hi! foo'}}
{'node_2': {'foo': 'hi! foobar'}}
```

请注意，`subgraph_node_2` 确实更新了父图状态的 `foo` 键。

## 持久化 (Persistence)

如果父图有检查点（checkpointer），子图将自动使用它。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

# Note that we're passing the checkpointer to the parent graph only
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}

for chunk in graph.stream({"foo": "foo"}, config=config):
    print(chunk)

print("-" * 10)

# The subgraph state is persisted!
snapshot = graph.get_state(config)
print(snapshot.values)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

// Note that we're passing the checkpointer to the parent graph only
const graph = builder.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };

for await (const chunk of await graph.stream({ foo: "foo" }, config)) {
  console.log(chunk);
}

console.log("-".repeat(10));

// The subgraph state is persisted!
const snapshot = await graph.getState(config);
console.log(snapshot.values);
```
</details>

```python
{'node_1': {'foo': 'hi! foo'}}
{'node_2': {'foo': 'hi! foobar'}}
----------
{'foo': 'hi! foobar'}
```

## 流式传输子图事件 (Stream subgraph events)

在流式传输父图时，您可能还希望看到子图中发生的事件。您可以通过在调用 `.stream` 时设置 `subgraphs=True` 来实现。

<details>
<summary>查看 Python 代码</summary>

```python
# Re-using the graph defined above

for chunk in graph.stream(
    {"foo": "foo"}, 
    stream_mode="updates", 
    subgraphs=True, # (1)!
):
    print(chunk)
```

1. 将 `subgraphs=True` 设置为流式传输子图的输出。
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
// Re-using the graph defined above

for await (const chunk of await graph.stream(
  { foo: "foo" },
  {
    streamMode: "updates",
    subgraphs: true, // (1)!
  }
)) {
  console.log(chunk);
}
```

1. 将 `subgraphs: true` 设置为流式传输子图的输出。
</details>

```python
((), {'node_1': {'foo': 'hi! foo'}})
(('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_1': {'bar': 'bar'}})
(('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
((), {'node_2': {'foo': 'hi! foobar'}})
```
