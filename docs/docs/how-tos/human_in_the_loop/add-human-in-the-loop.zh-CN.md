---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - interrupt
hide:
  - tags
---

# 启用人工干预 (Enable human intervention)

要在代理或工作流中审查、编辑和批准工具调用，可以使用中断 (interrupts) 来暂停图并等待人工输入。中断使用 LangGraph 的 [持久性 (persistence)](../../concepts/persistence.md) 层，该层保存图状态，以无限期暂停图执行，直到您恢复它。

!!! info

    有关人在回路 (human-in-the-loop) 工作流的更多信息，请参阅 [人在回路](../../concepts/human_in_the_loop.md) 概念指南。

## 使用 `interrupt` 暂停 (Pause using `interrupt`)

:::python
[动态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为动态断点）是根据图的当前状态触发的。您可以通过在适当的位置调用 @[`interrupt` 函数][interrupt] 来设置动态中断。图将暂停，允许人工干预，然后使用他们的输入恢复图。这对于审批、编辑或收集额外上下文等任务非常有用。

!!! note

    截至 v1.0，`interrupt` 是暂停图的推荐方式。`NodeInterrupt` 已弃用，且将在 v2.0 中移除。

:::

:::js
[动态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为动态断点）是根据图的当前状态触发的。您可以通过在适当的位置调用 @[`interrupt` 函数][interrupt] 来设置动态中断。图将暂停，允许人工干预，然后使用他们的输入恢复图。这对于审批、编辑或收集额外上下文等任务非常有用。
:::

要在图中使用 `interrupt`，您需要：

1. [**指定检查点保存器 (checkpointer)**](../../concepts/persistence.md#checkpoints) 以在每一步后保存图状态。
2. 在适当的位置 **调用 `interrupt()`**。有关示例，请参阅 [常见模式](#common-patterns) 部分。
3. 使用 [**线程 ID**](../../concepts/persistence.md#threads) **运行图**，直到触发 `interrupt`。
4. 使用 `invoke`/`stream` **恢复执行**（请参阅 [**`Command` 原语**](#resume-using-the-command-primitive)）。

:::python

```python
# highlight-next-line
from langgraph.types import interrupt, Command

def human_node(state: State):
    # highlight-next-line
    value = interrupt( # (1)!
        {
            "text_to_revise": state["some_text"] # (2)!
        }
    )
    return {
        "some_text": value # (3)!
    }


graph = graph_builder.compile(checkpointer=checkpointer) # (4)!

# Run the graph until the interrupt is hit.
config = {"configurable": {"thread_id": "some_id"}}
result = graph.invoke({"some_text": "original text"}, config=config) # (5)!
print(result['__interrupt__']) # (6)!
# > [
# >    Interrupt(
# >       value={'text_to_revise': 'original text'},
# >       resumable=True,
# >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
# >    )
# > ]

# highlight-next-line
print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
# > {'some_text': 'Edited text'}
```

1. `interrupt(...)` 暂停 `human_node` 的执行，将给定的有效负载呈现给人类。
2. 任何 JSON 可序列化的值都可以传递给 `interrupt` 函数。在这里，是一个包含要修改文本的字典。
3. 恢复后，`interrupt(...)` 的返回值是人类提供的输入，用于更新状态。
4. 持久化图状态需要检查点保存器。在生产中，这应该是持久的（例如，由数据库支持）。
5. 使用一些初始状态调用图。
6. 当图遇到中断时，它会返回一个包含有效负载和元数据的 `Interrupt` 对象。
7. 图使用 `Command(resume=...)` 恢复，注入人类的输入并继续执行。
   :::

:::js

```typescript
// highlight-next-line
import { interrupt, Command } from "@langchain/langgraph";

const graph = graphBuilder
  .addNode("humanNode", (state) => {
    // highlight-next-line
    const value = interrupt(
      // (1)!
      {
        textToRevise: state.someText, // (2)!
      }
    );
    return {
      someText: value, // (3)!
    };
  })
  .addEdge(START, "humanNode")
  .compile({ checkpointer }); // (4)!

// Run the graph until the interrupt is hit.
const config = { configurable: { thread_id: "some_id" } };
const result = await graph.invoke({ someText: "original text" }, config); // (5)!
console.log(result.__interrupt__); // (6)!
// > [
// >   {
// >     value: { textToRevise: 'original text' },
// >     resumable: true,
// >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
// >     when: 'during'
// >   }
// > ]

// highlight-next-line
console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
// > { someText: 'Edited text' }
```

1. `interrupt(...)` 暂停 `humanNode` 的执行，将给定的有效负载呈现给人类。
2. 任何 JSON 可序列化的值都可以传递给 `interrupt` 函数。在这里，是一个包含要修改文本的对象。
3. 恢复后，`interrupt(...)` 的返回值是人类提供的输入，用于更新状态。
4. 持久化图状态需要检查点保存器。在生产中，这应该是持久的（例如，由数据库支持）。
5. 使用一些初始状态调用图。
6. 当图遇到中断时，它会返回一个对象，其中 `__interrupt__` 包含有效负载和元数据。
7. 图使用 `Command({ resume: ... })` 恢复，注入人类的输入并继续执行。
   :::

??? example "扩展示例：使用 `interrupt`"

    :::python
    ```python
    from typing import TypedDict
    import uuid
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph

    # highlight-next-line
    from langgraph.types import interrupt, Command


    class State(TypedDict):
        some_text: str


    def human_node(state: State):
        # highlight-next-line
        value = interrupt(  # (1)!
            {
                "text_to_revise": state["some_text"]  # (2)!
            }
        )
        return {
            "some_text": value  # (3)!
        }


    # Build the graph
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")
    checkpointer = InMemorySaver()  # (4)!
    graph = graph_builder.compile(checkpointer=checkpointer)
    # Pass a thread ID to the graph to run it.
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    # Run the graph until the interrupt is hit.
    result = graph.invoke({"some_text": "original text"}, config=config)  # (5)!

    print(result['__interrupt__']) # (6)!
    # > [
    # >    Interrupt(
    # >       value={'text_to_revise': 'original text'},
    # >       resumable=True,
    # >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
    # >    )
    # > ]
    print(result["__interrupt__"])  # (6)!
    # > [Interrupt(value={'text_to_revise': 'original text'}, id='6d7c4048049254c83195429a3659661d')]

    # highlight-next-line
    print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
    # > {'some_text': 'Edited text'}
    ```

    1. `interrupt(...)` 暂停 `human_node` 的执行，将给定的有效负载呈现给人类。
    2. 任何 JSON 可序列化的值都可以传递给 `interrupt` 函数。在这里，是一个包含要修改文本的字典。
    3. 恢复后，`interrupt(...)` 的返回值是人类提供的输入，用于更新状态。
    4. 持久化图状态需要检查点保存器。在生产中，这应该是持久的（例如，由数据库支持）。
    5. 使用一些初始状态调用图。
    6. 当图遇到中断时，它会返回一个含有有效负载和元数据的 `Interrupt` 对象。
    7. 图使用 `Command(resume=...)` 恢复，注入人类的输入并继续执行。
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, StateGraph, START, interrupt, Command } from "@langchain/langgraph";

    const StateAnnotation = z.object({
      someText: z.string(),
    });

    // Build the graph
    const graphBuilder = new StateGraph(StateAnnotation)
      .addNode("humanNode", (state) => {
        // highlight-next-line
        const value = interrupt( // (1)!
          {
            textToRevise: state.someText // (2)!
          }
        );
        return {
          someText: value // (3)!
        };
      })
      .addEdge(START, "humanNode");

    const checkpointer = new MemorySaver(); // (4)!

    const graph = graphBuilder.compile({ checkpointer });

    // Pass a thread ID to the graph to run it.
    const config = { configurable: { thread_id: uuidv4() } };

    // Run the graph until the interrupt is hit.
    const result = await graph.invoke({ someText: "original text" }, config); // (5)!

    console.log(result.__interrupt__); // (6)!
    // > [
    // >   {
    // >     value: { textToRevise: 'original text' },
    // >     resumable: true,
    // >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
    // >     when: 'during'
    // >   }
    // > ]

    // highlight-next-line
    console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
    // > { someText: 'Edited text' }
    ```

    1. `interrupt(...)` 暂停 `humanNode` 的执行，将给定的有效负载呈现给人类。
    2. 任何 JSON 可序列化的值都可以传递给 `interrupt` 函数。在这里，是一个包含要修改文本的对象。
    3. 恢复后，`interrupt(...)` 的返回值是人类提供的输入，用于更新状态。
    4. 持久化图状态需要检查点保存器。在生产中，这应该是持久的（例如，由数据库支持）。
    5. 使用一些初始状态调用图。
    6. 当图遇到中断时，它会返回一个对象，其中 `__interrupt__` 包含有效负载和元数据。
    7. 图使用 `Command({ resume: ... })` 恢复，注入人类的输入并继续执行。
    :::

!!! tip "v0.4.0 新增功能"

    :::python
    `__interrupt__` 是一个特殊的键，如果图被中断，运行图时将返回该键。从 v0.4.0 版本开始，`invoke` 和 `ainvoke` 中增加了对 `__interrupt__` 的支持。如果您使用的是旧版本，只有使用 `stream` 或 `astream` 时才能在结果中看到 `__interrupt__`。您也可以使用 `graph.get_state(thread_id)` 获取中断值。
    :::

    :::js
    `__interrupt__` 是一个特殊的键，如果图被中断，运行图时将返回该键。从 v0.4.0 版本开始，`invoke` 中增加了对 `__interrupt__` 的支持。如果您使用的是旧版本，只有使用 `stream` 时才能在结果中看到 `__interrupt__`。您也可以使用 `graph.getState(config)` 获取中断值。
    :::

!!! warning

    :::python
    就开发者体验而言，中断类似于 Python 的 input() 函数，但它们不会从中断点自动恢复执行。相反，它们会重新运行使用了中断的整个节点。因此，中断通常最好放置在节点的开始处或专用节点中。
    :::

    :::js
    中断功能强大且符合人体工程学，但需要注意的是，它们不会从中断点自动恢复执行。相反，它们会重新运行使用了中断的整个位置。因此，中断通常最好放置在节点的状态或专用节点中。
    :::

## 使用 `Command` 原语恢复 (Resume using the `Command` primitive)

:::python
!!! warning

    从 `interrupt` 恢复与 Python 的 `input()` 函数不同，后者从调用 `input()` 函数的确切点恢复执行。

:::

当在图中使用 `interrupt` 函数时，执行会在该点暂停并等待用户输入。

:::python
要恢复执行，请使用 @[`Command`][Command] 原语，它可以通过 `invoke` 或 `stream` 方法提供。图从最初调用 `interrupt(...)` 的节点的开头恢复执行。这一次，`interrupt` 函数将返回 `Command(resume=value)` 中提供的值，而不是再次暂停。从节点开头到 `interrupt` 的所有代码都将重新执行。

```python
# Resume graph execution by providing the user's input.
graph.invoke(Command(resume={"age": "25"}), thread_config)
```

:::

:::js
要恢复执行，请使用 @[`Command`][Command] 原语，它可以通过 `invoke` 或 `stream` 方法提供。图从最初调用 `interrupt(...)` 的节点的开头恢复执行。这一次，`interrupt` 函数将返回 `Command(resume=value)` 中提供的值，而不是再次暂停。从节点开头到 `interrupt` 的所有代码都将重新执行。

```typescript
// Resume graph execution by providing the user's input.
await graph.invoke(new Command({ resume: { age: "25" } }), threadConfig);
```

:::

### 一次调用恢复多个中断 (Resume multiple interrupts with one invocation)

当具有中断条件的节点并行运行时，任务队列中可能会有多个中断。
例如，下图有两个并行运行的节点需要人工输入：

<figure markdown="1">
![image](../assets/human_in_loop_parallel.png){: style="max-height:400px"}
</figure>

:::python
一旦您的图被中断并停滞，您可以使用 `Command.resume` 一次恢复所有中断，传递中断 ID 到恢复值的字典映射。

```python
from typing import TypedDict
import uuid
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.constants import START
from langgraph.graph import StateGraph
from langgraph.types import interrupt, Command


class State(TypedDict):
    text_1: str
    text_2: str


def human_node_1(state: State):
    value = interrupt({"text_to_revise": state["text_1"]})
    return {"text_1": value}


def human_node_2(state: State):
    value = interrupt({"text_to_revise": state["text_2"]})
    return {"text_2": value}


graph_builder = StateGraph(State)
graph_builder.add_node("human_node_1", human_node_1)
graph_builder.add_node("human_node_2", human_node_2)

# Add both nodes in parallel from START
graph_builder.add_edge(START, "human_node_1")
graph_builder.add_edge(START, "human_node_2")

checkpointer = InMemorySaver()
graph = graph_builder.compile(checkpointer=checkpointer)

thread_id = str(uuid.uuid4())
config: RunnableConfig = {"configurable": {"thread_id": thread_id}}
result = graph.invoke(
    {"text_1": "original text 1", "text_2": "original text 2"}, config=config
)

# Resume with mapping of interrupt IDs to values
resume_map = {
    i.id: f"edited text for {i.value['text_to_revise']}"
    for i in graph.get_state(config).interrupts
}
print(graph.invoke(Command(resume=resume_map), config=config))
# > {'text_1': 'edited text for original text 1', 'text_2': 'edited text for original text 2'}
```

:::

:::js

```typescript
const state = await parentGraph.getState(threadConfig);
const resumeMap = Object.fromEntries(
  state.interrupts.map((i) => [
    i.interruptId,
    `human input for prompt ${i.value}`,
  ])
);

await parentGraph.invoke(new Command({ resume: resumeMap }), threadConfig);
```

:::

## 常见模式 (Common patterns)

下面我们展示可以使用 `interrupt` 和 `Command` 实现的不同设计模式。

### 批准或拒绝 (Approve or reject)

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/approve-or-reject.png){: style="max-height:400px"}
<figcaption>根据人类的批准或拒绝，图可以继续执行该动作或采取替代路径。</figcaption>
</figure>

在关键步骤（如 API 调用）之前暂停图，以审查并批准该操作。如果操作被拒绝，您可以阻止图执行该步骤，并可能采取替代操作。

:::python

```python
from typing import Literal
from langgraph.types import interrupt, Command

def human_approval(state: State) -> Command[Literal["some_node", "another_node"]]:
    is_approved = interrupt(
        {
            "question": "Is this correct?",
            # Surface the output that should be
            # reviewed and approved by the human.
            "llm_output": state["llm_output"]
        }
    )

    if is_approved:
        return Command(goto="some_node")
    else:
        return Command(goto="another_node")

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_approval", human_approval)
graph = graph_builder.compile(checkpointer=checkpointer)

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with either an approval or rejection.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(Command(resume=True), config=thread_config)
```

:::

:::js

```typescript
import { interrupt, Command } from "@langchain/langgraph";

// Add the node to the graph in an appropriate location
// and connect it to the relevant nodes.
graphBuilder.addNode("humanApproval", (state) => {
  const isApproved = interrupt({
    question: "Is this correct?",
    // Surface the output that should be
    // reviewed and approved by the human.
    llmOutput: state.llmOutput,
  });

  if (isApproved) {
    return new Command({ goto: "someNode" });
  } else {
    return new Command({ goto: "anotherNode" });
  }
});
const graph = graphBuilder.compile({ checkpointer });

// After running the graph and hitting the interrupt, the graph will pause.
// Resume it with either an approval or rejection.
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(new Command({ resume: true }), threadConfig);
```

:::

??? example "扩展示例：带中断的批准或拒绝"

    :::python
    ```python
    from typing import Literal, TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the shared graph state
    class State(TypedDict):
        llm_output: str
        decision: str

    # Simulate an LLM output node
    def generate_llm_output(state: State) -> State:
        return {"llm_output": "This is the generated output."}

    # Human approval node
    def human_approval(state: State) -> Command[Literal["approved_path", "rejected_path"]]:
        decision = interrupt({
            "question": "Do you approve the following output?",
            "llm_output": state["llm_output"]
        })

        if decision == "approve":
            return Command(goto="approved_path", update={"decision": "approved"})
        else:
            return Command(goto="rejected_path", update={"decision": "rejected"})

    # Next steps after approval
    def approved_node(state: State) -> State:
        print("✅ Approved path taken.")
        return state

    # Alternative path after rejection
    def rejected_node(state: State) -> State:
        print("❌ Rejected path taken.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_llm_output", generate_llm_output)
    builder.add_node("human_approval", human_approval)
    builder.add_node("approved_path", approved_node)
    builder.add_node("rejected_path", rejected_node)

    builder.set_entry_point("generate_llm_output")
    builder.add_edge("generate_llm_output", "human_approval")
    builder.add_edge("approved_path", END)
    builder.add_edge("rejected_path", END)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run until interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])
    # Output:
    # Interrupt(value={'question': 'Do you approve the following output?', 'llm_output': 'This is the generated output.'}, ...)

    # Simulate resuming with human input
    # To test rejection, replace resume="approve" with resume="reject"
    final_result = graph.invoke(Command(resume="approve"), config=config)
    print(final_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the shared graph state
    const StateAnnotation = z.object({
      llmOutput: z.string(),
      decision: z.string(),
    });

    // Simulate an LLM output node
    function generateLlmOutput(state: z.infer<typeof StateAnnotation>) {
      return { llmOutput: "This is the generated output." };
    }

    // Human approval node
    function humanApproval(state: z.infer<typeof StateAnnotation>): Command {
      const decision = interrupt({
        question: "Do you approve the following output?",
        llmOutput: state.llmOutput
      });

      if (decision === "approve") {
        return new Command({
          goto: "approvedPath",
          update: { decision: "approved" }
        });
      } else {
        return new Command({
          goto: "rejectedPath",
          update: { decision: "rejected" }
        });
      }
    }

    // Next steps after approval
    function approvedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("✅ Approved path taken.");
      return state;
    }

    // Alternative path after rejection
    function rejectedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("❌ Rejected path taken.");
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateLlmOutput", generateLlmOutput)
      .addNode("humanApproval", humanApproval, {
        ends: ["approvedPath", "rejectedPath"]
      })
      .addNode("approvedPath", approvedNode)
      .addNode("rejectedPath", rejectedNode)
      .addEdge(START, "generateLlmOutput")
      .addEdge("generateLlmOutput", "humanApproval")
      .addEdge("approvedPath", END)
      .addEdge("rejectedPath", END);

    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run until interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);
    console.log(result.__interrupt__);
    // Output:
    // [{
    //   value: {
    //     question: 'Do you approve the following output?',
    //     llmOutput: 'This is the generated output.'
    //   },
    //   ...
    // }]

    // Simulate resuming with human input
    // To test rejection, replace resume: "approve" with resume: "reject"
    const finalResult = await graph.invoke(
      new Command({ resume: "approve" }),
      config
    );
    console.log(finalResult);
    ```
    :::

### 审查和编辑状态 (Review and edit state)

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/edit-graph-state-simple.png){: style="max-height:400px"}
<figcaption>人可以审查和编辑图的状态。这对于纠正错误或使用附加信息更新状态非常有用。
</figcaption>
</figure>

:::python

```python
from langgraph.types import interrupt

def human_editing(state: State):
    ...
    result = interrupt(
        # Interrupt information to surface to the client.
        # Can be any JSON serializable value.
        {
            "task": "Review the output from the LLM and make any necessary edits.",
            "llm_generated_summary": state["llm_generated_summary"]
        }
    )

    # Update the state with the edited text
    return {
        "llm_generated_summary": result["edited_text"]
    }

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_editing", human_editing)
graph = graph_builder.compile(checkpointer=checkpointer)

...

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with the edited text.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(
    Command(resume={"edited_text": "The edited text"}),
    config=thread_config
)
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

function humanEditing(state: z.infer<typeof StateAnnotation>) {
  const result = interrupt({
    // Interrupt information to surface to the client.
    // Can be any JSON serializable value.
    task: "Review the output from the LLM and make any necessary edits.",
    llmGeneratedSummary: state.llmGeneratedSummary,
  });

  // Update the state with the edited text
  return {
    llmGeneratedSummary: result.editedText,
  };
}

// Add the node to the graph in an appropriate location
// and connect it to the relevant nodes.
graphBuilder.addNode("humanEditing", humanEditing);
const graph = graphBuilder.compile({ checkpointer });

// After running the graph and hitting the interrupt, the graph will pause.
// Resume it with the edited text.
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(
  new Command({ resume: { editedText: "The edited text" } }),
  threadConfig
);
```

:::

??? example "扩展示例：带中断的编辑状态"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the graph state
    class State(TypedDict):
        summary: str

    # Simulate an LLM summary generation
    def generate_summary(state: State) -> State:
        return {
            "summary": "The cat sat on the mat and looked at the stars."
        }

    # Human editing node
    def human_review_edit(state: State) -> State:
        result = interrupt({
            "task": "Please review and edit the generated summary if necessary.",
            "generated_summary": state["summary"]
        })
        return {
            "summary": result["edited_summary"]
        }

    # Simulate downstream use of the edited summary
    def downstream_use(state: State) -> State:
        print(f"✅ Using edited summary: {state['summary']}")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_summary", generate_summary)
    builder.add_node("human_review_edit", human_review_edit)
    builder.add_node("downstream_use", downstream_use)

    builder.set_entry_point("generate_summary")
    builder.add_edge("generate_summary", "human_review_edit")
    builder.add_edge("human_review_edit", "downstream_use")
    builder.add_edge("downstream_use", END)

    # Set up in-memory checkpointing for interrupt support
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke the graph until it hits the interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)

    # Output interrupt payload
    print(result["__interrupt__"])
    # Example output:
    # > [
    # >     Interrupt(
    # >         value={
    # >             'task': 'Please review and edit the generated summary if necessary.',
    # >             'generated_summary': 'The cat sat on the mat and looked at the stars.'
    # >         },
    # >         id='...'
    # >     )
    # > ]

    # Resume the graph with human-edited input
    edited_summary = "The cat lay on the rug, gazing peacefully at the night sky."
    resumed_result = graph.invoke(
        Command(resume={"edited_summary": edited_summary}),
        config=config
    )
    print(resumed_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the graph state
    const StateAnnotation = z.object({
      summary: z.string(),
    });

    // Simulate an LLM summary generation
    function generateSummary(state: z.infer<typeof StateAnnotation>) {
      return {
        summary: "The cat sat on the mat and looked at the stars."
      };
    }

    // Human editing node
    function humanReviewEdit(state: z.infer<typeof StateAnnotation>) {
      const result = interrupt({
        task: "Please review and edit the generated summary if necessary.",
        generatedSummary: state.summary
      });
      return {
        summary: result.editedSummary
      };
    }

    // Simulate downstream use of the edited summary
    function downstreamUse(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Using edited summary: ${state.summary}`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateSummary", generateSummary)
      .addNode("humanReviewEdit", humanReviewEdit)
      .addNode("downstreamUse", downstreamUse)
      .addEdge(START, "generateSummary")
      .addEdge("generateSummary", "humanReviewEdit")
      .addEdge("humanReviewEdit", "downstreamUse")
      .addEdge("downstreamUse", END);

    // Set up in-memory checkpointing for interrupt support
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Invoke the graph until it hits the interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);

    // Output interrupt payload
    console.log(result.__interrupt__);
    // Example output:
    // [{
    //   value: {
    //     task: 'Please review and edit the generated summary if necessary.',
    //     generatedSummary: 'The cat sat on the mat and looked at the stars.'
    //   },
    //   resumable: true,
    //   ...
    // }]

    // Resume the graph with human-edited input
    const editedSummary = "The cat lay on the rug, gazing peacefully at the night sky.";
    const resumedResult = await graph.invoke(
      new Command({ resume: { editedSummary } }),
      config
    );
    console.log(resumedResult);
    ```
    :::

### 审查工具调用 (Review tool calls)

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
<figcaption>除了在继续之前审查和编辑来自 LLM 的输出。在 LLM 请求的工具调用可能是敏感的或需要人工监督的应用程序中，这一点尤其关键。
</figcaption>
</figure>

要向工具添加人工批准步骤：

1. 在工具中使用 `interrupt()` 来暂停执行。
2. 使用 `Command` 恢复以根据人工输入继续。

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt
from langgraph.prebuilt import create_react_agent

# An example of a sensitive tool that requires human review / approval
def book_hotel(hotel_name: str):
    """Book a hotel"""
    # highlight-next-line
    response = interrupt(  # (1)!
        f"Trying to call `book_hotel` with args {{'hotel_name': {hotel_name}}}. "
        "Please approve or suggest edits."
    )
    if response["type"] == "accept":
        pass
    elif response["type"] == "edit":
        hotel_name = response["args"]["hotel_name"]
    else:
        raise ValueError(f"Unknown response type: {response['type']}")
    return f"Successfully booked a stay at {hotel_name}."

# highlight-next-line
checkpointer = InMemorySaver() # (2)!

agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[book_hotel],
    # highlight-next-line
    checkpointer=checkpointer, # (3)!
)
```

1. @[`interrupt` 函数][interrupt] 在特定节点暂停代理图。在这种情况下，我们在工具函数的开头调用 `interrupt()`，这会在执行该工具的节点处暂停图。`interrupt()` 内的信息（例如工具调用）可以呈现给人类，并且可以用用户输入（工具调用批准、编辑或反馈）恢复图。
2. `InMemorySaver` 用于在工具调用循环的每一步存储代理状态。这实现了 [短期记忆](../memory/add-memory.md#add-short-term-memory) 和 [人在回路](../../concepts/human_in_the_loop.md) 功能。在此示例中，我们使用 `InMemorySaver` 将代理状态存储在内存中。在生产应用程序中，代理状态将存储在数据库中。
3. 使用 `checkpointer` 初始化代理。
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { interrupt } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// An example of a sensitive tool that requires human review / approval
const bookHotel = tool(
  async ({ hotelName }) => {
    // highlight-next-line
    const response = interrupt(
      // (1)!
      `Trying to call \`bookHotel\` with args {"hotelName": "${hotelName}"}. ` +
        "Please approve or suggest edits."
    );
    if (response.type === "accept") {
      // Continue with original args
    } else if (response.type === "edit") {
      hotelName = response.args.hotelName;
    } else {
      throw new Error(`Unknown response type: ${response.type}`);
    }
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

// highlight-next-line
const checkpointer = new MemorySaver(); // (2)!

const agent = createReactAgent({
  llm: model,
  tools: [bookHotel],
  // highlight-next-line
  checkpointSaver: checkpointer, // (3)!
});
```

1. @[`interrupt` 函数][interrupt] 在特定节点暂停代理图。在这种情况下，我们在工具函数的开头调用 `interrupt()`，这会在执行该工具的节点处暂停图。`interrupt()` 内的信息（例如工具调用）可以呈现给人类，并且可以用用户输入（工具调用批准、编辑或反馈）恢复图。
2. `MemorySaver` 用于在工具调用循环的每一步存储代理状态。这实现了 [短期记忆](../memory/add-memory.md#add-short-term-memory) 和 [人在回路](../../concepts/human_in_the_loop.md) 功能。在此示例中，我们使用 `MemorySaver` 将代理状态存储在内存中。在生产应用程序中，代理状态将存储在数据库中。
3. 使用 `checkpointSaver` 初始化代理。
   :::

使用 `stream()` 方法运行代理，传递 `config` 对象以指定线程 ID。这允许代理在将来的调用中恢复同一对话。

:::python

```python
config = {
   "configurable": {
      # highlight-next-line
      "thread_id": "1"
   }
}

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
  },
};

const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

> 您应该看到代理运行直到它到达 `interrupt()` 调用，此时它暂停并等待人工输入。

使用 `Command` 恢复代理以根据人工输入继续。

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume={"type": "accept"}),  # (1)!
    # Command(resume={"type": "edit", "args": {"hotel_name": "McKittrick Hotel"}}),
    config
):
    print(chunk)
    print("\n")
