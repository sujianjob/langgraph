# 使用 Cron 作业 (Use cron jobs)

有时您不想基于用户交互运行图，而是希望安排图按计划运行 - 例如，如果您希望图撰写并发送团队待办事项的每周电子邮件。LangGraph Platform 允许您使用 `Crons` 客户端执行此操作，而无需编写自己的脚本。要安排图作业，您需要传递一个 [cron 表达式](https://crontab.cronhub.io/) 以通知客户端您希望何时运行图。`Cron` 作业在后台运行，不会干扰图的正常调用。

## 设置 (Setup)

首先，让我们设置 SDK 客户端、助手和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    # create thread
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    # Using the graph deployed with the name "agent"
    const assistantId = "agent";
    # create thread
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0].graph_id' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

输出：

    {
        'thread_id': '9dde5490-2b67-47c8-aa14-4bfec88af217', 
        'created_at': '2024-08-30T23:07:38.242730+00:00', 
        'updated_at': '2024-08-30T23:07:38.242730+00:00', 
        'metadata': {}, 
        'status': 'idle', 
        'config': {}, 
        'values': None
    }

## 线程上的 Cron 作业 (Cron job on a thread)

要创建与特定线程关联的 cron 作业，您可以编写：


=== "Python"

    ```python
    # This schedules a job to run at 15:27 (3:27PM) every day
    cron_job = await client.crons.create_for_thread(
        thread["thread_id"],
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // This schedules a job to run at 15:27 (3:27PM) every day
    const cronJob = await client.crons.create_for_thread(
      thread["thread_id"],
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

请注意，删除不再有用的 `Cron` 作业 **非常** 重要。否则，您可能会向 LLM 支付不必要的 API 费用！您可以使用以下代码删除 `Cron` 作业：

=== "Python"

    ```python
    await client.crons.delete(cron_job["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJob["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```

## 无状态 Cron 作业 (Cron job stateless)

您还可以使用以下代码创建无状态 cron 作业：

=== "Python"

    ```python
    # This schedules a job to run at 15:27 (3:27PM) every day
    cron_job_stateless = await client.crons.create(
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    # This schedules a job to run at 15:27 (3:27PM) every day
    const cronJobStateless = await client.crons.create(
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

再次记住，完成后删除您的作业！

=== "Python"

    ```python
    await client.crons.delete(cron_job_stateless["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJobStateless["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```
