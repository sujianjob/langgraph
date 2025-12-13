# 多智能体系统 (Multi-agent systems)

[智能体](./agentic_concepts.md#agent-architectures) 是_一个使用 LLM 来决定应用程序控制流的系统_。随着您开发这些系统，它们可能会随着时间的推移变得更加复杂，从而使管理和扩展变得更加困难。例如，您可能会遇到以下问题：

- 智能体可使用的工具太多，并且对下一步调用哪个工具做出了糟糕的决定
- 上下文变得过于复杂，单个智能体无法跟踪
- 系统中需要多个专业领域（例如规划师、研究员、数学专家等）

为了解决这些问题，您可以考虑将应用程序分解为多个更小的、独立的智能体，并将它们组合成原本的 **多智能体系统**。这些独立的智能体可以像提示和 LLM 调用一样简单，也可以像 [ReAct](./agentic_concepts.md#tool-calling-agent) 代理（甚至更多！）一样复杂。

使用多智能体系统的主要好处是：

- **模块化**：独立的智能体使开发、测试和维护智能体系统变得更加容易。
- **专业化**：您可以创建专注于特定领域的专家智能体，这有助于提高整体系统性能。
- **控制**：您可以明确控制智能体之间的通信方式（而不是仅仅依赖于函数调用）。

## 多智能体架构

![](./img/multi_agent/architectures.png)

连接多智能体系统中的智能体有几种方法：

- **网络 (Network)**：每个智能体都可以与 [所有其他智能体](../tutorials/multi_agent/multi-agent-collaboration.ipynb/) 通信。任何智能体都可以决定下一步调用哪个其他智能体。
- **主管 (Supervisor)**：每个智能体都与单个 [主管](../tutorials/multi_agent/agent_supervisor.md/) 智能体通信。主管智能体决定下一步应该调用哪个智能体。
- **主管（工具调用）(Supervisor (tool-calling))**：这是主管架构的一种特殊情况。个别智能体可以表示为工具。在这种情况下，主管智能体使用工具调用 LLM 来决定调用哪个智能体工具，以及传递给这些智能体的参数。
- **分层 (Hierarchical)**：您可以定义一个 [主管的主管](../tutorials/multi_agent/hierarchical_agent_teams.ipynb/) 的多智能体系统。这是主管架构的泛化，允许更复杂的控制流。
- **自定义多智能体工作流 (Custom multi-agent workflow)**：每个智能体仅与智能体的一个子集通信。部分流程是确定性的，只有部分智能体可以决定下一步调用哪个其他智能体。

### 切换 (Handoffs)

在多智能体架构中，智能体可以表示为图节点。每个智能体节点执行其步骤并决定是完成执行还是路由到另一个智能体，包括可能路由到自身（例如，在循环中运行）。多智能体交互中的一种常见模式是 **切换 (handoffs)**，其中一个智能体将控制权 _移交_ 给另一个智能体。切换允许您指定：

- **目的地 (destination)**：要导航到的目标智能体（例如，要转到的节点名称）
- **有效负载 (payload)**：[传递给该智能体的信息](#communication-and-state-management)（例如，状态更新）

为了在 LangGraph 中实现切换，智能体节点可以返回 [`Command`](./low_level.md#command) 对象，该对象允许您结合控制流和状态更新：

:::python

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # the condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # Specify which agent to call next
        goto=goto,
        # Update the graph state
        update={"my_state_key": "my_state_value"}
    )
```

:::

:::js

```typescript
graph.addNode((state) => {
    // the condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    const goto = getNextAgent(...); // 'agent' / 'another_agent'
    return new Command({
      // Specify which agent to call next
      goto,
      // Update the graph state
      update: { myStateKey: "myStateValue" }
    });
})
```

:::

:::python
在更复杂的场景中，每个智能体节点本身都是一个图（即 [子图](./subgraphs.md)），其中一个智能体子图中的节点可能希望导航到不同的智能体。例如，如果您有两个智能体 `alice` 和 `bob`（父图中的子图节点），并且 `alice` 需要导航到 `bob`，则可以在 `Command` 对象中设置 `graph=Command.PARENT`：

```python
def some_node_inside_alice(state):
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # specify which graph to navigate to (defaults to the current graph)
        graph=Command.PARENT,
    )
```

:::

:::js
在更复杂的场景中，每个智能体节点本身都是一个图（即 [子图](./subgraphs.md)），其中一个智能体子图中的节点可能希望导航到不同的智能体。例如，如果您有两个智能体 `alice` 和 `bob`（父图中的子图节点），并且 `alice` 需要导航到 `bob`，则可以在 `Command` 对象中设置 `graph: Command.PARENT`：

```typescript
alice.addNode((state) => {
  return new Command({
    goto: "bob",
    update: { myStateKey: "myStateValue" },
    // specify which graph to navigate to (defaults to the current graph)
    graph: Command.PARENT,
  });
});
```

:::

!!! note
    
    :::python

    如果您需要支持使用 `Command(graph=Command.PARENT)` 进行通信的子图的可视化，则需要将它们包装在带有 `Command` 注释的节点函数中：
    不是这样：

    ```python
    builder.add_node(alice)
    ```

    您需要这样做：

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

    :::

    :::js
    如果您需要支持使用 `Command({ graph: Command.PARENT })` 进行通信的子图的可视化，则需要将它们包装在带有 `Command` 注释的节点函数中：

    不是这样：

    ```typescript
    builder.addNode("alice", alice);
    ```

    您需要这样做：

    ```typescript
    builder.addNode("alice", (state) => alice.invoke(state), { ends: ["bob"] });
    ```

    :::

#### 切换作为工具 (Handoffs as tools)

最常见的智能体类型之一是 [工具调用智能体](../agents/overview.md)。对于这些类型的智能体，一种常见的模式是将切换包装在工具调用中：

:::python

```python
from langchain_core.tools import tool

@tool
def transfer_to_bob():
    """Transfer to bob."""
    return Command(
        # name of the agent (node) to go to
        goto="bob",
        # data to send to the agent
        update={"my_state_key": "my_state_value"},
        # indicate to LangGraph that we need to navigate to
        # agent node in a parent graph
        graph=Command.PARENT,
    )
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const transferToBob = tool(
  async () => {
    return new Command({
      // name of the agent (node) to go to
      goto: "bob",
      // data to send to the agent
      update: { myStateKey: "myStateValue" },
      // indicate to LangGraph that we need to navigate to
      // agent node in a parent graph
      graph: Command.PARENT,
    });
  },
  {
    name: "transfer_to_bob",
    description: "Transfer to bob.",
    schema: z.object({}),
  }
);
```

:::

这是从工具更新图状态的一种特殊情况，除了状态更新之外，还包括控制流。

!!! important

      :::python
      如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] 组件，或者自己实现逻辑：

      ```python
      def call_tools(state):
          ...
          commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
          return commands
      ```
      :::

      :::js
      如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`createReactAgent`][create_react_agent] / @[ToolNode] 组件，或者自己实现逻辑：

      ```typescript
      graph.addNode("call_tools", async (state) => {
        // ... tool execution logic
        const commands = toolCalls.map((toolCall) =>
          toolsByName[toolCall.name].invoke(toolCall)
        );
        return commands;
      });
      ```
      :::

现在让我们仔细看看不同的多智能体架构。

### 网络 (Network)

在这个架构中，智能体被定义为图节点。每个智能体都可以与所有其他智能体通信（多对多连接），并且可以决定下一步调用哪个智能体。这种架构适用于没有明确的智能体层次结构或特定的智能体调用顺序的问题。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which agent to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_agent" field)
    response = model.invoke(...)
    # route to one of the agents or exit based on the LLM's decision
    # if the LLM returns "__end__", the graph will finish execution
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    ...
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which agent to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_agent" field)
  const response = await model.invoke(...);
  // route to one of the agents or exit based on the LLM's decision
  // if the LLM returns "__end__", the graph will finish execution
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent3 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1, {
    ends: ["agent2", "agent3", END]
  })
  .addNode("agent2", agent2, {
    ends: ["agent1", "agent3", END]
  })
  .addNode("agent3", agent3, {
    ends: ["agent1", "agent2", END]
  })
  .addEdge(START, "agent1");

const network = builder.compile();
```

:::

### 主管 (Supervisor)

在这个架构中，我们将智能体定义为节点，并添加一个主管节点 (LLM)，该节点决定下一步应该调用哪个智能体节点。我们使用 [`Command`](./low_level.md#command) 根据主管的决定将执行路由到适当的智能体节点。这种架构也非常适合并行运行多个智能体或使用 [map-reduce](../how-tos/graph-api.md#map-reduce-and-the-send-api) 模式。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which agent to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_agent" field)
    response = model.invoke(...)
    # route to one of the agents or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")

supervisor = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which agent to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_agent" field)
  const response = await model.invoke(...);
  // route to one of the agents or exit based on the supervisor's decision
  // if the supervisor returns "__end__", the graph will finish execution
  return new Command({ goto: response.nextAgent });
};

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // and add any additional logic (different models, custom prompts, structured output, etc.)
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("supervisor", supervisor, {
    ends: ["agent1", "agent2", END]
  })
  .addNode("agent1", agent1, {
    ends: ["supervisor"]
  })
  .addNode("agent2", agent2, {
    ends: ["supervisor"]
  })
  .addEdge(START, "supervisor");

const supervisorGraph = builder.compile();
```

:::

有关主管多智能体架构的示例，请查看此 [教程](../tutorials/multi_agent/agent_supervisor.md)。

### 主管（工具调用） (Supervisor (tool-calling))

在 [主管](#supervisor) 架构的这种变体中，我们定义了一个负责调用子智能体的主管 [智能体](./agentic_concepts.md#agent-architectures)。子智能体作为工具暴露给主管，主管智能体决定下一步调用哪个工具。主管智能体遵循 [标准实现](./agentic_concepts.md#tool-calling-agent)，作为一个 LLM 在 while 循环中运行，调用工具直到它决定停止。

:::python

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# this is the agent function that will be called as tool
# notice that you can pass the state to the tool via InjectedState annotation
def agent_1(state: Annotated[dict, InjectedState]):
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    # return the LLM response as a string (expected tool response format)
    # this will be automatically turned to ToolMessage
    # by the prebuilt create_react_agent (supervisor)
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# the simplest way to build a supervisor w/ tool-calling is to use prebuilt ReAct agent graph
# that consists of a tool-calling LLM node (i.e. supervisor) and a tool-executing node
supervisor = create_react_agent(model, tools)
```

:::

:::js

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const model = new ChatOpenAI();

// this is the agent function that will be called as tool
// notice that you can pass the state to the tool via config parameter
const agent1 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    // you can pass relevant parts of the state to the LLM (e.g., state.messages)
    // and add any additional logic (different models, custom prompts, structured output, etc.)
    const response = await model.invoke(...);
    // return the LLM response as a string (expected tool response format)
    // this will be automatically turned to ToolMessage
    // by the prebuilt createReactAgent (supervisor)
    return response.content;
  },
  {
    name: "agent1",
    description: "Agent 1 description",
    schema: z.object({}),
  }
);