```

1. @[`interrupt` 函数][interrupt] 与 @[`Command`][Command] 对象结合使用，以使用人类提供的值恢复图。
   :::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: { type: "accept" } }), // (1)!
  // new Command({ resume: { type: "edit", args: { hotelName: "McKittrick Hotel" } } }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

1. @[`interrupt` 函数][interrupt] 与 @[`Command`][Command] 对象结合使用，以使用人类提供的值恢复图。
   :::

### 向任何工具添加中断 (Add interrupts to any tool)

您可以创建一个包装器将中断添加到 _任何_ 工具。下面的示例提供了一个与 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 和 [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 兼容的参考实现。

:::python

```python title="Wrapper that adds human-in-the-loop to any tool"
from typing import Callable
from langchain_core.tools import BaseTool, tool as create_tool
from langchain_core.runnables import RunnableConfig
from langgraph.types import interrupt
from langgraph.prebuilt.interrupt import HumanInterruptConfig, HumanInterrupt

def add_human_in_the_loop(
    tool: Callable | BaseTool,
    *,
    interrupt_config: HumanInterruptConfig = None,
) -> BaseTool:
    """Wrap a tool to support human-in-the-loop review."""
    if not isinstance(tool, BaseTool):
        tool = create_tool(tool)

    if interrupt_config is None:
        interrupt_config = {
            "allow_accept": True,
            "allow_edit": True,
            "allow_respond": True,
        }

    @create_tool(  # (1)!
        tool.name,
        description=tool.description,
        args_schema=tool.args_schema
    )
    def call_tool_with_interrupt(config: RunnableConfig, **tool_input):
        request: HumanInterrupt = {
            "action_request": {
                "action": tool.name,
                "args": tool_input
            },
            "config": interrupt_config,
            "description": "Please review the tool call"
        }
        # highlight-next-line
        response = interrupt([request])[0]  # (2)!
        # approve the tool call
        if response["type"] == "accept":
            tool_response = tool.invoke(tool_input, config)
        # update tool call args
        elif response["type"] == "edit":
            tool_input = response["args"]["args"]
            tool_response = tool.invoke(tool_input, config)
        # respond to the LLM with user feedback
        elif response["type"] == "response":
            user_feedback = response["args"]
            tool_response = user_feedback
        else:
            raise ValueError(f"Unsupported interrupt response type: {response['type']}")

        return tool_response

    return call_tool_with_interrupt
