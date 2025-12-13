# 使用 Webhooks (Use webhooks)

在使用 LangGraph Platform 时，您可能希望使用 webhook 在 API 调用完成后接收更新。Webhooks 对于在运行完成处理后触发服务中的操作非常有用。要实现这一点，您需要公开一个可以接受 `POST` 请求的端点，并将此端点作为 `webhook` 参数在您的 API 请求中传递。

目前，SDK 不提供用于定义 webhook 端点的内置支持，但您可以使用 API 请求手动指定它们。

## 支持的端点 (Supported endpoints)

以下 API 端点接受 `webhook` 参数：

| 操作 (Operation) | HTTP 方法 (HTTP Method) | 端点 (Endpoint) |
|----------------------|-------------|-----------------------------------|
| 创建运行 (Create Run)           | `POST`      | `/thread/{thread_id}/runs`        |
| 创建线程 Cron (Create Thread Cron)   | `POST`      | `/thread/{thread_id}/runs/crons`  |
| 流式运行 (Stream Run)           | `POST`      | `/thread/{thread_id}/runs/stream` |
| 等待运行 (Wait Run)             | `POST`      | `/thread/{thread_id}/runs/wait`   |
| 创建 Cron (Create Cron)          | `POST`      | `/runs/crons`                     |
| 无状态流式运行 (Stream Run Stateless) | `POST`      | `/runs/stream`                    |
| 无状态等待运行 (Wait Run Stateless)   | `POST`      | `/runs/wait`                      |

在本指南中，我们将展示如何在流式运行后触发 webhook。

## 设置您的助手和线程 (Set up your assistant and thread)

在进行 API 调用之前，设置您的助手和线程。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    assistant_id = "agent"
    thread = await client.threads.create()
    print(thread)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const assistantID = "agent";
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{ "limit": 10, "offset": 0 }' | jq -c 'map(select(.config == null or .config == {})) | .[0]' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

响应示例：

```json
{
    "thread_id": "9dde5490-2b67-47c8-aa14-4bfec88af217",
    "created_at": "2024-08-30T23:07:38.242730+00:00",
    "updated_at": "2024-08-30T23:07:38.242730+00:00",
    "metadata": {},
    "status": "idle",
    "config": {},
    "values": null
}
```

## 在图运行中使用 webhook (Use a webhook with a graph run)

要使用 webhook，请在您的 API 请求中指定 `webhook` 参数。当运行完成时，LangGraph Platform 会向指定的 webhook URL 发送 `POST` 请求。

例如，如果您的服务器在 `https://my-server.app/my-webhook-endpoint` 监听 webhook 事件，请在您的请求中包含它：

=== "Python"

    ```python
    input = { "messages": [{ "role": "user", "content": "Hello!" }] }

    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        stream_mode="events",
        webhook="https://my-server.app/my-webhook-endpoint"
    ):
        pass
    ```

=== "JavaScript"

    ```js
    const input = { messages: [{ role: "human", content: "Hello!" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input: input,
        webhook: "https://my-server.app/my-webhook-endpoint"
      }
    );

    for await (const chunk of streamResponse) {
      // Handle stream output
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
            "input": {"messages": [{"role": "user", "content": "Hello!"}]},
            "webhook": "https://my-server.app/my-webhook-endpoint"
        }'
    ```

## Webhook 有效负载 (Webhook payload)

LangGraph Platform 以 [Run](../../concepts/assistants.md#execution) 的格式发送 webhook 通知。有关详细信息，请参阅 [API 参考](https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html#model/run)。请求有效负载在 `kwargs` 字段中包括运行输入、配置和其他元数据。

## 安全 Webhooks (Secure webhooks)

为确保只有授权的请求到达您的 webhook 端点，请考虑将安全令牌添加为查询参数：

```
https://my-server.app/my-webhook-endpoint?token=YOUR_SECRET_TOKEN
```

您的服务器应在处理请求之前提取并验证此令牌。

## 禁用 Webhooks (Disable webhooks)

从 `langgraph-api>=0.2.78` 开始，开发人员可以在 `langgraph.json` 文件中禁用 webhook：

```json
{
  "http": {
    "disable_webhooks": true
  }
}
```

此功能主要用于自托管部署，平台管理员或开发人员可能希望禁用 webhook 以简化其安全状况（特别是如果他们未配置防火墙规则或其他网络控制）。禁用 webhook 有助于防止将不受信任的有效负载发送到内部端点。

有关完整的配置详细信息，请参阅 [配置文件参考](https://langchain-ai.github.io/langgraph/cloud/reference/cli/?h=disable_webhooks#configuration-file)。

## 测试 Webhooks (Test webhooks)

您可以使用以下在线服务测试您的 webhook：

- **[Beeceptor](https://beeceptor.com/)** – 快速创建测试端点并检查传入的 webhook 有效负载。
- **[Webhook.site](https://webhook.site/)** – 实时查看、调试和记录传入的 webhook 请求。

这些工具有助于您验证 LangGraph Platform 是否正确触发并将 webhook 发送到您的服务。
