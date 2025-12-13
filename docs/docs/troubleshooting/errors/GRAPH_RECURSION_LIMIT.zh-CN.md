# 图递归限制 (GRAPH_RECURSION_LIMIT)

您的 LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 在达到停止条件之前达到了最大步骤数。
这通常是由于如下所示的代码导致的无限循环引起的：

:::python

```python
class State(TypedDict):
    some_key: str

builder = StateGraph(State)
builder.add_node("a", ...)
builder.add_node("b", ...)
builder.add_edge("a", "b")
builder.add_edge("b", "a")
...

graph = builder.compile()
```

:::

:::js

```typescript
import { StateGraph } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: z.string(),
});

const builder = new StateGraph(State)
  .addNode("a", ...)
  .addNode("b", ...)
  .addEdge("a", "b")
  .addEdge("b", "a")
  ...

const graph = builder.compile();
```

:::

但是，复杂的图可能会自然地达到默认限制。

## 故障排除 (Troubleshooting)

- 如果您不希望图进行多次迭代，则可能存在循环。检查您的逻辑是否存在无限循环。

:::python

- 如果您有一个复杂的图，您可以在调用图时将更高的 `recursion_limit` 值传递给 `config` 对象，如下所示：

```python
graph.invoke({...}, {"recursion_limit": 100})
```

:::

:::js

- 如果您有一个复杂的图，您可以在调用图时将更高的 `recursionLimit` 值传递给 `config` 对象，如下所示：

```typescript
await graph.invoke({...}, { recursionLimit: 100 });
```

:::
