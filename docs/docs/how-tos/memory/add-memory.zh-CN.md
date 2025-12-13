# 添加和管理记忆 (Add and manage memory)

AI 应用程序需要 [记忆 (memory)](../../concepts/memory.md) 来在多次交互中共享上下文。在 LangGraph 中，您可以添加两种类型的记忆：

- [添加短期记忆](#add-short-term-memory) 作为代理 [状态 (state)](../../concepts/low_level.md#state) 的一部分，以启用多轮对话。
- [添加长期记忆](#add-long-term-memory) 以跨会话存储特定于用户或应用程序级别的数据。

## 添加短期记忆 (Add short-term memory)

**短期** 记忆（线程级 [持久性 (persistence)](../../concepts/persistence.md)）使代理能够跟踪多轮对话。要添加短期记忆：

:::python

```python
# highlight-next-line
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

# highlight-next-line
checkpointer = InMemorySaver()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    # highlight-next-line
    {"configurable": {"thread_id": "1"}},
)
```

:::

:::js

```typescript
import { MemorySaver, StateGraph } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });

await graph.invoke(
  { messages: [{ role: "user", content: "hi! i am Bob" }] },
  { configurable: { thread_id: "1" } }
);
```

:::

### 在生产环境中使用 (Use in production)

在生产中，使用由数据库支持的检查点保存器：

:::python

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const checkpointer = PostgresSaver.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });
```

:::

??? example "示例：使用 Postgres 检查点保存器"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres 检查点保存器时，即使不需要手动设置，也请确保数据库已运行且可访问。

    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres import PostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.setup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres 检查点保存器时，即使不需要手动设置，也请确保数据库已运行且可访问。

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
    import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await checkpointer.setup();

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", async (state) => {
        const response = await model.invoke(state.messages);
        return { messages: [response] };
      })
      .addEdge(START, "call_model");

    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: "1"
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "hi! I'm bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what's my name?" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "示例：使用 [MongoDB](https://pypi.org/project/langgraph-checkpoint-mongodb/) 检查点保存器"

    ```
    pip install -U pymongo langgraph langgraph-checkpoint-mongodb
    ```

    !!! note "Setup"

        要使用 MongoDB 检查点保存器，您需要一个 MongoDB 集群。如果您还没有，请按照 [此指南](https://www.mongodb.com/docs/guides/atlas/cluster/) 创建一个集群。

    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb import MongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        with MongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        async with AsyncMongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) 检查点保存器"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        首次使用 Redis 检查点保存器时，您需要调用 `checkpointer.setup()`。


    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis import RedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        with RedisSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.asetup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### 在子图中使用 (Use in subgraphs)

如果您的图包含 [子图 (subgraphs)](../../concepts/subgraphs.md)，您只需要在编译父图时提供检查点保存器。LangGraph 将自动将检查点保存器传播到子图。

:::python

```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class State(TypedDict):
    foo: str

# Subgraph

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
# highlight-next-line
subgraph = subgraph_builder.compile()

# Parent graph

builder = StateGraph(State)
# highlight-next-line
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ foo: z.string() });

const subgraphBuilder = new StateGraph(State)
  .addNode("subgraph_node_1", (state) => {
    return { foo: state.foo + "bar" };
  })
  .addEdge(START, "subgraph_node_1");
const subgraph = subgraphBuilder.compile();

const builder = new StateGraph(State)
  .addNode("node_1", subgraph)
  .addEdge(START, "node_1");

const checkpointer = new MemorySaver();
const graph = builder.compile({ checkpointer });
```

:::

如果您希望子图拥有自己的记忆，您可以使用相应的检查点保存器选项编译它。在 [多代理 (multi-agent)](../../concepts/multi_agent.md) 系统中，如果您希望代理跟踪其内部消息历史记录，这非常有用。

:::python

```python
subgraph_builder = StateGraph(...)
# highlight-next-line
subgraph = subgraph_builder.compile(checkpointer=True)
```

:::

:::js

```typescript
const subgraphBuilder = new StateGraph(...);
// highlight-next-line
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```

:::

### 在工具中读取短期记忆 (Read short-term memory in tools) { #read-short-term }

LangGraph 允许代理在工具内部访问其短期记忆（状态）。

:::python

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState, create_react_agent

class CustomState(AgentState):
    # highlight-next-line
    user_id: str

def get_user_info(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """Look up user info."""
    # highlight-next-line
    user_id = state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    state_schema=CustomState,
)

agent.invoke({
    "messages": "look up user information",
    # highlight-next-line
    "user_id": "user_123"
})
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userId: z.string(),
});

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    return userId === "user_123" ? "User is John Smith" : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  stateSchema: CustomState,
});

await agent.invoke({
  messages: [{ role: "user", content: "look up user information" }],
  userId: "user_123",
});
```

