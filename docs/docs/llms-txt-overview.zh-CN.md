# llms.txt

下面您可以找到 [`llms.txt`](https://llmstxt.org/) 格式的文档文件列表，特别是 `llms.txt` 和 `llms-full.txt`。这些文件允许大型语言模型 (LLMs) 和代理 (agents) 访问编程文档和 API，这在集成开发环境 (IDEs) 中特别有用。

| 语言版本 | llms.txt                                                                                                   | llms-full.txt                                                                                                        |
|----------|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| LangGraph Python | [https://langchain-ai.github.io/langgraph/llms.txt](https://langchain-ai.github.io/langgraph/llms.txt)     | [https://langchain-ai.github.io/langgraph/llms-full.txt](https://langchain-ai.github.io/langgraph/llms-full.txt)     |
| LangGraph JS     | [https://langchain-ai.github.io/langgraphjs/llms.txt](https://langchain-ai.github.io/langgraphjs/llms.txt) | [https://langchain-ai.github.io/langgraphjs/llms-full.txt](https://langchain-ai.github.io/langgraphjs/llms-full.txt) |
| LangChain Python | [https://python.langchain.com/llms.txt](https://python.langchain.com/llms.txt)                             | N/A (不适用)                                                                                                          |
| LangChain JS     | [https://js.langchain.com/llms.txt](https://js.langchain.com/llms.txt)                                     | N/A (不适用)                                                                                                          |

!!! info "审查输出"

    即使可以访问最新的文档，当前最先进的模型也可能并不总是生成正确的代码。请将生成的代码作为起点，在将代码发布到生产环境之前，务必进行审查。

## `llms.txt` 和 `llms-full.txt` 之间的区别

- **`llms.txt`** 是一个索引文件，包含链接和对内容的简要说明。LLM 或代理必须跟随这些链接才能访问详细信息。

- **`llms-full.txt`** 直接在一个文件中包含所有详细内容，通过消除额外的导航需求。

使用 `llms-full.txt` 时的一个关键考虑因素是其大小。对于大量文档，此文件可能会变得太大而无法放入 LLM 的上下文窗口中。

## 通过 MCP 服务器使用 `llms.txt`

截至 2025 年 3 月 9 日，IDEs [尚未对 `llms.txt` 提供强大的原生支持](https://x.com/jeremyphoward/status/1902109312216129905?t=1eHFv2vdNdAckajnug0_Vw&s=19)。然而，您仍然可以通过 MCP 服务器有效地使用 `llms.txt`。

### 🚀 使用 `mcpdoc` 服务器

我们提供了一个专为服务于 LLMs 和 IDEs 文档而设计的 **MCP 服务器**：

👉 **[langchain-ai/mcpdoc GitHub 仓库](https://github.com/langchain-ai/mcpdoc)**

此 MCP 服务器允许将 `llms.txt` 集成到 **Cursor**、**Windsurf**、**Claude** 和 **Claude Code** 等工具中。

📘 **设置说明和使用示例** 可在仓库中找到。

## 使用 `llms-full.txt`

LangGraph `llms-full.txt` 文件通常包含数十万个 token，超出了大多数 LLMs 的上下文窗口限制。要有效地使用此文件：

1. **使用 IDEs (例如 Cursor, Windsurf)**:
    - 将 `llms-full.txt` 添加为自定义文档。IDE 将自动对内容进行分块和索引，实施检索增强生成 (RAG)。

2. **没有 IDE 支持**:
    - 使用具有大上下文窗口的聊天模型。
    - 实施 RAG 策略以有效地管理和查询文档。

=== "OpenAI"

    ```shell
    pip install -U "langchain[openai]"
    ```
    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["OPENAI_API_KEY"] = "sk-..."

    llm = init_chat_model("openai:gpt-4.1")
    ```

    👉 阅读 [OpenAI 集成文档](https://python.langchain.com/docs/integrations/chat/openai/)

=== "Anthropic"

    ```shell
    pip install -U "langchain[anthropic]"
    ```
    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["ANTHROPIC_API_KEY"] = "sk-..."

    llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
    ```

    👉 阅读 [Anthropic 集成文档](https://python.langchain.com/docs/integrations/chat/anthropic/)

=== "Azure"

    ```shell
    pip install -U "langchain[openai]"
    ```
    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["AZURE_OPENAI_API_KEY"] = "..."
    os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
    os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

    llm = init_chat_model(
        "azure_openai:gpt-4.1",
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
    )
    ```
 
    👉 阅读 [Azure 集成文档](https://python.langchain.com/docs/integrations/chat/azure_chat_openai/)

=== "Google Gemini"

    ```shell
    pip install -U "langchain[google-genai]"
    ```
    ```python
    import os
    from langchain.chat_models import init_chat_model

    os.environ["GOOGLE_API_KEY"] = "..."

    llm = init_chat_model("google_genai:gemini-2.0-flash")
    ```

    👉 阅读 [Google GenAI 集成文档](https://python.langchain.com/docs/integrations/chat/google_generative_ai/)

=== "AWS Bedrock"

    ```shell
    pip install -U "langchain[aws]"
    ```
    ```python
    from langchain.chat_models import init_chat_model

    # 按照此处的步骤配置您的凭据：
    # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

    llm = init_chat_model(
        "anthropic.claude-3-5-sonnet-20240620-v1:0",
        model_provider="bedrock_converse",
    )
    ```

    👉 阅读 [AWS Bedrock 集成文档](https://python.langchain.com/docs/integrations/chat/bedrock/)
