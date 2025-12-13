# 使用服务器 API 进行人机回环 (Human-in-the-loop using Server API)

要在代理或工作流中审查、编辑和批准工具调用，请使用 LangGraph 的 [人机回环](../../concepts/human_in_the_loop.md) 功能。

## 动态中断 (Dynamic interrupts)

=== "Python"

    ```python
    from langgraph_sdk import get_client
    # highlight-next-line
    from langgraph_sdk.schema import Command
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph until the interrupt is hit.
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"some_text": "original text"}   # (1)!
    )

    print(result['__interrupt__']) # (2)!
    # > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'id': '...',
    # >     }
    # > ]


    # Resume the graph
    print(await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        command=Command(resume="Edited text")   # (3)!
    ))
    # > {'some_text': 'Edited text'}
    ```

    1. 图以某个初始状态被调用。
    2. 当图命中中断时，它返回一个包含有效负载和元数据的中断对象。
    3. 图通过 `Command(resume=...)` 恢复，注入人类的输入并继续执行。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // Run the graph until the interrupt is hit.
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: { "some_text": "original text" } }   // (1)!
    );

    console.log(result['__interrupt__']); // (2)!
    // > [
    // >     {
    // >         'value': {'text_to_revise': 'original text'},
    // >         'resumable': True,
    // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    // >         'when': 'during'
    // >     }
    // > ]

    // Resume the graph
    console.log(await client.runs.wait(
        threadID,
        assistantID,
        # highlight-next-line
        { command: { resume: "Edited text" }}   // (3)!
    ));
    // > {'some_text': 'Edited text'}
    ```

    1. 图以某个初始状态被调用。
    2. 当图命中中断时，它返回一个包含有效负载和元数据的中断对象。
    3. 图通过 `{ resume: ... }` 命令对象恢复，注入人类的输入并继续执行。

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph until the interrupt is hit.:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"some_text\": \"original text\"}
    }"
    ```

    Resume the graph:

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"command\": {
         \"resume\": \"Edited text\"
       }
     }"
    ```

??? example "Extended example: using `interrupt`"

    这是一个您可以在 LangGraph API 服务器中运行的示例图。
    有关更多详细信息，请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

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
        value = interrupt( # (1)!
            {
                "text_to_revise": state["some_text"] # (2)!
            }
        )
        return {
            "some_text": value # (3)!
        }


    # Build the graph
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")

    graph = graph_builder.compile()
    ```

    1. `interrupt(...)` 在 `human_node` 暂停执行，将给定的有效负载呈现给人类。
    2. 任何 JSON 可序列化的值都可以传递给 `interrupt` 函数。在这里，是一个包含要修改的文本的字典。
    3. 一旦恢复，`interrupt(...)` 的返回值就是人类提供的输入，用于更新状态。

    一旦您有一个运行的 LangGraph API 服务器，您可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 与之交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        # highlight-next-line
        from langgraph_sdk.schema import Command
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # Run the graph until the interrupt is hit.
        result = await client.runs.wait(
            thread_id,
            assistant_id,
            input={"some_text": "original text"}   # (1)!
        )

        print(result['__interrupt__']) # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'id': '...',
        # >     }
        # > ]


        # Resume the graph
        print(await client.runs.wait(
            thread_id,
            assistant_id,
            # highlight-next-line
            command=Command(resume="Edited text")   # (3)!
        ))
        # > {'some_text': 'Edited text'}
        ```

        1. 图以某个初始状态被调用。
        2. 当图命中中断时，它返回一个包含有效负载和元数据的中断对象。
        3. 图通过 `Command(resume=...)` 恢复，注入人类的输入并继续执行。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        # Using the graph deployed with the name "agent"
        const assistantID = "agent";

        # create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        # Run the graph until the interrupt is hit.
        const result = await client.runs.wait(
          threadID,
          assistantID,
          { input: { "some_text": "original text" } }   # (1)!
        );

        console.log(result['__interrupt__']); # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'resumable': True,
        # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        # >         'when': 'during'
        # >     }
        # > ]

        # Resume the graph
        console.log(await client.runs.wait(
            threadID,
            assistantID,
            # highlight-next-line
            { command: { resume: "Edited text" }}   # (3)!
        ));
        # > {'some_text': 'Edited text'}
        ```

        1. 图以某个初始状态被调用。
        2. 当图命中中断时，它返回一个包含有效负载和元数据的中断对象。
        3. 图通过 `{ resume: ... }` 命令对象恢复，注入人类的输入并继续执行。

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Run the graph until the interrupt is hit:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"some_text\": \"original text\"}
        }"
        ```

        Resume the graph:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"command\": {
            \"resume\": \"Edited text\"
          }
        }"
        ```

