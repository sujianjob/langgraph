---
search:
  boost: 2
---

# LangGraph 运行时 (LangGraph runtime)

:::python
@[Pregel] 实现了 LangGraph 的运行时，管理 LangGraph 应用程序的执行。

编译 @[StateGraph][StateGraph] 或创建 @[entrypoint][entrypoint] 会生成一个 @[Pregel] 实例，该实例可以使用输入调用。
:::

:::js
@[Pregel] 实现了 LangGraph 的运行时，管理 LangGraph 应用程序的执行。

编译 @[StateGraph][StateGraph] 或创建 @[entrypoint][entrypoint] 会生成一个 @[Pregel] 实例，该实例可以使用输入调用。
:::

本指南从高层次解释了运行时，并提供了直接使用 Pregel 实现应用程序的说明。

:::python

> **Note:** @[Pregel] 运行时以 [Google 的 Pregel 算法](https://research.google/pubs/pub37252/) 命名，该算法描述了一种使用图进行大规模并行计算的高效方法。

:::

:::js

> **Note:** @[Pregel] 运行时以 [Google 的 Pregel 算法](https://research.google/pubs/pub37252/) 命名，该算法描述了一种使用图进行大规模并行计算的高效方法。

:::

## 概述 (Overview)

在 LangGraph 中，Pregel 将 [**actors**](https://en.wikipedia.org/wiki/Actor_model) 和 **channels** 组合成一个应用程序。**Actors** 从通道读取数据并将数据写入通道。Pregel 遵循 **Pregel 算法**/**批量同步并行 (Bulk Synchronous Parallel)** 模型，将应用程序的执行组织成多个步骤。

每个步骤包含三个阶段：

- **计划 (Plan)**: 确定在此步骤中要执行哪些 **actor**。例如，在第一步中，选择订阅特殊 **input** 通道的 **actor**；在后续步骤中，选择订阅在上一步中更新的通道的 **actor**。
- **执行 (Execution)**: 并行执行所有选定的 **actor**，直到全部完成、一个失败或达到超时。在此阶段，通道更新对 actor 不可见，直到下一步。
- **更新 (Update)**: 使用 **actor** 在此步骤中写入的值更新通道。

重复执行，直到没有选择要执行的 **actor**，或者达到最大步骤数。

## Actors

**actor** 是一个 `PregelNode`。它订阅通道，从通道读取数据，并向通道写入数据。它可以被认为是 Pregel 算法中的 **actor**。`PregelNode` 实现了 LangChain 的 Runnable 接口。

## 通道 (Channels)

通道用于在 actor (PregelNodes) 之间进行通信。每个通道都有一个值类型、一个更新类型和一个更新函数——该函数接受一系列更新并修改存储的值。通道可用于将数据从一个链发送到另一个链，或在未来的步骤中将数据从一个链发送到其自身。LangGraph 提供了许多内置通道：

:::python

- @[LastValue][LastValue]: 默认通道，存储发送到通道的最后一个值，对于输入和输出值，或将数据从一步发送到下一步很有用。
- @[Topic][Topic]: 一个可配置的 PubSub 主题，对于在 **actor** 之间发送多个值或累积输出很有用。可以配置为对值进行重复数据删除或在多个步骤的过程中累积值。
- @[BinaryOperatorAggregate][BinaryOperatorAggregate]: 存储一个持久值，通过将二元运算符应用于当前值和发送到通道的每个更新来更新，对于计算多个步骤的聚合很有用；例如，`total = BinaryOperatorAggregate(int, operator.add)`
  :::

:::js

- @[LastValue]: 默认通道，存储发送到通道的最后一个值，对于输入和输出值，或将数据从一步发送到下一步很有用。
- @[Topic]: 一个可配置的 PubSub 主题，对于在 **actor** 之间发送多个值或累积输出很有用。可以配置为对值进行重复数据删除或在多个步骤的过程中累积值。
- @[BinaryOperatorAggregate]: 存储一个持久值，通过将二元运算符应用于当前值和发送到通道的每个更新来更新，对于计算多个步骤的聚合很有用；例如，`total = BinaryOperatorAggregate(int, operator.add)`
  :::

## 示例 (Examples)

:::python
虽然大多数用户将通过 @[StateGraph][StateGraph] API 或 @[entrypoint][entrypoint] 装饰器与 Pregel 交互，但也可以直接与 Pregel 交互。
:::

:::js
虽然大多数用户将通过 @[StateGraph] API 或 @[entrypoint] 装饰器与 Pregel 交互，但也可以直接与 Pregel 交互。
:::

下面是几个不同的示例，让您了解 Pregel API。

=== "Single node"

    :::python
    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder
    
    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )
    
    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )
    
    app.invoke({"a": "foo"})
    ```
    
    ```con
    {'b': 'foofoo'}
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";
    
    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b");
    
    const app = new Pregel({
      nodes: { node1 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
      },
      inputChannels: ["a"],
      outputChannels: ["b"],
    });
    
    await app.invoke({ a: "foo" });
    ```
    
    ```console
    { b: 'foofoo' }
    ```
    :::

=== "Multiple nodes"

    :::python
    ```python
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder
    
    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )
    
    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )
    
    
    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )
    
    app.invoke({"a": "foo"})
    ```
    
    ```con
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```
    :::

    :::js
    ```typescript
    import { LastValue, EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";
    
    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b");
    
    const node2 = new NodeBuilder()
      .subscribeOnly("b")
      .do((x: string) => x + x)
      .writeTo("c");
    
    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new LastValue<string>(),
        c: new EphemeralValue<string>(),
      },
      inputChannels: ["a"],
      outputChannels: ["b", "c"],
    });
    
    await app.invoke({ a: "foo" });
    ```
    
    ```console
    { b: 'foofoo', c: 'foofoofoofoo' }
    ```
    :::

=== "Topic"

    :::python
    ```python
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, NodeBuilder
    
    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )
    
    node2 = (
        NodeBuilder().subscribe_to("b")
        .do(lambda x: x["b"] + x["b"])
        .write_to("c")
    )
    
    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )
    
    app.invoke({"a": "foo"})
    ```
    
    ```pycon
    {'c': ['foofoo', 'foofoofoofoo']}
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue, Topic } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";
    
    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b", "c");
    
    const node2 = new NodeBuilder()
      .subscribeTo("b")
      .do((x: { b: string }) => x.b + x.b)
      .writeTo("c");
    
    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
        c: new Topic<string>({ accumulate: true }),
      },
      inputChannels: ["a"],
      outputChannels: ["c"],
    });
    
    await app.invoke({ a: "foo" });
    ```
    
    ```console
    { c: ['foofoo', 'foofoofoofoo'] }
    ```
    :::

=== "BinaryOperatorAggregate"

    此示例演示了如何使用 BinaryOperatorAggregate 通道来实现 reducer。

    :::python
    ```python
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, NodeBuilder
    
    
    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )
    
    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )
    
    def reducer(current, update):
        if current:
            return current + " | " + update
        else:
            return update
    
    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )
    
    app.invoke({"a": "foo"})
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue, BinaryOperatorAggregate } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";
    
    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b", "c");
    
    const node2 = new NodeBuilder()
      .subscribeOnly("b")
      .do((x: string) => x + x)
      .writeTo("c");
    
    const reducer = (current: string, update: string) => {
      if (current) {
        return current + " | " + update;
      } else {
        return update;
      }
    };
    
    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
        c: new BinaryOperatorAggregate<string>({ operator: reducer }),
      },
      inputChannels: ["a"],
      outputChannels: ["c"],
    });
    
    await app.invoke({ a: "foo" });
    ```
    :::

=== "Cycle"

    :::python
    
    此示例演示了如何通过使链写入其订阅的通道来在图中引入循环。执行将继续，直到将 `None` 值写入通道。
    
    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry
    
    example_node = (
        NodeBuilder().subscribe_only("value")
        .do(lambda x: x + x if len(x) < 10 else None)
        .write_to(ChannelWriteEntry("value", skip_none=True))
    )
    
    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )
    
    app.invoke({"value": "a"})
    ```
    
    ```pycon
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```
    :::

    :::js
    
    此示例演示了如何通过使链写入其订阅的通道来在图中引入循环。执行将继续，直到将 `null` 值写入通道。
    
    ```typescript
    import { EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder, ChannelWriteEntry } from "@langchain/langgraph/pregel";
    
    const exampleNode = new NodeBuilder()
      .subscribeOnly("value")
      .do((x: string) => x.length < 10 ? x + x : null)
      .writeTo(new ChannelWriteEntry("value", { skipNone: true }));
    
    const app = new Pregel({
      nodes: { exampleNode },
      channels: {
        value: new EphemeralValue<string>(),
      },
      inputChannels: ["value"],
      outputChannels: ["value"],
    });
    
    await app.invoke({ value: "a" });
    ```
    
    ```console
    { value: 'aaaaaaaaaaaaaaaa' }
    ```
    :::

