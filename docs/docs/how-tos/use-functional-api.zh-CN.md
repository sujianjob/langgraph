# 如何使用函数式 API (How to use the Functional API)

LangGraph 提供了两种构建图的 API：

1. [Functional API (beta)](../concepts/functional_api.md): 一个用户友好的库，用于定义工作流。如果您想要类似 Python 的体验且不需要对图结构（如循环）进行细粒度控制，可以使用此 API。这在功能上类似于 [LangChain RunnableLambda](https://python.langchain.com/docs/concepts/lcel/#runnables)，但增加了持久性。
2. [Graph API](../concepts/low_level.md): 一个低级库，用于定义图。如果您需要构建复杂的代理应用程序，并且需要对图结构、循环、状态等进行细粒度控制，可以使用此 API。

本指南提供了一个关于如何使用 Functional API 的快速概览。特别是，我们将介绍以下概念：

* `task` 和 `entrypoint` 装饰器
* [Checkpointing (持久性)](../concepts/persistence.md)
* [Memory (记忆)](../concepts/memory.md) (线程级和跨线程)
* [Human-in-the-loop (人在回路)](../concepts/human_in_the_loop.md)
* [Streaming (流式传输)](../concepts/streaming.md)

!!! note
    Functional API 目前处于 **beta** 阶段，可能会发生变化。

## 设置 (Setup)

首先，安装所需的包：

```python
%%capture --no-stderr
%pip install -U langgraph langchain-anthropic
```

接下来，设置 API 密钥：

```python
import getpass
import os

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("ANTHROPIC_API_KEY")
```

<details>
<summary>设置 TypeScript 环境</summary>

```typescript
import { z } from "zod";

// Define the state schema
const State = z.object({
  messages: z.array(z.any()),
  topic: z.string(),
});
```
</details>

## 简单工作流 (Simple workflow)

最简单的工作流可以使用 `@task` 和 `@entrypoint` 装饰器定义。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task

@task
def call_model(question: str):
    return f"Answer to {question}"

@entrypoint()
def workflow(question: str):
    return call_model(question).result()

print(workflow.invoke("what is the capital of france?"))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task } from "@langchain/langgraph";

const callModel = task("callModel", async (question: string) => {
  return `Answer to ${question}`;
});

const workflow = entrypoint("workflow", async (question: string) => {
  return await callModel(question);
});

console.log(await workflow.invoke("what is the capital of france?"));
```
</details>

## 多个参数 (Multiple arguments)

您也可以向任务传递多个参数。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task

@task
def call_model(question: str, context: str):
    return f"Answer to {question} with {context}"

@entrypoint()
def workflow(question: str):
    return call_model(question, "some context").result()

print(workflow.invoke("what is the capital of france?"))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task } from "@langchain/langgraph";

const callModel = task("callModel", async (question: string, context: string) => {
  return `Answer to ${question} with ${context}`;
});

const workflow = entrypoint("workflow", async (question: string) => {
  return await callModel(question, "some context");
});

console.log(await workflow.invoke("what is the capital of france?"));
```
</details>

## 序列 (Sequences)

您可以将多个任务串联起来。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task

@task
def step_1(question: str):
    return f"Step 1: {question}"

@task
def step_2(context: str):
    return f"Step 2: {context}"

@entrypoint()
def workflow(question: str):
    result_1 = step_1(question).result()
    result_2 = step_2(result_1).result()
    return result_2

print(workflow.invoke("what is the capital of france?"))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task } from "@langchain/langgraph";

const step1 = task("step1", async (question: string) => {
  return `Step 1: ${question}`;
});

const step2 = task("step2", async (context: string) => {
  return `Step 2: ${context}`;
});

const workflow = entrypoint("workflow", async (question: string) => {
  const result1 = await step1(question);
  const result2 = await step2(result1);
  return result2;
});

console.log(await workflow.invoke("what is the capital of france?"));
```
</details>

## 并行执行 (Parallel execution)

如果您在不调用 `.result()` (Python) 或 `await` (JS) 的情况下调用多个任务，它们将并行运行。请注意，`entrypoint` 仍然是同步的，只有其内部的任务是并行调度的。

<details>
<summary>查看 Python 代码</summary>

```python
# highlight-next-line
import time
from langgraph.func import entrypoint, task

@task
def step_1(question: str):
    time.sleep(1)
    return f"Step 1: {question}"

@task
def step_2(context: str):
    time.sleep(1)
    return f"Step 2: {context}"

