---
search:
  boost: 2
---

# 持久执行 (Durable Execution)

**持久执行** 是一种技术，在该技术中，流程或工作流会在关键时刻保存其进度，从而使其能够暂停并在以后从中断的确切位置恢复。这在需要 [人在回路 (human-in-the-loop)](./human_in_the_loop.md) 的场景中特别有用，用户可以在继续之前检查、验证或修改流程；以及在可能会遇到中断或错误（例如，LLM 调用超时）的长时间运行的任务中。通过保存已完成的工作，持久执行使流程能够恢复而无需重新处理先前的步骤——即使是在长时间延迟（例如，一周后）之后。

LangGraph 的内置 [持久性 (persistence)](./persistence.md) 层为工作流提供持久执行，确
保每个执行步骤的状态都保存到持久存储中。此功能保证即使工作流被中断——无论是由于系统故障还是为了 [人在回路](./human_in_the_loop.md) 交互——也可以从其上次记录的状态恢复。

!!! tip
    
    如果您将 LangGraph 与检查点指针 (checkpointer) 一起使用，则您已经启用了持久执行。即使在中断或故障之后，您也可以在任何时候暂停和恢复工作流。
    为了充分利用持久执行，请确保您的工作流被设计为 [确定性的和幂等的](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作包装在 [任务 (tasks)](./functional_api.md#task) 内。您可以从 [StateGraph (Graph API)](./low_level.md) 和 [Functional API](./functional_api.md) 中使用 [任务](./functional_api.md#task)。

## 要求

要在 LangGraph 中利用持久执行，您需要：

1. 通过指定一个将保存工作流进度的 [检查点指针 (checkpointer)](./persistence.md#checkpointer-libraries)，在您的工作流中启用 [持久性 (persistence)](./persistence.md)。
2. 在执行工作流时指定 [线程标识符 (thread)](./persistence.md#threads)。这将跟踪特定工作流实例的执行历史。

:::python

3. 将任何非确定性操作（例如，随机数生成）或具有副作用的操作（例如，文件写入、API 调用）包装在 @[tasks][task] 内，以确保在恢复工作流时，不会为特定运行重复这些操作，而是从持久性层检索其结果。有关更多信息，请参阅 [确定性和一致重放](#determinism-and-consistent-replay)。

:::

:::js

3. 将任何非确定性操作（例如，随机数生成）或具有副作用的操作（例如，文件写入、API 调用）包装在 @[tasks][task] 内，以确保在恢复工作流时，不会为特定运行重复这些操作，而是从持久性层检索其结果。有关更多信息，请参阅 [确定性和一致重放](#determinism-and-consistent-replay)。

:::

## 确定性和一致重放 (Determinism and Consistent Replay)

当您恢复工作流运行时，代码 **不会** 从停止执行的 **同一行代码** 恢复；相反，它将识别一个合适的 [起点](#starting-points-for-resuming-workflows)，从那里继续。这意味着工作流将从 [起点](#starting-points-for-resuming-workflows) 重放所有步骤，直到到达停止点。

因此，当您编写用于持久执行的工作流时，您必须将任何非确定性操作（例如，随机数生成）和任何具有副作用的操作（例如，文件写入、API 调用）包装在 [任务 (tasks)](./functional_api.md#task) 或 [节点 (nodes)](./low_level.md#nodes) 内。

为确保您的工作流是确定性的并且可以一致地重放，请遵循以下准则：

- **避免重复工作**：如果一个 [节点 (nodes)](./low_level.md#nodes) 包含多个具有副作用的操作（例如，日志记录、文件写入或网络调用），请将每个操作包装在单独的 **任务** 中。这确保了在恢复工作流时，不会重复操作，并且从持久性层检索其结果。
- **封装非确定性操作**：将任何可能产生非确定性结果的代码（例如，随机数生成）包装在 **任务** 或 **节点** 内。这确保了在恢复时，工作流遵循与相同结果完全相同的记录步骤序列。
- **使用幂等操作**：尽可能确保副作用（例如，API 调用、文件写入）是幂等的。这意味着如果操作在工作流失败后重试，它将具有与第一次执行时相同的效果。这对于导致数据写入的操作尤为重要。如果 **任务** 启动但未能成功完成，工作流的恢复将重新运行 **任务**，依靠记录的结果来保持一致性。使用幂等键或验证现有结果以避免意外重复，确保平滑和可预测的工作流执行。

:::python
有关要避免的陷阱的一些示例，请参阅 functional API 中的 [常见陷阱 (Common Pitfalls)](./functional_api.md#common-pitfalls) 部分，其中展示了如何使用 **任务** 构建代码以避免这些问题。相同的原则适用于 @[StateGraph (Graph API)][StateGraph]。
:::

:::js
有关要避免的陷阱的一些示例，请参阅 functional API 中的 [常见陷阱 (Common Pitfalls)](./functional_api.md#common-pitfalls) 部分，其中展示了如何使用 **任务** 构建代码以避免这些问题。相同的原则适用于 @[StateGraph (Graph API)][StateGraph]。
:::

## 持久性模式 (Durability modes)

LangGraph 支持三种持久性模式，允许您根据应用程序的要求平衡性能和数据一致性。持久性模式，从最不持久到最持久，如下所示：

- [`"exit"`](#exit)
- [`"async"`](#async)
- [`"sync"`](#sync)

较高的持久性模式会给工作流执行增加更多开销。

!!! version-added "在版本 0.6.0 中添加"

    使用 `durability` 参数代替 `checkpoint_during`（在 v0.6.0 中已弃用）进行持久性策略管理：
    
    * `durability="async"` 替换 `checkpoint_during=True`
    * `durability="exit"` 替换 `checkpoint_during=False`
    
    对于持久性策略管理，具有以下映射：

    * `checkpoint_during=True` -> `durability="async"`
    * `checkpoint_during=False` -> `durability="exit"`

### `"exit"`

仅当图执行完成（成功或出错）时才会持久保存更改。这为长时间运行的图提供了最佳性能，但意味着中间状态未保存，因此您无法从执行中途的失败中恢复或中断图执行。

### `"async"`

更改将在执行下一步骤时异步持久保存。这提供了良好的性能和持久性，但如果进程在执行期间崩溃，则存在检查点可能无法写入的小风险。

### `"sync"`

更改将在下一步骤开始之前同步持久保存。这确保在继续执行之前写入每个检查点，以牺牲一定的性能开销为代价提供高持久性。

您可以在调用任何图执行方法时指定持久性模式：

:::python

```python
graph.stream(
    {"input": "test"}, 
    durability="sync"
)
```

:::

## 在节点中使用任务 (Using tasks in nodes)

如果一个 [节点 (node)](./low_level.md#nodes) 包含多个操作，您可能会发现将每个操作转换为 **任务** 比将操作重构为单独的节点更容易。

:::python
=== "Original"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        result = requests.get(state['url']).text[:100]  # Side-effect
        return {
            "result": result
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "With task"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """Make a request."""
        # highlight-next-line
        return requests.get(url).text[:100]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        requests = [_make_request(url) for url in state['urls']]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

:::

:::js
=== "Original"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      url: z.string(),
      result: z.string().optional(),
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const response = await fetch(state.url);
      const text = await response.text();
      const result = text.slice(0, 100); // Side-effect
      return {
        result,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ url: "https://www.example.com" }, config);
    ```

=== "With task"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { task } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      urls: z.array(z.string()),
      results: z.array(z.string()).optional(),
    });

    const makeRequest = task("makeRequest", async (url: string) => {
      // highlight-next-line
      const response = await fetch(url);
      const text = await response.text();
      return text.slice(0, 100);
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const requests = state.urls.map((url) => makeRequest(url));
      const results = await Promise.all(requests);
      return {
        results,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ urls: ["https://www.example.com"] }, config);
    ```

:::

## 恢复工作流 (Resuming Workflows)

一旦您在工作流中启用了持久执行，您可以针对以下场景恢复执行：

:::python

- **暂停和恢复工作流：** 使用 @[interrupt][interrupt] 函数在特定点暂停工作流，并使用 @[Command] 原语以更新的状态恢复它。有关更多详细信息，请参阅 [**人在回路**](./human_in_the_loop.md)。
- **从故障中恢复：** 在异常（例如，LLM 提供商中断）后从上一个成功的检查点自动恢复工作流。这涉及使用相同的线程标识符执行工作流，并为其提供 `None` 作为输入值（请参阅 functional API 的此 [示例](../how-tos/use-functional-api.md#resuming-after-an-error)）。

  :::

:::js

- **暂停和恢复工作流：** 使用 @[interrupt][interrupt] 函数在特定点暂停工作流，并使用 @[Command] 原语以更新的状态恢复它。有关更多详细信息，请参阅 [**人在回路**](./human_in_the_loop.md)。
- **从故障中恢复：** 在异常（例如，LLM 提供商中断）后从上一个成功的检查点自动恢复工作流。这涉及使用相同的线程标识符执行工作流，并为其提供 `null` 作为输入值（请参阅 functional API 的此 [示例](../how-tos/use-functional-api.md#resuming-after-an-error)）。

  :::

## 恢复工作流的起点 (Starting Points for Resuming Workflows)

:::python

- 如果您正在使用 @[StateGraph (Graph API)][StateGraph]，则起点是停止执行的 [**节点 (node)**](./low_level.md#nodes) 的开头。
- 如果您在节点内进行子图调用，则起点将是调用已停止子图的 **父** 节点。
  在子图内，起点将是停止执行的特定 [**节点 (node)**](./low_level.md#nodes)。
- 如果您正在使用 Functional API，则起点是停止执行的 [**入口点 (entrypoint)**](./functional_api.md#entrypoint) 的开头。

  :::

:::js

- 如果您正在使用 [StateGraph (Graph API)](./low_level.md)，则起点是停止执行的 [**节点 (node)**](./low_level.md#nodes) 的开头。
- 如果您在节点内进行子图调用，则起点将是调用已停止子图的 **父** 节点。
  在子图内，起点将是停止执行的特定 [**节点 (node)**](./low_level.md#nodes)。
- 如果您正在使用 Functional API，则起点是停止执行的 [**入口点 (entrypoint)**](./functional_api.md#entrypoint) 的开头。

  :::