:::

有关更多信息，请参阅 [上下文 (Context)](../../agents/context.md) 指南。

### 从工具写入短期记忆 (Write short-term memory from tools) { #write-short-term }

要在执行期间修改代理的短期记忆（状态），您可以直接从工具返回状态更新。这对于持久化中间结果或使后续工具或提示可以访问信息非常有用。

:::python

```python
from typing import Annotated
from langchain_core.tools import InjectedToolCallId
from langchain_core.runnables import RunnableConfig
from langchain_core.messages import ToolMessage
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.types import Command

class CustomState(AgentState):
    # highlight-next-line
    user_name: str

def update_user_info(
    tool_call_id: Annotated[str, InjectedToolCallId],
    config: RunnableConfig
) -> Command:
    """Look up and update user info."""
    user_id = config["configurable"].get("user_id")
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    # highlight-next-line
    return Command(update={
        # highlight-next-line
        "user_name": name,
        # update the message history
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=tool_call_id
            )
        ]
    })

def greet(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """Use this to greet the user once you found their info."""
    user_name = state["user_name"]
    return f"Hello {user_name}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[update_user_info, greet],
    # highlight-next-line
    state_schema=CustomState
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
  Command,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userName: z.string().optional(),
});

const updateUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    const name = userId === "user_123" ? "John Smith" : "Unknown user";
    return new Command({
      update: {
        userName: name,
        // update the message history
        messages: [
          {
            role: "tool",
            content: "Successfully looked up user information",
            tool_call_id: config.toolCall?.id,
          },
        ],
      },
    });
  },
  {
    name: "update_user_info",
    description: "Look up and update user info.",
    schema: z.object({}),
  }
);

const greet = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userName = config.configurable?.userName;
    return `Hello ${userName}!`;
  },
  {
    name: "greet",
    description: "Use this to greet the user once you found their info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [updateUserInfo, greet],
  stateSchema: CustomState,
});

await agent.invoke(
  { messages: [{ role: "user", content: "greet the user" }] },
  { configurable: { userId: "user_123" } }
);
```

:::

## 添加长期记忆 (Add long-term memory)

使用长期记忆跨会话存储特定于用户或应用程序的数据。

:::python

```python
# highlight-next-line
from langgraph.store.memory import InMemoryStore
from langgraph.graph import StateGraph

# highlight-next-line
store = InMemoryStore()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { InMemoryStore, StateGraph } from "@langchain/langgraph";

const store = new InMemoryStore();

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

### 在生产环境中使用 (Use in production)

在生产中，使用由数据库支持的存储：

:::python

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresStore.from_conn_string(DB_URI) as store:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const store = PostgresStore.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

??? example "示例：使用 Postgres 存储"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres 存储时，您需要调用 `store.setup()`

    === "Sync"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver
        # highlight-next-line
        from langgraph.store.postgres import PostgresStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        with (
            # highlight-next-line
            PostgresStore.from_conn_string(DB_URI) as store,
            PostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # store.setup()
            # checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        # highlight-next-line
        from langgraph.store.postgres.aio import AsyncPostgresStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        async with (
            # highlight-next-line
            AsyncPostgresStore.from_conn_string(DB_URI) as store,
            AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.setup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres 存储时，您需要调用 `store.setup()`

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START, LangGraphRunnableConfig } from "@langchain/langgraph";
    import { PostgresSaver, PostgresStore } from "@langchain/langgraph-checkpoint-postgres";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";

    const store = PostgresStore.fromConnString(DB_URI);
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await store.setup();
    // await checkpointer.setup();

    const callModel = async (
      state: z.infer<typeof MessagesZodState>,
      config: LangGraphRunnableConfig,
    ) => {
      const userId = config.configurable?.userId;
      const namespace = ["memories", userId];
      const memories = await config.store?.search(namespace, { query: state.messages.at(-1)?.content });
      const info = memories?.map(d => d.value.data).join("\n") || "";
      const systemMsg = `You are a helpful assistant talking to the user. User info: ${info}`;

      // Store new memories if the user asks the model to remember
      const lastMessage = state.messages.at(-1);
      if (lastMessage?.content?.toLowerCase().includes("remember")) {
        const memory = "User name is Bob";
        await config.store?.put(namespace, uuidv4(), { data: memory });
      }

      const response = await model.invoke([
        { role: "system", content: systemMsg },
        ...state.messages
      ]);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");

    const graph = builder.compile({
      checkpointer,
      store,
    });

    const config = {
      configurable: {
        thread_id: "1",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "Hi! Remember: my name is Bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    const config2 = {
      configurable: {
        thread_id: "2",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what is my name?" }] },
      { ...config2, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) 存储"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        首次使用 Redis 存储时，您需要调用 `store.setup()`

    === "Sync"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver
        # highlight-next-line
        from langgraph.store.redis import RedisStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        with (
            # highlight-next-line
            RedisStore.from_conn_string(DB_URI) as store,
            RedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            store.setup()
            checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver
        # highlight-next-line
        from langgraph.store.redis.aio import AsyncRedisStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        async with (
            # highlight-next-line
            AsyncRedisStore.from_conn_string(DB_URI) as store,
            AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.asetup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### 在工具中读取长期记忆 (Read long-term memory in tools) { #read-long-term }

:::python

```python title="代理可用于查找用户信息的工具"
from langchain_core.runnables import RunnableConfig
from langgraph.config import get_store
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

