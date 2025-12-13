# 如何传递自定义运行 ID 或为 LangSmith 中的图运行设置标签和元数据

!!! tip "先决条件 (Prerequisites)"
    本指南假定您熟悉以下内容：
    
    - [LangSmith 文档](https://docs.smith.langchain.com)
    - [LangSmith 平台](https://smith.langchain.com)
    - [RunnableConfig](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.config.RunnableConfig.html#langchain_core.runnables.config.RunnableConfig)
    - [向追踪添加元数据和标签](https://docs.smith.langchain.com/how_to_guides/tracing/trace_with_langchain#add-metadata-and-tags-to-traces)
    - [自定义运行名称](https://docs.smith.langchain.com/how_to_guides/tracing/trace_with_langchain#customize-run-name)

在 IDE 或终端中调试图运行有时会很困难。[LangSmith](https://docs.smith.langchain.com) 允许您使用追踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用程序 — 阅读 [LangSmith 文档](https://docs.smith.langchain.com) 以获取有关如何开始的更多信息。

为了更容易地识别和分析图调用期间生成的追踪，您可以在运行时设置其他配置（请参阅 [RunnableConfig](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.config.RunnableConfig.html#langchain_core.runnables.config.RunnableConfig)）：

| **字段**   | **类型**            | **描述**                                                                                                    |
|-------------|---------------------|--------------------------------------------------------------------------------------------------------------------|
| run_name    | `str`               | 此调用的追踪器运行名称。默认为类的名称。                                          |
| run_id      | `UUID`              | 此调用的追踪器运行的唯一标识符。如果未提供，将生成一个新的 UUID。                 |
| tags        | `List[str]`         | 此调用及任何子调用（例如，调用 LLM 的链）的标签。您可以使用这些来过滤调用。            |
| metadata    | `Dict[str, Any]`    | 此调用及任何子调用（例如，调用 LLM 的链）的元数据。键应为字符串，值应为 JSON 可序列化的。 |

LangGraph 图实现了 [LangChain Runnable 接口](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.Runnable.html)，并在 `invoke`、`ainvoke`、`stream` 等方法中接受第二个参数（`RunnableConfig`）。

LangSmith 平台允许您基于 `run_name`、`run_id`、`tags` 和 `metadata` 搜索和过滤追踪。

## TLDR

```python
import uuid
# Generate a random UUID -- it must be a UUID
config = {"run_id": uuid.uuid4(), "tags": ["my_tag1"], "metadata": {"a": 5}}
# Works with all standard Runnable methods 
# like invoke, batch, ainvoke, astream_events etc
graph.stream(inputs, config, stream_mode="values")
```

本操作指南的其余部分将展示一个完整的智能体。

## 设置 (Setup)

首先，让我们安装所需的包并设置我们的 API 密钥

```python
%%capture --no-stderr
%pip install --quiet -U langgraph langchain_openai
```

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("OPENAI_API_KEY")
_set_env("LANGSMITH_API_KEY")
```

!!! tip
    注册 LangSmith 以快速发现问题并提高 LangGraph 项目的性能。[LangSmith](https://docs.smith.langchain.com) 允许您使用追踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用程序 — 在 [此处](https://docs.smith.langchain.com) 阅读更多关于如何开始的信息。

## 定义图 (Define the graph)

对于此示例，我们将使用 [预构建的 ReAct 智能体](https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/)。

```python
from langchain_openai import ChatOpenAI
from typing import Literal
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

# First we initialize the model we want to use.
model = ChatOpenAI(model="gpt-4o", temperature=0)


# For this tutorial we will use custom tool that returns pre-defined values for weather in two cities (NYC & SF)
@tool
def get_weather(city: Literal["nyc", "sf"]):
    """Use this to get weather information."""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")


tools = [get_weather]


# Define the graph
graph = create_react_agent(model, tools=tools)
```

## 运行您的图 (Run your graph)

现在我们已经定义了图，让我们运行它一次并在 LangSmith 中查看追踪。为了使我们的追踪在 LangSmith 中易于访问，我们将在配置中传入自定义 `run_id`。

这假设您已设置 `LANGSMITH_API_KEY` 环境变量。

请注意，您还可以通过设置 `LANGCHAIN_PROJECT` 环境变量来配置要追踪到的项目，默认情况下，运行将被追踪到 `default` 项目。

```python
import uuid


def print_stream(stream):
    for s in stream:
        message = s["messages"][-1]
        if isinstance(message, tuple):
            print(message)
        else:
            message.pretty_print()


inputs = {"messages": [("user", "what is the weather in sf")]}

config = {"run_name": "agent_007", "tags": ["cats are awesome"]}

print_stream(graph.stream(inputs, config, stream_mode="values"))
```

**Output:**
```
================================ Human Message ==================================

what is the weather in sf
================================== Ai Message ===================================
Tool Calls:
  get_weather (call_9ZudXyMAdlUjptq9oMGtQo8o)
 Call ID: call_9ZudXyMAdlUjptq9oMGtQo8o
  Args:
    city: sf
================================= Tool Message ==================================
Name: get_weather

It's always sunny in sf
================================== Ai Message ===================================

The weather in San Francisco is currently sunny.
```

## 在 LangSmith 中查看追踪 (View the trace in LangSmith)

现在我们已经运行了图，让我们前往 LangSmith 查看我们的追踪。首先点击您追踪到的项目（在我们的例子中是默认项目）。您应该能看到一个具有自定义运行名称 "agent_007" 的运行。

![LangSmith Trace View](assets/d38d1f2b-0f4c-4707-b531-a3c749de987f.png)

此外，您还可以使用提供的标签或元数据在事后过滤追踪。例如，

![LangSmith Filter View](assets/410e0089-2ab8-46bb-a61a-827187fd46b3.png)
