# 构建多智能体系统 (Build multi-agent systems)

单个智能体如果需要专精于多个领域或管理许多工具，可能会显得力不从心。为了解决这个问题，您可以将智能体分解为更小的、独立的智能体，并将它们组合成一个 [多智能体系统](../concepts/multi_agent.md)。

在多智能体系统中，智能体之间需要进行通信。它们通过 [交接 (handoffs)](#handoffs) 来实现——这是一种原语，用于描述将控制权交给哪个智能体以及发送给该智能体的有效载荷。

本指南涵盖以下内容：

* 实现智能体之间的 [交接 (handoffs)](#handoffs)
* 使用交接和预构建的 [智能体](../agents/agents.md) 来 [构建自定义多智能体系统](#build-a-multi-agent-system)

要开始构建多智能体系统，请查看 LangGraph 中两种最流行的多智能体架构的 [预构建实现](#prebuilt-implementations) —— [supervisor（监督者）](../agents/multi-agent.md#supervisor) 和 [swarm（蜂群）](../agents/multi-agent.md#swarm)。

## 交接 (Handoffs)

要在多智能体系统中设置智能体之间的通信，您可以使用 [**交接**](../concepts/multi_agent.md#handoffs) —— 一种一个智能体将控制权 *交接* 给另一个智能体的模式。交接允许您指定：

- **目的地 (destination)**：要导航到的目标智能体（例如，要前往的 LangGraph 节点的名称）
- **有效载荷 (payload)**：传递给该智能体的信息（例如，状态更新）

### 创建交接 (Create handoffs)

要实现交接，您可以从您的智能体节点或工具中返回 `Command` 对象：

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import create_react_agent, InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Transfer to {agent_name}"

    @tool(name, description=description)
    def handoff_tool(
        # highlight-next-line
        state: Annotated[MessagesState, InjectedState], # (1)!
        # highlight-next-line
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        return Command(  # (2)!
            # highlight-next-line
            goto=agent_name,  # (3)!
            # highlight-next-line
            update={"messages": state["messages"] + [tool_message]},  # (4)!
            # highlight-next-line
            graph=Command.PARENT,  # (5)!
        )
    return handoff_tool
```

1. 使用 @[InjectedState] 注解访问调用交接工具的智能体的 [状态](../concepts/low_level.md#state)。
2. `Command` 原语允许将状态更新和节点转换指定为单个操作，使其对于实现交接非常有用。
3. 要交接给的智能体或节点的名称。
4. 获取智能体的消息并将它们 **添加** 到父级的 **状态** 中，作为交接的一部分。下一个智能体将看到父级状态。
5. 向 LangGraph 指示我们需要导航到 **父** 多智能体图中的智能体节点。

!!! tip
    如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] 组件，或者实现您自己的工具执行节点，该节点收集工具返回的 `Command` 对象并返回它们的列表，例如：
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import { Command, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  const name = `transfer_to_${agentName}`;
  const toolDescription = description || `Transfer to ${agentName}`;

  return tool(
    async (_, config) => {
      // (1)!
      const state = config.state;
      const toolCallId = config.toolCall.id;

      const toolMessage = {
        role: "tool" as const,
        content: `Successfully transferred to ${agentName}`,
        name: name,
        tool_call_id: toolCallId,
      };

      return new Command({
        // (3)!
        goto: agentName,
        // (4)!
        update: { messages: [...state.messages, toolMessage] },
        // (5)!
        graph: Command.PARENT,
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({}),
    }
  );
}
```

1. 通过 `config` 参数访问调用交接工具的智能体的 [状态](../concepts/low_level.md#state)。
2. `Command` 原语允许将状态更新和节点转换指定为单个操作，使其对于实现交接非常有用。
3. 要交接给的智能体或节点的名称。
4. 获取智能体的消息并将它们 **添加** 到父级的 **状态** 中，作为交接的一部分。下一个智能体将看到父级状态。
5. 向 LangGraph 指示我们需要导航到 **父** 多智能体图中的智能体节点。

!!! tip
    如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] 组件，或者实现您自己的工具执行节点，该节点收集工具返回的 `Command` 对象并返回它们的列表，例如：
    
    ```typescript
    const callTools = async (state) => {
      // ...
      const commands = await Promise.all(
        toolCalls.map(toolCall => toolsByName[toolCall.name].invoke(toolCall))
      );
      return commands;
    };
    ```
</details>

!!! Important
    此交接实现假设：
    
    - 多智能体系统中的每个智能体都接收整体消息历史记录（跨所有智能体）作为其输入。如果您想要更好地控制智能体输入，请参阅 [此部分](#control-agent-inputs)
    - 每个智能体将其内部消息历史记录输出到多智能体系统的整体消息历史记录中。如果您想要更好地控制 **如何添加智能体输出**，请将智能体包装在一个单独的节点函数中：

    <details>
    <summary>查看 Python 代码</summary>

    ```python
    def call_hotel_assistant(state):
        # return agent's final response,
        # excluding inner monologue
        response = hotel_assistant.invoke(state)
        # highlight-next-line
        return {"messages": response["messages"][-1]}
    ```
    </details>

    <details>
    <summary>查看 TypeScript 代码</summary>

    ```typescript
    const callHotelAssistant = async (state) => {
      # return agent's final response,
      # excluding inner monologue
      const response = await hotelAssistant.invoke(state);
      # highlight-next-line
      return { messages: [response.messages.at(-1)] };
    };
    ```
    </details>

### 控制智能体输入 (Control agent inputs)

您可以使用 @[`Send()`][Send] 原语直接在交接期间向工作智能体发送数据。例如，您可以请求调用智能体为下一个智能体填充任务描述：

<details>
<summary>查看 Python 代码</summary>

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
# highlight-next-line
from langgraph.types import Command, Send

def create_task_description_handoff_tool(
    *, agent_name: str, description: str | None = None
):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        # this is populated by the calling agent
        task_description: Annotated[
            str,
            "Description of what the next agent should do, including all of the relevant context.",
        ],
        # these parameters are ignored by the LLM
        state: Annotated[MessagesState, InjectedState],
    ) -> Command:
        task_description_message = {"role": "user", "content": task_description}
        agent_input = {**state, "messages": [task_description_message]}
        return Command(
            # highlight-next-line
            goto=[Send(agent_name, agent_input)],
            graph=Command.PARENT,
        )

    return handoff_tool
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { tool } from "@langchain/core/tools";
import { Command, Send, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createTaskDescriptionHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  const name = `transfer_to_${agentName}`;
  const toolDescription = description || `Ask ${agentName} for help.`;

  return tool(
    async (
      { taskDescription },
      config
    ) => {
      const state = config.state;
      
      const taskDescriptionMessage = {
        role: "user" as const,
        content: taskDescription,
      };
      const agentInput = {
        ...state,
        messages: [taskDescriptionMessage],
      };
      
      return new Command({
        // highlight-next-line
        goto: [new Send(agentName, agentInput)],
        graph: Command.PARENT,
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({
        taskDescription: z
          .string()
          .describe(
            "Description of what the next agent should do, including all of the relevant context."
          ),
      }),
    }
  );
}
```
</details>

有关在交接中使用 @[`Send()`][Send] 的完整示例，请参阅多智能体 [supervisor（监督者）](../tutorials/multi_agent/agent_supervisor.md#4-create-delegation-tasks) 示例。

## 构建多智能体系统 (Build a multi-agent system)

您可以在任何使用 LangGraph 构建的智能体中使用交接。我们建议使用预构建的 [智能体](../agents/overview.md) 或 [`ToolNode`](./tool-calling.md#toolnode)，因为它们原生支持返回 `Command` 的交接工具。以下是如何使用交接实现用于预订旅行的多智能体系统的示例：

<details>
<summary>查看 Python 代码</summary>

```python
from langgraph.prebuilt import create_react_agent
from langgraph.graph import StateGraph, START, MessagesState

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    # same implementation as above
    ...
    return Command(...)

# Handoffs
transfer_to_hotel_assistant = create_handoff_tool(agent_name="hotel_assistant")
transfer_to_flight_assistant = create_handoff_tool(agent_name="flight_assistant")

# Define agents
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_hotel_assistant],
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_flight_assistant],
    # highlight-next-line
    name="hotel_assistant"
)

# Define multi-agent graph
multi_agent_graph = (
    StateGraph(MessagesState)
    # highlight-next-line
    .add_node(flight_assistant)
    # highlight-next-line
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { StateGraph, START, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  // same implementation as above
  // ...
  return new Command(/* ... */);
}

// Handoffs
const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
});
const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
});

// Define agents
const flightAssistant = createReactAgent({
  llm: model,
  // highlight-next-line
  tools: [/* ... */, transferToHotelAssistant],
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: model,
  // highlight-next-line
  tools: [/* ... */, transferToFlightAssistant],
  // highlight-next-line
  name: "hotel_assistant",
});

// Define multi-agent graph
const multiAgentGraph = new StateGraph(MessagesZodState)
  // highlight-next-line
  .addNode("flight_assistant", flightAssistant)
  // highlight-next-line
  .addNode("hotel_assistant", hotelAssistant)
  .addEdge(START, "flight_assistant")
  .compile();
```
</details>

<details>
<summary>完整示例：用于预订旅行的多智能体系统</summary>

这里展示了完整的 Python 和 TypeScript 代码示例，包含美观的消息打印辅助函数、交接工具创建以及多智能体图的定义和运行。请参考上面代码块中的详细实现。
</details>

## 多轮对话 (Multi-turn conversation)

用户可能希望与一个或多个智能体进行 *多轮对话*。要构建能够处理这种情况的系统，您可以创建一个使用 @[`interrupt`][interrupt] 来收集用户输入并路由回 **活动** 智能体的节点。

然后，智能体可以作为图中的节点来实现，该图执行智能体步骤并确定下一个操作：

1. **等待用户输入** 以继续对话，或者
2. 通过 [交接](#handoffs) **路由到另一个智能体**（或路由回自身，如在循环中）

<details>
<summary>查看 Python 代码</summary>

```python
def human(state) -> Command[Literal["agent", "another_agent"]]:
    """A node for collecting user input."""
    user_input = interrupt(value="Ready for user input.")

    # Determine the active agent.
    active_agent = ...

    ...
    return Command(
        update={
            "messages": [{
                "role": "human",
                "content": user_input,
            }]
        },
        goto=active_agent
    )

def agent(state) -> Command[Literal["agent", "another_agent", "human"]]:
    # The condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    if goto:
        return Command(goto=goto, update={"my_state_key": "my_state_value"})
    else:
        return Command(goto="human") # Go to human node
```
</details>

<details>
<summary>查看 TypeScript 代码</summary>

```typescript
import { interrupt, Command } from "@langchain/langgraph";

function human(state: MessagesState): Command {
  const userInput: string = interrupt("Ready for user input.");

  // Determine the active agent
  const activeAgent = /* ... */;

  return new Command({
    update: {
      messages: [{
        role: "human",
        content: userInput,
      }]
    },
    goto: activeAgent,
  });
}

function agent(state: MessagesState): Command {
  // The condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
  const goto = getNextAgent(/* ... */); // 'agent' / 'anotherAgent'

  if (goto) {
    return new Command({
      goto,
      update: { myStateKey: "myStateValue" }
    });
  }

  return new Command({ goto: "human" });
}
```
</details>

<details>
<summary>完整示例：用于旅行推荐的多智能体系统</summary>

这个例子展示了如何构建一个旅行顾问和酒店顾问的团队，它们可以通过交接相互通信，并且包含一个收集人类输入的节点来支持多轮对话。请参考上面代码块中的详细 Python 和 TypeScript 实现。
</details>

## 预构建实现 (Prebuilt implementations)

LangGraph 附带了两种最流行的多智能体架构的预构建实现：

- [supervisor（监督者）](../agents/multi-agent.md#supervisor) — 单个智能体由中央监督者智能体协调。监督者控制所有通信流和任务委托，根据当前上下文和任务要求决定调用哪个智能体。您可以使用 `langgraph-supervisor` 库来创建监督者多智能体系统。
- [swarm（蜂群）](../agents/multi-agent.md#supervisor) — 智能体根据其专长动态地将控制权交给彼此。系统会记住哪个智能体是最后活动的，确保在后续交互中，对话会与该智能体恢复。您可以使用 `langgraph-swarm` 库来创建蜂群多智能体系统。