# highlight-next-line
store = InMemoryStore() # (1)!

# highlight-next-line
store.put(  # (2)!
    ("users",),  # (3)!
    "user_123",  # (4)!
    {
        "name": "John Smith",
        "language": "English",
    } # (5)!
)

def get_user_info(config: RunnableConfig) -> str:
    """Look up user info."""
    # Same as that provided to `create_react_agent`
    # highlight-next-line
    store = get_store() # (6)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    user_info = store.get(("users",), user_id) # (7)!
    return str(user_info.value) if user_info else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    store=store # (8)!
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user information"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产设置中，您通常会使用数据库或其他持久性存储。请查看 [存储文档](../../reference/store.md) 以获取更多选项。如果您使用的是 **LangGraph Platform** 部署，平台将为您提供生产就绪的存储。
2. 对于此示例，我们使用 `put` 方法将一些示例数据写入存储。有关更多详细信息，请参阅 @[BaseStore.put] API 参考。
3. 第一个参数是命名空间。这用于将相关数据分组。在这种情况下，我们使用 `users` 命名空间对用户数据进行分组。
4. 命名空间内的键。此示例使用用户 ID 作为键。
5. 我们要为该用户存储的数据。
6. `get_store` 函数用于访问存储。您可以在代码中的任何位置（包括工具和提示）调用它。此函数返回在创建代理时传递给代理的存储。
7. `get` 方法用于从存储中检索数据。第一个参数是命名空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关值的元数据。
8. `store` 传递给代理。这使代理能够在运行工具时访问存储。您还可以使用 `get_store` 函数从代码中的任何位置访问存储。
   :::

:::js

```typescript title="代理可用于查找用户信息的工具"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

await store.put(
  // (2)!
  ["users"], // (3)!
  "user_123", // (4)!
  {
    name: "John Smith",
    language: "English",
  } // (5)!
);

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    /**Look up user info.*/
    // Same as that provided to `createReactAgent`
    const store = config.store; // (6)!
    const userId = config.configurable?.userId;
    const userInfo = await store?.get(["users"], userId); // (7)!
    return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  store, // (8)!
});

// Run the agent
await agent.invoke(
  { messages: [{ role: "user", content: "look up user information" }] },
  { configurable: { userId: "user_123" } }
);
```

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产设置中，您通常会使用数据库或其他持久性存储。请查看 [存储文档](../../reference/store.md) 以获取更多选项。如果您使用的是 **LangGraph Platform** 部署，平台将为您提供生产就绪的存储。
2. 对于此示例，我们使用 `put` 方法将一些示例数据写入存储。有关更多详细信息，请参阅 @[BaseStore.put] API 参考。
3. 第一个参数是命名空间。这用于将相关数据分组。在这种情况下，我们使用 `users` 命名空间对用户数据进行分组。
4. 命名空间内的键。此示例使用用户 ID 作为键。
5. 我们要为该用户存储的数据。
6. 存储可通过配置访问。您可以在代码中的任何位置（包括工具和提示）调用它。此函数返回在创建代理时传递给代理的存储。
7. `get` 方法用于从存储中检索数据。第一个参数是命名空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关值的元数据。
8. `store` 传递给代理。这使代理能够在运行工具时访问存储。您还可以使用配置中的存储从代码中的任何位置访问它。
   :::

### 从工具写入长期记忆 (Write long-term memory from tools) { #write-long-term }

:::python

