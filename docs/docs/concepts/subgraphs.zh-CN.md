# 子图 (Subgraphs)

子图是一个 [图](./low_level.md#graphs)，它被用作另一个图中的 [节点](./low_level.md#nodes) —— 这是应用于 LangGraph 的封装概念。子图允许您构建具有本身就是图的多个组件的复杂系统。

![Subgraph](./img/subgraph.png)

使用子图的一些原因是：

- 构建 [多代理系统](./multi_agent.md)
- 当你想在多个图中重用一组节点时
- 当你想让不同的团队独立地处理图的不同部分时，你可以将每个部分定义为一个子图，只要遵守子图接口（输入和输出模式），父图就可以在不知道子图任何细节的情况下构建

添加子图时的主要问题是父图和子图如何通信，即它们如何在图执行期间相互传递 [状态](./low_level.md#state)。有两种情况：

- 父图和子图在其状态 [模式](./low_level.md#state) 中具有 **共享的状态键**。在这种情况下，您可以 [将子图包含为父图中的一个节点](../how-tos/subgraph.ipynb#shared-state-schemas)

  :::python

  ```python
  from langgraph.graph import StateGraph, MessagesState, START

  # Subgraph

  def call_model(state: MessagesState):
      response = model.invoke(state["messages"])
      return {"messages": response}

  subgraph_builder = StateGraph(State)
  subgraph_builder.add_node(call_model)
  ...
  # highlight-next-line
  subgraph = subgraph_builder.compile()

  # Parent graph

  builder = StateGraph(State)
  # highlight-next-line
  builder.add_node("subgraph_node", subgraph)
  builder.add_edge(START, "subgraph_node")
  graph = builder.compile()
  ...
  graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
  ```

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";

  // Subgraph

  const subgraphBuilder = new StateGraph(MessagesZodState).addNode(
    "callModel",
    async (state) => {
      const response = await model.invoke(state.messages);
      return { messages: response };
    }
  );
  // ... other nodes and edges
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // Parent graph

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", subgraph)
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::

- 父图和子图具有 **不同的模式**（在其状态 [模式](./low_level.md#state) 中没有共享的状态键）。在这种情况下，您必须 [从父图中的节点内部调用子图](../how-tos/subgraph.ipynb#different-state-schemas)：当父图和子图具有不同的状态模式并且您需要在调用子图之前或之后转换状态时，这很有用

  :::python

  ```python
  from typing_extensions import TypedDict, Annotated
  from langchain_core.messages import AnyMessage
  from langgraph.graph import StateGraph, MessagesState, START
  from langgraph.graph.message import add_messages

  class SubgraphMessagesState(TypedDict):
      # highlight-next-line
      subgraph_messages: Annotated[list[AnyMessage], add_messages]

  # Subgraph

  # highlight-next-line
  def call_model(state: SubgraphMessagesState):
      response = model.invoke(state["subgraph_messages"])
      return {"subgraph_messages": response}

  subgraph_builder = StateGraph(SubgraphMessagesState)
  subgraph_builder.add_node("call_model_from_subgraph", call_model)
  subgraph_builder.add_edge(START, "call_model_from_subgraph")
  ...
  # highlight-next-line
  subgraph = subgraph_builder.compile()

  # Parent graph

  def call_subgraph(state: MessagesState):
      response = subgraph.invoke({"subgraph_messages": state["messages"]})
      return {"messages": response["subgraph_messages"]}

  builder = StateGraph(State)
  # highlight-next-line
  builder.add_node("subgraph_node", call_subgraph)
  builder.add_edge(START, "subgraph_node")
  graph = builder.compile()
  ...
  graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
  ```

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
  import { z } from "zod";

  const SubgraphState = z.object({
    // highlight-next-line
    subgraphMessages: MessagesZodState.shape.messages,
  });

  // Subgraph

  const subgraphBuilder = new StateGraph(SubgraphState)
    // highlight-next-line
    .addNode("callModelFromSubgraph", async (state) => {
      const response = await model.invoke(state.subgraphMessages);
      return { subgraphMessages: response };
    })
    .addEdge(START, "callModelFromSubgraph");
  // ...
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // Parent graph

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", async (state) => {
      const response = await subgraph.invoke({
        subgraphMessages: state.messages,
      });
      return { messages: response.subgraphMessages };
    })
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::
