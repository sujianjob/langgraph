---
search:
  boost: 2
---

# 函数式 API 概念 (Functional API concepts)

## 概述 (Overview)

**函数式 API (Functional API)** 允许您以对现有代码进行最少更改的方式，将 LangGraph 的关键功能 —— [持久性 (persistence)](./persistence.md)、[记忆 (memory)](../how-tos/memory/add-memory.md)、[人在回路 (human-in-the-loop)](./human_in_the_loop.md) 和 [流式传输 (streaming)](./streaming.md) —— 添加到您的应用程序中。

它的设计目的是将这些功能集成到可能使用标准语言原语（如 `if` 语句、`for` 循环和函数调用）进行分支和控制流的现有代码中。与许多需要将代码重组为显式管道或 DAG 的数据编排框架不同，函数式 API 允许您在不强制执行严格执行模型的情况下整合这些功能。

函数式 API 使用两个关键构建块：

:::python

- **`@entrypoint`** – 将函数标记为工作流的起点，封装逻辑并管理执行流程，包括处理长时间运行的任务和中断。
- **`@task`** – 表示离散的工作单元，例如 API 调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似 future 的对象，该对象可以被等待或同步解析。
  :::

:::js

- **`entrypoint`** – 一个入口点封装了工作流逻辑并管理执行流程，包括处理长时间运行的任务和中断。
- **`task`** – 表示离散的工作单元，例如 API 调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似 future 的对象，该对象可以被等待或同步解析。
  :::

这为构建具有状态管理和流式传输的工作流提供了最小抽象。

!!! tip

    有关如何使用函数式 API 的信息，请参阅 [使用函数式 API](../how-tos/use-functional-api.md)。

## 函数式 API 与 图 API (Functional API vs. Graph API)

对于喜欢更声明式方法的用，LangGraph 的 [图 API (Graph API)](./low_level.md) 允许您使用图范式定义工作流。两个 API 共享相同的底层运行时，因此您可以在同一个应用程序中一起使用它们。

以下是一些主要区别：

- **控制流 (Control flow)**: 函数式 API 不需要考虑图结构。您可以使用标准 Python 结构来定义工作流。这通常会减少您需要编写的代码量。
- **短期记忆 (Short-term memory)**: **GraphAPI** 需要声明一个 [**State**](./low_level.md#state)，并且可能需要定义 [**reducers**](./low_level.md#reducers) 来管理图状态的更新。`@entrypoint` 和 `@tasks` 不需要显式状态管理，因为它们的状态作用于函数，并且不在函数之间共享。
- **检查点 (Checkpointing)**: 两个 API 都会生成和使用检查点。在 **图 API** 中，每一步 [superstep](./low_level.md) 之后都会生成一个新的检查点。在 **函数式 API** 中，当执行任务时，它们的结果将保存到与给定入口点关联的现有检查点中，而不是创建新的检查点。
- **可视化 (Visualization)**: 图 API 使您可以轻松地将工作流可视化为图表，这对于调试、理解工作流以及与他人共享非常有用。函数式 API 不支持可视化，因为图是在运行时动态生成的。

## 示例 (Example)

下面我们演示一个简单的应用程序，它编写一篇文章并在请求人工审核时 [中断](human_in_the_loop.md)。

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1) # A placeholder for a long-running task.
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # Any json-serializable payload provided to interrupt as argument.
        # It will be surfaced on the client side as an Interrupt when streaming data
        # from the workflow.
        "essay": essay, # The essay we want reviewed.
        # We can add any additional information that we need.
        # For example, introduce a key called "action" with some instructions.
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay, # The essay that was generated
        "is_approved": is_approved, # Response from HIL
    }
```

:::

:::js

```typescript
import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

const writeEssay = task("writeEssay", async (topic: string) => {
  // A placeholder for a long-running task.
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return `An essay about topic: ${topic}`;
});

