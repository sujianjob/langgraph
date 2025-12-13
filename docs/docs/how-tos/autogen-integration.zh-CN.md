# 如何将 LangGraph 与 AutoGen 结合使用

[AutoGen](https://microsoft.github.io/autogen/) 是一个用于构建多智能体应用程序的框架。LangGraph 可以与 AutoGen 集成，为您现有的 AutoGen 工作流添加持久化、流式传输、人工介入和从 LangSmith 获得的额外可观测性等功能。

要将 AutoGen 与 LangGraph 结合使用，您只需：

1. 定义您的 [AutoGen 智能体](https://microsoft.github.io/autogen/docs/tutorial/introduction)
2. 定义一个调用 AutoGen 智能体的 LangGraph 节点
3. 照常 [编译和运行图](../concepts/low_level.md#compiling-your-graph)

这样，您就可以部署您的 AutoGen 智能体并在 [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) 上运行它们。

## 设置 (Setup)

从安装 AutoGen 和 LangGraph 开始：

```bash
pip install autogen-agentchat~=0.2 langgraph
```

然后，我们将设置一些环境变量。

```python
import os
import getpass

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")

_set_env("OPENAI_API_KEY")
```

## 定义 AutoGen 智能体 (Define AutoGen Agents)

请参照 [AutoGen 文档](https://microsoft.github.io/autogen/docs/tutorial/introduction) 来定义您的智能体。在这里，我们将创建一个简单的助手智能体和一个用户代理智能体。

```python
from autogen import AssistantAgent, UserProxyAgent

config_list = [{"model": "gpt-4o", "api_key": os.environ["OPENAI_API_KEY"]}]

def get_agents():
    assistant = AssistantAgent(
        "assistant",
        llm_config={"config_list": config_list},
    )
    user_proxy = UserProxyAgent(
        "user_proxy",
        human_input_mode="NEVER",
        code_execution_config={"work_dir": "coding", "use_docker": False},
        max_consecutive_auto_reply=1,
    )
    return assistant, user_proxy
```

## 包装在 LangGraph 中 (Wrap in LangGraph)

我们将定义一个简单的图，它只有一步：运行 AutoGen 对话。

<details>
<summary>点击展开查看 AutoGen 兼容性包装器代码</summary>

```python
import asyncio
from typing import Any, Dict, List, Optional, Union
from autogen import Agent, GroupChat, GroupChatManager
from langchain_core.messages import (
    AIMessage,
    BaseMessage,
    FunctionMessage,
    HumanMessage,
    SystemMessage,
)
from pydantic import BaseModel, Field


class AutoGenMessage(BaseModel):
    content: str
    role: str
    name: Optional[str] = None
    function_call: Optional[Dict[str, Any]] = None


def _langchain_to_autogen(message: BaseMessage) -> Dict[str, Any]:
    if isinstance(message, HumanMessage):
        return {"content": message.content, "role": "user", "name": message.name}
    elif isinstance(message, AIMessage):
        return {"content": message.content, "role": "assistant", "name": message.name}
    elif isinstance(message, SystemMessage):
        return {"content": message.content, "role": "system", "name": message.name}
    elif isinstance(message, FunctionMessage):
        return {
            "content": message.content,
            "role": "function",
            "name": message.name,
        }
    else:
        raise ValueError(f"Unsupported message type: {type(message)}")


def _autogen_to_langchain(message: Dict[str, Any]) -> BaseMessage:
    role = message.get("role")
    content = message.get("content")
    name = message.get("name")
    if role == "user":
        return HumanMessage(content=content, name=name)
    elif role == "assistant":
        return AIMessage(content=content, name=name)
    elif role == "system":
        return SystemMessage(content=content, name=name)
    elif role == "function":
        return FunctionMessage(content=content, name=name)
    else:
        # Fallback for other types or custom roles
        return HumanMessage(content=str(content), name=name or role)


class AutoGenGraphState(BaseModel):
    messages: List[BaseMessage] = Field(default_factory=list)


async def run_autogen_conversation(state: AutoGenGraphState):
    """Running AutoGen conversation and syncing the state."""
    assistant, user_proxy = get_agents()
    
    # Initialize chat history from LangGraph state
    if state.messages:
        # AutoGen expects the first message to be from the user to start the chat
        # or we need to use `initiate_chat` which handles the first message.
        # For simplicity, we'll assume the last message in LangGraph is the trigger
        last_msg = state.messages[-1]
        
        # We need to run this in a thread because AutoGen is synchronous by default
        # and we are in an async context
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(
            None, 
            lambda: user_proxy.initiate_chat(
                assistant, 
                message=last_msg.content, 
                clear_history=False
            )
        )
        
    # Extract messages from AutoGen agent
    # Note: This is a simplification. You might need to access the group chat messages
    # or the specific agent's chat history depending on your setup.
    autogen_messages = user_proxy.chat_messages[assistant]
    
    # Convert new AutoGen messages back to LangChain format
    # We skip the ones we already have (simplification)
    new_messages = []
    # Simple logic: take all messages that came after our initial trigger
    # In a real app, you'd want more robust deduplication
    for msg in autogen_messages:
        langchain_msg = _autogen_to_langchain(msg)
        # Rudimentary check to avoid duplication of the trigger message
        # In production, use IDs or more sophisticated state management
        if langchain_msg.content != state.messages[-1].content:
             new_messages.append(langchain_msg)

    return {"messages": new_messages}
```
</details>

在上面的 `run_autogen_conversation` 节点中，我们：

1. **初始化智能体**：创建 AutoGen 智能体。
2. **同步状态**：可以使用 `user_proxy.initiate_chat` 的参数或预先填充聊天记录，将 `LangGraph` 状态中的消息加载到 AutoGen 中。
3. **运行 AutoGen**：调用 `user_proxy.initiate_chat` 开始对话。
4. **提取结果**：从 AutoGen 智能体的历史记录中提取消息，并将其转换回 LangChain 格式以更新图状态。

## 定义并运行图 (Define and Run Graph)

现在我们将节点放入图中。

```python
from langgraph.graph import StateGraph, END

builder = StateGraph(AutoGenGraphState)
builder.add_node("autogen", run_autogen_conversation)
builder.set_entry_point("autogen")
builder.add_edge("autogen", END)

graph = builder.compile()
```

### 测试本地运行 (Test Run Locally)

```python
messages = [HumanMessage(content="Plot a chart of NVDA and TSLA stock price change YTD.")]
for event in graph.stream({"messages": messages}):
    for value in event.values():
        print("---")
        for msg in value["messages"]:
            print(f"{msg.type}: {msg.content[:100]}...")
```

## 为部署做准备 (Prepare for Deployment)

要从 [LangGraph Platform](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/) 部署此应用程序，您必须：

1. 将上面定义的 `graph` 放入 Python 文件中。
2. 添加 `langgraph.json` 配置文件。
3. 指定依赖项（包括 `autogen-agentchat`）。

有关更多详细信息，请参阅[部署指南](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/)。

这种集成的关键优势是，您现在可以获得 LangGraph 的所有生产就绪功能：

- **持久化 (Persistence)**：图状态（包括 AutoGen 对话历史记录）会自动保存。
- **流式传输 (Streaming)**：如上所示，您可以流式传输执行更新。
- **记忆 (Memory)**：跨会话维护用户上下文。
- **人工介入 (Human-in-the-loop)**：您可以在执行 AutoGen 之前或之后（或期间，如果您将 AutoGen 步骤分解）使用 LangGraph 的 `interrupt` 来批准操作。