```

1. 此包装器创建一个新工具，该工具在执行包装的工具 **之前** 调用 `interrupt()`。
2. `interrupt()` 使用 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 期望的特殊输入和输出格式： - 将 @[`HumanInterrupt`][HumanInterrupt] 对象列表发送到 `AgentInbox` 以呈现中断信息给最终用户 - 恢复值由 `AgentInbox` 作为列表提供 (即 `Command(resume=[...])`)
   :::

:::js

```typescript title="Wrapper that adds human-in-the-loop to any tool"
import { StructuredTool, tool } from "@langchain/core/tools";
import { RunnableConfig } from "@langchain/core/runnables";
import { interrupt } from "@langchain/langgraph";

interface HumanInterruptConfig {
  allowAccept?: boolean;
  allowEdit?: boolean;
  allowRespond?: boolean;
}

interface HumanInterrupt {
  actionRequest: {
    action: string;
    args: Record<string, any>;
  };
  config: HumanInterruptConfig;
  description: string;
}

function addHumanInTheLoop(
  originalTool: StructuredTool,
  interruptConfig: HumanInterruptConfig = {
    allowAccept: true,
    allowEdit: true,
    allowRespond: true,
  }
): StructuredTool {
  // Wrap the original tool to support human-in-the-loop review
  return tool(
    // (1)!
    async (toolInput: Record<string, any>, config?: RunnableConfig) => {
      const request: HumanInterrupt = {
        actionRequest: {
          action: originalTool.name,
          args: toolInput,
        },
        config: interruptConfig,
        description: "Please review the tool call",
      };

      // highlight-next-line
      const response = interrupt([request])[0]; // (2)!

      // approve the tool call
      if (response.type === "accept") {
        return await originalTool.invoke(toolInput, config);
      }
      // update tool call args
      else if (response.type === "edit") {
        const updatedArgs = response.args.args;
        return await originalTool.invoke(updatedArgs, config);
      }
      // respond to the LLM with user feedback
      else if (response.type === "response") {
        return response.args;
      } else {
        throw new Error(
          `Unsupported interrupt response type: ${response.type}`
        );
      }
    },
    {
      name: originalTool.name,
      description: originalTool.description,
      schema: originalTool.schema,
    }
  );
}
```

1. 此包装器创建一个新工具，该工具在执行包装的工具 **之前** 调用 `interrupt()`。
2. `interrupt()` 使用 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 期望的特殊输入和输出格式： - 将 [`HumanInterrupt`] 对象列表发送到 `AgentInbox` 以呈现中断信息给最终用户 - 恢复值由 `AgentInbox` 作为列表提供 (即 `Command({ resume: [...] })`)
   :::

您可以使用包装器将 `interrupt()` 添加到任何工具，而无需将其添加到工具 _内部_：

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.prebuilt import create_react_agent

# highlight-next-line
checkpointer = InMemorySaver()

def book_hotel(hotel_name: str):
   """Book a hotel"""
   return f"Successfully booked a stay at {hotel_name}."


agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[
        # highlight-next-line
        add_human_in_the_loop(book_hotel), # (1)!
    ],
    # highlight-next-line
    checkpointer=checkpointer,
)

config = {"configurable": {"thread_id": "1"}}

# Run the agent
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

1. `add_human_in_the_loop` 包装器用于将 `interrupt()` 添加到工具。这允许代理暂停执行并等待人工输入，然后再继续调用工具。
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const checkpointer = new MemorySaver();

const bookHotel = tool(
  async ({ hotelName }) => {
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [
    // highlight-next-line
    addHumanInTheLoop(bookHotel), // (1)!
  ],
  // highlight-next-line
  checkpointSaver: checkpointer,
});

const config = { configurable: { thread_id: "1" } };

// Run the agent
const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

1. `addHumanInTheLoop` 包装器用于将 `interrupt()` 添加到工具。这允许代理暂停执行并等待人工输入，然后再继续调用工具。
   :::

> 您应该看到代理运行直到它到达 `interrupt()` 调用，此时它暂停并等待人工输入。

使用 `Command` 恢复代理以根据人工输入继续。

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume=[{"type": "accept"}]),
    # Command(resume=[{"type": "edit", "args": {"args": {"hotel_name": "McKittrick Hotel"}}}]),
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: [{ type: "accept" }] }),
  // new Command({ resume: [{ type: "edit", args: { args: { hotelName: "McKittrick Hotel" } } }] }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

### 验证人工输入 (Validate human input)

如果您需要在图本身（而不是客户端）内验证人类提供的输入，可以通过在单个节点中使用多个中断调用来实现。

:::python

```python
from langgraph.types import interrupt