```python title="更新用户信息的工具示例"
from typing_extensions import TypedDict

from langgraph.config import get_store
from langchain_core.runnables import RunnableConfig
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

store = InMemoryStore() # (1)!

class UserInfo(TypedDict): # (2)!
    name: str

def save_user_info(user_info: UserInfo, config: RunnableConfig) -> str: # (3)!
    """Save user info."""
    # Same as that provided to `create_react_agent`
    # highlight-next-line
    store = get_store() # (4)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    store.put(("users",), user_id, user_info) # (5)!
    return "Successfully saved user info."

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[save_user_info],
    # highlight-next-line
    store=store
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}} # (6)!
)

# You can access the store directly to get the value
store.get(("users",), "user_123").value
```

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产设置中，您通常会使用数据库或其他持久性存储。请查看 [存储文档](../../reference/store.md) 以获取更多选项。如果您使用的是 **LangGraph Platform** 部署，平台将为您提供生产就绪的存储。
2. `UserInfo` 类是一个 `TypedDict`，它定义了用户信息的结构。LLM 将使用此结构根据模式格式化响应。
3. `save_user_info` 函数是一个允许代理更新用户信息的工具。这对于希望更新个人资料信息的聊天应用程序非常有用。
4. `get_store` 函数用于访问存储。您可以在代码中的任何位置（包括工具和提示）调用它。此函数返回在创建代理时传递给代理的存储。
5. `put` 方法用于在存储中存储数据。第一个参数是命名空间，第二个参数是键。这将把用户信息存储在存储中。
6. `user_id` 在配置中传递。这用于识别我们正在更新信息的用户。
   :::

:::js

```typescript title="更新用户信息的工具示例"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

const UserInfo = z.object({
  // (2)!
  name: z.string(),
});

const saveUserInfo = tool(
  async (
    userInfo: z.infer<typeof UserInfo>,
    config: LangGraphRunnableConfig
  ) => {
    // (3)!
    /**Save user info.*/
    // Same as that provided to `createReactAgent`
    const store = config.store; // (4)!
    const userId = config.configurable?.userId;
    await store?.put(["users"], userId, userInfo); // (5)!
    return "Successfully saved user info.";
  },
  {
    name: "save_user_info",
    description: "Save user info.",
    schema: UserInfo,
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [saveUserInfo],
  store,
});

// Run the agent
await agent.invoke(
  { messages: [{ role: "user", content: "My name is John Smith" }] },
  { configurable: { userId: "user_123" } } // (6)!
);

// You can access the store directly to get the value
const result = await store.get(["users"], "user_123");
console.log(result?.value);
```

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产设置中，您通常会使用数据库或其他持久性存储。请查看 [存储文档](../../reference/store.md) 以获取更多选项。如果您使用的是 **LangGraph Platform** 部署，平台将为您提供生产就绪的存储。
2. `UserInfo` 模式定义了用户信息的结构。LLM 将使用此结构根据模式格式化响应。
3. `saveUserInfo` 函数是一个允许代理更新用户信息的工具。这对于希望更新个人资料信息的聊天应用程序非常有用。
4. 存储可通过配置访问。您可以在代码中的任何位置（包括工具和提示）调用它。此函数返回在创建代理时传递给代理的存储。
5. `put` 方法用于在存储中存储数据。第一个参数是命名空间，第二个参数是键。这将把用户信息存储在存储中。
6. `userId` 在配置中传递。这用于识别我们正在更新信息的用户。
   :::

### 使用语义搜索 (Use semantic search)

在您的图存储中启用语义搜索，让图代理通过语义相似性在存储中搜索项目。

:::python

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# Create store with semantic search enabled
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";
import { InMemoryStore } from "@langchain/langgraph";

// Create store with semantic search enabled
const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const store = new InMemoryStore({
  index: {
    embeddings,
    dims: 1536,
  },
});

await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