@entrypoint()
def workflow(question: str):
    # This will return a future
    future_1 = step_1(question)
    # This will return a future
    future_2 = step_2(question)
    # The tasks will run in parallel
    # highlight-next-line
    return future_1.result() + future_2.result()

print(workflow.invoke("what is the capital of france?"))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task } from "@langchain/langgraph";

const step1 = task("step1", async (question: string) => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return `Step 1: ${question}`;
});

const step2 = task("step2", async (context: string) => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return `Step 2: ${context}`;
});

const workflow = entrypoint("workflow", async (question: string) => {
  // This will return a promise
  const future1 = step1(question);
  // This will return a promise
  const future2 = step2(question);
  // The tasks will run in parallel
  // highlight-next-line
  const [result1, result2] = await Promise.all([future1, future2]);
  return result1 + result2;
});

console.log(await workflow.invoke("what is the capital of france?"));
```
</details>

## 兼容性 (Compatibility)

### 从 Functional API 调用 Graph API

您可以从 `entrypoint` 调用使用 `StateGraph` 创建的 `CompiledGraph`。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import StateGraph, START, END
from langgraph.func import entrypoint, task

# Define a simple graph
builder = StateGraph(dict)
builder.add_node("node_1", lambda state: {"a": "b"})
builder.add_edge(START, "node_1")
builder.add_edge("node_1", END)
graph = builder.compile()

@entrypoint()
def workflow(question: str):
    # Call the graph
    return graph.invoke({"question": question})

print(workflow.invoke("what is the capital of france?"))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START, END, entrypoint } from "@langchain/langgraph";

// Define a simple graph
const builder = new StateGraph<any>({
  channels: {
    question: null,
    a: null,
  },
})
  .addNode("node1", () => ({ a: "b" }))
  .addEdge(START, "node1")
  .addEdge("node1", END);
const graph = builder.compile();

const workflow = entrypoint("workflow", async (question: string) => {
  // Call the graph
  return await graph.invoke({ question });
});

console.log(await workflow.invoke("what is the capital of france?"));
```
</details>

### 从 Graph API 调用 Functional API

您也可以从 `StateGraph` 调用 `entrypoint`。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.graph import StateGraph, START, END
from langgraph.func import entrypoint, task

@entrypoint()
def workflow(question: str):
    return f"Answer to {question}"

# Define a simple graph
builder = StateGraph(dict)
# Call the entrypoint from a node
builder.add_node("node_1", lambda state: {"a": workflow.invoke(state["question"])})
builder.add_edge(START, "node_1")
builder.add_edge("node_1", END)
graph = builder.compile()

print(graph.invoke({"question": "what is the capital of france?"}))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, START, END, entrypoint } from "@langchain/langgraph";

const workflow = entrypoint("workflow", async (question: string) => {
  return `Answer to ${question}`;
});

// Define a simple graph
const builder = new StateGraph<any>({
  channels: {
    question: null,
    a: null,
  },
})
  // Call the entrypoint from a node
  .addNode("node1", async (state) => ({ a: await workflow.invoke(state.question) }))
  .addEdge(START, "node1")
  .addEdge("node1", END);
const graph = builder.compile();

console.log(await graph.invoke({ question: "what is the capital of france?" }));
```
</details>

## 缓存与重试 (Cache and Retry)

LangGraph 允许您为特定任务定义缓存策略和重试策略。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy, CachePolicy

# Retry configuration
retry_policy = RetryPolicy(max_attempts=3)
# Cache configuration (time-to-live)
cache_policy = CachePolicy(ttl=3600)  # cache results for 1 hour

@task(retry_policy=retry_policy, cache_policy=cache_policy)
def robust_task(param: str):
    # This function will be retried on failure
    # and its successful results will be cached
    return f"Processed {param}"

@entrypoint()
def workflow(question: str):
    return robust_task(question).result()
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task, RetryPolicy } from "@langchain/langgraph";

// Retry configuration
const retryPolicy: RetryPolicy = { maxAttempts: 3 };

const robustTask = task(
  "robustTask",
  async (param: string) => {
    // This function will be retried on failure
    return `Processed ${param}`;
  },
  {
    retryPolicy,
  }
);

const workflow = entrypoint("workflow", async (question: string) => {
  return await robustTask(question);
});
```
</details>

## 持久性 (Persistence)

通过传入 [checkpointer](../concepts/persistence.md) 对象并设置 `checkpointer` 参数，可以为 `entrypoint` 启用持久性。这将会在每一步之后自动保存工作流的状态。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task

