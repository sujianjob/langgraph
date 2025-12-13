---
search:
  boost: 2
---

# 模板应用程序 (Template Applications)

模板是开源参考应用程序，旨在帮助您在构建 LangGraph 时快速上手。它们提供了常见的代理工作流的工作示例，可以根据您的需求进行自定义。

您可以使用 LangGraph CLI 从模板创建应用程序。

:::python
!!! info "Requirements"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/): 需要 langchain-cli[inmem] >= 0.1.58

## 安装 LangGraph CLI (Install the LangGraph CLI)

```bash
pip install "langgraph-cli[inmem]" --upgrade
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (推荐):

```bash
uvx --from "langgraph-cli[inmem]" langgraph dev --help
```

:::

:::js

```bash
npx @langchain/langgraph-cli --help
```

:::

## 可用模板 (Available Templates)

:::python
| 模板 (Template) | 描述 (Description) | 链接 (Link) |
| -------- | ----------- | ------ |
| **New LangGraph Project** | 一个简单的、具有记忆功能的最小聊天机器人。 | [仓库](https://github.com/langchain-ai/new-langgraph-project) |
| **ReAct Agent** | 一个简单的代理，可以灵活地扩展到许多工具。 | [仓库](https://github.com/langchain-ai/react-agent) |
| **Memory Agent** | 一个 ReAct 风格的代理，带有额外的工具来存储记忆以供跨线程使用。 | [仓库](https://github.com/langchain-ai/memory-agent) |
| **Retrieval Agent** | 一个包含基于检索的问答系统的代理。 | [仓库](https://github.com/langchain-ai/retrieval-agent-template) |
| **Data-Enrichment Agent** | 一个执行网络搜索并将结果组织成结构化格式的代理。 | [仓库](https://github.com/langchain-ai/data-enrichment) |

:::

:::js
| 模板 (Template) | 描述 (Description) | 链接 (Link) |
| -------- | ----------- | ------ |
| **New LangGraph Project** | 一个简单的、具有记忆功能的最小聊天机器人。 | [仓库](https://github.com/langchain-ai/new-langgraphjs-project) |
| **ReAct Agent** | 一个简单的代理，可以灵活地扩展到许多工具。 | [仓库](https://github.com/langchain-ai/react-agent-js) |
| **Memory Agent** | 一个 ReAct 风格的代理，带有额外的工具来存储记忆以供跨线程使用。 | [仓库](https://github.com/langchain-ai/memory-agent-js) |
| **Retrieval Agent** | 一个包含基于检索的问答系统的代理。 | [仓库](https://github.com/langchain-ai/retrieval-agent-template-js) |
| **Data-Enrichment Agent** | 一个执行网络搜索并将结果组织成结构化格式的代理。 | [仓库](https://github.com/langchain-ai/data-enrichment-js) |
:::

## 🌱 创建 LangGraph 应用程序 (Create a LangGraph App)

要从模板创建新应用程序，请使用 `langgraph new` 命令。

:::python

```bash
langgraph new
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (推荐):

```bash
uvx --from "langgraph-cli[inmem]" langgraph new
```

:::

:::js

```bash
npm create langgraph
```

:::

## 下一步 (Next Steps)

查看新的 LangGraph 应用程序根目录中的 `README.md` 文件，以获取有关模板以及如何自定义它的更多信息。

正确配置应用程序并添加 API 密钥后，您可以使用 LangGraph CLI 启动应用程序：

:::python

```bash
langgraph dev
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (推荐):

```bash
uvx --from "langgraph-cli[inmem]" --with-editable . langgraph dev
```

!!! info "Missing Local Package?"

    如果您不使用 `uv` 并且即使在安装了本地包 (`pip install -e .`) 之后仍遇到 "`ModuleNotFoundError`" 或 "`ImportError`"，则很可能需要将 CLI 安装到本地虚拟环境中，以使 CLI "意识到" 本地包。您可以通过运行 `python -m pip install "langgraph-cli[inmem]"` 并在运行 `langgraph dev` 之前重新激活虚拟环境来完成此操作。

:::

:::js

```bash
npx @langchain/langgraph-cli dev
```

:::

有关如何部署应用程序的更多信息，请参阅以下指南：

- **[启动本地 LangGraph 服务器](../tutorials/langgraph-platform/local-server.md)**: 本快速入门指南展示了如何在本地为 **ReAct Agent** 模板启动 LangGraph 服务器。其他模板的步骤类似。
- **[部署到 LangGraph 平台](../cloud/quick_start.md)**: 使用 LangGraph 平台部署您的 LangGraph 应用程序。
