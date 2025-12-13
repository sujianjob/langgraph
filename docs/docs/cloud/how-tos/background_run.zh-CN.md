# 如何启动后台运行 (How to kick off background runs)

本指南涵盖了如何启动代理的后台运行。
这对于长时间运行的作业很有用。

## 设置 (Setup)

首先，让我们设置我们的客户端和线程：

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
    // Using the graph deployed with the name "agent"
    const assistantID = "agent";
    // create thread
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

输出：

    {
        'thread_id': '5cb1e8a1-34b3-4a61-a34e-71a9799bd00d',
        'created_at': '2024-08-30T20:35:52.062934+00:00',
        'updated_at': '2024-08-30T20:35:52.062934+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## 检查线程上的运行 (Check runs on thread)

如果我们列出此线程上的当前运行，我们将看到它是空的：

=== "Python"

    ```python
    runs = await client.runs.list(thread["thread_id"])
    print(runs)
    ```

=== "Javascript"

    ```js
    let runs = await client.runs.list(thread['thread_id']);
    console.log(runs);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs
    ```

输出：

    []

## 在线程上启动运行 (Start runs on thread)

现在让我们启动一个运行：

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
    run = await client.runs.create(thread["thread_id"], assistant_id, input=input)
    ```

=== "Javascript"

    ```js
    let input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]};
    let run = await client.runs.create(thread["thread_id"], assistantID, { input });
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>
        }'
    ```

第一次轮询它时，我们可以看到 `status=pending`：

=== "Python"

    ```python
    print(await client.runs.get(thread["thread_id"], run["run_id"]))
    ```

=== "Javascript"

    ```js
    console.log(await client.runs.get(thread["thread_id"], run["run_id"]));
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>
    ```

现在我们可以加入运行，等待它完成并再次检查该状态：

=== "Python"

    ```python
    await client.runs.join(thread["thread_id"], run["run_id"])
    print(await client.runs.get(thread["thread_id"], run["run_id"]))
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], run["run_id"]);
    console.log(await client.runs.get(thread["thread_id"], run["run_id"]));
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join &&
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>
    ```

完美！运行如我们预期的那样成功。我们可以通过打印出最终状态来仔细检查运行是否按预期工作：

=== "Python"

    ```python
    final_result = await client.threads.get_state(thread["thread_id"])
    print(final_result)
    ```

=== "Javascript"

    ```js
    let finalResult = await client.threads.getState(thread["thread_id"]);
    console.log(finalResult);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state
    ```

我们也可以只打印最后一条 AIMessage 的内容：

=== "Python"

    ```python
    print(final_result['values']['messages'][-1]['content'][0]['text'])
    ```

=== "Javascript"

    ```js
    console.log(finalResult['values']['messages'][finalResult['values']['messages'].length-1]['content'][0]['text']);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | jq -r '.values.messages[-1].content.[0].text'
    ```

输出：

    The search results provide the current weather conditions in San Francisco. According to the data, as of 2:00 PM on August 30, 2024, the temperature in San Francisco is 70°F (21.1°C) with partly cloudy skies. The wind is blowing from the west-northwest at around 12 mph (19 km/h). The humidity is 59% and visibility is 9 miles (16 km). Overall, it looks like a nice late summer day in San Francisco with comfortable temperatures and partly sunny conditions.
