# 如何使用 requirements.txt 设置 LangGraph 应用程序 (How to Set Up a LangGraph Application with requirements.txt)

必须使用 [LangGraph 配置文件](../reference/cli.md#configuration-file) 配置 LangGraph 应用程序，以便将其部署到 LangGraph Platform（或进行自托管）。本操作指南讨论了使用 `requirements.txt` 指定项目依赖项来设置 LangGraph 应用程序以进行部署的基本步骤。

本演练基于 [此存储库](https://github.com/langchain-ai/langgraph-example)，您可以试用它以了解有关如何设置 LangGraph 应用程序以进行部署的更多信息。

!!! tip "使用 pyproject.toml 设置"
    如果您更喜欢使用 poetry 进行依赖项管理，请查看 [此操作指南](./setup_pyproject.md) 关于将 `pyproject.toml` 用于 LangGraph Platform。

!!! tip "使用 Monorepo 设置"
    如果您有兴趣部署位于 monorepo 内的图，请查看 [此存储库](https://github.com/langchain-ai/langgraph-example-monorepo) 以获取有关如何执行此操作的示例。

最终的存储库结构将如下所示：

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

每一步之后，都会提供一个示例文件目录来演示如何组织代码。

## 指定依赖项 (Specify Dependencies)

可以选择在以下文件之一中指定依赖项：`pyproject.toml`、`setup.py` 或 `requirements.txt`。如果未创建任何这些文件，则可以在 [LangGraph 配置文件](#create-langgraph-configuration-file) 中稍后指定依赖项。

以下依赖项将包含在镜像中，您也可以在代码中使用它们，只要版本范围兼容即可：

```
langgraph>=0.3.27
langgraph-sdk>=0.1.66
langgraph-checkpoint>=2.0.23
langchain-core>=0.2.38
langsmith>=0.1.63
orjson>=3.9.7,<3.10.17
httpx>=0.25.0
tenacity>=8.0.0
uvicorn>=0.26.0
sse-starlette>=2.1.0,<2.2.0
uvloop>=0.18.0
httptools>=0.5.0
jsonschema-rs>=0.20.0
structlog>=24.1.0
cloudpickle>=3.0.0
```

示例 `requirements.txt` 文件：

```
langgraph
langchain_anthropic
tavily-python
langchain_community
langchain_openai

```

示例文件目录：

```bash
my-app/
├── my_agent # all project code lies within here
│   └── requirements.txt # package dependencies
```

## 指定环境变量 (Specify Environment Variables)

可以选择在文件（例如 `.env`）中指定环境变量。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
```

示例文件目录：

```bash
my-app/
├── my_agent # all project code lies within here
│   └── requirements.txt # package dependencies
└── .env # environment variables
```

## 定义图 (Define Graphs)

实现您的图！图可以在单个文件或多个文件中定义。记下要包含在 LangGraph 应用程序中的每个 @[CompiledStateGraph][CompiledStateGraph] 的变量名称。这些变量名称稍后将在创建 [LangGraph 配置文件](../reference/cli.md#configuration-file) 时使用。

示例 `agent.py` 文件，显示如何从您定义的其他模块导入（此处未显示模块的代码，请参阅 [此存储库](https://github.com/langchain-ai/langgraph-example) 以查看其实陈）：

```python
# my_agent/agent.py
from typing import Literal
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, END, START
from my_agent.utils.nodes import call_model, should_continue, tool_node # import nodes
from my_agent.utils.state import AgentState # import state

# Define the runtime context
class GraphContext(TypedDict):
    model_name: Literal["anthropic", "openai"]

workflow = StateGraph(AgentState, context_schema=GraphContext)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)
workflow.add_edge("action", "agent")

graph = workflow.compile()
```

示例文件目录：

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
└── .env # environment variables
```

## 创建 LangGraph 配置文件 (Create LangGraph Configuration File)

创建一个名为 `langgraph.json` 的 [LangGraph 配置文件](../reference/cli.md#configuration-file)。有关配置文件的 JSON 对象中每个键的详细说明，请参阅 [LangGraph 配置文件参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "dependencies": ["./my_agent"],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名称出现在顶级 `graphs` 键中每个子键值的末尾（即 `:<variable_name>`）。

!!! warning "Configuration File Location"
    LangGraph 配置文件必须放置在包含已编译图及相关依赖项的 Python 文件的同一级别或更高级别的目录中。

示例文件目录：

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   └── agent.py # code for constructing your graph
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

## 下一步 (Next)

设置项目并将其放置在 GitHub 存储库中后，就可以 [部署您的应用程序](./cloud.md) 了。
