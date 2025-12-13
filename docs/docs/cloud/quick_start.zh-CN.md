# 部署快速入门 (Deployment quickstart)

本指南向您展示如何设置和使用 LangGraph Platform 进行云部署。

## 先决条件 (Prerequisites)

在开始之前，请确保您拥有以下内容：

- 一个 [GitHub 帐户](https://github.com/)
- 一个 [LangSmith 帐户](https://smith.langchain.com/) – 免费注册

## 1. 在 GitHub 上创建存储库

要将应用程序部署到 **LangGraph Platform**，您的应用程序代码必须位于 GitHub 存储库中。支持公共和私有存储库。对于本快速入门，请为您的应用程序使用 [`new-langgraph-project` 模板](https://github.com/langchain-ai/react-agent)：

1. 转到 [`new-langgraph-project` 存储库](https://github.com/langchain-ai/new-langgraph-project) 或 [`new-langgraphjs-project` 模板](https://github.com/langchain-ai/new-langgraphjs-project)。
2. 单击右上角的 `Fork` 按钮将存储库分叉到您的 GitHub 帐户。
3. 单击 **Create fork**（创建分叉）。

## 2. 部署到 LangGraph Platform

1. 登录到 [LangSmith](https://smith.langchain.com/)。
2. 在左侧边栏中，选择 **Deployments**（部署）。
3. 单击 **+ New Deployment**（新建部署）按钮。将打开一个窗格，您可以在其中填写必填字段。
4. 如果您是首次用户或添加以前未连接的私有存储库，请单击 **Import from GitHub**（从 GitHub 导入）按钮并按照说明连接您的 GitHub 帐户。
5. 选择您的 New LangGraph Project 存储库。
6. 单击 **Submit**（提交）进行部署。

    这可能需要大约 15 分钟才能完成。您可以在 **Deployment details**（部署详情）视图中查看状态。

## 3. 在 LangGraph Studio 中测试您的应用程序

部署应用程序后：

1. 选择您刚刚创建的部署以查看更多详细信息。
2. 单击右上角的 **LangGraph Studio** 按钮。

    LangGraph Studio 将打开以显示您的图。

    <figure markdown="1">
    [![image](deployment/img/langgraph_studio.png){: style="max-height:400px"}](deployment/img/langgraph_studio.png)
    <figcaption>
        LangGraph Studio 中的示例图运行。
    </figcaption>
    </figure>

## 4. 获取部署的 API URL

1. 在 LangGraph 的 **Deployment details**（部署详情）视图中，单击 **API URL** 将其复制到剪贴板。
2. 单击 `URL` 将其复制到剪贴板。

## 5. 测试 API

您现在可以测试 API：

=== "Python SDK (Async)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    2. 向助手发送消息（无线程运行）：

        ```python
        from langgraph_sdk import get_client

        client = get_client(url="your-deployment-url", api_key="your-langsmith-api-key")

        async for chunk in client.runs.stream(
            None,  # Threadless run
            "agent", # Name of assistant. Defined in langgraph.json.
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="updates",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "Python SDK (Sync)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    2. 向助手发送消息（无线程运行）：

        ```python
        from langgraph_sdk import get_sync_client

        client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

        for chunk in client.runs.stream(
            None,  # Threadless run
            "agent", # Name of assistant. Defined in langgraph.json.
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="updates",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "JavaScript SDK"

    1. 安装 LangGraph JS SDK

        ```shell
        npm install @langchain/langgraph-sdk
        ```

    2. 向助手发送消息（无线程运行）：

        ```js
        const { Client } = await import("@langchain/langgraph-sdk");

        const client = new Client({ apiUrl: "your-deployment-url", apiKey: "your-langsmith-api-key" });

        const streamResponse = client.runs.stream(
            null, // Threadless run
            "agent", // Assistant ID
            {
                input: {
                    "messages": [
                        { "role": "user", "content": "What is LangGraph?"}
                    ]
                },
                streamMode: "messages",
            }
        );

        for await (const chunk of streamResponse) {
            console.log(`Receiving new event of type: ${chunk.event}...`);
            console.log(JSON.stringify(chunk.data));
            console.log("\n\n");
        }
        ```

=== "Rest API"

    ```bash
    curl -s --request POST \
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --header "X-Api-Key: <LANGSMITH API KEY> \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"updates\"
        }" 
    ```


## 下一步 (Next steps)

恭喜！您已使用 LangGraph Platform 部署了应用程序。

以下是一些其他可供查看的资源：

- [LangGraph Platform 概述](../concepts/langgraph_platform.md)
- [部署选项](../concepts/deployment_options.md)
