---
tags:
  - mcp
  - platform
hide:
  - tags
---

# LangGraph Server 中的 MCP 端点 (MCP endpoint in LangGraph Server)

[模型上下文协议 (Model Context Protocol, MCP)](./mcp.md) 是一个开放协议，用于以模型无关的格式描述工具和数据源，使 LLM 能够通过结构化 API 发现和使用它们。

[LangGraph Server](./langgraph_server.md) 使用 [Streamable HTTP 传输](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/transports/#streamable-http) 实现了 MCP。这允许将 LangGraph **代理** 作为 **MCP 工具** 公开，使其可以与任何支持 Streamable HTTP 的 MCP 兼容客户端一起使用。

MCP 端点位于 [LangGraph Server](./langgraph_server.md) 的 `/mcp`。

## 要求 (Requirements)

:::python
要使用 MCP，请确保您已安装以下依赖项：

- `langgraph-api >= 0.2.3`
- `langgraph-sdk >= 0.1.61`

使用以下命令安装它们：

```bash
pip install "langgraph-api>=0.2.3" "langgraph-sdk>=0.1.61"
```

:::

:::js
要使用 MCP，请确保您已同时安装 api 和 sdk 包。

```bash
npm install @langchain/langgraph-api @langchain/langgraph-sdk
```

:::

## 将代理公开为 MCP 工具 (Exposing an agent as MCP tool)

部署后，您的代理将作为 MCP 端点中的一个工具出现，具有以下配置：

- **工具名称 (Tool name)**: 代理的名称。
- **工具描述 (Tool description)**: 代理的描述。
- **工具输入模式 (Tool input schema)**: 代理的输入模式。

### 设置名称和描述 (Setting name and description)

您可以在 `langgraph.json` 中设置代理的名称和描述：

:::python

```json
{
  "graphs": {
    "my_agent": {
      "path": "./my_agent/agent.py:graph",
      "description": "A description of what the agent does"
    }
  },
  "env": ".env"
}
```

:::
:::js

```json
{
  "graphs": {
    "my_agent": {
      "path": "./my_agent/agent.ts:graph",
      "description": "A description of what the agent does"
    }
  },
  "env": ".env"
}
```

:::

部署后，您可以使用 LangGraph SDK 更新名称和描述。

### 模式 (Schema)

定义清晰、最小的输入和输出模式，以避免向 LLM 暴露不必要的内部复杂性。

:::python
默认的 [MessagesState](./low_level.md#messagesstate) 使用 `AnyMessage`，它支持多种消息类型，但对于直接 LLM 暴露来说太通用了。
:::

相反，定义使用显式类型输入和输出结构的 **自定义代理或工作流**。

例如，回答文档问题的工作流可能如下所示：

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# Define input schema
class InputState(TypedDict):
    question: str

# Define output schema
class OutputState(TypedDict):
    answer: str

# Combine input and output
class OverallState(InputState, OutputState):
    pass

# Define the processing node
def answer_node(state: InputState):
    # Replace with actual logic and do something useful
    return {"answer": "bye", "question": state["question"]}

# Build the graph with explicit schemas
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)
builder.add_edge(START, "answer_node")
builder.add_edge("answer_node", END)
graph = builder.compile()

# Run the graph
print(graph.invoke({"question": "hi"}))
```

有关更多详细信息，请参阅 [低级概念指南](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)。

## 使用概述 (Usage overview)

要启用 MCP：

- 升级以使用 langgraph-api>=0.2.3。如果您正在部署 LangGraph Platform，如果您创建新修订，这将自动为您完成。
- MCP 工具（代理）将被自动公开。
- 连接任何支持 Streamable HTTP 的 MCP 兼容客户端。

### 客户端 (Client)

:::python
使用 MCP 兼容客户端连接到 LangGraph 服务器。以下示例显示如何使用 [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) 进行连接。

使用以下命令安装适配器：

```bash
pip install langchain-mcp-adapters
```

以下是如何连接到远程 MCP 端点并将代理用作工具的示例：

```python
# Create server parameters for stdio connection
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client
import asyncio

from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent
from langgraph.prebuilt import create_react_agent

server_params = {
    "url": "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
    "headers": {
        "X-Api-Key":"lsv2_pt_your_api_key"
    }
}

async def main():
    async with streamablehttp_client(**server_params) as (read, write, _):
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # Load the remote graph as if it was a tool
            tools = await load_mcp_tools(session)

            # Create and run a react agent with the tools
            agent = create_react_agent("openai:gpt-4.1", tools)

            # Invoke the agent with a message
            agent_response = await agent.ainvoke({"messages": "What can the finance agent do for me?"})
            print(agent_response)

if __name__ == "__main__":
    asyncio.run(main())
```

:::

:::js
使用 MCP 兼容客户端连接到 LangGraph 服务器。以下示例显示如何使用 [`@langchain/mcp-adapters`](https://npmjs.com/package/@langchain/mcp-adapters) 进行连接。

```bash
npm install @langchain/mcp-adapters
```

以下是如何连接到远程 MCP 端点并将代理用作工具的示例：

```typescript
import { MultiServerMCPClient } from "@langchain/mcp-adapters";
import { createReactAgent } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";

async function main() {
  const client = new MultiServerMCPClient({
    mcpServers: {
      "finance-agent": {
        url: "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
        headers: {
          "X-Api-Key": "lsv2_pt_your_api_key",
        },
      },
    },
  });

  const tools = await client.getTools();

  const model = new ChatOpenAI({
    model: "gpt-4o-mini",
    temperature: 0,
  });

  const agent = createReactAgent({
    model,
    tools,
  });

  const response = await agent.invoke({
    input: "What can the finance agent do for me?",
  });

  console.log(response);
}

main();
```

:::

## 会话行为 (Session behavior)

当前的 LangGraph MCP 实现不支持会话。每个 `/mcp` 请求都是无状态且独立的。

## 身份验证 (Authentication)

`/mcp` 端点使用与 LangGraph API 其余部分相同的身份验证。有关设置详细信息，请参阅 [身份验证指南](./auth.md)。

## 禁用 MCP (Disable MCP)

要禁用 MCP 端点，请在您的 `langgraph.json` 配置文件中将 `disable_mcp` 设置为 `true`：

```json
{
  "http": {
    "disable_mcp": true
  }
}
```

这将阻止服务器公开 `/mcp` 端点。
