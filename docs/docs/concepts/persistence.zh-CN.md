---
search:
  boost: 2
---

# 持久化 (Persistence)

LangGraph 具有内置的持久层，通过检查点 (checkpointers) 实现。当您使用检查点编译图时，检查点会在每个超步 (super-step) 保存图状态的“快照” (checkpoint)。这些检查点保存到“线程” (thread) 中，可以在图执行后访问。因为“线程”允许在执行后访问图的状态，所以包括人机回圈 (human-in-the-loop)、记忆、时间旅行和容错在内的几种强大功能都是可能的。下面，我们将更详细地讨论这些概念中的每一个。

![Checkpoints](img/persistence/checkpoints.jpg)

!!! info "LangGraph API handles checkpointing automatically"

    使用 LangGraph API 时，您无需手动实现或配置检查点。API 会在后台为您处理所有持久化基础设施。

## 线程 (Threads)

线程是分配给检查点保存的每个检查点的唯一 ID 或线程标识符。它包含一系列 [运行](./assistants.md#execution) 的累积状态。执行运行时，助手的底层图的 [状态](../concepts/low_level.md#state) 将持久保存到线程中。

当使用检查点调用图时，您 **必须** 指定 `thread_id` 作为配置的 `configurable` 部分的一部分：

:::python

```python
{"configurable": {"thread_id": "1"}}
```

:::

:::js

```typescript
{
  configurable: {
    thread_id: "1";
  }
}
```

:::

可以检索线程的当前和历史状态。要持久保存状态，必须在执行运行之前创建线程。LangGraph Platform API 提供了多个端点用于创建和管理线程和线程状态。有关更多详细信息，请参阅 [API 参考](../cloud/reference/api/api_ref.html#tag/threads)。

## 检查点 (Checkpoints)

线程在特定时间点的状态称为检查点。检查点是在每个超步保存的图状态的快照，由具有以下关键属性的 `StateSnapshot` 对象表示：

- `config`: 与此检查点关联的配置。
- `metadata`: 与此检查点关联的元数据。
- `values`: 此时间点状态通道的值。
- `next`: 图中接下来要执行的节点名称的元组。
- `tasks`: 包含有关接下来要执行的任务信息的 `PregelTask` 对象元组。如果之前尝试过该步骤，它将包含错误信息。如果图从节点内 [动态](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt) 中断，任务将包含与中断关联的其他数据。

检查点被持久化，可用于在以后恢复线程的状态。

让我们看看当如下调用简单图时会保存哪些检查点：

:::python

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::python

运行图后，我们希望看到正好 4 个检查点：

- 空检查点，`START` 作为下一个要执行的节点
- 包含用户输入 `{'foo': '', 'bar': []}` 的检查点，`node_a` 作为下一个要执行的节点
- 包含 `node_a` 输出 `{'foo': 'a', 'bar': ['a']}` 的检查点，`node_b` 作为下一个要执行的节点
- 包含 `node_b` 输出 `{'foo': 'b', 'bar': ['a', 'b']}` 的检查点，没有下一个要执行的节点

请注意，我们的 `bar` 通道值包含来自两个节点的输出，因为我们有 `bar` 通道的缩减器 (reducer)。

:::

:::js

运行图后，我们希望看到正好 4 个检查点：

- 空检查点，`START` 作为下一个要执行的节点
- 包含用户输入 `{'foo': '', 'bar': []}` 的检查点，`nodeA` 作为下一个要执行的节点
- 包含 `nodeA` 输出 `{'foo': 'a', 'bar': ['a']}` 的检查点，`nodeB` 作为下一个要执行的节点
- 包含 `nodeB` 输出 `{'foo': 'b', 'bar': ['a', 'b']}` 的检查点，没有下一个要执行的节点

请注意，我们的 `bar` 通道值包含来自两个节点的输出，因为我们有 `bar` 通道的缩减器 (reducer)。
:::

### 获取状态 (Get state)

:::python
与保存的图状态交互时，您 **必须** 指定 [线程标识符](#threads)。您可以通过调用 `graph.get_state(config)` 查看图的 *最新* 状态。这将返回一个 `StateSnapshot` 对象，该对象对应于与配置中提供的线程 ID 关联的最新检查点，或者如果提供了检查点 ID，则对应于与该线程的检查点 ID 关联的检查点。

```python
# get the latest state snapshot
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# get a state snapshot for a specific checkpoint_id
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)
```

:::

:::js
与保存的图状态交互时，您 **必须** 指定 [线程标识符](#threads)。您可以通过调用 `graph.getState(config)` 查看图的 *最新* 状态。这将返回一个 `StateSnapshot` 对象，该对象对应于与配置中提供的线程 ID 关联的最新检查点，或者如果提供了检查点 ID，则对应于与该线程的检查点 ID 关联的检查点。

```typescript
// get the latest state snapshot
const config = { configurable: { thread_id: "1" } };
await graph.getState(config);

// get a state snapshot for a specific checkpoint_id
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "1ef663ba-28fe-6528-8002-5a559208592c",
  },
};
await graph.getState(config);
```

:::

:::python
在我们的示例中，`get_state` 的输出将如下所示：

```
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

:::

:::js
在我们的示例中，`getState` 的输出将如下所示：

```
StateSnapshot {
  values: { foo: 'b', bar: ['a', 'b'] },
  next: [],
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
    }
  },
  metadata: {
    source: 'loop',
    writes: { nodeB: { foo: 'b', bar: ['b'] } },
    step: 2
  },
  createdAt: '2024-08-29T19:19:38.821749+00:00',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
    }
  },
  tasks: []
}
```

:::

### 获取状态历史 (Get state history)

:::python
您可以通过调用 `graph.get_state_history(config)` 获取给定线程的完整图执行历史记录。这将返回与配置中提供的线程 ID 关联的 `StateSnapshot` 对象列表。重要的是，检查点将按时间顺序排列，最新的检查点 / `StateSnapshot` 位于列表的第一个。

```python
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