def human_node(state: State):
    """Human node with validation."""
    question = "What is your age?"

    while True:
        answer = interrupt(question)

        # Validate answer, if the answer isn't valid ask for input again.
        if not isinstance(answer, int) or answer < 0:
            question = f"'{answer} is not a valid age. What is your age?"
            answer = None
            continue
        else:
            # If the answer is valid, we can proceed.
            break

    print(f"The human in the loop is {answer} years old.")
    return {
        "age": answer
    }
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

graphBuilder.addNode("humanNode", (state) => {
  // Human node with validation.
  let question = "What is your age?";

  while (true) {
    const answer = interrupt(question);

    // Validate answer, if the answer isn't valid ask for input again.
    if (typeof answer !== "number" || answer < 0) {
      question = `'${answer}' is not a valid age. What is your age?`;
      continue;
    } else {
      // If the answer is valid, we can proceed.
      break;
    }
  }

  console.log(`The human in the loop is ${answer} years old.`);
  return {
    age: answer,
  };
});
```

:::

??? example "扩展示例：验证用户输入"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define graph state
    class State(TypedDict):
        age: int

    # Node that asks for human input and validates it
    def get_valid_age(state: State) -> State:
        prompt = "Please enter your age (must be a non-negative integer)."

        while True:
            user_input = interrupt(prompt)

            # Validate the input
            try:
                age = int(user_input)
                if age < 0:
                    raise ValueError("Age must be non-negative.")
                break  # Valid input received
            except (ValueError, TypeError):
                prompt = f"'{user_input}' is not valid. Please enter a non-negative integer for age."

        return {"age": age}

    # Node that uses the valid input
    def report_age(state: State) -> State:
        print(f"✅ Human is {state['age']} years old.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("get_valid_age", get_valid_age)
    builder.add_node("report_age", report_age)

    builder.set_entry_point("get_valid_age")
    builder.add_edge("get_valid_age", "report_age")
    builder.add_edge("report_age", END)

    # Create the graph with a memory checkpointer
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run the graph until the first interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])  # First prompt: "Please enter your age..."

    # Simulate an invalid input (e.g., string instead of integer)
    result = graph.invoke(Command(resume="not a number"), config=config)
    print(result["__interrupt__"])  # Follow-up prompt with validation message

    # Simulate a second invalid input (e.g., negative number)
    result = graph.invoke(Command(resume="-10"), config=config)
    print(result["__interrupt__"])  # Another retry

    # Provide valid input
    final_result = graph.invoke(Command(resume="25"), config=config)
    print(final_result)  # Should include the valid age
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define graph state
    const StateAnnotation = z.object({
      age: z.number(),
    });

    // Node that asks for human input and validates it
    function getValidAge(state: z.infer<typeof StateAnnotation>) {
      let prompt = "Please enter your age (must be a non-negative integer).";

      while (true) {
        const userInput = interrupt(prompt);

        // Validate the input
        try {
          const age = parseInt(userInput as string);
          if (isNaN(age) || age < 0) {
            throw new Error("Age must be non-negative.");
          }
          return { age };
        } catch (error) {
          prompt = `'${userInput}' is not valid. Please enter a non-negative integer for age.`;
        }
      }
    }

    // Node that uses the valid input
    function reportAge(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Human is ${state.age} years old.`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("getValidAge", getValidAge)
      .addNode("reportAge", reportAge)
      .addEdge(START, "getValidAge")
      .addEdge("getValidAge", "reportAge")
      .addEdge("reportAge", END);

    // Create the graph with a memory checkpointer
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run the graph until the first interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    let result = await graph.invoke({}, config);
    console.log(result.__interrupt__);  // First prompt: "Please enter your age..."

    // Simulate an invalid input (e.g., string instead of integer)
    result = await graph.invoke(new Command({ resume: "not a number" }), config);
    console.log(result.__interrupt__);  // Follow-up prompt with validation message

    // Simulate a second invalid input (e.g., negative number)
    result = await graph.invoke(new Command({ resume: "-10" }), config);
    console.log(result.__interrupt__);  // Another retry

    // Provide valid input
    const finalResult = await graph.invoke(new Command({ resume: "25" }), config);
    console.log(finalResult);  // Should include the valid age
    ```
    :::

### 在单个节点中使用多个中断 (Using multiple interrupts in a single node)

在 **单个** 节点中使用多个中断对于诸如 [验证人工输入](#validate-human-input) 之类的模式很有用。但是，如果在处理不当时，在同一节点中使用多个中断可能会导致意外行为。

当节点包含多个中断调用时，LangGraph 会保留特定于执行该节点的任务的恢复值列表。每当恢复执行时，它都会从节点的开头开始。对于遇到的每个中断，LangGraph 都会检查任务的恢复列表中是否存在匹配的值。匹配 **严格基于索引**，因此节点内中断调用的顺序至关重要。

为避免问题，请避免在执行之间动态更改节点的结构。这包括添加、删除或重新排序中断调用，因为此类更改可能导致索引不匹配。这些问题通常源于非常规模式，例如通过 `Command(resume=..., update=SOME_STATE_MUTATION)` 改变状态或依靠全局变量动态修改节点的结构。

:::python
??? example "扩展示例：引入非确定性的错误代码"

    ```python
    import uuid
    from typing import TypedDict, Optional

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""

        age: Optional[str]
        name: Optional[str]


    def human_node(state: State):
        if not state.get('name'):
            name = interrupt("what is your name?")
        else:
            name = "N/A"

        if not state.get('age'):
            age = interrupt("what is your age?")
        else:
            age = "N/A"

        print(f"Name: {name}. Age: {age}")

        return {
            "age": age,
            "name": name,
        }


    builder = StateGraph(State)
    builder.add_node("human_node", human_node)
    builder.add_edge(START, "human_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
            "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"age": None, "name": None}, config):
        print(chunk)

    for chunk in graph.stream(Command(resume="John", update={"name": "foo"}), config):
        print(chunk)
    ```

    ```pycon
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    Name: N/A. Age: John
    {'human_node': {'age': 'John', 'name': 'N/A'}}
    ```

:::

## 使用中断进行调试 (Debug with interrupts)

要调试和测试图，请使用 [静态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为静态断点）一次一步地执行图，或在特定节点暂停图执行。静态中断在节点执行之前或之后的定义点触发。您可以通过在编译时或运行时指定 `interrupt_before` 和 `interrupt_after` 来设置静态中断。

!!! warning

    **不建议** 将静态中断用于人在回路工作流。请改用 [动态中断](#pause-using-interrupt)。

=== "编译时 (Compile time)"

    :::python
    ```python
    # highlight-next-line
    graph = graph_builder.compile( # (1)!
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"], # (3)!
        checkpointer=checkpointer, # (4)!
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=thread_config) # (5)!

    # Resume the graph
    graph.invoke(None, config=thread_config) # (6)!
    ```

    1. 断点在 `compile` 时设置。
    2. `interrupt_before` 指定在执行节点之前应暂停的节点。
    3. `interrupt_after` 指定在执行节点之后应暂停的节点。
    4. 启用断点需要检查点保存器。
    5. 运行图直到遇到第一个断点。
    6. 通过传入 `None` 作为输入来恢复图。这将运行图直到遇到下一个断点。
    :::
    :::js
    ```typescript
    // highlight-next-line
    const graph = graphBuilder.compile({
      // highlight-next-line
      interruptBefore: ["nodeA"], // (1)!
      // highlight-next-line
      interruptAfter: ["nodeB", "nodeC"], // (2)!
      checkpointer, // (3)!
    });

    const config = {
      configurable: {
        thread_id: "some_thread",
      },
    };

    // Run the graph until the breakpoint
    await graph.invoke(inputs, config); // (4)!

    // Resume the graph
    await graph.invoke(null, config); // (5)!
    ```
    1. `interruptBefore` 指定在执行节点之前应暂停的节点。
    2. `interruptAfter` 指定在执行节点之后应暂停的节点。
    3. 启用断点需要检查点保存器。
    4. 运行图直到遇到第一个断点。
    5. 通过传入 `null` 作为输入来恢复图。这将运行图直到遇到下一个断点。
    :::

=== "运行时 (Run time)"

    :::python
    ```python
    # highlight-next-line
    graph.invoke( # (1)!
        inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
        config={
            "configurable": {"thread_id": "some_thread"}
        },
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=config) # (4)!

    # Resume the graph
    graph.invoke(None, config=config) # (5)!
    ```

    1. 调用 `graph.invoke` 时带有 `interrupt_before` 和 `interrupt_after` 参数。这是一个运行时配置，可以为每次调用更改。
    2. `interrupt_before` 指定在执行节点之前应暂停的节点。
    3. `interrupt_after` 指定在执行节点之后应暂停的节点。
    4. 运行图直到遇到第一个断点。
    5. 通过传入 `None` 作为输入来恢复图。这将运行图直到遇到下一个断点。

    !!! note

        您不能在运行时为 **子图** 设置静态断点。
        如果有一个子图，您必须在编译时设置断点。
    :::
    :::js
    ```typescript
    // highlight-next-line
    await graph.invoke(inputs, {
      // highlight-next-line
      interruptBefore: ["nodeA"], // (1)!
      // highlight-next-line
      interruptAfter: ["nodeB", "nodeC"], // (2)!
      configurable: {
        thread_id: "some_thread",
      },
    });

    const config = {
      configurable: {
        thread_id: "some_thread",
      },
    };

    // Run the graph until the breakpoint
    await graph.invoke(inputs, config); // (3)!

    // Resume the graph
    await graph.invoke(null, config); // (4)!
    ```
    1. `interruptBefore` 指定在执行节点之前应暂停的节点。
    2. `interruptAfter` 指定在执行节点之后应暂停的节点。
    3. 运行图直到遇到第一个断点。
    4. 通过传入 `null` 作为输入来恢复图。这将运行图直到遇到下一个断点。
    
    !!! note

        您不能在运行时为 **子图** 设置静态断点。
        如果有一个子图，您必须在编译时设置断点。
    :::

??? example "设置静态断点"

    :::python
    ```python
    from IPython.display import Image, display
    from typing_extensions import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END


    class State(TypedDict):
        input: str


    def step_1(state):
        print("---Step 1---")
        pass


    def step_2(state):
        print("---Step 2---")
        pass


    def step_3(state):
        print("---Step 3---")
        pass


    builder = StateGraph(State)
    builder.add_node("step_1", step_1)
    builder.add_node("step_2", step_2)
    builder.add_node("step_3", step_3)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    # Set up a checkpointer
    checkpointer = InMemorySaver() # (1)!

    graph = builder.compile(
        checkpointer=checkpointer, # (2)!
        interrupt_before=["step_3"] # (3)!
    )

    # View
    display(Image(graph.get_graph().draw_mermaid_png()))


    # Input
    initial_input = {"input": "hello world"}

    # Thread
    thread = {"configurable": {"thread_id": "1"}}

    # Run the graph until the first interruption
    for event in graph.stream(initial_input, thread, stream_mode="values"):
        print(event)

    # This will run until the breakpoint
    # You can get the state of the graph at this point
    print(graph.get_state(config))

    # You can continue the graph execution by passing in `None` for the input
    for event in graph.stream(None, thread, stream_mode="values"):
        print(event)
    ```
    :::

### 在 LangGraph Studio 中使用静态中断 (Use static interrupts in LangGraph Studio)

您可以使用 [LangGraph Studio](../../concepts/langgraph_studio.md) 来调试您的图。您可以在 UI 中设置静态断点，然后运行图。您还可以使用 UI 在执行的任何点检查图状态。

![image](../../concepts/img/human_in_the_loop/static-interrupt.png){: style="max-height:400px"}

使用 `langgraph dev` 的 [本地部署应用程序](../../tutorials/langgraph-platform/local-server.md) 可以免费使用 LangGraph Studio。

## 注意事项 (Considerations)

使用人在回路时，需要牢记一些注意事项。

### 与具有副作用的代码一起使用 (Using with code with side-effects)

将具有副作用的代码（例如 API 调用）放置在 `interrupt` 之后或单独的节点中，以避免重复，因为每次恢复节点时都会重新触发这些代码。

=== "中断后的副作用 (Side effects after interrupt)"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        api_call(answer) # OK as it's after the interrupt
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      apiCall(answer); // OK as it's after the interrupt
    }
    ```
    :::

=== "单独节点中的副作用 (Side effects in a separate node)"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        return {
            "answer": answer
        }

    def api_call_node(state: State):
        api_call(...) # OK as it's in a separate node
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      return {
        answer
      };
    }

    function apiCallNode(state: z.infer<typeof StateAnnotation>) {
      apiCall(state.answer); // OK as it's in a separate node
    }
    ```
    :::

### 与作为函数调用的子图一起使用 (Using with subgraphs called as functions)

当作为函数调用子图时，父图将从触发 `interrupt` 的 **调用子图的节点开头** 恢复执行。同样，**子图** 将从调用 `interrupt()` 函数的 **节点开头** 恢复。

:::python

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- This will re-execute when the subgraph is resumed.
    # Invoke a subgraph as a function.
    # The subgraph contains an `interrupt` call.
    subgraph_result = subgraph.invoke(some_input)
    ...
```

