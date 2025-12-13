# 无效的并发图更新 (INVALID_CONCURRENT_GRAPH_UPDATE)

LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从多个节点接收到对不支持并发更新的状态属性的并发更新。

发生这种情况的一种方式是，如果您在图中使用了 [扇出 (fanout)](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 或其他并行执行，并且您定义了如下所示的图：

:::python

```python hl_lines="2"
class State(TypedDict):
    some_key: str

def node(state: State):
    return {"some_key": "some_string_value"}

def other_node(state: State):
    return {"some_key": "some_string_value"}


builder = StateGraph(State)
builder.add_node(node)
builder.add_node(other_node)
builder.add_edge(START, "node")
builder.add_edge(START, "other_node")
graph = builder.compile()
```

:::

:::js

```typescript hl_lines="2"
import { StateGraph, Annotation, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: z.string(),
});

const builder = new StateGraph(State)
  .addNode("node", (state) => {
    return { someKey: "some_string_value" };
  })
  .addNode("otherNode", (state) => {
    return { someKey: "some_string_value" };
  })
  .addEdge(START, "node")
  .addEdge(START, "otherNode");

const graph = builder.compile();
```

:::

:::python
如果上图中的节点返回 `{ "some_key": "some_string_value" }`，这将用 `"some_string_value"` 覆盖 `"some_key"` 的状态值。
但是，如果单个步骤中的多个节点（例如扇出）返回 `"some_key"` 的值，图将抛出此错误，因为如何更新内部状态存在不确定性。
:::

:::js
如果上图中的节点返回 `{ someKey: "some_string_value" }`，这将用 `"some_string_value"` 覆盖 `someKey` 的状态值。
但是，如果单个步骤中的多个节点（例如扇出）返回 `someKey` 的值，图将抛出此错误，因为如何更新内部状态存在不确定性。
:::

为了解决这个问题，您可以定义一个结合多个值的归约器 (reducer)：

:::python

```python hl_lines="5-6"
import operator
from typing import Annotated

class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    some_key: Annotated[list, operator.add]
```

:::

:::js

```typescript hl_lines="4-7"
import { withLangGraph } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (existing, update) => existing.concat(update),
    },
    default: () => [],
  }),
});
```

:::

这将允许您定义逻辑来处理从并行执行的多个节点返回的相同键。

## 故障排除 (Troubleshooting)

以下方法可能有助于解决此错误：

- 如果您的图并行执行节点，请确保您已定义带有归约器的相关状态键。