:::

:::js
您可以通过调用 `graph.getStateHistory(config)` 获取给定线程的完整图执行历史记录。这将返回与配置中提供的线程 ID 关联的 `StateSnapshot` 对象列表。重要的是，检查点将按时间顺序排列，最新的检查点 / `StateSnapshot` 位于列表的第一个。

```typescript
const config = { configurable: { thread_id: "1" } };
for await (const state of graph.getStateHistory(config)) {
  console.log(state);
}
```

:::

:::python
在我们的示例中，`get_state_history` 的输出将如下所示：

```
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

:::

:::js
在我们的示例中，`getStateHistory` 的输出将如下所示：

```
[
  StateSnapshot {
    values: { foo: 'b', bar: ['a', 'b'] },
    next: [],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeB: { foo: 'b', bar: ['b'] } },
      step: 2
    },
    createdAt: '2024-08-29T19:19:38.821749+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    tasks: []
  },
  StateSnapshot {
    values: { foo: 'a', bar: ['a'] },
    next: ['nodeB'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeA: { foo: 'a', bar: ['a'] } },
      step: 1
    },
    createdAt: '2024-08-29T19:19:38.819946+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    tasks: [
      PregelTask {
        id: '6fb7314f-f114-5413-a1f3-d37dfe98ff44',
        name: 'nodeB',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { foo: '', bar: [] },
    next: ['node_a'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    metadata: {
      source: 'loop',
      writes: null,
      step: 0
    },
    createdAt: '2024-08-29T19:19:38.817813+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    tasks: [
      PregelTask {
        id: 'f1b14528-5ee5-579c-949b-23ef9bfbed58',
        name: 'node_a',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { bar: [] },
    next: ['__start__'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    metadata: {
      source: 'input',
      writes: { foo: '' },
      step: -1
    },
    createdAt: '2024-08-29T19:19:38.816205+00:00',
    parentConfig: null,
    tasks: [
      PregelTask {
        id: '6d27aa2e-d72b-5504-a36f-8620e54a76dd',
        name: '__start__',
        error: null,
        interrupts: []
      }
    ]
  }
]
```

:::

![State](img/persistence/get_state.jpg)

### 重放 (Replay)

也可以重放之前的图执行。如果我们使用 `thread_id` 和 `checkpoint_id` 来 `invoke` 一个图，那么我们将 *重放* 对应于 `checkpoint_id` 的检查点 *之前* 执行的步骤，并且只执行检查点 *之后* 的步骤。

- `thread_id` 是线程的 ID。
- `checkpoint_id` 是引用线程内特定检查点的标识符。

调用图时，您必须将这些作为配置的 `configurable` 部分传递：

:::python

```python
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}
graph.invoke(None, config=config)
```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "0c62ca34-ac19-445d-bbb0-5b4984975b2a",
  },
};
await graph.invoke(null, config);
```

:::

重要的是，LangGraph 知道特定步骤以前是否执行过。如果是，LangGraph 只是简单地 *重放* 图中的那个特定步骤而不重新执行该步骤，但这仅适用于提供的 `checkpoint_id` *之前* 的步骤。`checkpoint_id` *之后* 的所有步骤都将被执行（即，一个新的分叉），即使它们以前已经执行过。请参阅 [有关时间旅行的操作指南以了解有关重放的更多信息](../how-tos/human_in_the_loop/time-travel.md)。

![Replay](img/persistence/re_play.png)

### 更新状态 (Update state)

:::python

除了从特定的 `checkpoints` 重放图之外，我们还可以 *编辑* 图状态。我们使用 `graph.update_state()` 来做到这一点。此方法接受三个不同的参数：

:::

:::js

除了从特定的 `checkpoints` 重放图之外，我们还可以 *编辑* 图状态。我们使用 `graph.updateState()` 来做到这一点。此方法接受三个不同的参数：

:::

#### `config`

配置应包含指定要更新哪个线程的 `thread_id`。如果仅传递 `thread_id`，我们将更新（或分叉）当前状态。或者，如果我们包含 `checkpoint_id` 字段，那么我们将分叉该选定的检查点。

#### `values`

这些是用于更新状态的值。请注意，此更新的处理方式与处理来自节点的任何更新完全相同。这意味着这些值将被传递给 [reducer](./low_level.md#reducers) 函数，如果它们为图状态中的某些通道定义了的话。这意味着 `update_state` **不会** 自动覆盖每个通道的通道值，而仅覆盖没有 reducer 的通道。让我们来看一个例子。

假设您使用以下模式定义了图的状态（参见上面的完整示例）：

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

:::

:::js

```typescript
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});
```

:::

现在假设图的当前状态是

:::python

```
{"foo": 1, "bar": ["a"]}
```

:::

:::js

```typescript
{ foo: 1, bar: ["a"] }
```

:::

如果您如下更新状态：

:::python

```python
graph.update_state(config, {"foo": 2, "bar": ["b"]})
```

:::

:::js

```typescript
await graph.updateState(config, { foo: 2, bar: ["b"] });
```

:::

那么图的新状态将是：

:::python

```
{"foo": 2, "bar": ["a", "b"]}
```

`foo` 键（通道）被完全更改（因为该通道没有指定 reducer，所以 `update_state` 覆盖了它）。但是，`bar` 键指定了一个 reducer，因此它将 `"b"` 附加到 `bar` 的状态。
:::

:::js

```typescript
{ foo: 2, bar: ["a", "b"] }
```

`foo` 键（通道）被完全更改（因为该通道没有指定 reducer，所以 `updateState` 覆盖了它）。但是，`bar` 键指定了一个 reducer，因此它将 `"b"` 附加到 `bar` 的状态。
:::

#### `as_node`

:::python
调用 `update_state` 时，您可以选择指定的最后一件事是 `as_node`。如果您提供了它，更新将像来自节点 `as_node` 一样应用。如果未提供 `as_node`，且没有歧义，它将被设置为更新状态的最后一个节点。这很重要的原因是，接下来的执行步骤取决于最后一个给出更新的节点，因此这可用于控制接下来执行哪个节点。请参阅 [有关时间旅行的操作指南以了解有关分叉状态的更多信息](../how-tos/human_in_the_loop/time-travel.md)。
:::

:::js
调用 `updateState` 时，您可以选择指定的最后一件事是 `asNode`。如果您提供了它，更新将像来自节点 `asNode` 一样应用。如果未提供 `asNode`，且没有歧义，它将被设置为更新状态的最后一个节点。这很重要的原因是，接下来的执行步骤取决于最后一个给出更新的节点，因此这可用于控制接下来执行哪个节点。请参阅 [有关时间旅行的操作指南以了解有关分叉状态的更多信息](../how-tos/human_in_the_loop/time-travel.md)。
:::

![Update](img/persistence/checkpoints_full_story.jpg)

## 记忆存储 (Memory Store)

![Model of shared state](img/persistence/shared_state.png)

[状态模式 (state schema)](low_level.md#schema) 指定了一组在图执行时填充的键。如上所述，检查点可以在每个图步骤将状态写入线程，从而实现状态持久化。

但是，如果我们想 *跨线程* 保留一些信息怎么办？考虑一个聊天机器人的情况，我们希望在与该用户的 *所有* 聊天对话（即线程）中保留有关用户的特定信息！

仅使用检查点，我们无法跨线程共享信息。这激发了对 [`Store`](../reference/store.md#langgraph.store.base.BaseStore) 接口的需求。作为一个例子，我们可以定义一个 `InMemoryStore` 来跨线程存储有关用户的信息。我们只需像以前一样使用检查点编译我们的图，并使用我们的新 `in_memory_store` 变量。

!!! info "LangGraph API handles stores automatically"

    使用 LangGraph API 时，您无需手动实现或配置存储。API 会在后台为您处理所有存储基础设施。

### 基本用法 (Basic Usage)

首先，让我们在不使用 LangGraph 的情况下单独展示这一点。

:::python

```python
from langgraph.store.memory import InMemoryStore
in_memory_store = InMemoryStore()
```

:::

:::js

```typescript
import { MemoryStore } from "@langchain/langgraph";

const memoryStore = new MemoryStore();
```

:::

记忆由一个 `tuple` 命名空间，在此特定示例中将是 `(<user_id>, "memories")`。命名空间可以是任何长度并表示任何内容，不必是特定于用户的。

:::python

```python
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

:::

:::js

```typescript
const userId = "1";
const namespaceForMemory = [userId, "memories"];
```

:::

我们使用 `store.put` 方法将记忆保存到我们在存储中的命名空间。当我们这样做时，我们会指定如上定义的命名空间，以及记忆的键值对：键仅仅是记忆的唯一标识符 (`memory_id`)，值（一个字典）是记忆本身。

:::python

```python
memory_id = str(uuid.uuid4())
memory = {"food_preference" : "I like pizza"}
in_memory_store.put(namespace_for_memory, memory_id, memory)
```

:::

:::js

```typescript
import { v4 as uuidv4 } from "uuid";

const memoryId = uuidv4();
const memory = { food_preference: "I like pizza" };
await memoryStore.put(namespaceForMemory, memoryId, memory);
```

:::

我们可以使用 `store.search` 方法读出我们命名空间中的记忆，该方法将作为列表返回给定用户的所有记忆。最近的记忆在列表的最后。

:::python

```python
memories = in_memory_store.search(namespace_for_memory)
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

每种记忆类型都是具有某些属性的 Python 类 ([`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item))。我们可以像上面一样通过 `.dict` 转换将其作为字典访问。

它具有的属性是：

- `value`: 此记忆的值（本身是一个字典）
- `key`: 此命名空间中此记忆的唯一键
- `namespace`: 字符串列表，此记忆类型的命名空间
- `created_at`: 此记忆创建的时间戳
- `updated_at`: 此记忆更新的时间戳

:::

:::js

```typescript
const memories = await memoryStore.search(namespaceForMemory);
memories[memories.length - 1];

// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

它具有的属性是：

- `value`: 此记忆的值
- `key`: 此命名空间中此记忆的唯一键
- `namespace`: 字符串列表，此记忆类型的命名空间
- `createdAt`: 此记忆创建的时间戳
- `updatedAt`: 此记忆更新的时间戳

:::

### 语义搜索 (Semantic Search)

除了简单的检索，存储还支持语义搜索，允许您根据含义而不是精确匹配来查找记忆。要启用此功能，请使用嵌入模型配置存储：

:::python

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # Embedding provider
        "dims": 1536,                              # Embedding dimensions
        "fields": ["food_preference", "$"]              # Fields to embed
    }
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";

const store = new InMemoryStore({
  index: {
    embeddings: new OpenAIEmbeddings({ model: "text-embedding-3-small" }),
    dims: 1536,
    fields: ["food_preference", "$"], // Fields to embed
  },
});
```

:::

现在搜索时，您可以使用自然语言查询来查找相关记忆：

:::python

```python
# Find memories about food preferences
# (This can be done after putting memories into the store)
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # Return top 3 matches
)
```

:::

:::js

```typescript
// Find memories about food preferences
// (This can be done after putting memories into the store)
const memories = await store.search(namespaceForMemory, {
  query: "What does the user like to eat?",
  limit: 3, // Return top 3 matches
});
```

:::

您可以通过配置 `fields` 参数或在存储记忆时指定 `index` 参数来控制嵌入记忆的哪些部分：

:::python

```python
# Store with specific fields to embed
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # Only embed "food_preferences" field
)

# Store without embedding (still retrievable, but not searchable)
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

:::

:::js

```typescript
// Store with specific fields to embed
await store.put(
  namespaceForMemory,
  uuidv4(),
  {
    food_preference: "I love Italian cuisine",
    context: "Discussing dinner plans",
  },
  { index: ["food_preference"] } // Only embed "food_preferences" field
);

// Store without embedding (still retrievable, but not searchable)
await store.put(
  namespaceForMemory,
  uuidv4(),
  { system_info: "Last updated: 2024-01-01" },
  { index: false }
);
```

:::

### 在 LangGraph 中使用 (Using in LangGraph)

:::python
做好这一切后，我们在 LangGraph 中使用 `in_memory_store`。`in_memory_store` 与检查点协同工作：如上所述，检查点将状态保存到线程，而 `in_memory_store` 允许我们存储任意信息以便 *跨* 线程访问。我们使用检查点和 `in_memory_store` 编译图，如下所示。

```python
from langgraph.checkpoint.memory import InMemorySaver

# We need this because we want to enable threads (conversations)
checkpointer = InMemorySaver()

# ... Define the graph ...

# Compile the graph with the checkpointer and store
graph = graph.compile(checkpointer=checkpointer, store=in_memory_store)
```

:::

:::js
做好这一切后，我们在 LangGraph 中使用 `memoryStore`。`memoryStore` 与检查点协同工作：如上所述，检查点将状态保存到线程，而 `memoryStore` 允许我们存储任意信息以便 *跨* 线程访问。我们使用检查点和 `memoryStore` 编译图，如下所示。

```typescript
import { MemorySaver } from "@langchain/langgraph";

// We need this because we want to enable threads (conversations)
const checkpointer = new MemorySaver();

// ... Define the graph ...

// Compile the graph with the checkpointer and store
const graph = workflow.compile({ checkpointer, store: memoryStore });
```

:::

我们像以前一样使用 `thread_id` 调用图，并且还使用 `user_id`，正如我们上面所示，我们将使用它来将我们的记忆命名空间到该特定用户。

:::python

```python
# Invoke the graph
user_id = "1"
config = {"configurable": {"thread_id": "1", "user_id": user_id}}

# First let's just say hi to the AI
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// Invoke the graph
const userId = "1";
const config = { configurable: { thread_id: "1", user_id: userId } };

// First let's just say hi to the AI
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

:::python
我们可以通过传递 `store: BaseStore` 和 `config: RunnableConfig` 作为节点参数，在 *任何节点* 中访问 `in_memory_store` 和 `user_id`。以下是我们如何在节点中使用语义搜索来查找相关记忆：

```python
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):

    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # ... Analyze conversation and create a new memory

    # Create a new memory ID
    memory_id = str(uuid.uuid4())

    # We create a new memory
    store.put(namespace, memory_id, {"memory": memory})

```

:::

:::js
我们可以通过访问 `config` 和 `store` 作为节点参数，在 *任何节点* 中访问 `memoryStore` 和 `user_id`。以下是我们如何在节点中使用语义搜索来查找相关记忆：

```typescript
import {
  LangGraphRunnableConfig,
  BaseStore,
  MessagesZodState,
} from "@langchain/langgraph";
import { z } from "zod";

const updateMemory = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // Get the user id from the config
  const userId = config.configurable?.user_id;

  // Namespace the memory
  const namespace = [userId, "memories"];

  // ... Analyze conversation and create a new memory

  // Create a new memory ID
  const memoryId = uuidv4();

  // We create a new memory
  await store.put(namespace, memoryId, { memory });
};
```

:::

如上所示，我们还可以访问任何节点中的存储，并使用 `store.search` 方法获取记忆。回想一下，记忆作为对象列表返回，可以转换为字典。

:::python

```python
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

:::

:::js

```typescript
memories[memories.length - 1];
// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

:::

我们可以访问记忆并在我们的模型调用中使用它们。

:::python

```python
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # Search based on the most recent message
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... Use memories in the model call
```

:::

:::js

```typescript
const callModel = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // Get the user id from the config
  const userId = config.configurable?.user_id;

  // Namespace the memory
  const namespace = [userId, "memories"];

  // Search based on the most recent message
  const memories = await store.search(namespace, {
    query: state.messages[state.messages.length - 1].content,
    limit: 3,
  });
  const info = memories.map((d) => d.value.memory).join("\n");

  // ... Use memories in the model call
};
```

:::

如果我们创建一个新线程，只要 `user_id` 相同，我们仍然可以访问相同的记忆。

:::python

```python
# Invoke the graph
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# Let's say hi again
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// Invoke the graph
const config = { configurable: { thread_id: "2", user_id: "1" } };