checkpointer = InMemorySaver()

@task
def step_1(question: str):
    return f"Step 1: {question}"

@entrypoint(checkpointer=checkpointer)
def workflow(question: str):
    return step_1(question).result()

config = {"configurable": {"thread_id": "1"}}
print(workflow.invoke("what is the capital of france?", config=config))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const step1 = task("step1", async (question: string) => {
  return `Step 1: ${question}`;
});

const workflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (question: string) => {
    return await step1(question);
  }
);

const config = { configurable: { thread_id: "1" } };
console.log(await workflow.invoke("what is the capital of france?", config));
```
</details>

## 人在回路 (Human-in-the-loop)

启用持久性后，您可以使用 [interrupt](../concepts/human_in_the_loop.md) 函数暂停工作流并等待用户输入。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt

checkpointer = InMemorySaver()

@task
def step_1(question: str):
    return f"Step 1: {question}"

@entrypoint(checkpointer=checkpointer)
def workflow(question: str):
    step_1_result = step_1(question).result()
    # Pause and wait for user input
    user_approval = interrupt(f"Approve {step_1_result}?")
    if user_approval == "yes":
        return f"Approved: {step_1_result}"
    else:
        return "Rejected"

config = {"configurable": {"thread_id": "1"}}
# First run will pause at interrupt
print(workflow.invoke("start", config=config))

# Resume with user input
# highlight-next-line
print(workflow.invoke(
    # highlight-next-line
    Command(resume="yes"), 
    config=config
))
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task, MemorySaver, interrupt, Command } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const step1 = task("step1", async (question: string) => {
  return `Step 1: ${question}`;
});

const workflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (question: string) => {
    const step1Result = await step1(question);
    // Pause and wait for user input
    const userApproval = interrupt(`Approve ${step1Result}?`);
    if (userApproval === "yes") {
      return `Approved: ${step1Result}`;
    } else {
      return "Rejected";
    }
  }
);

const config = { configurable: { thread_id: "1" } };
// First run will pause at interrupt
console.log(await workflow.invoke("start", config));

// Resume with user input
// highlight-next-line
console.log(await workflow.invoke(
  // highlight-next-line
  new Command({ resume: "yes" }),
  config
));
```
</details>

## 流式传输 (Streaming)

Functional API 支持流式传输更新。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task

@task
def step_1(question: str):
    return f"Step 1: {question}"

@entrypoint()
def workflow(question: str):
    return step_1(question).result()

for event in workflow.stream("hello", stream_mode="updates"):
    print(event)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, task } from "@langchain/langgraph";

const step1 = task("step1", async (question: string) => {
  return `Step 1: ${question}`;
});

const workflow = entrypoint("workflow", async (question: string) => {
  return await step1(question);
});

for await (const event of await workflow.stream("hello", { streamMode: "updates" })) {
  console.log(event);
}
```
</details>

## 生成器 (Generators)

您可以使用 Python 生成器来流式传输自定义更新。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.func import entrypoint, task
from langchain_core.messages import AIMessage

@task
def call_model_stream(messages):
    # This is a generator
    yield AIMessage(content="Chunk 1")
    yield AIMessage(content="Chunk 2")

@entrypoint()
def workflow(messages):
    final_message = ""
    # Iterate over the generator
    for chunk in call_model_stream(messages):
        final_message += chunk.content
    return final_message
```
</details>

## 记忆 (Memory)

使用 `MemorySaver` 和 `entrypoint.final` 来管理记忆。通过返回 `save` 参数，您可以将值保存到检查点中，以便在下一次运行中通过 `previous` 参数检索。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: int = 0):
    new_total = previous + n
    # Return result and save state for next time
    return entrypoint.final(value=new_total, save=new_total)

config = {"configurable": {"thread_id": "my_thread"}}

print(accumulate.invoke(1, config=config))  # 1
print(accumulate.invoke(2, config=config))  # 3 (1 + 2)
print(accumulate.invoke(3, config=config))  # 6 (3 + 3)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { entrypoint, MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const accumulate = entrypoint(
  { checkpointer, name: "accumulate" },
  async (n: number, previous?: number) => {
    const prev = previous || 0;
    const total = prev + n;
    // Return result and save state for next time
    return entrypoint.final({ value: total, save: total });
  }
);

const config = { configurable: { thread_id: "my-thread" } };

console.log(await accumulate.invoke(1, config)); // 1
console.log(await accumulate.invoke(2, config)); // 3
console.log(await accumulate.invoke(3, config)); // 6
```
</details>
