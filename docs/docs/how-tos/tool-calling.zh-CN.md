# 如何调用工具 (How to call tools)

[工具调用](../concepts/tools.md) 是一种允许模型与外部系统（如数据库、API 等）进行交互的功能。

本指南涵盖了以下内容：

- [定义工具](#define-tools): 创建与 Python/JavaScript 函数和 Pydantic/Zod 模型相绑定的工具。
- [运行时绑定](#runtime-binding): 在图运行时动态地将工具绑定到模型。
- [工具节点](#toolnode): 使用预构建的 `ToolNode` 来执行工具。
- [工具定制](#tool-customization): 自定义工具执行、异常处理和结果返回。
- [高级功能](#advanced-tool-features): 强制工具使用、禁用并行调用、处理大量工具和动态工具选择。

!!! tip "Prebuilt Agent"
    如果您正在寻找能够开箱即用调用工具的预构建代理，请参阅 `create_react_agent` [Python](https://langchain-ai.github.io/langgraph/reference/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent) / [JS](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.createReactAgent.html) 参考文档。

## 定义工具 (Define tools)

工具是 LLM 如何与外界交互的接口。最简单的工具定义方式是使用装饰器 `@tool` (Python) 或 `tool` 函数 (JS)。

<details>
<summary>查看 Python 代码</summary>

```python
from langchain_core.tools import tool

@tool
def get_weather(location: str):
    """Call to get the current weather."""
    if location.lower() in ["sf", "san francisco"]:
        return "It's 60 degrees and foggy."
    else:
        return "It's 90 degrees and sunny."

@tool
def get_coolest_cities():
    """Get a list of coolest cities"""
    return "nyc, sf"

tools = [get_weather, get_coolest_cities]
```

您还可以使用 Pydantic 模型来定义工具架构：

```python
from pydantic import BaseModel, Field

class GetWeather(BaseModel):
    """Call to get the current weather."""
    location: str = Field(description="Location to get the weather for.")

tools = [GetWeather]
```

</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(
  (input) => {
    if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
      return "It's 60 degrees and foggy.";
    } else {
      return "It's 90 degrees and sunny.";
    }
  },
  {
    name: "get_weather",
    description: "Call to get the current weather.",
    schema: z.object({
      location: z.string().describe("Location to get the weather for."),
    }),
  }
);

const getCoolestCities = tool(
  () => {
    return "nyc, sf";
  },
  {
    name: "get_coolest_cities",
    description: "Get a list of coolest cities",
    schema: z.object({}),
  }
);

const tools = [getWeather, getCoolestCities];
```
</details>

## 运行时绑定 (Runtime Binding)

要在运行时将这些工具绑定到 LLM，您可以使用 `bind_tools` 方法。此方法接受工具列表（Pydantic/Zod 模型、函数或 LangChain 工具），并配置模型以知道可以使用这些工具。

请注意，调用 `bind_tools` 并*不*执行工具。它只是通过将工具架构作为参数传递给 LLM，使其意识到工具的存在。

<details>
<summary>查看 Python 代码</summary>

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")
model_with_tools = model.bind_tools(tools)

# Verify the binding
model_with_tools.invoke("what is the weather in sf?")
```

```python
AIMessage(
    content="",
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "sf"},
        "id": "call_12345",
        "type": "tool_call"
    }]
)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ model: "gpt-4o" });
const modelWithTools = model.bindTools(tools);

// Verify the binding
await modelWithTools.invoke("what is the weather in sf?");
```

```typescript
AIMessage {
  content: "",
  tool_calls: [{
    name: "get_weather",
    args: { location: "sf" },
    id: "call_12345",
    type: "tool_call"
  }]
}
```
</details>

## 工具节点 (ToolNode)

LangGraph 提供了一个预构建的 `ToolNode` 组件，用于在图中执行工具调用。它接受工具列表，并将根据状态中的最后一条消息执行它们。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.prebuilt import ToolNode

tool_node = ToolNode(tools)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { ToolNode } from "@langchain/langgraph/prebuilt";

const toolNode = new ToolNode(tools);
```
</details>

### 与图集成 (Integration with Graph)

以下是如何将工具定义、模型绑定和 `ToolNode` 执行结合在一起，构建一个完整的工具调用代理。

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Literal
from langgraph.graph import StateGraph, MessagesState, START, END

def should_continue(state: MessagesState) -> Literal["tools", END]:
    messages = state['messages']
    last_message = messages[-1]
    if last_message.tool_calls:
        return "tools"
    return END

def call_model(state: MessagesState):
    messages = state['messages']
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}

workflow = StateGraph(MessagesState)

# Define the two nodes we will cycle between
workflow.add_node("agent", call_model)
# highlight-next-line
workflow.add_node("tools", tool_node)

workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, ["tools", END])
workflow.add_edge("tools", "agent")

app = workflow.compile()
```

```python
for chunk in app.stream(
    {"messages": [("human", "what's the weather in sf?")]}, stream_mode="values"
):
    chunk["messages"][-1].pretty_print()
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { isAIMessage } from "@langchain/core/messages";

const shouldContinue = (state: z.infer<typeof MessagesZodState>) => {
  const messages = state.messages;
  const lastMessage = messages.at(-1);
  if (lastMessage && isAIMessage(lastMessage) && lastMessage.tool_calls?.length) {
    return "tools";
  }
  return END;
};

const callModel = async (state: z.infer<typeof MessagesZodState>) => {
  const messages = state.messages;
  const response = await modelWithTools.invoke(messages);
  return { messages: [response] };
};

const workflow = new StateGraph(MessagesZodState)
  // Define the two nodes we will cycle between
  .addNode("agent", callModel)
  // highlight-next-line
  .addNode("tools", toolNode)
  .addEdge(START, "agent")
  .addConditionalEdges("agent", shouldContinue, ["tools", END])
  .addEdge("tools", "agent");

const app = workflow.compile();

const inputs = {
  messages: [{ role: "user", content: "what's the weather in sf?" }],
};

for await (const chunk of await app.stream(inputs, { streamMode: "values" })) {
  console.log(chunk.messages.at(-1));
}
```
</details>

### 手动执行工具 (Manually execute tools)

虽然 `ToolNode` 处理了常见的工具执行逻辑，但有时您可能需要手动调用工具节点。

<details>
<summary>查看 Python 代码</summary>

```python
from langchain_core.messages import AIMessage

# Manually create a message with a tool call
message_with_tool_call = AIMessage(
    content="",
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "San Francisco"},
        "id": "tool_call_id",
        "type": "tool_call"
    }]
)

# Invoke ToolNode manually
tool_node.invoke({"messages": [message_with_tool_call]})
```

```python
{'messages': [
    ToolMessage(
        content="It's 60 degrees and foggy.",
        name='get_weather',
        tool_call_id='tool_call_id'
    )
]}
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { AIMessage } from "@langchain/core/messages";

// Manually create a message with a tool call
const messageWithToolCall = new AIMessage({
  content: "",
  tool_calls: [{
    name: "get_weather",
    args: { location: "San Francisco" },
    id: "tool_call_id",
    type: "tool_call"
  }]
});

// Invoke ToolNode manually
await toolNode.invoke({ messages: [messageWithToolCall] });
```
</details>

## 工具定制 (Tool customization)

### 上下文管理 (Context management)

工具通常需要运行时上下文，例如用户 ID 或会话详细信息。LangGraph 提供了三种管理上下文的方法：

| 类型 | 使用场景 | 可变性 | 生命周期 |
| --- | --- | --- | --- |
| [配置 (Configuration)](../concepts/low_level.md#configuration) | 静态、不可变的运行时数据（如 user_id） | ❌ | 单次调用 |
| [短期记忆 (Short-term memory)](#short-term-memory) | 在单个执行过程中更改的动态数据 | ✅ | 单次调用 |
| [长期记忆 (Long-term memory)](#long-term-memory) | 跨会话持久化的数据 | ✅ | 跨会话 |

#### 配置 (Configuration)

当工具有 **不可变** 的运行时数据需求时使用配置。通过 `RunnableConfig` 传递参数。

<details>
<summary>查看 Python 代码</summary>

```python
from langchain_core.tools import tool
from langchain_core.runnables import RunnableConfig

@tool
def get_user_info(config: RunnableConfig) -> str:
    """Retrieve user information based on user ID."""
    user_id = config["configurable"].get("user_id")
    return "User is John Smith" if user_id == "user_123" else "Unknown user"
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config?.configurable?.user_id;
    return userId === "user_123" ? "User is John Smith" : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Retrieve user information based on user ID.",
    schema: z.object({}),
  }
);
```
</details>

#### 短期记忆 (Short-term memory)

短期记忆维持在单次执行期间更改的 **动态** 状态。使用 `InjectedState` 注解来访问状态。

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState
from langgraph.prebuilt.chat_agent_executor import AgentState

@tool
def get_user_name(state: Annotated[AgentState, InjectedState]) -> str:
    """Retrieve the current user-name from state."""
    return state.get("user_name", "Unknown user")
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { getContextVariable } from "@langchain/core/context";

const getUserName = tool(
  async (_, config) => {
    const currentState = getContextVariable("currentState");
    return currentState?.userName || "Unknown user";
  },
  {
    name: "get_user_name",
    description: "Retrieve the current user name from state.",
    schema: z.object({}),
  }
);
```
</details>

要 **更新** 状态，工具可以返回 `Command` 对象：

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.types import Command

@tool
def update_user_name(new_name: str, tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
    """Update user-name in short-term memory."""
    return Command(update={
        "user_name": new_name,
        "messages": [ToolMessage(f"Updated user name to {new_name}", tool_call_id=tool_call_id)]
    })
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { Command } from "@langchain/langgraph";

const updateUserName = tool(
  async (input) => {
    return new Command({
      update: {
        userName: input.newName,
        messages: [{
          role: "tool",
          content: `Updated user name to ${input.newName}`,
          // ...
        }],
      },
    });
  },
  // ...
);
```
</details>

#### 长期记忆 (Long-term memory)

[长期记忆](../concepts/memory.md#long-term-memory) 允许跨线程存储信息。使用 `get_store()` 访问存储。

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.config import get_store

@tool
def get_user_info(config: RunnableConfig) -> str:
    store = get_store()
    user_id = config["configurable"].get("user_id")
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const store = config.store;
    if (!store) throw new Error("Store not provided");
    
    const userId = config?.configurable?.user_id;
    const userInfo = await store.get(["users"], userId);
    return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
  },
  // ...
);
```
</details>

## 高级工具功能 (Advanced tool features)

### 立即返回 (Immediate return)

使用 `return_direct=True` (Python) 或 `returnDirect: true` (JS) 来立即返回工具的结果给用户，而不执行后续逻辑。

<details>
<summary>查看 Python 代码</summary>

```python
@tool(return_direct=True)
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const add = tool(
  (input) => input.a + input.b,
  {
    name: "add",
    description: "Add two numbers",
    schema: z.object({ a: z.number(), b: z.number() }),
    returnDirect: true,
  }
);
```
</details>

### 强制使用工具 (Force tool use)

您可以通过 `tool_choice` 参数强制模型使用特定工具：

<details>
<summary>查看 Python 代码</summary>

```python
configured_model = model.bind_tools(
    tools,
    tool_choice={"type": "tool", "name": "greet"}
)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
const configuredModel = model.bindTools(
  tools,
  { tool_choice: { type: "tool", name: "greet" } }
);
```
</details>

### 禁用并行调用 (Disable parallel calls)

对于支持的提供商，您可以禁用并行工具调用：

<details>
<summary>查看 Python 代码</summary>

```python
model.bind_tools(tools, parallel_tool_calls=False)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
model.bindTools(tools, { parallel_tool_calls: false });
```
</details>

### 错误处理 (Handle errors)

`ToolNode` 默认捕获工具执行期间的异常并返回包含错误状态的 `ToolMessage`。您可以自定义错误消息或禁用错误捕获。

<details>
<summary>查看 Python 代码</summary>

```python
# Custom error message
tool_node = ToolNode(
    [multiply],
    handle_tool_errors="Can't use 42 as the first operand!"
)

# Disable error handling (propagate exceptions)
tool_node = ToolNode([multiply], handle_tool_errors=False)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
// Custom error message
const toolNode = new ToolNode([multiply], {
  handleToolErrors: "Can't use 42 as the first operand!",
});

// Disable error handling (propagate exceptions)
const toolNode = new ToolNode([multiply], { handleToolErrors: false });
```
</details>

### 处理大量工具 (Handle large numbers of tools)

当工具数量众多时，可以使用 [`langgraph-bigtool`](https://github.com/langchain-ai/langgraph-bigtool) 库通过语义搜索动态检索相关工具，以减少令牌消耗并提高推理准确性。

## 预构建工具 (Prebuilt tools)

LangGraph 支持来自模型提供商（如 OpenAI 的 web_search_preview）和 LangChain 集成（如 Bing, SerpAPI, SQL, MongoDB 等）的预构建工具。这些工具可以像自定义工具一样绑定并添加到您的代理中。
