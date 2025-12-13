---
search:
  boost: 2
---

# 应用程序结构 (Application Structure)

## 概述 (Overview)

LangGraph 应用程序由一个或多个图、一个配置文件 (`langgraph.json`)、一个指定依赖项的文件以及一个指定环境变量的可选 `.env` 文件组成。

本指南展示了应用程序的典型结构，并展示了如何指定使用 LangGraph Platform 部署应用程序所需的信息。

## 关键概念 (Key Concepts)

要使用 LangGraph Platform 进行部署，应提供以下信息：

1. 一个 [LangGraph 配置文件](#configuration-file-concepts) (`langgraph.json`)，用于指定应用程序使用的依赖项、图和环境变量。
2. 实现应用程序逻辑的 [图](#graphs)。
3. 一个指定运行应用程序所需的 [依赖项](#dependencies) 的文件。
4. 应用程序运行所需的 [环境变量](#environment-variables)。

## 文件结构 (File Structure)

以下是应用程序的目录结构示例：

:::python
=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都在这里
    │   ├── utils # 您的图的实用程序
    │   │   ├── __init__.py
    │   │   ├── tools.py # 您的图的工具
    │   │   ├── nodes.py # 您的图的节点函数
    │   │   └── state.py # 您的图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── requirements.txt # 包依赖项
    └── langgraph.json # LangGraph 的配置文件
    ```

=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都在这里
    │   ├── utils # 您的图的实用程序
    │   │   ├── __init__.py
    │   │   ├── tools.py # 您的图的工具
    │   │   ├── nodes.py # 您的图的节点函数
    │   │   └── state.py # 您的图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── langgraph.json  # LangGraph 的配置文件
    └── pyproject.toml # 项目的依赖项
    ```

:::

:::js

```plaintext
my-app/
├── src # 所有项目代码都在这里
│   ├── utils # 您的图的可选实用程序
│   │   ├── tools.ts # 您的图的工具
│   │   ├── nodes.ts # 您的图的节点函数
│   │   └── state.ts # 您的图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

:::

!!! note

    LangGraph 应用程序的目录结构可能因编程语言和使用的包管理器而异。

## 配置文件 (Configuration File) {#configuration-file-concepts}

`langgraph.json` 文件是一个 JSON 文件，用于指定部署 LangGraph 应用程序所需的依赖项、图、环境变量和其他设置。

有关 JSON 文件中所有支持的键的详细信息，请参阅 [LangGraph 配置文件参考](../cloud/reference/cli.md#configuration-file)。

!!! tip

    [LangGraph CLI](./langgraph_cli.md) 默认使用当前目录中的配置文件 `langgraph.json`。

### 示例 (Examples)

:::python

- 依赖项涉及自定义本地包和 `langchain_openai` 包。
- 将从带有变量 `variable` 的文件 `./your_package/your_file.py` 加载单个图。
- 从 `.env` 文件加载环境变量。

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

:::

:::js

- 依赖项将从本地目录中的依赖文件（例如 `package.json`）加载。
- 将从带有函数 `agent` 的文件 `./your_package/your_file.js` 加载单个图。
- 环境变量 `OPENAI_API_KEY` 是内联设置的。

```json
{
  "dependencies": ["."],
  "graphs": {
    "my_agent": "./your_package/your_file.js:agent"
  },
  "env": {
    "OPENAI_API_KEY": "secret-key"
  }
}
```

:::

## 依赖项 (Dependencies)

:::python
LangGraph 应用程序可能依赖于其他 Python 包。
:::

:::js
LangGraph 应用程序可能依赖于其他 TypeScript/JavaScript 库。
:::

您通常需要指定以下信息才能正确设置依赖项：

:::python

1. 目录中指定依赖项的文件（例如 `requirements.txt`、`pyproject.toml` 或 `package.json`）。
   :::

:::js

1. 目录中指定依赖项的文件（例如 `package.json`）。
   :::

2. [LangGraph 配置文件](#configuration-file-concepts) 中的 `dependencies` 键，指定运行 LangGraph 应用程序所需的依赖项。
3. 可以使用 [LangGraph 配置文件](#configuration-file-concepts) 中的 `dockerfile_lines` 键指定任何其他二进制文件或系统库。

## 图 (Graphs)

使用 [LangGraph 配置文件](#configuration-file-concepts) 中的 `graphs` 键指定哪些图将在已部署的 LangGraph 应用程序中可用。

您可以在配置文件中指定一个或多个图。每个图都由一个名称（应该是唯一的）和一个路径标识，该路径指向：(1) 编译图或 (2) 定义图的函数。

## 环境变量 (Environment Variables)

如果您在本地使用已部署的 LangGraph 应用程序，则可以在 [LangGraph 配置文件](#configuration-file-concepts) 的 `env` 键中配置环境变量。

对于生产部署，您通常需要在部署环境中配置环境变量。
