# 如何使用远程图 (How to use a remote graph)

`RemoteGraph` 允许您像使用本地的 [CompiledGraph](../concepts/low_level.md#compiledgraph) 一样与 [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) 部署进行交互。

它可以通过其 URL 或显式传入客户端实例进行初始化。

## 使用 URL 初始化 (Initialize with URL)

```python
from langgraph.prebuilt import RemoteGraph

url = "https://..."
graph_name = "my_graph"
remote_graph = RemoteGraph(graph_name, url=url)
```

## 使用客户端初始化 (Initialize with client)

```python
from langgraph.prebuilt import RemoteGraph
from langgraph_sdk import get_client

client = get_client(url="https://...")
graph_name = "my_graph"
remote_graph = RemoteGraph(graph_name, client=client)
```

## 同步调用 (Synchronous invocation)

!!! note
    同步调用需要 `langgraph-sdk>=0.1.34`。

您可以像使用 `CompiledGraph` 一样使用 `invoke` 方法。

```python
remote_graph.invoke({
    "messages": [
        {"role": "user", "content": "What's the weather in SF?"}
    ]
})
```

## 异步调用 (Asynchronous invocation)

您可以像使用 `CompiledGraph` 一样使用 `ainvoke`、`astream` 和 `astream_events` 方法。

```python
await remote_graph.ainvoke({
    "messages": [
        {"role": "user", "content": "What's the weather in SF?"}
    ]
})
```

```python
async for chunk in remote_graph.astream({
    "messages": [
        {"role": "user", "content": "What's the weather in SF?"}
    ]
}):
    print(chunk)
```

```python
async for event in remote_graph.astream_events({
    "messages": [
        {"role": "user", "content": "What's the weather in SF?"}
    ]
}, version="v2"):
    print(event)
```

## 持久化 (Persistence)

`RemoteGraph` 支持线程级持久化，类似于 `CompiledGraph`。

```python
config = {"configurable": {"thread_id": "thread-1"}}

await remote_graph.ainvoke({
    "messages": [
        {"role": "user", "content": "What's the weather in SF?"}
    ]
}, config=config)

await remote_graph.ainvoke({
    "messages": [
        {"role": "user", "content": "And in LA?"}
    ]
}, config=config)
```

## 作为子图使用 (Use as a subgraph)

`RemoteGraph` 可以作为子图在父图中使用。

```python
from langgraph.prebuilt import RemoteGraph
from langgraph.graph import StateGraph, START, END

# Define the remote graph
url = "https://..."
graph_name = "my_graph"
remote_graph = RemoteGraph(graph_name, url=url)

# Define the parent graph
builder = StateGraph(State)
builder.add_node("remote_graph", remote_graph)
builder.add_edge(START, "remote_graph")
builder.add_edge("remote_graph", END)
graph = builder.compile()
```