const items = await store.search(["user_123", "memories"], {
  query: "I'm hungry",
  limit: 1,
});
```

:::

??? example "具有语义搜索的长期记忆"

    :::python
    ```python
    from typing import Optional

    from langchain.embeddings import init_embeddings
    from langchain.chat_models import init_chat_model
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore
    from langgraph.graph import START, MessagesState, StateGraph

    llm = init_chat_model("openai:gpt-4o-mini")

    # Create store with semantic search enabled
    embeddings = init_embeddings("openai:text-embedding-3-small")
    store = InMemoryStore(
        index={
            "embed": embeddings,
            "dims": 1536,
        }
    )

    store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
    store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

    def chat(state, *, store: BaseStore):
        # Search based on user's last message
        items = store.search(
            ("user_123", "memories"), query=state["messages"][-1].content, limit=2
        )
        memories = "\n".join(item.value["text"] for item in items)
        memories = f"## Memories of user\n{memories}" if memories else ""
        response = llm.invoke(
            [
                {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
                *state["messages"],
            ]
        )
        return {"messages": [response]}


    builder = StateGraph(MessagesState)
    builder.add_node(chat)
    builder.add_edge(START, "chat")
    graph = builder.compile(store=store)

    for message, metadata in graph.stream(
        input={"messages": [{"role": "user", "content": "I'm hungry"}]},
        stream_mode="messages",
    ):
        print(message.content, end="")
    ```
    :::

    :::js
    ```typescript
    import { OpenAIEmbeddings, ChatOpenAI } from "@langchain/openai";
    import { StateGraph, START, MessagesZodState, InMemoryStore } from "@langchain/langgraph";
    import { z } from "zod";

    const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

    // Create store with semantic search enabled
    const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
    const store = new InMemoryStore({
      index: {
        embeddings,
        dims: 1536,
      }
    });

    await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
    await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

    const chat = async (state: z.infer<typeof MessagesZodState>, config) => {
      // Search based on user's last message
      const items = await config.store.search(
        ["user_123", "memories"],
        { query: state.messages.at(-1)?.content, limit: 2 }
      );
      const memories = items.map(item => item.value.text).join("\n");
      const memoriesText = memories ? `## Memories of user\n${memories}` : "";

      const response = await llm.invoke([
        { role: "system", content: `You are a helpful assistant.\n${memoriesText}` },
        ...state.messages,
      ]);

      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("chat", chat)
      .addEdge(START, "chat");
    const graph = builder.compile({ store });

    for await (const [message, metadata] of await graph.stream(
      { messages: [{ role: "user", content: "I'm hungry" }] },
      { streamMode: "messages" }
    )) {
      if (message.content) {
        console.log(message.content);
      }
    }
    ```
    :::

有关如何在 LangGraph 记忆存储中使用语义搜索的更多信息，请参阅 [此指南](../../cloud/deployment/semantic_search.md)。

## 管理短期记忆 (Manage short-term memory)

启用 [短期记忆](#add-short-term-memory) 后，长时间的对话可能会超出 LLM 的上下文窗口。常见的解决方案有：

- [修剪消息](#trim-messages)：移除前 N 条或后 N 条消息（在调用 LLM 之前）
- [删除消息](#delete-messages)：从 LangGraph 状态中永久删除消息
- [总结消息](#summarize-messages)：总结历史记录中的早期消息并将其替换为摘要
- [管理检查点](#manage-checkpoints)：存储和检索消息历史记录
- 自定义策略（例如，消息过滤等）

这允许代理在不超出 LLM 上下文窗口的情况下跟踪对话。

### 修剪消息 (Trim messages)

大多数 LLM 都有最大支持的上下文窗口（以 token 为单位）。决定何时截断消息的一种方法是计算消息历史记录中的 token，并在其接近该限制时进行截断。如果您使用的是 LangChain，则可以使用修剪消息实用程序并指定要从列表中保留的 token 数量，以及用于处理边界的 `strategy`（例如，保留最后的 `maxTokens`）。

=== "在代理中 (In an agent)"

    :::python
    要在代理中修剪消息历史记录，请在 @[`pre_model_hook`][create_react_agent] 中使用 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langgraph.prebuilt import create_react_agent

    # This function will be called every time before the node that calls LLM
    def pre_model_hook(state):
        trimmed_messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=384,
            start_on="human",
            end_on=("human", "tool"),
        )
        # highlight-next-line
        return {"llm_input_messages": trimmed_messages}

    checkpointer = InMemorySaver()
    agent = create_react_agent(
        model,
        tools,
        # highlight-next-line
        pre_model_hook=pre_model_hook,
        checkpointer=checkpointer,
    )
    ```
    :::

    :::js
    要在代理中修剪消息历史记录，请在 `stateModifier` 中使用 [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) 函数：

    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    // This function will be called every time before the node that calls LLM
    const stateModifier = async (state) => {
      return trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 384,
        startOn: "human",
        endOn: ["human", "tool"],
      });
    };

    const checkpointer = new MemorySaver();
    const agent = createReactAgent({
      llm: model,
      tools,
      stateModifier,
      checkpointer,
    });
    ```
    :::

=== "在工作流中 (In a workflow)"

    :::python
    要修剪消息历史记录，请使用 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    ...
    ```
    :::

    :::js
    要修剪消息历史记录，请使用 [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) 函数：

    ```typescript
    import { trimMessages } from "@langchain/core/messages";

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel);
    // ...
    ```
    :::

??? example "完整示例：修剪消息"

    :::python
    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START, MessagesState

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    ```

    ```
    ================================== Ai Message ==================================

    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

    :::js
    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, MemorySaver } from "@langchain/langgraph";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20241022" });

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const checkpointer = new MemorySaver();
    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");
    const graph = builder.compile({ checkpointer });

    const config = { configurable: { thread_id: "1" } };
    await graph.invoke({ messages: [{ role: "user", content: "hi, my name is bob" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "write a short poem about cats" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "now do the same but for dogs" }] }, config);
    const finalResponse = await graph.invoke({ messages: [{ role: "user", content: "what's my name?" }] }, config);

    console.log(finalResponse.messages.at(-1)?.content);
    ```

    ```
    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

### 总结消息 (Summarize messages)

