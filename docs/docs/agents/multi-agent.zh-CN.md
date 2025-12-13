---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 多代理 (Multi-agent)

如果单个代理需要在多个领域进行专业化或管理许多工具，它可能会很吃力。为了解决这个问题，您可以将代理分解为更小的、独立的代理，并将它们组合成一个 [多代理系统 (multi-agent system)](../concepts/multi_agent.md)。

在多代理系统中，代理需要相互通信。它们通过 [handoffs (移交)](#handoffs) 来实现这一点——这是一个原始操作，描述了将控制权移交给哪个代理以及发送给该代理的有效负载。

两种最流行的多代理架构是：

- [主管 (supervisor)](#supervisor) — 各个代理作为一个中央主管代理进行协调。主管控制所有通信流和任务委派，根据当前上下文和任务要求决定调用哪个代理。
- [群体 (swarm)](#swarm) — 代理根据其专业化动态地互相移交控制权。系统会记住哪个代理是最后活跃的，确保在随后的交互中，对话从该代理恢复。

## 主管 (Supervisor)

![Supervisor](./assets/supervisor.png)

:::python
使用 [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py) 库创建主管多代理系统：

```bash
pip install langgraph-supervisor
```

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_supervisor import create_supervisor

def book_hotel(hotel_name: str):
    """Book a hotel"""
    return f"Successfully booked a stay at {hotel_name}."

def book_flight(from_airport: str, to_airport: str):
    """Book a flight"""
    return f"Successfully booked a flight from {from_airport} to {to_airport}."

flight_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_flight],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)

hotel_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_hotel],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
supervisor = create_supervisor(
    agents=[flight_assistant, hotel_assistant],
    model=ChatOpenAI(model="gpt-4o"),
    prompt=(
        "You manage a hotel booking assistant and a"
        "flight booking assistant. Assign work to them."
    )
).compile()

for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

:::

:::js
使用 [`@langchain/langgraph-supervisor`](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-supervisor) 库创建主管多代理系统：

```bash
npm install @langchain/langgraph-supervisor
```

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
// highlight-next-line
import { createSupervisor } from "langgraph-supervisor";

function bookHotel(hotelName: string) {
  /**Book a hotel*/
  return `Successfully booked a stay at ${hotelName}.`;
}

function bookFlight(fromAirport: string, toAirport: string) {
  /**Book a flight*/
  return `Successfully booked a flight from ${fromAirport} to ${toAirport}.`;
}

const flightAssistant = createReactAgent({
  llm: "openai:gpt-4o",
  tools: [bookFlight],
  stateModifier: "You are a flight booking assistant",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: "openai:gpt-4o",
  tools: [bookHotel],
  stateModifier: "You are a hotel booking assistant",
  // highlight-next-line
  name: "hotel_assistant",
});

// highlight-next-line
const supervisor = createSupervisor({
  agents: [flightAssistant, hotelAssistant],
  llm: new ChatOpenAI({ model: "gpt-4o" }),
  systemPrompt:
    "You manage a hotel booking assistant and a " +
    "flight booking assistant. Assign work to them.",
});

for await (const chunk of supervisor.stream({
  messages: [
    {
      role: "user",
      content: "book a flight from BOS to JFK and a stay at McKittrick Hotel",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

:::

## 群体 (Swarm)

![Swarm](./assets/swarm.png)

:::python
使用 [`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-py) 库创建群体多代理系统：

```bash
pip install langgraph-swarm
```

```python
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_swarm import create_swarm, create_handoff_tool

transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="Transfer user to the hotel-booking assistant.",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="Transfer user to the flight-booking assistant.",
)

flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
swarm = create_swarm(
    agents=[flight_assistant, hotel_assistant],
    default_active_agent="flight_assistant"
).compile()

for chunk in swarm.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

:::

:::js
使用 [`@langchain/langgraph-swarm`](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-swarm) 库创建群体多代理系统：

```bash
npm install @langchain/langgraph-swarm
```

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
// highlight-next-line
import { createSwarm, createHandoffTool } from "@langchain/langgraph-swarm";

const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
  description: "Transfer user to the hotel-booking assistant.",
});

const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
  description: "Transfer user to the flight-booking assistant.",
});

const flightAssistant = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  // highlight-next-line
  tools: [bookFlight, transferToHotelAssistant],
  stateModifier: "You are a flight booking assistant",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  // highlight-next-line
  tools: [bookHotel, transferToFlightAssistant],
  stateModifier: "You are a hotel booking assistant",
  // highlight-next-line
  name: "hotel_assistant",
});