:::

:::js

```typescript
async function nodeInParentGraph(state: z.infer<typeof StateAnnotation>) {
  someCode(); // <-- This will re-execute when the subgraph is resumed.
  // Invoke a subgraph as a function.
  // The subgraph contains an `interrupt` call.
  const subgraphResult = await subgraph.invoke(someInput);
  // ...
}
```

:::

??? example "扩展示例：父图和子图执行流程"

    假设我们有一个包含 3 个节点的父图：

    **Parent Graph**: `node_1` → `node_2` (调用子图) → `node_3`

    子图有 3 个节点，其中第二个节点包含一个 `interrupt`：

    **Subgraph**: `sub_node_1` → `sub_node_2` (`interrupt`) → `sub_node_3`

    当恢复图时，执行将按如下方式进行：

    1. **跳过 `node_1`** 在父图中（已执行，图状态保存在快照中）。
    2. **重新执行 `node_2`** 在父图中，从头开始。
    3. **跳过 `sub_node_1`** 在子图中（已执行，图状态保存在快照中）。
    4. **重新执行 `sub_node_2`** 在子图中，从头开始。
    5. 继续执行 `sub_node_3` 和后续节点。

    以下是可以用来理解子图如何与中断配合使用的简略示例代码。
    它计算每个节点进入的次数并打印计数。

    :::python
    ```python
    import uuid
    from typing import TypedDict

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""
        state_counter: int


    counter_node_in_subgraph = 0

    def node_in_subgraph(state: State):
        """A node in the sub-graph."""
        global counter_node_in_subgraph
        counter_node_in_subgraph += 1  # This code will **NOT** run again!
        print(f"Entered `node_in_subgraph` a total of {counter_node_in_subgraph} times")

    counter_human_node = 0

    def human_node(state: State):
        global counter_human_node
        counter_human_node += 1 # This code will run again!
        print(f"Entered human_node in sub-graph a total of {counter_human_node} times")
        answer = interrupt("what is your name?")
        print(f"Got an answer of {answer}")


    checkpointer = InMemorySaver()

    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node("some_node", node_in_subgraph)
    subgraph_builder.add_node("human_node", human_node)
    subgraph_builder.add_edge(START, "some_node")
    subgraph_builder.add_edge("some_node", "human_node")
    subgraph = subgraph_builder.compile(checkpointer=checkpointer)


    counter_parent_node = 0

    def parent_node(state: State):
        """This parent node will invoke the subgraph."""
        global counter_parent_node

        counter_parent_node += 1 # This code will run again on resuming!
        print(f"Entered `parent_node` a total of {counter_parent_node} times")

        # Please note that we're intentionally incrementing the state counter
        # in the graph state as well to demonstrate that the subgraph update
        # of the same key will not conflict with the parent graph (until
        subgraph_state = subgraph.invoke(state)
        return subgraph_state


    builder = StateGraph(State)
    builder.add_node("parent_node", parent_node)
    builder.add_edge(START, "parent_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
          "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"state_counter": 1}, config):
        print(chunk)

    print('--- Resuming ---')

    for chunk in graph.stream(Command(resume="35"), config):
        print(chunk)
    ```

    这将打印出：

    ```pycon
    Entered `parent_node` a total of 1 times
    Entered `node_in_subgraph` a total of 1 times
    Entered human_node in sub-graph a total of 1 times
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    --- Resuming ---
    Entered `parent_node` a total of 2 times
    Entered human_node in sub-graph a total of 2 times
    Got an answer of 35
    {'parent_node': {'state_counter': 1}}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";
    import { z } from "zod";

    const StateAnnotation = z.object({
      stateCounter: z.number(),
    });

    // Global variable to track the number of attempts
    let counterNodeInSubgraph = 0;

    function nodeInSubgraph(state: z.infer<typeof StateAnnotation>) {
      // A node in the sub-graph.
      counterNodeInSubgraph += 1; // This code will **NOT** run again!
      console.log(`Entered 'nodeInSubgraph' a total of ${counterNodeInSubgraph} times`);
      return {};
    }

    let counterHumanNode = 0;

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      counterHumanNode += 1; // This code will run again!
      console.log(`Entered humanNode in sub-graph a total of ${counterHumanNode} times`);
      const answer = interrupt("what is your name?");
      console.log(`Got an answer of ${answer}`);
      return {};
    }

    const checkpointer = new MemorySaver();

    const subgraphBuilder = new StateGraph(StateAnnotation)
      .addNode("someNode", nodeInSubgraph)
      .addNode("humanNode", humanNode)
      .addEdge(START, "someNode")
      .addEdge("someNode", "humanNode");
    const subgraph = subgraphBuilder.compile({ checkpointer });

    let counterParentNode = 0;

    async function parentNode(state: z.infer<typeof StateAnnotation>) {
      // This parent node will invoke the subgraph.
      counterParentNode += 1; // This code will run again on resuming!
      console.log(`Entered 'parentNode' a total of ${counterParentNode} times`);

      // Please note that we're intentionally incrementing the state counter
      // in the graph state as well to demonstrate that the subgraph update
      // of the same key will not conflict with the parent graph (until
      const subgraphState = await subgraph.invoke(state);
      return subgraphState;
    }

    const builder = new StateGraph(StateAnnotation)
      .addNode("parentNode", parentNode)
      .addEdge(START, "parentNode");

    // A checkpointer must be enabled for interrupts to work!
    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: uuidv4(),
      }
    };

    const stream = await graph.stream({ stateCounter: 1 }, config);
    for await (const chunk of stream) {
      console.log(chunk);
    }

    console.log('--- Resuming ---');

    const resumeStream = await graph.stream(new Command({ resume: "35" }), config);
    for await (const chunk of resumeStream) {
      console.log(chunk);
    }
    ```

    这将打印出：

    ```
    Entered 'parentNode' a total of 1 times
    Entered 'nodeInSubgraph' a total of 1 times
    Entered humanNode in sub-graph a total of 1 times
    { __interrupt__: [{ value: 'what is your name?', resumable: true, ns: ['parentNode:4c3a0248-21f0-1287-eacf-3002bc304db4', 'humanNode:2fe86d52-6f70-2a3f-6b2f-b1eededd6348'], when: 'during' }] }
    --- Resuming ---
    Entered 'parentNode' a total of 2 times
    Entered humanNode in sub-graph a total of 2 times
    Got an answer of 35
    { parentNode: null }
    ```
    :::

