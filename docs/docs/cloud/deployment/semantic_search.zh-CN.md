# 如何将语义搜索添加到您的 LangGraph 部署 (How to add semantic search to your LangGraph deployment)

本指南介绍如何将语义搜索添加到 LangGraph 部署的跨线程 [存储 (store)](../../concepts/persistence.md#memory-store)，以便您的代理可以按语义相似性搜索记忆和其他文档。

## 先决条件 (Prerequisites)

- LangGraph 部署（请参阅 [如何部署](setup_pyproject.md)）
- 嵌入提供商的 API 密钥（在本例中为 OpenAI）
- `langchain >= 0.3.8`（如果您指定使用下面的字符串格式）

## 步骤 (Steps)

1. 更新您的 `langgraph.json` 配置文件以包含存储配置：

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embedding-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

此配置：

- 使用 OpenAI 的 text-embedding-3-small 模型生成嵌入
- 将嵌入维度设置为 1536（匹配模型的输出）
- 索引存储数据中的所有字段（`["$"]` 表示索引所有内容，或指定特定字段，如 `["text", "metadata.title"]`）

2. 要使用上面的字符串嵌入格式，请确保您的依赖项包含 `langchain >= 0.3.8`：

```toml
# In pyproject.toml
[project]
dependencies = [
    "langchain>=0.3.8"
]
```

或者如果使用 requirements.txt：

```
langchain>=0.3.8
```

## 用法 (Usage)

配置完成后，您可以在 LangGraph 节点中使用语义搜索。存储需要命名空间元组来组织记忆：

```python
def search_memory(state: State, *, store: BaseStore):
    # Search the store using semantic similarity
    # The namespace tuple helps organize different types of memories
    # e.g., ("user_facts", "preferences") or ("conversation", "summaries")
    results = store.search(
        namespace=("memory", "facts"),  # Organize memories by type
        query="your search query",
        limit=3  # number of results to return
    )
    return results
```

## 自定义嵌入 (Custom Embeddings)

如果您想使用自定义嵌入，可以传递自定义嵌入函数的路径：

```json
{
    ...
    "store": {
        "index": {
            "embed": "path/to/embedding_function.py:embed",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

部署将在指定路径中查找该函数。该函数必须是异步的并且接受字符串列表：

```python
# path/to/embedding_function.py
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def aembed_texts(texts: list[str]) -> list[list[float]]:
    """Custom embedding function that must:
    1. Be async
    2. Accept a list of strings
    3. Return a list of float arrays (embeddings)
    """
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [e.embedding for e in response.data]
```

## 通过 API 查询 (Querying via the API)

您还可以使用 LangGraph SDK 查询存储。由于 SDK 使用异步操作：

```python
from langgraph_sdk import get_client

async def search_store():
    client = get_client()
    results = await client.store.search_items(
        ("memory", "facts"),
        query="your search query",
        limit=3  # number of results to return
    )
    return results

# Use in an async context
results = await search_store()
```
