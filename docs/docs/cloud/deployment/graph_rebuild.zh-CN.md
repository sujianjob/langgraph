# 在运行时重建图 (Rebuild Graph at Runtime)

您可能需要使用不同的配置为新的运行重建图。例如，您可能需要根据配置使用不同的图状态或图结构。本指南向您展示如何执行此操作。

!!! note "Note"
    在大多数情况下，基于配置的自定义行为应由单个图处理，其中每个节点都可以读取配置并根据该配置更改其行为。

## 先决条件 (Prerequisites)

请务必先查看 [关于设置应用程序以进行部署的操作指南](./setup.md)。

## 定义图 (Define graphs)

假设您有一个应用程序，其中包含一个简单的图，该图调用 LLM 并将响应返回给用户。应用程序文件目录如下所示：

```
my-app/
|-- requirements.txt
|-- .env
|-- openai_agent.py     # code for your graph
```

其中图在 `openai_agent.py` 中定义。

### 无重建 (No rebuild)

在标准的 LangGraph API 配置中，服务器使用定义在 `openai_agent.py` 顶层的已编译图实例，如下所示：

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph

model = ChatOpenAI(temperature=0)

graph_workflow = MessageGraph()

graph_workflow.add_node("agent", model)
graph_workflow.add_edge("agent", END)
graph_workflow.add_edge(START, "agent")

agent = graph_workflow.compile()
```

为了让服务器知道您的图，您需要在 LangGraph API 配置 (`langgraph.json`) 中指定包含 `CompiledStateGraph` 实例的变量的路径，例如：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env"
}
```

### 重建 (Rebuild)

为了让您的图在每次使用自定义配置的新运行时重建，您需要重写 `openai_agent.py`，以提供一个 _函数_，该函数接受配置并返回图（或已编译的图）实例。假设我们想为用户 ID '1' 返回现有的图，并为其他用户返回一个工具调用代理。我们可以修改 `openai_agent.py` 如下：

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph
from langgraph.graph.state import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool
from langchain_core.messages import BaseMessage
from langchain_core.runnables import RunnableConfig


class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


model = ChatOpenAI(temperature=0)

def make_default_graph():
    """Make a simple LLM agent"""
    graph_workflow = StateGraph(State)
    def call_model(state):
        return {"messages": [model.invoke(state["messages"])]}

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_edge("agent", END)
    graph_workflow.add_edge(START, "agent")

    agent = graph_workflow.compile()
    return agent


def make_alternative_graph():
    """Make a tool-calling agent"""

    @tool
    def add(a: float, b: float):
        """Adds two numbers."""
        return a + b

    tool_node = ToolNode([add])
    model_with_tools = model.bind_tools([add])
    def call_model(state):
        return {"messages": [model_with_tools.invoke(state["messages"])]}

    def should_continue(state: State):
        if state["messages"][-1].tool_calls:
            return "tools"
        else:
            return END

    graph_workflow = StateGraph(State)

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_node("tools", tool_node)
    graph_workflow.add_edge("tools", "agent")
    graph_workflow.add_edge(START, "agent")
    graph_workflow.add_conditional_edges("agent", should_continue)

    agent = graph_workflow.compile()
    return agent


# this is the graph making function that will decide which graph to
# build based on the provided config
def make_graph(config: RunnableConfig):
    user_id = config.get("configurable", {}).get("user_id")
    # route to different graph state / structure based on the user ID
    if user_id == "1":
        return make_default_graph()
    else:
        return make_alternative_graph()
```

最后，您需要在 `langgraph.json` 中指定图制作函数 (`make_graph`) 的路径：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:make_graph",
    },
    "env": "./.env"
}
```

在此处查看有关 LangGraph API 配置文件的更多信息 [这里](../reference/cli.md#configuration-file)