const workflow = entrypoint(
  { checkpointer: new MemorySaver(), name: "workflow" },
  async (topic: string) => {
    const essay = await writeEssay(topic);
    const isApproved = interrupt({
      // Any json-serializable payload provided to interrupt as argument.
      // It will be surfaced on the client side as an Interrupt when streaming data
      // from the workflow.
      essay, // The essay we want reviewed.
      // We can add any additional information that we need.
      // For example, introduce a key called "action" with some instructions.
      action: "Please approve/reject the essay",
    });

    return {
      essay, // The essay that was generated
      isApproved, // Response from HIL
    };
  }
);
```

:::

??? example "详细说明 (Detailed Explanation)"

    此工作流将撰写一篇关于主题 "cat" 的文章，然后暂停以获得人工审核。工作流可以中断无限长的时间，直到提供审核为止。

    当工作流恢复时，它将从一开始就执行，但是因为 `writeEssay` 任务的结果已经保存，所以任务结果将从检查点加载，而不是重新计算。

    :::python
    ```python
    import time
    import uuid
    from langgraph.func import entrypoint, task
    from langgraph.types import interrupt
    from langgraph.checkpoint.memory import InMemorySaver


    @task
    def write_essay(topic: str) -> str:
        """Write an essay about the given topic."""
        time.sleep(1)  # This is a placeholder for a long-running task.
        return f"An essay about topic: {topic}"

    @entrypoint(checkpointer=InMemorySaver())
    def workflow(topic: str) -> dict:
        """A simple workflow that writes an essay and asks for a review."""
        essay = write_essay("cat").result()
        is_approved = interrupt(
            {
                # Any json-serializable payload provided to interrupt as argument.
                # It will be surfaced on the client side as an Interrupt when streaming data
                # from the workflow.
                "essay": essay,  # The essay we want reviewed.
                # We can add any additional information that we need.
                # For example, introduce a key called "action" with some instructions.
                "action": "Please approve/reject the essay",
            }
        )
        return {
            "essay": essay,  # The essay that was generated
            "is_approved": is_approved,  # Response from HIL
        }


    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}
    for item in workflow.stream("cat", config):
        print(item)
    # > {'write_essay': 'An essay about topic: cat'}
    # > {
    # >     '__interrupt__': (
    # >        Interrupt(
    # >            value={
    # >                'essay': 'An essay about topic: cat',
    # >                'action': 'Please approve/reject the essay'
    # >            },
    # >            id='b9b2b9d788f482663ced6dc755c9e981'
    # >        ),
    # >    )
    # > }
    ```

    一篇文章已写好并准备好接受审核。一旦提供审核，我们可以恢复工作流：

    ```python
    from langgraph.types import Command

    # Get review from a user (e.g., via a UI)
    # In this case, we're using a bool, but this can be any json-serializable value.
    human_review = True

    for item in workflow.stream(Command(resume=human_review), config):
        print(item)
    ```

    ```pycon
    {'workflow': {'essay': 'An essay about topic: cat', 'is_approved': False}}
    ```

    工作流已完成，审核已添加到文章中。
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

    const writeEssay = task("writeEssay", async (topic: string) => {
      // This is a placeholder for a long-running task.
      await new Promise(resolve => setTimeout(resolve, 1000));
      return `An essay about topic: ${topic}`;
    });

    const workflow = entrypoint(
      { checkpointer: new MemorySaver(), name: "workflow" },
      async (topic: string) => {
        const essay = await writeEssay(topic);
        const isApproved = interrupt({
          // Any json-serializable payload provided to interrupt as argument.
          // It will be surfaced on the client side as an Interrupt when streaming data
          // from the workflow.
          essay, // The essay we want reviewed.
          // We can add any additional information that we need.
          // For example, introduce a key called "action" with some instructions.
          action: "Please approve/reject the essay",
        });

        return {
          essay, // The essay that was generated
          isApproved, // Response from HIL
        };
      }
    );

    const threadId = uuidv4();

    const config = {
      configurable: {
        thread_id: threadId
      }
    };

    for await (const item of workflow.stream("cat", config)) {
      console.log(item);
    }
    ```

    ```console
    { writeEssay: 'An essay about topic: cat' }
    {
      __interrupt__: [{
        value: { essay: 'An essay about topic: cat', action: 'Please approve/reject the essay' },
        resumable: true,
        ns: ['workflow:f7b8508b-21c0-8b4c-5958-4e8de74d2684'],
        when: 'during'
      }]
    }
    ```

    一篇文章已写好并准备好接受审核。一旦提供审核，我们可以恢复工作流：

    ```typescript
    import { Command } from "@langchain/langgraph";

    // Get review from a user (e.g., via a UI)
    // In this case, we're using a bool, but this can be any json-serializable value.
    const humanReview = true;

    for await (const item of workflow.stream(new Command({ resume: humanReview }), config)) {
      console.log(item);
    }
    ```

    ```console
    { workflow: { essay: 'An essay about topic: cat', isApproved: true } }
    ```

    工作流已完成，审核已添加到文章中。
    :::