// Let's say hi again
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi, tell me about my memories" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

当我们使用 LangGraph Platform 时，无论是在本地（例如，在 LangGraph Studio 中）还是使用 LangGraph Platform，基础存储默认可用，无需在图编译期间指定。但是，要启用语义搜索，您 **确实** 需要在 `langgraph.json` 文件中配置索引设置。例如：

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

有关更多详细信息和配置选项，请参阅 [部署指南](../cloud/deployment/semantic_search.md)。

## 检查点库 (Checkpointer libraries)

在后台，检查点由符合 @[BaseCheckpointSaver] 接口的检查点对象提供支持。LangGraph 提供了多种检查点实现，全部通过独立的、可安装的库实现：

:::python

- `langgraph-checkpoint`: 检查点保存器 (@[BaseCheckpointSaver]) 和序列化/反序列化接口 (@[SerializerProtocol][SerializerProtocol]) 的基本接口。包括用于实验的内存检查点实现 (@[InMemorySaver][InMemorySaver])。LangGraph 附带 `langgraph-checkpoint`。
- `langgraph-checkpoint-sqlite`: 使用 SQLite 数据库的 LangGraph 检查点实现 (@[SqliteSaver][SqliteSaver] / @[AsyncSqliteSaver])。非常适合实验和本地工作流。需要单独安装。
- `langgraph-checkpoint-postgres`: 使用 Postgres 数据库的高级检查点 (@[PostgresSaver][PostgresSaver] / @[AsyncPostgresSaver])，在 LangGraph Platform 中使用。非常适合生产环境。需要单独安装。