## 高级 API (High-level API)

LangGraph 提供了两个用于创建 Pregel 应用程序的高级 API：[StateGraph (Graph API)](./low_level.md) 和 [Functional API](functional_api.md)。

=== "StateGraph (Graph API)"

    :::python
    
    @[StateGraph (Graph API)][StateGraph] 是一个更高级别的抽象，它简化了 Pregel 应用程序的创建。它允许您定义节点和边的图。当您编译图时，StateGraph API 会自动为您创建 Pregel 应用程序。
    
    ```python
    from typing import TypedDict, Optional
    
    from langgraph.constants import START
    from langgraph.graph import StateGraph
    
    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]
    
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }
    
    def score_essay(essay: Essay):
        return {
            "score": 10
        }
    
    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")
    
    # Compile the graph.
    # This will return a Pregel instance.
    graph = builder.compile()
    ```
    :::
    
    :::js
    
    @[StateGraph (Graph API)][StateGraph] 是一个更高级别的抽象，它简化了 Pregel 应用程序的创建。它允许您定义节点和边的图。当您编译图时，StateGraph API 会自动为您创建 Pregel 应用程序。
    
    ```typescript
    import { START, StateGraph } from "@langchain/langgraph";
    
    interface Essay {
      topic: string;
      content?: string;
      score?: number;
    }
    
    const writeEssay = (essay: Essay) => {
      return {
        content: `Essay about ${essay.topic}`,
      };
    };
    
    const scoreEssay = (essay: Essay) => {
      return {
        score: 10
      };
    };
    
    const builder = new StateGraph<Essay>({
      channels: {
        topic: null,
        content: null,
        score: null,
      }
    })
      .addNode("writeEssay", writeEssay)
      .addNode("scoreEssay", scoreEssay)
      .addEdge(START, "writeEssay");
    
    // Compile the graph.
    // This will return a Pregel instance.
    const graph = builder.compile();
    ```
    :::
    
    编译后的 Pregel 实例将与节点和通道列表相关联。您可以通过打印来检查节点和通道。
    
    :::python
    ```python
    print(graph.nodes)
    ```
    
    您将看到类似这样的内容：
    
    ```pycon
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
    ```
    
    ```python
    print(graph.channels)
    ```
    
    您应该看到类似这样的内容：
    
    ```pycon
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```
    :::
    
    :::js
    ```typescript
    console.log(graph.nodes);
    ```
    
    您将看到类似这样的内容：
    
    ```console
    {
      __start__: PregelNode { ... },
      writeEssay: PregelNode { ... },
      scoreEssay: PregelNode { ... }
    }
    ```
    
    ```typescript
    console.log(graph.channels);
    ```
    
    您应该看到类似这样的内容：
    
    ```console
    {
      topic: LastValue { ... },
      content: LastValue { ... },
      score: LastValue { ... },
      __start__: EphemeralValue { ... },
      writeEssay: EphemeralValue { ... },
      scoreEssay: EphemeralValue { ... },
      'branch:__start__:__self__:writeEssay': EphemeralValue { ... },
      'branch:__start__:__self__:scoreEssay': EphemeralValue { ... },
      'branch:writeEssay:__self__:writeEssay': EphemeralValue { ... },
      'branch:writeEssay:__self__:scoreEssay': EphemeralValue { ... },
      'branch:scoreEssay:__self__:writeEssay': EphemeralValue { ... },
      'branch:scoreEssay:__self__:scoreEssay': EphemeralValue { ... },
      'start:writeEssay': EphemeralValue { ... }
    }
    ```
    :::