## Entrypoint

:::python
@[`@entrypoint`][entrypoint] 装饰器可用于从函数创建工作流。它封装了工作流逻辑并管执行流程，包括处理 _长时间运行的任务_ 和 [中断](./human_in_the_loop.md)。
:::

:::js
@[`entrypoint`][entrypoint] 函数可用于从函数创建工作流。它封装了工作流逻辑并管理执行流程，包括处理 _长时间运行的任务_ 和 [中断](./human_in_the_loop.md)。
:::

### 定义 (Definition)

:::python
**entrypoint** 是通过使用 `@entrypoint` 装饰器装饰函数来定义的。

该函数 **必须接受单个位置参数**，作为工作流输入。如果您需要传递多条数据，请使用字典作为第一个参数的输入类型。

使用 `entrypoint` 装饰函数会生成一个 @[`Pregel`][Pregel.stream] 实例，该实例有助于管理工作流的执行（例如，处理流式传输、恢复和检查点）。

您通常希望将 **checkpointer** 传递给 `@entrypoint` 装饰器，以启用持久性并使用 **人在回路** 等功能。

=== "Sync"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop.
        ...
        return result
    ```

=== "Async"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop
        ...
        return result
    ```

:::

:::js
**entrypoint** 是通过使用配置和函数调用 `entrypoint` 函数来定义的。

该函数 **必须接受单个位置参数**，作为工作流输入。如果您需要传递多条数据，请使用对象作为第一个参数的输入类型。

使用函数创建 entrypoint 会生成一个工作流实例，该实例有助于管理工作流的执行（例如，处理流式传输、恢复和检查点）。

您通常希望将 **checkpointer** 传递给 `entrypoint` 函数，以启用持久性并使用 **人在回路** 等功能。

```typescript
import { entrypoint } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (someInput: Record<string, any>): Promise<number> => {
    // some logic that may involve long-running tasks like API calls,
    // and may be interrupted for human-in-the-loop
    return result;
  }
);
```

:::

