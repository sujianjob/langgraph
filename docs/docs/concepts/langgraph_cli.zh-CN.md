---
search:
  boost: 2
---

# LangGraph CLI

**LangGraph CLI** 是一个多平台命令行工具，用于在本地构建和运行 [LangGraph API 服务器](./langgraph_server.md)。生成的服务器包含图的运行、线程、助手等的所有 API 端点，以及运行代理所需的其他服务，包括用于检查点和存储的托管数据库。

:::python

## 安装 (Installation)

LangGraph CLI 可以通过 pip 或 [Homebrew](https://brew.sh/) 安装：

=== "pip"

    ```bash
    pip install langgraph-cli
    ```

=== "Homebrew"

    ```bash
    brew install langgraph-cli
    ```
:::

:::js

## 安装 (Installation)

LangGraph.js CLI 可以从 NPM 注册表安装：

=== "npx"
    ```bash
    npx @langchain/langgraph-cli
    ```

=== "npm"
    ```bash
    npm install @langchain/langgraph-cli
    ```

=== "yarn"
    ```bash
    yarn add @langchain/langgraph-cli
    ```

=== "pnpm"
    ```bash
    pnpm add @langchain/langgraph-cli
    ```

=== "bun"
    ```bash
    bun add @langchain/langgraph-cli
    ```
:::

## 命令 (Commands)

LangGraph CLI 提供以下核心功能：

| 命令 (Command)                                                 | 描述 (Description)                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`langgraph build`](../cloud/reference/cli.md#build)           | 为 [LangGraph API 服务器](./langgraph_server.md) 构建一个可以直接部署的 Docker 映像。                                                                                                                                                                             |
| [`langgraph dev`](../cloud/reference/cli.md#dev)               | 启动一个不需要安装 Docker 的轻量级开发服务器。此服务器非常适合快速开发和测试。                                                                                                                                                  |
| [`langgraph dockerfile`](../cloud/reference/cli.md#dockerfile) | 生成一个 [Dockerfile](https://docs.docker.com/reference/dockerfile/)，可用于构建和部署 [LangGraph API 服务器](./langgraph_server.md) 的实例。如果您想进一步自定义 dockerfile 或以更自定义的方式部署，这将非常有用。|
| [`langgraph up`](../cloud/reference/cli.md#up)                 | 在 Docker 容器中本地启动 [LangGraph API 服务器](./langgraph_server.md) 实例。这需要 docker 服务器在本地运行。它还需要用于本地开发的 LangSmith API 密钥或用于生产用途的许可证密钥。                          |

有关更多信息，请参阅 [LangGraph CLI 参考](../cloud/reference/cli.md)。