// highlight-next-line
const swarm = createSwarm({
  agents: [flightAssistant, hotelAssistant],
  defaultActiveAgent: "flight_assistant",
});

for await (const chunk of swarm.stream({
  messages: [
    {
      role: "user",
      content: "book a flight from BOS to JFK and a stay at McKittrick Hotel",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

:::

## 移交 (Handoffs)

多代理交互中的一个常见模式是 **移交 (handoffs)**，其中一个代理将控制权 _移交_ 给另一个代理。移交允许您指定：

- **destination**: 要导航到的目标代理
- **payload**: 要传递给该代理的信息

:::python
这被 `langgraph-supervisor`（主管移交给单个代理）和 `langgraph-swarm`（单个代理可以移交给其他代理）使用。

要使用 `create_react_agent` 实现移交，您需要：

1.  创建一个特殊的工具，可以将控制权转移到不同的代理

    ```python
    def transfer_to_bob():
        """Transfer to bob."""
        return Command(
            # name of the agent (node) to go to
            # highlight-next-line
            goto="bob",
            # data to send to the agent
            # highlight-next-line
            update={"messages": [...]},
            # indicate to LangGraph that we need to navigate to
            # agent node in a parent graph
            # highlight-next-line
            graph=Command.PARENT,
        )
    ```

2.  创建可以访问移交工具的单个代理：

    ```python
    flight_assistant = create_react_agent(
        ..., tools=[book_flight, transfer_to_hotel_assistant]
    )
    hotel_assistant = create_react_agent(
        ..., tools=[book_hotel, transfer_to_flight_assistant]
    )
    ```

3.  定义包含单个代理作为节点的父图：

    ```python
    from langgraph.graph import StateGraph, MessagesState
    multi_agent_graph = (
        StateGraph(MessagesState)
        .add_node(flight_assistant)
        .add_node(hotel_assistant)
        ...
    )
    ```

:::

:::js
这被 `@langchain/langgraph-supervisor`（主管移交给单个代理）和 `@langchain/langgraph-swarm`（单个代理可以移交给其他代理）使用。

要使用 `createReactAgent` 实现移交，您需要：

1.  创建一个特殊的工具，可以将控制权转移到不同的代理

    ```typescript
    function transferToBob() {
      /**Transfer to bob.*/
      return new Command({
        // name of the agent (node) to go to
        // highlight-next-line
        goto: "bob",
        // data to send to the agent
        // highlight-next-line
        update: { messages: [...] },
        // indicate to LangGraph that we need to navigate to
        // agent node in a parent graph
        // highlight-next-line
        graph: Command.PARENT,
      });
    }
    ```

2.  创建可以访问移交工具的单个代理：

    ```typescript
    const flightAssistant = createReactAgent({
      ..., tools: [bookFlight, transferToHotelAssistant]
    });
    const hotelAssistant = createReactAgent({
      ..., tools: [bookHotel, transferToFlightAssistant]
    });
    ```

3.  定义包含单个代理作为节点的父图：

    ```typescript
    import { StateGraph, MessagesZodState } from "@langchain/langgraph";
    const multiAgentGraph = new StateGraph(MessagesZodState)
      .addNode("flight_assistant", flightAssistant)
      .addNode("hotel_assistant", hotelAssistant)
      // ...
    ```

    :::

综合起来，以下是如何实现一个简单的多代理系统，包含两个代理——一个航班预订助手和一个酒店预订助手：

:::python

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

# Handoffs
transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="Transfer user to the hotel-booking assistant.",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="Transfer user to the flight-booking assistant.",
)

# Simple agent tools
def book_hotel(hotel_name: str):
    """Book a hotel"""
    return f"Successfully booked a stay at {hotel_name}."

def book_flight(from_airport: str, to_airport: str):
    """Book a flight"""
    return f"Successfully booked a flight from {from_airport} to {to_airport}."

# Define agents
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# Define multi-agent graph
multi_agent_graph = (
    StateGraph(MessagesState)
    .add_node(flight_assistant)
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)

# Run the multi-agent graph
for chunk in multi_agent_graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

1. 访问代理的状态
2. `Command` 原语允许将状态更新和节点转换指定为单个操作，使其对实现移交非常有用。
3. 要移交到的代理或节点的名称。
4. 获取代理的消息并将其 **添加** 到父级的 **状态** 中，作为移交的一部分。下一个代理将看到父级状态。
5. 向 LangGraph 指示我们需要导航到 **父** 多代理图中的代理节点。
   :::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { ChatAnthropic } from "@langchain/anthropic";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import {
  StateGraph,
  START,
  MessagesZodState,
  Command,
} from "@langchain/langgraph";
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
      const toolMessage = {
        role: "tool" as const,
        content: `Successfully transferred to ${agentName}`,
        name: name,
        tool_call_id: config.toolCall?.id!,
      };
      return new Command({
        // (2)!
        // highlight-next-line
        goto: agentName, // (3)!
        // highlight-next-line
        update: { messages: [toolMessage] }, // (4)!
        // highlight-next-line
        graph: Command.PARENT, // (5)!
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({}),
    }
  );
}

// Handoffs
const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
  description: "Transfer user to the hotel-booking assistant.",
});

const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
  description: "Transfer user to the flight-booking assistant.",
});

