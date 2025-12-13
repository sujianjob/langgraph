# 流式 API (Streaming API)

[LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 允许您从 LangGraph API 服务器 [流式传输输出](../../concepts/streaming.md)。

!!! note

    LangGraph SDK 和 LangGraph Server 是 [LangGraph Platform](../../concepts/langgraph_platform.md) 的一部分。

## 基本用法 (Basic usage)

基本用法示例：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # create a streaming run
    # highlight-next-line
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // create a streaming run
    // highlight-next-line
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Create a streaming run:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <inputs>,
      \"stream_mode\": \"updates\"
    }"
    ```

??? example "Extended example: streaming updates"

    这是一个您可以在 LangGraph API 服务器中运行的示例图。
    有关更多详细信息，请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

    ```python
    # graph.py
    from typing import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        topic: str
        joke: str

    def refine_topic(state: State):
        return {"topic": state["topic"] + " and cats"}

    def generate_joke(state: State):
        return {"joke": f"This is a joke about {state['topic']}"}

    graph = (
        StateGraph(State)
        .add_node(refine_topic)
        .add_node(generate_joke)
        .add_edge(START, "refine_topic")
        .add_edge("refine_topic", "generate_joke")
        .add_edge("generate_joke", END)
        .compile()
    )
    ```

    一旦您有一个运行的 LangGraph API 服务器，您可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 与之交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # create a streaming run
        # highlight-next-line
        async for chunk in client.runs.stream(  # (1)!
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"  # (2)!
        ):
            print(chunk.data)
        ```

        1. `client.runs.stream()` 方法返回一个生成流式输出的迭代器。
        2. 设置 `stream_mode="updates"` 以仅流式传输每个节点后对图状态的更新。其他流模式也可用。有关详细信息，请参阅 [支持的流模式](#supported-stream-modes)。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        # Using the graph deployed with the name "agent"
        const assistantID = "agent";

        # create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        # create a streaming run
        # highlight-next-line
        const streamResponse = client.runs.stream(  # (1)!
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            # highlight-next-line
            streamMode: "updates"  # (2)!
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

        1. `client.runs.stream()` 方法返回一个生成流式输出的迭代器。
        2. 设置 `streamMode: "updates"` 以仅流式传输每个节点后对图状态的更新。其他流模式也可用。有关详细信息，请参阅 [支持的流模式](#supported-stream-modes)。

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Create a streaming run:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

    ```output
    {'run_id': '1f02c2b3-3cef-68de-b720-eec2a4a8e920', 'attempt': 1}
    {'refine_topic': {'topic': 'ice cream and cats'}}
    {'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}
    ```


### 支持的流模式 (Supported stream modes)

| 模式 (Mode)                             | 描述 (Description)                                                                                                                                                                         | LangGraph 库方法 (LangGraph Library Method)                                                                                 |
|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| [`values`](#stream-graph-state)  | 每个 [超级步骤](../../concepts/low_level.md#graphs) 后流式传输完整的图状态。                                                                                            | `.stream()` / `.astream()` with [`stream_mode="values"`](../../how-tos/streaming.md#stream-graph-state)  |
| [`updates`](#stream-graph-state) | 流式传输图的每个步骤后对状态的更新。如果在同一步骤中进行了多次更新（例如，运行了多个节点），则这些更新将分别流式传输。 | `.stream()` / `.astream()` with [`stream_mode="updates"`](../../how-tos/streaming.md#stream-graph-state) |
| [`messages-tuple`](#messages)    | 对于调用 LLM 的图节点，流式传输 LLM 令牌和元数据（对聊天应用程序很有用）。                                                                                 | `.stream()` / `.astream()` with [`stream_mode="messages"`](../../how-tos/streaming.md#messages)          |
| [`debug`](#debug)                | 在图的执行过程中流式传输尽可能多的信息。                                                                                                      | `.stream()` / `.astream()` with [`stream_mode="debug"`](../../how-tos/streaming.md#stream-graph-state)   |
| [`custom`](#stream-custom-data)  | 从图内部流式传输自定义数据                                                                                                                                          | `.stream()` / `.astream()` with [`stream_mode="custom"`](../../how-tos/streaming.md#stream-custom-data)  |
| [`events`](#stream-events)       | 流式传输所有事件（包括图的状态）；在迁移大型 LCEL 应用程序时主要有用。                                                                                 | `.astream_events()`                                                                                      |

### 流式传输多种模式 (Stream multiple modes)

您可以传递一个列表作为 `stream_mode` 参数，一次流式传输多种模式。

流式输出将是 `(mode, chunk)` 元组，其中 `mode` 是流模式的名称，`chunk` 是该模式流式传输的数据。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode=["updates", "custom"]
    ):
        print(chunk)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: ["updates", "custom"]
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": <inputs>,
       \"stream_mode\": [
         \"updates\"
         \"custom\"
       ]
     }"
    ```

## 流式图状态 (Stream graph state)

使用流模式 `updates` 和 `values` 在图执行时流式传输图的状态。

* `updates` 流式传输每个步骤后对状态的 **更新 (updates)**。
* `values` 流式传输每个步骤后状态的 **完整值 (full value)**。

??? example "Example graph"

    ```python
    from typing import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
      topic: str
      joke: str

    def refine_topic(state: State):
        return {"topic": state["topic"] + " and cats"}

    def generate_joke(state: State):
        return {"joke": f"This is a joke about {state['topic']}"}

    graph = (
      StateGraph(State)
      .add_node(refine_topic)
      .add_node(generate_joke)
      .add_edge(START, "refine_topic")
      .add_edge("refine_topic", "generate_joke")
      .add_edge("generate_joke", END)
      .compile()
    )
    ```

!!! note "Stateful runs"

    下面的示例假设您希望将流式传输运行的 **输出持久化 (persist the outputs)** 在 [检查点](../../concepts/persistence.md) 数据库中，并已创建一个线程。要创建线程：

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"
        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
        ```

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        # Using the graph deployed with the name "agent"
        const assistantID = "agent";
        # create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"]
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

    如果您不需要持久化运行的输出，则在流式传输时可以传递 `None` 而不是 `thread_id`。

=== "updates"

    使用此选项仅流式传输每个步骤后节点返回的 **状态更新**。流式输出包括节点名称以及更新。

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            # highlight-next-line
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

===  "values"

    使用此选项流式传输每个步骤后图的 **完整状态**。

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="values"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            # highlight-next-line
            streamMode: "values"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"values\"
        }"
        ```


## 子图 (Subgraphs)

要在流式输出中包含来自 [子图](../../concepts/subgraphs.md) 的输出，可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输来自父图和任何子图的输出。

```python
for chunk in client.runs.stream(
    thread_id,
    assistant_id,
    input={"foo": "foo"},
    # highlight-next-line
    stream_subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `stream_subgraphs=True` 以流式传输来自子图的输出。

??? example "Extended example: streaming from subgraphs"

    这是一个您可以在 LangGraph API 服务器中运行的示例图。
    有关更多详细信息，请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

    ```python
    # graph.py
    from langgraph.graph import START, StateGraph
    from typing import TypedDict

    # Define subgraph
    class SubgraphState(TypedDict):
        foo: str  # note that this key is shared with the parent graph state
        bar: str

    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}

    def subgraph_node_2(state: SubgraphState):
        return {"foo": state["foo"] + state["bar"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Define parent graph
    class ParentState(TypedDict):
        foo: str

    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}

    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", subgraph)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()
    ```

    一旦您有一个运行的 LangGraph API 服务器，您可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 与之交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
    
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"foo": "foo"},
            # highlight-next-line
            stream_subgraphs=True, # (1)!
            stream_mode="updates",
        ):
            print(chunk)
        ```
        
        1. 设置 `stream_subgraphs=True` 以流式传输来自子图的输出。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        # Using the graph deployed with the name "agent"
        const assistantID = "agent";

        # create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        # create a streaming run
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { foo: "foo" },
            # highlight-next-line
            streamSubgraphs: true,  # (1)!
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk);
        }
        ```

        1. 设置 `streamSubgraphs: true` 以流式传输来自子图的输出。

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Create a streaming run:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"foo\": \"foo\"},
          \"stream_subgraphs\": true,
          \"stream_mode\": [
            \"updates\"
          ]
        }"
        ```

    **注意**，我们不仅接收节点更新，还接收命名空间，这告诉我们正在从哪个图（或子图）进行流式传输。

## 调试 (Debugging) {#debug}

使用 `debug` 流模式在图的执行过程中流式传输尽可能多的信息。流式输出包括节点名称以及完整状态。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="debug"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        # highlight-next-line
        streamMode: "debug"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"debug\"
    }"
    ```

## LLM 令牌 (LLM tokens) {#messages}

使用 `messages-tuple` 流模式从图的任何部分（包括节点、工具、子图或任务）**逐个令牌 (token by token)** 流式传输大型语言模型 (LLM) 输出。

[`messages-tuple` 模式](#supported-stream-modes) 的流式输出是一个元组 `(message_chunk, metadata)`，其中：

- `message_chunk`: 来自 LLM 的令牌或消息段。
- `metadata`: 包含有关图节点和 LLM 调用的详细信息的字典。
 
??? example "Example graph"

    ```python
    from dataclasses import dataclass

    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START

    @dataclass
    class MyState:
        topic: str
        joke: str = ""

    llm = init_chat_model(model="openai:gpt-4o-mini")

    def call_model(state: MyState):
        """Call the LLM to generate a joke about a topic"""
        # highlight-next-line
        llm_response = llm.invoke( # (1)!
            [
                {"role": "user", "content": f"Generate a joke about {state.topic}"}
            ]
        )
        return {"joke": llm_response.content}

    graph = (
        StateGraph(MyState)
        .add_node(call_model)
        .add_edge(START, "call_model")
        .compile()
    )
    ```

    1. 请注意，即使使用 `.invoke` 而不是 `.stream` 运行 LLM，也会发出消息事件。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="messages-tuple",
    ):
        if chunk.event != "messages":
            continue

        message_chunk, metadata = chunk.data  # (1)!
        if message_chunk["content"]:
            print(message_chunk["content"], end="|", flush=True)
    ```

    1. "messages-tuple" 流模式返回 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的令牌，`metadata` 是包含有关调用 LLM 的图节点信息和其他信息的字典。

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        # highlight-next-line
        streamMode: "messages-tuple"
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.event !== "messages") {
        continue;
      }
      console.log(chunk.data[0]["content"]);  # (1)!
    }
    ```

    1. "messages-tuple" 流模式返回 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的令牌，`metadata` 是包含有关调用 LLM 的图节点信息和其他信息的字典。

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"messages-tuple\"
    }"
    ```

### 过滤 LLM 令牌 (Filter LLM tokens)

* 要按 LLM 调用过滤流式传输的令牌，您可以 [将 `tags` 与 LLM 调用相关联](../../how-tos/streaming.md#filter-by-llm-invocation)。
* 要仅从特定节点流式传输令牌，请使用 `stream_mode="messages"` 并 [通过流式元数据中的 `langgraph_node` 字段过滤输出](../../how-tos/streaming.md#filter-by-node)。

## 流式自定义数据 (Stream custom data)

发送 **自定义用户定义数据**：

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"query": "example"},
        # highlight-next-line
        stream_mode="custom"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { query: "example" },
        # highlight-next-line
        streamMode: "custom"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"query\": \"example\"},
      \"stream_mode\": \"custom\"
    }"
    ```

## 流式事件 (Stream events)

流式传输所有事件，包括图的状态：

=== "Python"

    ```python
    async for chunk in client.runs.list_events(
        thread_id,
        run_id,
    ):
        print(chunk)
    ```
