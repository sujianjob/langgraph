!!! info "Prerequisites"

    - [LangGraph Studio Overview](../../../concepts/langgraph_studio.md)

LangGraph Studio 支持连接到两种类型的图：

- 部署在 [LangGraph Platform](../../../cloud/quick_start.md) 上的图
- 通过 [LangGraph Server](../../../tutorials/langgraph-platform/local-server.md) 在本地运行的图。

LangGraph Studio 可以从 LangSmith UI 的 LangGraph Platform Deployments（LangGraph 平台部署）选项卡中访问。

## 已部署的应用程序 (Deployed application)

对于已 [部署](../../quick_start.md) 在 LangGraph Platform 上的应用程序，您可以作为该部署的一部分访问 Studio。为此，请在 LangSmith UI 内导航到 LangGraph Platform 中的部署，然后单击 "LangGraph Studio" 按钮。

这将加载连接到您的实时部署的 Studio UI，允许您在该部署中创建、读取和更新 [线程](../../../concepts/persistence.md#threads)、[助手](../../../concepts/assistants.md) 和 [内存](../../../concepts//memory.md)。

## 本地开发服务器 (Local development server)

要使用 LangGraph Studio 测试您在本地运行的应用程序，请确保按照 [本指南](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/) 设置您的应用程序。

!!! info "LangSmith Tracing"
    对于本地开发，如果您不希望将数据跟踪到 LangSmith，请在应用程序的 `.env` 文件中设置 `LANGSMITH_TRACING=false`。禁用跟踪后，任何数据都不会离开您的本地服务器。

接下来，安装 [LangGraph CLI](../../../concepts/langgraph_cli.md)：

```
pip install -U "langgraph-cli[inmem]"
```

然后运行：

```
langgraph dev
```

!!! warning "Browser Compatibility"
    Safari 阻止 `localhost` 连接到 Studio。要解决此问题，请使用 `--tunnel` 运行上述命令，以便通过安全隧道访问 Studio。

这将在本地启动 LangGraph Server，在内存中运行。服务器将以监视模式运行，监听并自动在代码更改时重新启动。阅读此 [参考](https://langchain-ai.github.io/langgraph/cloud/reference/cli/#dev) 以了解启动 API 服务器的所有选项。

如果成功，您将看到以下日志：

> Ready!
>
> - API: [http://localhost:2024](http://localhost:2024/)
>
> - Docs: http://localhost:2024/docs
>
> - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

一旦运行，您将自动被重定向到 LangGraph Studio。

对于已在运行的服务器，可以通过以下方式访问 Studio：

1.  直接导航到以下 URL：`https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`。
2.  在 LangSmith 中，导航到 LangGraph Platform Deployments 选项卡，单击 "LangGraph Studio" 按钮，输入 `http://127.0.0.1:2024` 并单击 "Connect"。

如果在不同的主机或端口上运行服务器，只需更新 `baseUrl` 以使其匹配。

### (可选) 附加调试器 ((Optional) Attach a debugger)

要进行带断点和变量检查的逐步调试：

```bash
# Install debugpy package
pip install debugpy

# Start server with debugging enabled
langgraph dev --debug-port 5678
```

然后附加您首选的调试器：

=== "VS Code"

    将此配置添加到 `launch.json`：

    ```json
    {
        "name": "Attach to LangGraph",
        "type": "debugpy",
        "request": "attach",
        "connect": {
          "host": "0.0.0.0",
          "port": 5678
        }
    }
    ```

=== "PyCharm" 

    1. 转到 Run → Edit Configurations
    2. 单击 + 并选择 "Python Debug Server"
    3. 设置 IDE 主机名：`localhost`
    4. 设置端口：`5678`（或您在上一步中选择的端口号）
    5. 单击 "OK" 并开始调试

## 故障排除 (Troubleshooting)

有关入门问题，请参阅此 [故障排除指南](../../../troubleshooting/studio.md)。

## 下一步 (Next steps)

有关如何使用 Studio 的更多信息，请参阅以下指南：

- [运行应用程序](../invoke_studio.md)
- [管理助手](./manage_assistants.md)
- [管理线程](../threads_studio.md)
- [迭代提示词](../iterate_graph_studio.md)
- [调试 LangSmith 跟踪](../clone_traces_studio.md)
- [将节点添加到数据集](../datasets_studio.md)