// Simple agent tools
const bookHotel = tool(
  async ({ hotelName }) => {
    /**Book a hotel*/
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "book_hotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string().describe("Name of the hotel to book"),
    }),
  }
);

const bookFlight = tool(
  async ({ fromAirport, toAirport }) => {
    /**Book a flight*/
    return `Successfully booked a flight from ${fromAirport} to ${toAirport}.`;
  },
  {
    name: "book_flight",
    description: "Book a flight",
    schema: z.object({
      fromAirport: z.string().describe("Departure airport code"),
      toAirport: z.string().describe("Arrival airport code"),
    }),
  }
);

// Define agents
const flightAssistant = createReactAgent({
  llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }),
  // highlight-next-line
  tools: [bookFlight, transferToHotelAssistant],
  stateModifier: "You are a flight booking assistant",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }),
  // highlight-next-line
  tools: [bookHotel, transferToFlightAssistant],
  stateModifier: "You are a hotel booking assistant",
  // highlight-next-line
  name: "hotel_assistant",
});

// Define multi-agent graph
const multiAgentGraph = new StateGraph(MessagesZodState)
  .addNode("flight_assistant", flightAssistant)
  .addNode("hotel_assistant", hotelAssistant)
  .addEdge(START, "flight_assistant")
  .compile();

// Run the multi-agent graph
for await (const chunk of multiAgentGraph.stream({
  messages: [
    {
      role: "user",
      content: "book a flight from BOS to JFK and a stay at McKittrick Hotel",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

1. 访问代理的状态
2. `Command` 原语允许将状态更新和节点转换指定为单个操作，使其对实现移交非常有用。
3. 要移交到的代理或节点的名称。
4. 获取代理的消息并将其 **添加** 到父级的 **状态** 中，作为移交的一部分。下一个代理将看到父级状态。
5. 向 LangGraph 指示我们需要导航到 **父** 多代理图中的代理节点。

:::

!!! Note

    此移交实现假设：

    - 每个代理接收多代理系统中的整体消息历史记录（跨所有代理）作为其输入
    - 每个代理将其内部消息历史记录输出到多代理系统的整体消息历史记录中

:::python
查看 LangGraph [supervisor](https://github.com/langchain-ai/langgraph-supervisor-py#customizing-handoff-tools) 和 [swarm](https://github.com/langchain-ai/langgraph-swarm-py#customizing-handoff-tools) 文档以了解如何自定义移交。
:::

:::js
查看 LangGraph [supervisor](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-supervisor#customizing-handoff-tools) 和 [swarm](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-swarm#customizing-handoff-tools) 文档以了解如何自定义移交。
:::