const agent2 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    const response = await model.invoke(...);
    return response.content;
  },
  {
    name: "agent2",
    description: "Agent 2 description",
    schema: z.object({}),
  }
);

const tools = [agent1, agent2];
// the simplest way to build a supervisor w/ tool-calling is to use prebuilt ReAct agent graph
// that consists of a tool-calling LLM node (i.e. supervisor) and a tool-executing node
const supervisor = createReactAgent({ llm: model, tools });
```

:::

### 分层 (Hierarchical)

随着您向系统中添加更多智能体，主管可能变得难以管理所有这些智能体。主管可能会开始对下一步调用哪个智能体做出糟糕的决定，或者上下文可能过于复杂，单个主管无法跟踪。换句话说，您最终会遇到最初促使多智能体架构产生相同的问题。

为了解决这个问题，您可以 _分层_ 设计您的系统。例如，您可以创建由单独主管管理的独立专业智能体团队，以及一个管理这些团队的顶级主管。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# define team 1 (same as the single supervisor example above)

def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(Team1State)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# define team 2 (same as the single supervisor example above)
class Team2State(MessagesState):
    next: Literal["team_2_agent_1", "team_2_agent_2", "__end__"]

def team_2_supervisor(state: Team2State):
    ...

def team_2_agent_1(state: Team2State):
    ...

def team_2_agent_2(state: Team2State):
    ...

team_2_builder = StateGraph(Team2State)
...
team_2_graph = team_2_builder.compile()


# define top-level supervisor

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which team to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_team" field)
    response = model.invoke(...)
    # route to one of the teams or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_team"])

builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

// define team 1 (same as the single supervisor example above)

const team1Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({ goto: response.nextAgent });
};

const team1Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Builder = new StateGraph(MessagesZodState)
  .addNode("team1Supervisor", team1Supervisor, {
    ends: ["team1Agent1", "team1Agent2", END]
  })
  .addNode("team1Agent1", team1Agent1, {
    ends: ["team1Supervisor"]
  })
  .addNode("team1Agent2", team1Agent2, {
    ends: ["team1Supervisor"]
  })
  .addEdge(START, "team1Supervisor");
const team1Graph = team1Builder.compile();

// define team 2 (same as the single supervisor example above)
const team2Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Builder = new StateGraph(MessagesZodState);
// ... build team2Graph
const team2Graph = team2Builder.compile();

// define top-level supervisor

const topLevelSupervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which team to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_team" field)
  const response = await model.invoke(...);
  // route to one of the teams or exit based on the supervisor's decision
  // if the supervisor returns "__end__", the graph will finish execution
  return new Command({ goto: response.nextTeam });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("topLevelSupervisor", topLevelSupervisor, {
    ends: ["team1Graph", "team2Graph", END]
  })
  .addNode("team1Graph", team1Graph)
  .addNode("team2Graph", team2Graph)
  .addEdge(START, "topLevelSupervisor")
  .addEdge("team1Graph", "topLevelSupervisor")
  .addEdge("team2Graph", "topLevelSupervisor");

const graph = builder.compile();
```