=== "Functional API"

    :::python
    
    在 [Functional API](functional_api.md) 中，您可以使用 @[`entrypoint`][entrypoint] 来创建 Pregel 应用程序。`entrypoint` 装饰器允许您定义一个接受输入并返回输出的函数。
    
    ```python
    from typing import TypedDict, Optional
    
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint
    
    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]
    
    
    checkpointer = InMemorySaver()
    
    @entrypoint(checkpointer=checkpointer)
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }
    
    print("Nodes: ")
    print(write_essay.nodes)
    print("Channels: ")
    print(write_essay.channels)
    ```
    
    ```pycon
    Nodes:
    {'write_essay': <langgraph.pregel.read.PregelNode object at 0x7d05e2f9aad0>}
    Channels:
    {'__start__': <langgraph.channels.ephemeral_value.EphemeralValue object at 0x7d05e2c906c0>, '__end__': <langgraph.channels.last_value.LastValue object at 0x7d05e2c90c40>, '__previous__': <langgraph.channels.last_value.LastValue object at 0x7d05e1007280>}
    ```
    :::
    
    :::js
    
    在 [Functional API](functional_api.md) 中，您可以使用 @[`entrypoint`][entrypoint] 来创建 Pregel 应用程序。`entrypoint` 装饰器允许您定义一个接受输入并返回输出的函数。
    
    ```typescript
    import { MemorySaver } from "@langchain/langgraph";
    import { entrypoint } from "@langchain/langgraph/func";
    
    interface Essay {
      topic: string;
      content?: string;
      score?: number;
    }
    
    const checkpointer = new MemorySaver();
    
    const writeEssay = entrypoint(
      { checkpointer, name: "writeEssay" },
      async (essay: Essay) => {
        return {
          content: `Essay about ${essay.topic}`,
        };
      }
    );
    
    console.log("Nodes: ");
    console.log(writeEssay.nodes);
    console.log("Channels: ");
    console.log(writeEssay.channels);
    ```
    
    ```console
    Nodes:
    { writeEssay: PregelNode { ... } }
    Channels:
    {
      __start__: EphemeralValue { ... },
      __end__: LastValue { ... },
      __previous__: LastValue { ... }
    }
    ```
    :::