!!! important "Serialization"

    entrypoint 的 **输入** 和 **输出** 必须是 JSON 可序列化的以支持检查点。请参阅 [序列化](#serialization) 部分以获取更多详细信息。

:::python

### 可注入参数 (Injectable parameters)

声明 `entrypoint` 时，您可以请求访问将在运行时自动注入的其他参数。这些参数包括：

| 参数 (Parameter) | 描述 (Description)                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **previous** | 访问与给定线程的上一个 `checkpoint` 关联的状态。请参阅 [短期记忆](#short-term-memory)。                                      |
| **store**    | [BaseStore][langgraph.store.base.BaseStore] 的实例。对于 [长期记忆](../how-tos/use-functional-api.md#long-term-memory) 很有用。                      |
| **writer**   | 在使用 Async Python < 3.11 时用于访问 StreamWriter。有关详细信息，请参阅 [使用函数式 API 进行流式传输](../how-tos/use-functional-api.md#streaming)。 |
| **config**   | 用于访问运行时配置。有关信息，请参阅 [RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig)。                  |

!!! important

    使用适当的名称和类型注释声明参数。

??? example "请求可注入参数 (Requesting Injectable Parameters)"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langgraph.func import entrypoint
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore

    in_memory_store = InMemoryStore(...)  # An instance of InMemoryStore for long-term memory

    @entrypoint(
        checkpointer=checkpointer,  # Specify the checkpointer
        store=in_memory_store  # Specify the store
    )
    def my_workflow(
        some_input: dict,  # The input (e.g., passed via `invoke`)
        *,
        previous: Any = None, # For short-term memory
        store: BaseStore,  # For long-term memory
        writer: StreamWriter,  # For streaming custom data
        config: RunnableConfig  # For accessing the configuration passed to the entrypoint
    ) -> ...:
    ```

:::

### 执行 (Executing)

:::python
使用 [`@entrypoint`](#entrypoint) 会生成一个 @[`Pregel`][Pregel.stream] 对象，该对象可以使用 `invoke`、`ainvoke`、`stream` 和 `astream` 方法执行。

=== "Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # Wait for the result synchronously
    ```

=== "Async Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # Await result asynchronously
    ```

=== "Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(some_input, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(some_input, config):
        print(chunk)
    ```

:::

:::js
使用 [`entrypoint`](#entrypoint) 函数将返回一个对象，该对象可以使用 `invoke` 和 `stream` 方法执行。

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };
    await myWorkflow.invoke(someInput, config); // Wait for the result
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(someInput, config)) {
      console.log(chunk);
    }
    ```

:::

### 恢复 (Resuming)

:::python
可以通过将 **resume** 值传递给 @[Command] 原语来在 @[interrupt][interrupt] 后恢复执行。

=== "Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```

=== "Async Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```

=== "Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

:::

:::js
可以通过将 **resume** 值传递给 @[`Command`][Command] 原语来在 @[interrupt][interrupt] 后恢复执行。

=== "Invoke"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(new Command({ resume: someResumeValue }), config);
    ```

=== "Stream"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    const stream = await myWorkflow.stream(
      new Command({ resume: someResumableValue }),
      config,
    )

    for await (const chunk of stream) {
      console.log(chunk);
    }
    ```

:::

:::python

**错误后恢复 (Resuming after an error)**

要在错误后恢复，请使用 `None` 和相同的 **thread id** (config) 运行 `entrypoint`。

这假设底层 **错误** 已解决，并且执行可以成功进行。

=== "Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(None, config)
    ```

=== "Async Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(None, config)
    ```

=== "Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(None, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(None, config):
        print(chunk)
    ```

:::

:::js

**错误后恢复 (Resuming after an error)**

要在错误后恢复，请使用 `null` 和相同的 **thread id** (config) 运行 `entrypoint`。

这假设底层 **错误** 已解决，并且执行可以成功进行。

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(null, config);
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(null, config)) {
      console.log(chunk);
    }
    ```

:::

### 短期记忆 (Short-term memory)

当使用 `checkpointer` 定义 `entrypoint` 时，它会在 [检查点](persistence.md#checkpoints) 中的同一 **线程 ID** 上的连续调用之间存储信息。

:::python
这允许使用 `previous` 参数访问先前调用的状态。

默认情况下，`previous` 参数是先前调用的返回值。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1 (previous was None)
my_workflow.invoke(2, config)  # 3 (previous was 1 from the previous invocation)
```

:::

:::js
这允许使用 `getPreviousState` 函数访问先前调用的状态。

默认情况下，`getPreviousState` 函数返回先前调用的返回值。

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    return number + previous;
  }
);

const config = {
  configurable: {
    thread_id: "some_thread_id",
  },
};

await myWorkflow.invoke(1, config); // 1 (previous was undefined)
await myWorkflow.invoke(2, config); // 3 (previous was 1 from the previous invocation)
```

:::

#### `entrypoint.final`

:::python
@[`entrypoint.final`][entrypoint.final] 是一个特殊的原语，可以从入口点返回，并允许 **解耦** 从 **入口点的返回值** 中 **保存在检查点中** 的值。

第一个值是入口点的返回值，第二个值是将保存在检查点中的值。类型注释是 `entrypoint.final[return_type, save_type]`。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # This will return the previous value to the caller, saving
    # 2 * number to the checkpoint, which will be used in the next invocation
    # for the `previous` parameter.
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
```

:::

:::js
@[`entrypoint.final`][entrypoint.final] 是一个特殊的原语，可以从入口点返回，并允许 **解耦** 从 **入口点的返回值** 中 **保存在检查点中** 的值。

第一个值是入口点的返回值，第二个值是将保存在检查点中的值。

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    // This will return the previous value to the caller, saving
    // 2 * number to the checkpoint, which will be used in the next invocation
    // for the `previous` parameter.
    return entrypoint.final({
      value: previous,
      save: 2 * number,
    });
  }
);

const config = {
  configurable: {
    thread_id: "1",
  },
};

await myWorkflow.invoke(3, config); // 0 (previous was undefined)
await myWorkflow.invoke(1, config); // 6 (previous was 3 * 2 from the previous invocation)
```