如上所示，修剪或删除消息的问题在于，您可能会因为消息队列的剔除而丢失信息。因此，一些应用程序受益于使用聊天模型总结消息历史记录的更复杂的方法。

![](../../concepts/img/memory/summary.png)

=== "在代理中 (In an agent)"

    :::python
    要在代理中总结消息历史记录，请在 @[`pre_model_hook`][create_react_agent] 中使用预构建的 [`SummarizationNode`](https://langchain-ai.github.io/langmem/reference/short_term/#langmem.short_term.SummarizationNode) 抽象：

    ```python
    from langchain_anthropic import ChatAnthropic
    from langmem.short_term import SummarizationNode, RunningSummary
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.prebuilt import create_react_agent
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.checkpoint.memory import InMemorySaver
    from typing import Any

    model = ChatAnthropic(model="claude-3-7-sonnet-latest")

    summarization_node = SummarizationNode( # (1)!
        token_counter=count_tokens_approximately,
        model=model,
        max_tokens=384,
        max_summary_tokens=128,
        output_messages_key="llm_input_messages",
    )

    class State(AgentState):
        # NOTE: we're adding this key to keep track of previous summary information
        # to make sure we're not summarizing on every LLM call
        # highlight-next-line
        context: dict[str, RunningSummary]  # (2)!


    checkpointer = InMemorySaver() # (3)!

    agent = create_react_agent(
        model=model,
        tools=tools,
        # highlight-next-line
        pre_model_hook=summarization_node, # (4)!
        # highlight-next-line
        state_schema=State, # (5)!
        checkpointer=checkpointer,
    )
    ```

    1. `InMemorySaver` 是一个将代理状态存储在内存中的检查点存储器。在生产设置中，通常会使用数据库或其他持久性存储。请查看 [检查点文档](../../reference/checkpoints.md) 以获取更多选项。如果您使用 **LangGraph Platform** 部署，平台将为您提供生产就绪的检查点存储器。
    2. `context` 键被添加到代理的状态中。该键包含总结节点的簿记信息。它用于跟踪最后的摘要信息，并确保代理不会在每次 LLM 调用时都进行总结，这可能是低效的。
    3. `checkpointer` 被传递给代理。这使代理能够在调用之间持久保存其状态。
    4. `pre_model_hook` 设置为 `SummarizationNode`。此节点将在将消息历史记录发送到 LLM 之前对其进行总结。总结节点将自动处理总结过程并使用新摘要更新代理的状态。如果您愿意，可以用自定义实现替换它。有关更多详细信息，请参阅 @[create_react_agent][create_react_agent] API 参考。
    5. `state_schema` 设置为 `State` 类，这是一个包含额外 `context` 键的自定义状态。
    :::

=== "在工作流中 (In a workflow)"

    :::python
    提示和编排逻辑可用于总结消息历史记录。例如，在 LangGraph 中，您可以扩展 [`MessagesState`](../../concepts/low_level.md#working-with-messages-in-graph-state) 以包含 `summary` 键：

    ```python
    from langgraph.graph import MessagesState
    class State(MessagesState):
        summary: str
    ```

    然后，您可以生成聊天记录的摘要，使用任何现有摘要作为下一个摘要的上下文。这个 `summarize_conversation` 节点可以在 `messages` 状态键中累积一定数量的消息后调用。

    ```python
    def summarize_conversation(state: State):

        # First, we get any existing summary
        summary = state.get("summary", "")

        # Create our summarization prompt
        if summary:

            # A summary already exists
            summary_message = (
                f"This is a summary of the conversation to date: {summary}\n\n"
                "Extend the summary by taking into account the new messages above:"
            )

        else:
            summary_message = "Create a summary of the conversation above:"

        # Add prompt to our history
        messages = state["messages"] + [HumanMessage(content=summary_message)]
        response = model.invoke(messages)

        # Delete all but the 2 most recent messages
        delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
        return {"summary": response.content, "messages": delete_messages}
    ```
    :::

    :::js
    提示和编排逻辑可用于总结消息历史记录。例如，在 LangGraph 中，您可以扩展 [`MessagesZodState`](../../concepts/low_level.md#working-with-messages-in-graph-state) 以包含 `summary` 键：

    ```typescript
    import { MessagesZodState } from "@langchain/langgraph";
    import { z } from "zod";

    const State = MessagesZodState.merge(z.object({
      summary: z.string().optional(),
    }));
    ```

    然后，您可以生成聊天记录的摘要，使用任何现有摘要作为下一个摘要的上下文。这个 `summarizeConversation` 节点可以在 `messages` 状态键中累积一定数量的消息后调用。

    ```typescript
    import { RemoveMessage, HumanMessage } from "@langchain/core/messages";

    const summarizeConversation = async (state: z.infer<typeof State>) => {
      // First, we get any existing summary
      const summary = state.summary || "";

      // Create our summarization prompt
      let summaryMessage: string;
      if (summary) {
        // A summary already exists
        summaryMessage =
          `This is a summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      // Add prompt to our history
      const messages = [
        ...state.messages,
        new HumanMessage({ content: summaryMessage })
      ];
      const response = await model.invoke(messages);

      // Delete all but the 2 most recent messages
      const deleteMessages = state.messages
        .slice(0, -2)
        .map(m => new RemoveMessage({ id: m.id }));

      return {
        summary: response.content,
        messages: deleteMessages
      };
    };
    ```
    :::

??? example "完整示例：总结消息"

    :::python
    ```python
    from typing import Any, TypedDict

    from langchain.chat_models import init_chat_model
    from langchain_core.messages import AnyMessage
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.checkpoint.memory import InMemorySaver
    # highlight-next-line
    from langmem.short_term import SummarizationNode, RunningSummary

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    class State(MessagesState):
        # highlight-next-line
        context: dict[str, RunningSummary]  # (1)!

    class LLMInputState(TypedDict):  # (2)!
        summarized_messages: list[AnyMessage]
        context: dict[str, RunningSummary]

    # highlight-next-line
    summarization_node = SummarizationNode(
        token_counter=count_tokens_approximately,
        model=summarization_model,
        max_tokens=256,
        max_tokens_before_summary=256,
        max_summary_tokens=128,
    )

    # highlight-next-line
    def call_model(state: LLMInputState):  # (3)!
        response = model.invoke(state["summarized_messages"])
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(State)
    builder.add_node(call_model)
    # highlight-next-line
    builder.add_node("summarize", summarization_node)
    builder.add_edge(START, "summarize")
    builder.add_edge("summarize", "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke the graph
    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    print("\nSummary:", final_response["context"]["running_summary"].summary)
    ```

    1. 我们将在 `context` 字段中跟踪我们的运行摘要（`SummarizationNode` 需要）。
    2. 定义仅用于过滤 `call_model` 节点输入的私有状态。
    3. 我们在这里传递一个私有输入状态，以隔离总结节点返回的消息。

    ```
    ================================== Ai Message ==================================

    From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

    Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
    ```
    :::

    :::js
    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import {
      SystemMessage,
      HumanMessage,
      RemoveMessage,
      type BaseMessage
    } from "@langchain/core/messages";
    import {
      MessagesZodState,
      StateGraph,
      START,
      END,
      MemorySaver,
    } from "@langchain/langgraph";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const memory = new MemorySaver();

    // We will add a `summary` attribute (in addition to `messages` key,
    // which MessagesZodState already has)
    const GraphState = z.object({
      messages: MessagesZodState.shape.messages,
      summary: z.string().default(""),
    });

    // We will use this model for both the conversation and the summarization
    const model = new ChatAnthropic({ model: "claude-3-haiku-20240307" });

    // Define the logic to call the model
    const callModel = async (state: z.infer<typeof GraphState>) => {
      // If a summary exists, we add this in as a system message
      const { summary } = state;
      let { messages } = state;
      if (summary) {
        const systemMessage = new SystemMessage({
          id: uuidv4(),
          content: `Summary of conversation earlier: ${summary}`,
        });
        messages = [systemMessage, ...messages];
      }
      const response = await model.invoke(messages);
      // We return an object, because this will get added to the existing state
      return { messages: [response] };
    };

    // We now define the logic for determining whether to end or summarize the conversation
    const shouldContinue = (state: z.infer<typeof GraphState>) => {
      const messages = state.messages;
      // If there are more than six messages, then we summarize the conversation
      if (messages.length > 6) {
        return "summarize_conversation";
      }
      // Otherwise we can just end
      return END;
    };

    const summarizeConversation = async (state: z.infer<typeof GraphState>) => {
      // First, we summarize the conversation
      const { summary, messages } = state;
      let summaryMessage: string;
      if (summary) {
        // If a summary already exists, we use a different system prompt
        // to summarize it than if one didn't
        summaryMessage =
          `This is summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      const allMessages = [
        ...messages,
        new HumanMessage({ id: uuidv4(), content: summaryMessage }),
      ];

      const response = await model.invoke(allMessages);

      // We now need to delete messages that we no longer want to show up
      // I will delete all but the last two messages, but you can change this
      const deleteMessages = messages
        .slice(0, -2)
        .map((m) => new RemoveMessage({ id: m.id! }));

      if (typeof response.content !== "string") {
        throw new Error("Expected a string response from the model");
      }

      return { summary: response.content, messages: deleteMessages };
    };

    // Define a new graph
    const workflow = new StateGraph(GraphState)
      // Define the conversation node and the summarize node
      .addNode("conversation", callModel)
      .addNode("summarize_conversation", summarizeConversation)
      // Set the entrypoint as conversation
      .addEdge(START, "conversation")
      // We now add a conditional edge
      .addConditionalEdges(
        // First, we define the start node. We use `conversation`.
        // This means these are the edges taken after the `conversation` node is called.
        "conversation",
        // Next, we pass in the function that will determine which node is called next.
        shouldContinue,
      )
      // We now add a normal edge from `summarize_conversation` to END.
      // This means that after `summarize_conversation` is called, we end.
      .addEdge("summarize_conversation", END);

    // Finally, we compile it!
    const app = workflow.compile({ checkpointer: memory });
    ```
    :::

### 管理检查点 (Manage checkpoints)

您可以查看和删除检查点存储器存储的信息。

#### 查看线程状态（检查点） (View thread state (checkpoint))

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # optionally provide an ID for a specific checkpoint,
            # otherwise the latest checkpoint is shown
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    graph.get_state(config)
    ```

    ```
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    )
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # optionally provide an ID for a specific checkpoint,
            # otherwise the latest checkpoint is shown
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    checkpointer.get_tuple(config)
    ```

    ```
    CheckpointTuple(
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        checkpoint={
            'v': 3,
            'ts': '2025-05-05T16:01:24.680462+00:00',
            'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
            'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        },
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        pending_writes=[]
    )
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    // optionally provide an ID for a specific checkpoint,
    // otherwise the latest checkpoint is shown
    // checkpoint_id: "1f029ca3-1f5b-6704-8004-820c16b69a5a"
  },
};
await graph.getState(config);
```

```
{
  values: { messages: [HumanMessage(...), AIMessage(...), HumanMessage(...), AIMessage(...)] },
  next: [],
  config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
  metadata: {
    source: 'loop',
    writes: { call_model: { messages: AIMessage(...) } },
    step: 4,
    parents: {},
    thread_id: '1'
  },
  createdAt: '2025-05-05T16:01:24.680462+00:00',
  parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
  tasks: [],
  interrupts: []
}
```

:::

#### 查看线程历史记录（检查点） (View the history of the thread (checkpoints))

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(graph.get_state_history(config))
    ```

    ```
    [
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            next=(),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:24.680462+00:00',
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
            next=('call_model',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863421+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=('__start__',),
            config={...},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863173+00:00',
            parent_config={...}
            tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=(),
            config={...},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.862295+00:00',
            parent_config={...}
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob")]},
            next=('call_model',),
            config={...},
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.278960+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': []},
            next=('__start__',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.277497+00:00',
            parent_config=None,
            tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
            interrupts=()
        )
    ]
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(checkpointer.list(config))
    ```

    ```
    [
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:24.680462+00:00',
                'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863421+00:00',
                'id': '1f029ca3-1790-6b0a-8003-baf965b6a38f',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8ab4155e-6b15-b885-9ce5-bed69a2c305c', 'messages', AIMessage(content='Your name is Bob.'))]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863173+00:00',
                'id': '1f029ca3-1790-616e-8002-9e021694a0cd',
                'channel_versions': {'__start__': '00000000000000000000000000000004.0.5736472536395331', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}, 'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'messages', [{'role': 'user', 'content': "what's my name?"}]), ('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'branch:to:call_model', None)]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.862295+00:00',
                'id': '1f029ca3-178d-6f54-8001-d7b180db0c89',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.278960+00:00',
                'id': '1f029ca3-0874-6612-8000-339f2abc83b1',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000002.0.30296526818059655', 'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8cbd75e0-3720-b056-04f7-71ac805140a0', 'messages', AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'))]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.277497+00:00',
                'id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565',
                'channel_versions': {'__start__': '00000000000000000000000000000001.0.7040775356287469'},
                'versions_seen': {'__input__': {}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            parent_config=None,
            pending_writes=[('d458367b-8265-812c-18e2-33001d199ce6', 'messages', [{'role': 'user', 'content': "hi! I'm bob"}]), ('d458367b-8265-812c-18e2-33001d199ce6', 'branch:to:call_model', None)]
        )
    ]
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
  },
};

const history = [];
for await (const state of graph.getStateHistory(config)) {
  history.push(state);
}
```

:::

#### 删除线程的所有检查点 (Delete all checkpoints for a thread)

:::python

```python
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

:::

:::js

```typescript
const threadId = "1";
await checkpointer.deleteThread(threadId);
```

:::

## 预构建的记忆工具 (Prebuilt memory tools)

**LangMem** 是一个由 LangChain 维护的库，提供用于管理代理中长期记忆的工具。请参阅 [LangMem 文档](https://langchain-ai.github.io/langmem/) 以获取使用示例。

