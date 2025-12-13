# 管理助手 (Manage assistants)

在本指南中，我们将展示如何创建、配置和管理 [助手](../../concepts/assistants.md)。

首先，简单回顾一下运行时上下文的概念，请考虑以下简单的 `call_model` 节点和上下文模式。观察到此节点尝试读取并使用由 `Runtime` 对象的 `context` 属性定义的 `model_provider`。

=== "Python"

    ```python
    @dataclass
    class ContextSchema:
        llm_provider: str = "anthropic"

    builder = StateGraph(AgentState, context_schema=ContextSchema)

    def call_model(state, runtime: Runtime[ContextSchema]):
        messages = state["messages"]
        model = _get_model(runtime.context.llm_provider)
        response = model.invoke(messages)
        # We return a list, because this will get added to the existing list
        return {"messages": [response]}
    ```

=== "Javascript"

    ```js
    import { Annotation } from "@langchain/langgraph";

    const ConfigSchema = Annotation.Root({
        model_name: Annotation<string>,
        system_prompt:
    });

    const builder = new StateGraph(AgentState, ConfigSchema)

    function callModel(state: State, config: RunnableConfig) {
      const messages = state.messages;
      const modelName = config.configurable?.model_name ?? "anthropic";
      const model = _getModel(modelName);
      const response = model.invoke(messages);
      // We return a list, because this will get added to the existing list
      return { messages: [response] };
    }
    ```

:::python
有关运行时上下文的更多信息，请参阅 [此处](../../concepts/low_level.md#runtime-context)。
:::

## 创建助手 (Create an assistant)

### LangGraph SDK

要创建助手，请使用 [LangGraph SDK](../../concepts/sdk.md) 的 `create` 方法。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.create) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create) SDK 参考文档。

此示例使用与上面相同的配置模式，并创建一个 `model_name` 设置为 `openai` 的助手。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    openai_assistant = await client.assistants.create(
        # "agent" is the name of a graph we deployed
        "agent", config={"configurable": {"model_name": "openai"}}, name="Open AI Assistant"
    )

    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const openAIAssistant = await client.assistants.create({
        graphId: 'agent',
        name: "Open AI Assistant",
        config: { "configurable": { "model_name": "openai" } },
    });

    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{"graph_id":"agent", "config":{"configurable":{"model_name":"openai"}}, "name": "Open AI Assistant"}'
    ```

由于输出较长，此处省略具体的 JSON 响应。

### LangGraph Platform UI

您还可以从 LangGraph Platform UI 创建助手。

在您的部署中，选择 "Assistants"（助手）此选项卡。这将加载您部署的所有图中所有助手的表格。

要创建新助手，请选择 "+ New assistant"（+ 新建助手）按钮。这将打开一个表单，您可以在其中指定此助手所属的图，并根据该图的配置模式提供名称、描述和所需的助手配置。

确认后，单击 "Create assistant"（创建助手）。这将带您进入 [LangGraph Studio](../../concepts/langgraph_studio.md)，您可以在其中测试助手。如果您返回部署中的 "Assistants" 选项卡，您将在表中看到新创建的助手。

## 使用助手 (Use an assistant)

### LangGraph SDK

我们现在创建了一个名为 "Open AI Assistant" 的助手，其 `model_name` 定义为 `openai`。我们现在可以使用此配置使用此助手：

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        # this is where we specify the assistant id to use
        openai_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const thread = await client.threads.create();
    const input = { "messages": [{ "role": "user", "content": "who made you?" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      // this is where we specify the assistant id to use
      openAIAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );

    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    thread_id=$(curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}' | jq -r '.thread_id') && \
    curl --request POST \
        --url "<DEPLOYMENT_URL>/threads/${thread_id}/runs/stream" \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <OPENAI_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "who made you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
        '
    ```

### LangGraph Platform UI

在您的部署中，选择 "Assistants" 选项卡。对于您想要使用的助手，单击 "Studio" 按钮。这将使用选定的助手打开 LangGraph Studio。当您提交输入（在图或聊天模式下）时，将使用选定的助手及其配置。

## 为您的助手创建新版本 (Create a new version for your assistant)

### LangGraph SDK

要编辑助手，请使用 `update` 方法。这将使用提供的编辑创建助手的新版本。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.update) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#update) SDK 参考文档。

!!! note "Note"

    您必须传入完整配置（以及元数据，如果您正在使用它）。更新端点完全从头开始创建新版本，不依赖于以前的版本。

例如，要更新助手的系统提示：

=== "Python"

    ```python
    openai_assistant_v2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        config={
            "configurable": {
                "model_name": "openai",
                "system_prompt": "You are an unhelpful assistant!",
            }
        },
    )
    ```

=== "Javascript"

    ```js
    const openaiAssistantV2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        {
            config: {
                configurable: {
                    model_name: 'openai',
                    system_prompt: 'You are an unhelpful assistant!',
                },
        },
    });
    ```

=== "CURL"

    ```bash
    curl --request PATCH \
    --url <DEPOLYMENT_URL>/assistants/<ASSISTANT_ID> \
    --header 'Content-Type: application/json' \
    --data '{
    "config": {"model_name": "openai", "system_prompt": "You are an unhelpful assistant!"}
    }'
    ```

这将使用更新的参数创建助手的新版本，并将其设置为助手的活动版本。如果您现在运行图并传入此助手 ID，它将使用此最新版本。

### LangGraph Platform UI

您还可以从 LangGraph Platform UI 编辑助手。

在您的部署中，选择 "Assistants" 选项卡。这将加载您部署的所有图中所有助手的表格。

要编辑现有助手，请选择指定助手的 "Edit"（编辑）按钮。这将打开一个表单，您可以在其中编辑助手的名称、描述和配置。

此外，如果使用 LangGraph Studio，您可以通过 "Manage Assistants"（管理助手）按钮编辑助手并创建新版本。

## 使用以前的助手版本 (Use a previous assistant version)

### LangGraph SDK

您还可以更改助手的活动版本。为此，请使用 `setLatest` 方法。

在上面的示例中，要回滚到助手的第一个版本：

=== "Python"

    ```python
    await client.assistants.set_latest(openai_assistant['assistant_id'], 1)
    ```

=== "Javascript"

    ```js
    await client.assistants.setLatest(openaiAssistant['assistant_id'], 1);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/assistants/<ASSISTANT_ID>/latest \
    --header 'Content-Type: application/json' \
    --data '{
    "version": 1
    }'
    ```

如果您现在运行图并传入此助手 ID，它将使用助手的第一个版本。

### LangGraph Platform UI

如果使用 LangGraph Studio，要设置助手的活动版本，请单击 "Manage Assistants" 按钮并找到您要使用的助手。选择助手和版本，然后单击 "Active" 切换开关。这将更新助手以使所选版本处于活动状态。

!!! warning "Deleting Assistants"
    删除助手将删除其 所有 版本。目前无法删除单个版本，但通过将助手指向正确的版本，您可以跳过任何不想使用的版本。
