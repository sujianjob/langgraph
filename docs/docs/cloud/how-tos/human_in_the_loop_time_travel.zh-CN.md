# 使用服务器 API 进行时间旅行 (Time travel using Server API)

LangGraph 提供 [**时间旅行**](../../concepts/time-travel.md) 功能，以从先前的检查点恢复执行，可以是重播相同的状态，也可以是修改它以探索替代方案。在所有情况下，恢复过去的执行都会在历史记录中产生一个新的分叉。

要使用 LangGraph Server API（通过 LangGraph SDK）进行时间旅行：

1. **运行图**：使用 [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 的 @[`client.runs.wait`][client.runs.wait] 或 @[`client.runs.stream`][client.runs.stream] API 使用初始输入运行图。
2. **识别现有线程中的检查点**：使用 @[`client.threads.get_history`][client.threads.get_history] 方法检索特定 `thread_id` 的执行历史记录并找到所需的 `checkpoint_id`。
   或者，在您希望暂停执行的节点之前设置 [断点](./human_in_the_loop_breakpoint.md)。然后，您可以找到在该断点之前记录的最近检查点。
3. **(可选) 修改图状态**：使用 @[`client.threads.update_state`][client.threads.update_state] 方法在检查点修改图的状态，并从替代状态恢复执行。
4. **从检查点恢复执行**：使用 @[`client.runs.wait`][client.runs.wait] 或 @[`client.runs.stream`][client.runs.stream] API，输入为 `None`，并使用适当的 `thread_id` 和 `checkpoint_id`。

## 在工作流中使用时间旅行 (Use time travel in a workflow)

??? example "Example graph"

    ```python
    from typing_extensions import TypedDict, NotRequired
    from langgraph.graph import StateGraph, START, END
    from langchain.chat_models import init_chat_model
    from langgraph.checkpoint.memory import InMemorySaver

    class State(TypedDict):
        topic: NotRequired[str]
        joke: NotRequired[str]

    llm = init_chat_model(
        "anthropic:claude-3-7-sonnet-latest",
        temperature=0,
    )

    def generate_topic(state: State):
        """LLM call to generate a topic for the joke"""
        msg = llm.invoke("Give me a funny topic for a joke")
        return {"topic": msg.content}

    def write_joke(state: State):
        """LLM call to write a joke based on the topic"""
        msg = llm.invoke(f"Write a short joke about {state['topic']}")
        return {"joke": msg.content}

    # Build workflow
    builder = StateGraph(State)

    # Add nodes
    builder.add_node("generate_topic", generate_topic)
    builder.add_node("write_joke", write_joke)

    # Add edges to connect nodes
    builder.add_edge(START, "generate_topic")
    builder.add_edge("generate_topic", "write_joke")

    # Compile
    graph = builder.compile()
    ```

### 1. 运行图 (Run the graph)

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={}
    )
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // Run the graph
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: {}}
    );
    ```

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {}
    }"
    ```

### 2. 识别检查点 (Identify a checkpoint)

=== "Python"

    ```python
    # The states are returned in reverse chronological order.
    states = await client.threads.get_history(thread_id)
    selected_state = states[1]
    print(selected_state)
    ```

=== "JavaScript"

    ```js
    # The states are returned in reverse chronological order.
    const states = await client.threads.getHistory(threadID);
    const selectedState = states[1];
    console.log(selectedState);
    ```

=== "cURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history \
    --header 'Content-Type: application/json'
    ```

### 3. 更新状态（可选） (Update the state (optional))

`update_state` 将创建一个新检查点。新检查点将与同一线程关联，但具有新的检查点 ID。

=== "Python"

    ```python
    new_config = await client.threads.update_state(
        thread_id,
        {"topic": "chickens"},
        # highlight-next-line
        checkpoint_id=selected_state["checkpoint_id"]
    )
    print(new_config)
    ```

=== "JavaScript"

    ```js
    const newConfig = await client.threads.updateState(
      threadID,
      {
        values: { "topic": "chickens" },
        checkpointId: selectedState["checkpoint_id"]
      }
    );
    console.log(newConfig);
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>,
      \"values\": {\"topic\": \"chickens\"}
    }"
    ```

### 4. 从检查点恢复执行 (Resume execution from the checkpoint)

=== "Python"

    ```python
    await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        input=None,
        # highlight-next-line
        checkpoint_id=new_config["checkpoint_id"]
    )
    ```

=== "JavaScript"

    ```js
    await client.runs.wait(
      threadID,
      assistantID,
      {
        # highlight-next-line
        input: null,
        # highlight-next-line
        checkpointId: newConfig["checkpoint_id"]
      }
    );
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>
    }"
    ```

## 学习更多 (Learn more)

- [**LangGraph 时间旅行指南**](../../how-tos/human_in_the_loop/time-travel.md): 了解有关在 LangGraph 中使用时间旅行的更多信息。
