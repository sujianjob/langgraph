# 无效的图节点返回值 (INVALID_GRAPH_NODE_RETURN_VALUE)

:::python
LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 接收到来自节点的非字典 (non-dict) 返回类型。示例如下：

```python
class State(TypedDict):
    some_key: str

def bad_node(state: State):
    # Should return a dict with a value for "some_key", not a list
    return ["whoops"]

builder = StateGraph(State)
builder.add_node(bad_node)
...

graph = builder.compile()
```

调用上述图将导致如下错误：

```python
graph.invoke({ "some_key": "someval" });
```

```
InvalidUpdateError: Expected dict, got ['whoops']
For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

图中的节点必须返回包含一个或多个在状态中定义的键的字典。
:::

:::js
LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 接收到来自节点的非对象 (non-object) 返回类型。示例如下：

```typescript
import { z } from "zod";
import { StateGraph } from "@langchain/langgraph";

const State = z.object({
  someKey: z.string(),
});

const badNode = (state: z.infer<typeof State>) => {
  // Should return an object with a value for "someKey", not an array
  return ["whoops"];
};

const builder = new StateGraph(State).addNode("badNode", badNode);
// ...

const graph = builder.compile();
```

调用上述图将导致如下错误：

```typescript
await graph.invoke({ someKey: "someval" });
```

```
InvalidUpdateError: Expected object, got ['whoops']
For troubleshooting, visit: https://langchain-ai.github.io/langgraphjs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

图中的节点必须返回包含一个或多个在状态中定义的键的对象。
:::

## 故障排除 (Troubleshooting)

以下方法可能有助于解决此错误：

:::python

- 如果您的节点中有复杂的逻辑，请确保所有代码路径都返回适合您定义状态的字典。
  :::

:::js

- 如果您的节点中有复杂的逻辑，请确保所有代码路径都返回适合您定义状态的对象。
  :::