:::

### 自定义多智能体工作流 (Custom multi-agent workflow)

在这个架构中，我们将单个智能体添加为图节点，并提前在自定义工作流中定义智能体被调用的顺序。在 LangGraph 中，工作流可以通过两种方式定义：

- **显式控制流 (普通边)**：LangGraph 允许您通过 [普通图边](./low_level.md#normal-edges) 显式定义应用程序的控制流（即智能体如何通信的顺序）。这是上述架构中最确定性的变体——我们总是提前知道下一步将调用哪个智能体。

- **动态控制流 (Command)**：在 LangGraph 中，您可以允许 LLM 决定部分应用程序控制流。这可以通过使用 [`Command`](./low_level.md#command) 来实现。这种方法的一个特例是 [主管工具调用](#supervisor-tool-calling) 架构。在这种情况下，为主管智能体提供支持的工具调用 LLM 将决定调用工具（智能体）的顺序。

:::python

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# define the flow explicitly
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1)
  .addNode("agent2", agent2)
  // define the flow explicitly
  .addEdge(START, "agent1")
  .addEdge("agent1", "agent2");
```

:::

## 通信和状态管理 (Communication and state management)

构建多智能体系统时最重要的事情是弄清楚智能体如何通信。

智能体之间最常见、通用的通信方式是通过消息列表。这引出了以下问题：

- 智能体通过 [**切换通信还是通过工具调用通信**](#handoffs-vs-tool-calls)？
- 什么消息 [**从一个智能体传递到下一个智能体**](#message-passing-between-agents)？
- [**切换如何在消息列表中表示**](#representing-handoffs-in-message-history)？
- 您如何 [**管理子智能体的状态**](#state-management-for-subagents)？

此外，如果您正在处理更复杂的智能体，或者希望将单个智能体状态与多智能体系统状态分开，您可能需要使用 [**不同的状态模式**](#using-different-state-schemas)。

### 切换 vs 工具调用 (Handoffs vs tool calls)

在智能体之间传递的“有效负载”是什么？在上面讨论的大多数架构中，智能体通过 [切换](#handoffs) 进行通信，并将 [图状态](./low_level.md#state) 作为切换有效负载的一部分进行传递。具体来说，智能体将消息列表作为图状态的一部分进行传递。在 [具有工具调用的主管](#supervisor-tool-calling) 的情况下，有效负载是工具调用参数。

![](./img/multi_agent/request.png)

### 智能体之间的消息传递 (Message passing between agents)

智能体之间最常见的通信方式是通过共享状态通道，通常是消息列表。这假设状态中至少有一个通道（键）由智能体共享（例如 `messages`）。当通过共享消息列表进行通信时，还有一个额外的考虑因素：智能体应该 [分享他们思维过程的完整历史](#sharing-full-thought-process)，还是仅分享 [最终结果](#sharing-only-final-results)？

![](./img/multi_agent/response.png)

#### 分享完整的思维过程 (Sharing full thought process)

智能体可以与所有其他智能体 **分享他们思维过程的完整历史**（即 “便签本”）。这个“便签本”通常看起来像一个 [消息列表](./low_level.md#why-use-messages)。分享完整思维过程的好处是，它可能有助于其他智能体做出更好的决策，并提高整个系统的推理能力。缺点是，随着智能体数量及其复杂性的增加，“便签本”将迅速增长，并且可能需要额外的 [内存管理](../how-tos/memory/add-memory.md) 策略。

#### 仅分享最终结果 (Sharing only final results)

智能体可以拥有自己的私有“便签本”，并且只与其余智能体 **分享最终结果**。这种方法可能更适合拥有许多智能体或更复杂智能体的系统。在这种情况下，您需要定义具有 [不同状态模式](#using-different-state-schemas) 的智能体。

对于作为工具调用的智能体，主管根据工具模式确定输入。此外，LangGraph 允许在运行时 [将状态传递](../how-tos/tool-calling.md#short-term-memory) 给各个工具，以便下级智能体可以访问父级状态（如果需要）。

#### 在消息中指示智能体名称 (Indicating agent name in messages)

指示特定 AI 消息来自哪个智能体可能会有所帮助，特别是对于长消息历史记录。一些 LLM 提供商（如 OpenAI）支持向消息添加 `name` 参数——您可以使用它将智能体名称附加到消息中。如果不支持，您可以考虑手动将智能体名称注入消息内容中，例如 `<agent>alice</agent><message>message from alice</message>`。

### 在消息历史记录中表示切换 (Representing handoffs in message history)

:::python
切换通常是通过 LLM 调用专门的 [切换工具](#handoffs-as-tools) 完成的。这表示为一个带有工具调用的 [AI 消息](https://python.langchain.com/docs/concepts/messages/#aimessage)，该消息被传递给下一个智能体 (LLM)。大多数 LLM 提供商不支持在 **没有** 相应工具消息的情况下接收带有工具调用的 AI 消息。
:::

:::js
切换通常是通过 LLM 调用专门的 [切换工具](#handoffs-as-tools) 完成的。这表示为一个带有工具调用的 [AI 消息](https://js.langchain.com/docs/concepts/messages/#aimessage)，该消息被传递给下一个智能体 (LLM)。大多数 LLM 提供商不支持在 **没有** 相应工具消息的情况下接收带有工具调用的 AI 消息。
:::

因此，您有两个选择：

:::python

1. 向消息列表添加额外的 [工具消息](https://python.langchain.com/docs/concepts/messages/#toolmessage)，例如，“Successfully transferred to agent X”
2. 删除带有工具调用的 AI 消息
   :::

:::js

1. 向消息列表添加额外的 [工具消息](https://js.langchain.com/docs/concepts/messages/#toolmessage)，例如，“Successfully transferred to agent X”
2. 删除带有工具调用的 AI 消息
:::

实际上，我们看到大多数开发人员选择选项 (1)。

### 子智能体的状态管理 (State management for subagents)

一种常见的做法是让多个智能体在共享消息列表上进行通信，但仅 [将其最终消息添加到列表中](#sharing-only-final-results)。这意味着任何中间消息（例如工具调用）都不会保存在此列表中。

如果您 **确实** 想要保存这些消息，以便在将来调用特定子智能体时可以将它们传回，该怎么办？

有两种高级方法可以实现这一点：

:::python

1. 将这些消息存储在共享消息列表中，但在将列表传递给子智能体 LLM 之前对其进行过滤。例如，您可以选择过滤掉来自 **其他** 智能体的所有工具调用。
2. 在子智能体的图状态中为每个智能体存储一个单独的消息列表（例如 `alice_messages`）。这将是他们对消息历史记录“视图”的看法。
:::

:::js

1. 将这些消息存储在共享消息列表中，但在将列表传递给子智能体 LLM 之前对其进行过滤。例如，您可以选择过滤掉来自 **其他** 智能体的所有工具调用。
2. 在子智能体的图状态中为每个智能体存储一个单独的消息列表（例如 `aliceMessages`）。这将是他们对消息历史记录“视图”的看法。
:::

### 使用不同的状态模式 (Using different state schemas)

智能体可能需要具有与其余智能体不同的状态模式。例如，搜索智能体可能只需要跟踪查询和检索到的文档。在 LangGraph 中有两种方法可以实现这一点：

- 定义具有单独状态模式的 [子图](./subgraphs.md) 智能体。如果子图和父图之间没有共享状态键（通道），则必须 [添加输入/输出转换](../how-tos/subgraph.md#different-state-schemas)，以便父图知道如何与子图通信。
- 定义具有 [私有输入状态模式](../how-tos/graph-api.md#pass-private-state-between-nodes) 的智能体节点函数，该模式与整体图状态模式不同。这允许传递仅执行该特定智能体所需的信息。
