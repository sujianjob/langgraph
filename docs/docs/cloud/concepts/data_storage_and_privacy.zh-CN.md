# 数据存储和隐私 (Data Storage and Privacy)

本文档描述了 LangGraph CLI 和 LangGraph Server 在内存服务器 (`langgraph dev`) 和本地 Docker 服务器 (`langgraph up`) 中如何处理数据。它还描述了与托管的 LangGraph Studio 前端交互时会跟踪哪些数据。

## CLI

LangGraph **CLI** 是用于构建和运行 LangGraph 应用程序的命令行界面；请参阅 [CLI 指南](../../concepts/langgraph_cli.md) 了解更多信息。

默认情况下，对大多数 CLI 命令的调用会在调用时记录一个单独的分析事件。这有助于我们更好地确定 CLI 体验改进的优先级。每个遥测事件都包含调用进程的操作系统、操作系统版本、Python 版本、CLI 版本、命令名称（`dev`、`up`、`run` 等）以及表示是否向命令传递了标志的布尔值。您可以在 [此处](https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/analytics.py) 查看完整的分析逻辑。

您可以通过设置 `LANGGRAPH_CLI_NO_ANALYTICS=1` 来禁用所有 CLI 遥测。

## LangGraph Server (内存 & docker)

[LangGraph Server](../../concepts/langgraph_server.md) 提供了一个持久执行运行时，它依赖于将应用程序状态、长期记忆、线程元数据、助手和类似资源的检查点持久化到本地文件系统或数据库。除非您特意自定义了存储位置，否则此信息将写入本地磁盘（对于 `langgraph dev`）或 PostgreSQL 数据库（对于 `langgraph up` 和所有部署）。

### LangSmith 跟踪 (LangSmith Tracing)

运行 LangGraph 服务器（内存中或 Docker 中）时，可以启用 LangSmith 跟踪以促进更快的调试并提供生产中图状态和 LLM 提示的可观测性。通过在服务器的运行时环境中设置 `LANGSMITH_TRACING=false`，您始终可以禁用跟踪。

### 内存开发服务器 (`langgraph dev`)

`langgraph dev` 运行一个 [内存开发服务器](../../tutorials/langgraph-platform/local-server.md) 作为单个 Python 进程，专为快速开发和测试而设计。它将所有检查点和内存数据保存到当前工作目录内的 `.langgraph_api` 目录中的磁盘。除了 [CLI](#cli) 部分中描述的遥测数据外，除非通过启用跟踪或您的图代码显式联系外部服务，否则任何数据都不会离开机器。

### 独立容器 (`langgraph up`)

`langgraph up` 将您的本地包构建为 Docker 镜像，并将服务器作为由三个容器组成的 [独立容器](../../concepts/deployment_options.md#standalone-container) 运行：API 服务器、PostgreSQL 容器和 Redis 容器。所有持久数据（检查点、助手等）都存储在 PostgreSQL 数据库中。Redis 用作 pubsub 连接，用于实时流式传输事件。您可以通过设置有效的 `LANGGRAPH_AES_KEY` 环境变量，在保存到数据库之前加密所有检查点。您还可以在 `langgraph.json` 中为检查点和跨线程记忆指定 [TTL](../../how-tos/ttl/configure_ttl.md)，以控制数据的存储时间。所有持久化线程、记忆和其他数据都可以通过相关的 API 端点删除。

会进行额外的 API 调用以确认服务器拥有有效许可证并跟踪已执行运行和任务的数量。API 服务器会定期验证提供的许可证密钥（或 API 密钥）。

如果您已禁用 [跟踪](#langsmith-tracing)，除非您的图代码显式联系外部服务，否则不会在外部持久保存任何用户数据。

## Studio

[LangGraph Studio](../../concepts/langgraph_studio.md) 是一个图形界面，用于与您的 LangGraph 服务器进行交互。它不持久化任何私有数据（您发送到服务器的数据不会发送到 LangSmith）。虽然 Studio 界面通过 [smith.langchain.com](https://smith.langchain.com) 提供服务，但它要在您的浏览器中运行并直接连接到您的本地 LangGraph 服务器，因此无需将任何数据发送到 LangSmith。

如果您已登录，LangSmith 确实会收集一些使用情况分析数据，以帮助改善 Studio 的用户体验。这包括：

- 页面访问和导航模式
- 用户操作（按钮点击）
- 浏览器类型和版本
- 屏幕分辨率和视口大小

重要的是，不会收集任何应用程序数据或代码（或其他敏感配置详细信息）。所有这些都存储在 LangGraph 服务器的持久层中。匿名使用 Studio 时，无需创建帐户，也不会收集使用情况分析数据。

## 快速参考 (Quick reference)

总之，您可以通过关闭 CLI 分析和禁用跟踪来选择退出服务器端遥测。

| 变量 | 用途 | 默认值 |
| ------------------------------ | ------------------------- | -------------------------------- |
| `LANGGRAPH_CLI_NO_ANALYTICS=1` | 禁用 CLI 分析 | 分析已启用 |
| `LANGSMITH_API_KEY` | 启用 LangSmith 跟踪 | 跟踪已禁用 |
| `LANGSMITH_TRACING=false` | 禁用 LangSmith 跟踪 | 取决于环境 |