:::

:::js

- `@langchain/langgraph-checkpoint`: 检查点保存器 (@[BaseCheckpointSaver][BaseCheckpointSaver]) 和序列化/反序列化接口 (@[SerializerProtocol][SerializerProtocol]) 的基本接口。包括用于实验的内存检查点实现 (@[MemorySaver])。LangGraph 附带 `@langchain/langgraph-checkpoint`。
- `@langchain/langgraph-checkpoint-sqlite`: 使用 SQLite 数据库的 LangGraph 检查点实现 (@[SqliteSaver])。非常适合实验和本地工作流。需要单独安装。
- `@langchain/langgraph-checkpoint-postgres`: 使用 Postgres 数据库的高级检查点 (@[PostgresSaver])，在 LangGraph Platform 中使用。非常适合生产环境。需要单独安装。

:::

### 检查点接口 (Checkpointer interface)

:::python
每个检查点都符合 @[BaseCheckpointSaver] 接口并实现以下方法：

- `.put` - 存储带有其配置和元数据的检查点。
- `.put_writes` - 存储链接到检查点的中间写入（即 [挂起写入](#pending-writes)）。
- `.get_tuple` - 使用给定配置（`thread_id` 和 `checkpoint_id`）获取检查点元组。用于在 `graph.get_state()` 中填充 `StateSnapshot`。
- `.list` - 列出与给定配置和过滤条件匹配的检查点。用于在 `graph.get_state_history()` 中填充状态历史记录。

如果检查点用于异步图执行（即通过 `.ainvoke`、`.astream`、`.abatch` 执行图），则将使用上述方法的异步版本（`.aput`、`.aput_writes`、`.aget_tuple`、`.alist`）。

!!! note

    要异步运行您的图，您可以使用 `InMemorySaver`，或 Sqlite/Postgres 检查点的异步版本 -- `AsyncSqliteSaver` / `AsyncPostgresSaver` 检查点。

:::

:::js
每个检查点都符合 @[BaseCheckpointSaver][BaseCheckpointSaver] 接口并实现以下方法：

- `.put` - 存储带有其配置和元数据的检查点。
- `.putWrites` - 存储链接到检查点的中间写入（即 [挂起写入](#pending-writes)）。
- `.getTuple` - 使用给定配置（`thread_id` 和 `checkpoint_id`）获取检查点元组。用于在 `graph.getState()` 中填充 `StateSnapshot`。
- `.list` - 列出与给定配置和过滤条件匹配的检查点。用于在 `graph.getStateHistory()` 中填充状态历史记录。
  :::

### 序列化器 (Serializer)

当检查点保存图状态时，它们需要序列化状态中的通道值。这是使用序列化器对象完成的。

:::python
`langgraph_checkpoint` 定义了用于实现序列化器的 @[protocol][SerializerProtocol]，并提供了一个默认实现 (@[JsonPlusSerializer][JsonPlusSerializer])，它可以处理各种类型，包括 LangChain 和 LangGraph 原语、日期时间、枚举等。

#### 使用 `pickle` 序列化

默认序列化器 @[`JsonPlusSerializer`][JsonPlusSerializer] 在后台使用 ormsgpack 和 JSON，这并不适用于所有类型的对象。

如果您想对当前我们的 msgpack 编码器不支持的对象（例如 Pandas dataframes）回退到 pickle，
您可以使用 `JsonPlusSerializer` 的 `pickle_fallback` 参数：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... Define the graph ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### 加密

检查点可以选择加密所有持久化状态。要启用此功能，请将 @[`EncryptedSerializer`][EncryptedSerializer] 的实例传递给任何 `BaseCheckpointSaver` 实现的 `serde` 参数。创建加密序列化器的最简单方法是通过 @[`from_pycryptodome_aes`][from_pycryptodome_aes]，它从 `LANGGRAPH_AES_KEY` 环境变量读取 AES 密钥（或接受 `key` 参数）：

```python
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # reads LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

在 LangGraph Platform 上运行时，只要存在 `LANGGRAPH_AES_KEY`，就会自动启用加密，因此您只需提供环境变量。可以通过实现 @[`CipherProtocol`][CipherProtocol] 并将其提供给 `EncryptedSerializer` 来使用其他加密方案。
:::

:::js
`@langchain/langgraph-checkpoint` 定义了用于实现序列化器的协议，并提供了一个默认实现，它可以处理各种类型，包括 LangChain 和 LangGraph 原语、日期时间、枚举等。
:::

## 功能 (Capabilities)

### 人机回圈 (Human-in-the-loop)

首先，检查点通过允许人工检查、中断和批准图步骤来促进 [人机回圈工作流](agentic_concepts.md#human-in-the-loop) 工作流。这些工作流需要检查点，因为人工必须能够随时查看图的状态，并且图必须能够在人工对状态进行任何更新后恢复执行。有关示例，请参阅 [操作指南](../how-tos/human_in_the_loop/add-human-in-the-loop.md)。

### 记忆 (Memory)

其次，检查点允许在交互之间进行 ["记忆"](../concepts/memory.md)。在重复的人工交互（如对话）的情况下，任何后续消息都可以发送到该线程，该线程将保留其对先前消息的记忆。有关如何使用检查点添加和管理对话记忆的信息，请参阅 [添加记忆](../how-tos/memory/add-memory.md)。

### 时间旅行 (Time Travel)

第三，检查点允许 ["时间旅行"](time-travel.md)，允许用户重放以前的图执行以查看和/或调试特定的图步骤。此外，检查点使得可以在任意检查点分叉图状态以探索替代轨迹成为可能。

### 容错 (Fault-tolerance)

最后，检查点还提供容错和错误恢复：如果一个或多个节点在给定的超步失败，您可以从上一个成功的步骤重新启动您的图。此外，当图节点在给定的超步中途执行失败时，LangGraph 会存储来自在该超步成功完成的任何其他节点的挂起检查点写入，以便每当我们从该超步恢复图执行时，我们不会重新运行成功的节点。

#### 挂起写入 (Pending writes)

此外，当图节点在给定的超步中途执行失败时，LangGraph 会存储来自在该超步成功完成的任何其他节点的挂起检查点写入，以便每当我们从该超步恢复图执行时，我们不会重新运行成功的节点。