## 静态中断 (Static interrupts)

静态中断（也称为静态断点）在节点执行之前或之后触发。

!!! warning

    静态中断 **不** 推荐用于人机回环工作流。它们最适合用于调试和测试。

您可以通过在编译时指定 `interrupt_before` 和 `interrupt_after` 来设置静态中断：

```python
# highlight-next-line
graph = graph_builder.compile( # (1)!
    # highlight-next-line
    interrupt_before=["node_a"], # (2)!
    # highlight-next-line
    interrupt_after=["node_b", "node_c"], # (3)!
)
```

1. 断点在 `compile`（编译）时设置。
2. `interrupt_before` 指定在节点执行之前应暂停执行的节点。
3. `interrupt_after` 指定在节点执行之后应暂停执行的节点。

或者，您可以在运行时设置静态中断：

=== "Python"

    ```python
    # highlight-next-line
    await client.runs.wait( # (1)!
        thread_id,
        assistant_id,
        inputs=inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
    )
    ```

    1. 调用 `client.runs.wait` 并带有 `interrupt_before` 和 `interrupt_after` 参数。这是运行时配置，可以为每次调用进行更改。
    2. `interrupt_before` 指定在节点执行之前应暂停执行的节点。
    3. `interrupt_after` 指定在节点执行之后应暂停执行的节点。

=== "JavaScript"

    ```js
    # highlight-next-line
    await client.runs.wait( # (1)!
        threadID,
        assistantID,
        {
        input: input,
        # highlight-next-line
        interruptBefore: ["node_a"], # (2)!
        # highlight-next-line
        interruptAfter: ["node_b", "node_c"] # (3)!
        }
    )
    ```

    1. 调用 `client.runs.wait` 并带有 `interruptBefore` 和 `interruptAfter` 参数。这是运行时配置，可以为每次调用进行更改。
    2. `interruptBefore` 指定在节点执行之前应暂停执行的节点。
    3. `interruptAfter` 指定在节点执行之后应暂停执行的节点。

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
        \"assistant_id\": \"agent\",
        \"interrupt_before\": [\"node_a\"],
        \"interrupt_after\": [\"node_b\", \"node_c\"],
        \"input\": <INPUT>
    }"
    ```

以下示例显示如何添加静态中断：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph until the breakpoint
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input=inputs   # (1)!
    )

    # Resume the graph
    await client.runs.wait(
        thread_id,
        assistant_id,
        input=None   # (2)!
    )
    ```

    1. 运行图直到遇到第一个断点。
    2. 通过传入 `None` 作为输入来恢复图。这将运行图直到遇到下一个断点。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    # Using the graph deployed with the name "agent"
    const assistantID = "agent";

    # create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    # Run the graph until the breakpoint
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: input }   # (1)!
    );

    # Resume the graph
    await client.runs.wait(
      threadID,
      assistantID,
      { input: null }   # (2)!
    );
    ```

    1. 运行图直到遇到第一个断点。
    2. 通过传入 `null` 作为输入来恢复图。这将运行图直到遇到下一个断点。

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph until the breakpoint:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <INPUT>
    }"
    ```

    Resume the graph:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\"
    }"
    ```


## 学习更多 (Learn more)

- [人机回环概念指南](../../concepts/human_in_the_loop.md): 了解有关 LangGraph 人机回环功能的更多信息。
- [常见模式](../../how-tos/human_in_the_loop/add-human-in-the-loop.md#common-patterns): 了解如何实现批准/拒绝操作、请求用户输入、工具调用审查和验证人工输入等模式。
