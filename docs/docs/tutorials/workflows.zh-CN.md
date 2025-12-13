---
search:
  boost: 2
---

# 工作流和智能体 (Workflows and Agents)

本指南回顾了智能体系统的常见模式。在描述这些系统时，区分“工作流”和“智能体”是很有用的。Anthropic 的 `Building Effective Agents` 博客文章中很好地解释了这种差异：

> 工作流是 LLM 和工具通过预定义代码路径进行编排的系统。
> 另一方面，智能体是 LLM 动态指导其自己的流程和工具使用的系统，保持对它们如何完成任务的控制。

这是一个可视化这些差异的简单方法：

![Agent Workflow](../concepts/img/agent_workflow.png)

在构建智能体和工作流时，LangGraph 提供了许多好处，包括持久性、流式传输以及对调试和部署的支持。

## 设置

:::python
您可以使用 [任何支持结构化输出和工具调用的聊天模型](https://python.langchain.com/docs/integrations/chat/)。下面，我们展示了安装包、设置 API 密钥以及测试 Anthropic 的结构化输出/工具调用的过程。

??? "Install dependencies"

    ```bash
    pip install langchain_core langchain-anthropic langgraph
    ```

初始化 LLM

```python
import os
import getpass

from langchain_anthropic import ChatAnthropic

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")

llm = ChatAnthropic(model="claude-3-5-sonnet-latest")
```

:::

:::js
您可以使用 [任何支持结构化输出和工具调用的聊天模型](https://js.langchain.com/docs/integrations/chat/)。下面，我们展示了安装包、设置 API 密钥以及测试 Anthropic 的结构化输出/工具调用的过程。

??? "Install dependencies"

    ```bash
    npm install @langchain/core @langchain/anthropic @langchain/langgraph
    ```

初始化 LLM

```typescript
import { ChatAnthropic } from "@langchain/anthropic";

process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";

const llm = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
```

:::

## 构建块：增强的 LLM

LLM 具有支持构建工作流和智能体的增强功能。这些包括结构化输出和工具调用，如 Anthropic 博客文章 `Building Effective Agents` 中的这张图所示：

![augmented_llm.png](./workflows/img/augmented_llm.png)

:::python

```python
# Schema for structured output
from pydantic import BaseModel, Field

class SearchQuery(BaseModel):
    search_query: str = Field(None, description="Query that is optimized web search.")
    justification: str = Field(
        None, description="Why this query is relevant to the user's request."
    )


# Augment the LLM with schema for structured output
structured_llm = llm.with_structured_output(SearchQuery)

# Invoke the augmented LLM
output = structured_llm.invoke("How does Calcium CT score relate to high cholesterol?")

# Define a tool
def multiply(a: int, b: int) -> int:
    return a * b

# Augment the LLM with tools
llm_with_tools = llm.bind_tools([multiply])

# Invoke the LLM with input that triggers the tool call
msg = llm_with_tools.invoke("What is 2 times 3?")

# Get the tool call
msg.tool_calls
```

:::

:::js

```typescript
import { z } from "zod";
import { tool } from "@langchain/core/tools";

// Schema for structured output
const SearchQuery = z.object({
  search_query: z.string().describe("Query that is optimized web search."),
  justification: z
    .string()
    .describe("Why this query is relevant to the user's request."),
});

// Augment the LLM with schema for structured output
const structuredLlm = llm.withStructuredOutput(SearchQuery);

// Invoke the augmented LLM
const output = await structuredLlm.invoke(
  "How does Calcium CT score relate to high cholesterol?"
);

// Define a tool
const multiply = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);

// Augment the LLM with tools
const llmWithTools = llm.bindTools([multiply]);

// Invoke the LLM with input that triggers the tool call
const msg = await llmWithTools.invoke("What is 2 times 3?");

// Get the tool call
console.log(msg.tool_calls);
```

:::

## 提示链 (Prompt chaining)

在提示链中，每个 LLM 调用处理前一个调用的输出。

正如 Anthropic 博客文章 `Building Effective Agents` 中所述：

> 提示链将任务分解为一系列步骤，其中每个 LLM 调用处理前一个调用的输出。您可以在任何中间步骤添加编程检查（参见下图中的“gate”），以确保流程仍在轨道上。

> 何时使用此工作流：此工作流非常适合任务可以轻松清晰地分解为固定子任务的情况。主要目标是通过使每个 LLM 调用成为更简单的任务，用延迟换取更高的准确性。

![prompt_chain.png](./workflows/img/prompt_chain.png)

=== "Graph API"

    :::python
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from IPython.display import Image, display


    # Graph state
    class State(TypedDict):
        topic: str
        joke: str
        improved_joke: str
        final_joke: str


    # Nodes
    def generate_joke(state: State):
        """First LLM call to generate initial joke"""

        msg = llm.invoke(f"Write a short joke about {state['topic']}")
        return {"joke": msg.content}


    def check_punchline(state: State):
        """Gate function to check if the joke has a punchline"""

        # Simple check - does the joke contain "?" or "!"
        if "?" in state["joke"] or "!" in state["joke"]:
            return "Pass"
        return "Fail"


    def improve_joke(state: State):
        """Second LLM call to improve the joke"""

        msg = llm.invoke(f"Make this joke funnier by adding wordplay: {state['joke']}")
        return {"improved_joke": msg.content}


    def polish_joke(state: State):
        """Third LLM call for final polish"""

        msg = llm.invoke(f"Add a surprising twist to this joke: {state['improved_joke']}")
        return {"final_joke": msg.content}


    # Build workflow
    workflow = StateGraph(State)

    # Add nodes
    workflow.add_node("generate_joke", generate_joke)
    workflow.add_node("improve_joke", improve_joke)
    workflow.add_node("polish_joke", polish_joke)

    # Add edges to connect nodes
    workflow.add_edge(START, "generate_joke")
    workflow.add_conditional_edges(
        "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
    )
    workflow.add_edge("improve_joke", "polish_joke")
    workflow.add_edge("polish_joke", END)

    # Compile
    chain = workflow.compile()

    # Show workflow
    display(Image(chain.get_graph().draw_mermaid_png()))

    # Invoke
    state = chain.invoke({"topic": "cats"})
    print("Initial joke:")
    print(state["joke"])
    print("\n--- --- ---\n")
    if "improved_joke" in state:
        print("Improved joke:")
        print(state["improved_joke"])
        print("\n--- --- ---\n")

        print("Final joke:")
        print(state["final_joke"])
    else:
        print("Joke failed quality gate - no punchline detected!")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/a0281fca-3a71-46de-beee-791468607b75/r

    **Resources:**

    **LangChain Academy**

    See our lesson on Prompt Chaining [here](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/chain.ipynb).
    :::

    :::js
    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { z } from "zod";

    // Graph state
    const State = z.object({
      topic: z.string(),
      joke: z.string().optional(),
      improved_joke: z.string().optional(),
      final_joke: z.string().optional(),
    });

    // Nodes
    const generateJoke = async (state: z.infer<typeof State>) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a short joke about ${state.topic}`);
      return { joke: msg.content };
    };

    const checkPunchline = (state: z.infer<typeof State>) => {
      // Gate function to check if the joke has a punchline
      // Simple check - does the joke contain "?" or "!"
      if (state.joke && (state.joke.includes("?") || state.joke.includes("!"))) {
        return "Pass";
      }
      return "Fail";
    };

    const improveJoke = async (state: z.infer<typeof State>) => {
      // Second LLM call to improve the joke
      const msg = await llm.invoke(`Make this joke funnier by adding wordplay: ${state.joke}`);
      return { improved_joke: msg.content };
    };

    const polishJoke = async (state: z.infer<typeof State>) => {
      // Third LLM call for final polish
      const msg = await llm.invoke(`Add a surprising twist to this joke: ${state.improved_joke}`);
      return { final_joke: msg.content };
    };

    // Build workflow
    const workflow = new StateGraph(State)
      .addNode("generate_joke", generateJoke)
      .addNode("improve_joke", improveJoke)
      .addNode("polish_joke", polishJoke)
      .addEdge(START, "generate_joke")
      .addConditionalEdges(
        "generate_joke",
        checkPunchline,
        { "Fail": "improve_joke", "Pass": END }
      )
      .addEdge("improve_joke", "polish_joke")
      .addEdge("polish_joke", END);

    // Compile
    const chain = workflow.compile();

    // Show workflow
    import * as fs from "node:fs/promises";
    const drawableGraph = await chain.getGraphAsync();
    const image = await drawableGraph.drawMermaidPng();
    const imageBuffer = new Uint8Array(await image.arrayBuffer());
    await fs.writeFile("workflow.png", imageBuffer);

    // Invoke
    const state = await chain.invoke({ topic: "cats" });
    console.log("Initial joke:");
    console.log(state.joke);
    console.log("\n--- --- ---\n");
    if (state.improved_joke) {
      console.log("Improved joke:");
      console.log(state.improved_joke);
      console.log("\n--- --- ---\n");

      console.log("Final joke:");
      console.log(state.final_joke);
    } else {
      console.log("Joke failed quality gate - no punchline detected!");
    }
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from langgraph.func import entrypoint, task


    # Tasks
    @task
    def generate_joke(topic: str):
        """First LLM call to generate initial joke"""
        msg = llm.invoke(f"Write a short joke about {topic}")
        return msg.content


    def check_punchline(joke: str):
        """Gate function to check if the joke has a punchline"""
        # Simple check - does the joke contain "?" or "!"
        if "?" in joke or "!" in joke:
            return "Fail"

        return "Pass"


    @task
    def improve_joke(joke: str):
        """Second LLM call to improve the joke"""
        msg = llm.invoke(f"Make this joke funnier by adding wordplay: {joke}")
        return msg.content


    @task
    def polish_joke(joke: str):
        """Third LLM call for final polish"""
        msg = llm.invoke(f"Add a surprising twist to this joke: {joke}")
        return msg.content


    @entrypoint()
    def prompt_chaining_workflow(topic: str):
        original_joke = generate_joke(topic).result()
        if check_punchline(original_joke) == "Pass":
            return original_joke

        improved_joke = improve_joke(original_joke).result()
        return polish_joke(improved_joke).result()

    # Invoke
    for step in prompt_chaining_workflow.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/332fa4fc-b6ca-416e-baa3-161625e69163/r
    :::

    :::js
    ```typescript
    import { entrypoint, task } from "@langchain/langgraph";

    // Tasks
    const generateJoke = task("generate_joke", async (topic: string) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a short joke about ${topic}`);
      return msg.content;
    });

    const checkPunchline = (joke: string) => {
      // Gate function to check if the joke has a punchline
      // Simple check - does the joke contain "?" or "!"
      if (joke.includes("?") || joke.includes("!")) {
        return "Pass";
      }
      return "Fail";
    };

    const improveJoke = task("improve_joke", async (joke: string) => {
      // Second LLM call to improve the joke
      const msg = await llm.invoke(`Make this joke funnier by adding wordplay: ${joke}`);
      return msg.content;
    });

    const polishJoke = task("polish_joke", async (joke: string) => {
      // Third LLM call for final polish
      const msg = await llm.invoke(`Add a surprising twist to this joke: ${joke}`);
      return msg.content;
    });

    const promptChainingWorkflow = entrypoint("promptChainingWorkflow", async (topic: string) => {
      const originalJoke = await generateJoke(topic);
      if (checkPunchline(originalJoke) === "Pass") {
        return originalJoke;
      }

      const improvedJoke = await improveJoke(originalJoke);
      return await polishJoke(improvedJoke);
    });

    // Invoke
    const stream = await promptChainingWorkflow.stream("cats", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## 并行化 (Parallelization)

通过并行化，LLM 同时处理任务：

> LLM 有时可以同时处理任务，并以编程方式聚合其输出。这种工作流（并行化）表现为两个关键变体：分段 (Sectioning)：将任务分解为并行运行的独立子任务。投票 (Voting)：多次运行同一任务以获得不同的输出。

> 何时使用此工作流：当划分的子任务可以并行化以提高速度时，或者当需要多个视角或尝试以获得更高置信度的结果时，并行化是有效的。对于具有多个考虑因素的复杂任务，当每个考虑因素由单独的 LLM 调用处理时，LLM 通常表现更好，从而可以专注于每个特定方面。

![parallelization.png](./workflows/img/parallelization.png)

=== "Graph API"

    :::python
    ```python
    # Graph state
    class State(TypedDict):
        topic: str
        joke: str
        story: str
        poem: str
        combined_output: str


    # Nodes
    def call_llm_1(state: State):
        """First LLM call to generate initial joke"""

        msg = llm.invoke(f"Write a joke about {state['topic']}")
        return {"joke": msg.content}


    def call_llm_2(state: State):
        """Second LLM call to generate story"""

        msg = llm.invoke(f"Write a story about {state['topic']}")
        return {"story": msg.content}


    def call_llm_3(state: State):
        """Third LLM call to generate poem"""

        msg = llm.invoke(f"Write a poem about {state['topic']}")
        return {"poem": msg.content}


    def aggregator(state: State):
        """Combine the joke and story into a single output"""

        combined = f"Here's a story, joke, and poem about {state['topic']}!\n\n"
        combined += f"STORY:\n{state['story']}\n\n"
        combined += f"JOKE:\n{state['joke']}\n\n"
        combined += f"POEM:\n{state['poem']}"
        return {"combined_output": combined}


    # Build workflow
    parallel_builder = StateGraph(State)

    # Add nodes
    parallel_builder.add_node("call_llm_1", call_llm_1)
    parallel_builder.add_node("call_llm_2", call_llm_2)
    parallel_builder.add_node("call_llm_3", call_llm_3)
    parallel_builder.add_node("aggregator", aggregator)

    # Add edges to connect nodes
    parallel_builder.add_edge(START, "call_llm_1")
    parallel_builder.add_edge(START, "call_llm_2")
    parallel_builder.add_edge(START, "call_llm_3")
    parallel_builder.add_edge("call_llm_1", "aggregator")
    parallel_builder.add_edge("call_llm_2", "aggregator")
    parallel_builder.add_edge("call_llm_3", "aggregator")
    parallel_builder.add_edge("aggregator", END)
    parallel_workflow = parallel_builder.compile()

    # Show workflow
    display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

    # Invoke
    state = parallel_workflow.invoke({"topic": "cats"})
    print(state["combined_output"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/3be2e53c-ca94-40dd-934f-82ff87fac277/r

    **Resources:**

    **Documentation**

    See our documentation on parallelization [here](https://langchain-ai.github.io/langgraph/how-tos/branching/).

    **LangChain Academy**

    See our lesson on parallelization [here](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/simple-graph.ipynb).
    :::

    :::js
    ```typescript
    // Graph state
    const State = z.object({
      topic: z.string(),
      joke: z.string().optional(),
      story: z.string().optional(),
      poem: z.string().optional(),
      combined_output: z.string().optional(),
    });

    // Nodes
    const callLlm1 = async (state: z.infer<typeof State>) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a joke about ${state.topic}`);
      return { joke: msg.content };
    };

    const callLlm2 = async (state: z.infer<typeof State>) => {
      // Second LLM call to generate story
      const msg = await llm.invoke(`Write a story about ${state.topic}`);
      return { story: msg.content };
    };

    const callLlm3 = async (state: z.infer<typeof State>) => {
      // Third LLM call to generate poem
      const msg = await llm.invoke(`Write a poem about ${state.topic}`);
      return { poem: msg.content };
    };

    const aggregator = (state: z.infer<typeof State>) => {
      // Combine the joke and story into a single output
      let combined = `Here's a story, joke, and poem about ${state.topic}!\n\n`;
      combined += `STORY:\n${state.story}\n\n`;
      combined += `JOKE:\n${state.joke}\n\n`;
      combined += `POEM:\n${state.poem}`;
      return { combined_output: combined };
    };

    // Build workflow
    const parallelBuilder = new StateGraph(State)
      .addNode("call_llm_1", callLlm1)
      .addNode("call_llm_2", callLlm2)
      .addNode("call_llm_3", callLlm3)
      .addNode("aggregator", aggregator)
      .addEdge(START, "call_llm_1")
      .addEdge(START, "call_llm_2")
      .addEdge(START, "call_llm_3")
      .addEdge("call_llm_1", "aggregator")
      .addEdge("call_llm_2", "aggregator")
      .addEdge("call_llm_3", "aggregator")
      .addEdge("aggregator", END);

    const parallelWorkflow = parallelBuilder.compile();

    // Invoke
    const state = await parallelWorkflow.invoke({ topic: "cats" });
    console.log(state.combined_output);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    @task
    def call_llm_1(topic: str):
        """First LLM call to generate initial joke"""
        msg = llm.invoke(f"Write a joke about {topic}")
        return msg.content


    @task
    def call_llm_2(topic: str):
        """Second LLM call to generate story"""
        msg = llm.invoke(f"Write a story about {topic}")
        return msg.content


    @task
    def call_llm_3(topic):
        """Third LLM call to generate poem"""
        msg = llm.invoke(f"Write a poem about {topic}")
        return msg.content


    @task
    def aggregator(topic, joke, story, poem):
        """Combine the joke and story into a single output"""

        combined = f"Here's a story, joke, and poem about {topic}!\n\n"
        combined += f"STORY:\n{story}\n\n"
        combined += f"JOKE:\n{joke}\n\n"
        combined += f"POEM:\n{poem}"
        return combined


    # Build workflow
    @entrypoint()
    def parallel_workflow(topic: str):
        joke_fut = call_llm_1(topic)
        story_fut = call_llm_2(topic)
        poem_fut = call_llm_3(topic)
        return aggregator(
            topic, joke_fut.result(), story_fut.result(), poem_fut.result()
        ).result()

    # Invoke
    for step in parallel_workflow.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/623d033f-e814-41e9-80b1-75e6abb67801/r
    :::

    :::js
    ```typescript
    const callLlm1 = task("call_llm_1", async (topic: string) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a joke about ${topic}`);
      return msg.content;
    });

    const callLlm2 = task("call_llm_2", async (topic: string) => {
      // Second LLM call to generate story
      const msg = await llm.invoke(`Write a story about ${topic}`);
      return msg.content;
    });

    const callLlm3 = task("call_llm_3", async (topic: string) => {
      // Third LLM call to generate poem
      const msg = await llm.invoke(`Write a poem about ${topic}`);
      return msg.content;
    });

    const aggregator = task("aggregator", (topic: string, joke: string, story: string, poem: string) => {
      // Combine the joke and story into a single output
      let combined = `Here's a story, joke, and poem about ${topic}!\n\n`;
      combined += `STORY:\n${story}\n\n`;
      combined += `JOKE:\n${joke}\n\n`;
      combined += `POEM:\n${poem}`;
      return combined;
    });

    // Build workflow
    const parallelWorkflow = entrypoint("parallelWorkflow", async (topic: string) => {
      const jokeFut = callLlm1(topic);
      const storyFut = callLlm2(topic);
      const poemFut = callLlm3(topic);

      return await aggregator(
        topic,
        await jokeFut,
        await storyFut,
        await poemFut
      );
    });

    // Invoke
    const stream = await parallelWorkflow.stream("cats", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## 路由 (Routing)

路由对输入进行分类，并将其定向到专门的后续任务。此工作流允许分离关注点，并构建更专业的提示。如果不进行路由，针对一种类型的输入进行优化可能会损害其它类型输入的性能。

正如 Anthropic 博客文章 `Building Effective Agents` 中所述：

> 路由将输入分类并将其定向到专门的后续任务。此工作流允许分离关注点，并构建更专业的提示。如果不进行路由，针对一种类型的输入进行优化可能会损害其它类型输入的性能。

> 何时使用此工作流：路由非常适合复杂的任务，其中有不同的类别，最好由单独的处理方式来处理，以及分类可以由 LLM 或传统的分类模型/算法准确处理的情况。

![routing.png](./workflows/img/routing.png)

=== "Graph API"

    :::python
    ```python
    from typing import Literal

    # Graph state
    class State(TypedDict):
        topic: str
        joke: str
        story: str
        poem: str


    # Nodes
    def call_llm_1(state: State):
        """First LLM call to generate initial joke"""

        msg = llm.invoke(f"Write a joke about {state['topic']}")
        return {"joke": msg.content}


    def call_llm_2(state: State):
        """Second LLM call to generate story"""

        msg = llm.invoke(f"Write a story about {state['topic']}")
        return {"story": msg.content}


    def call_llm_3(state: State):
        """Third LLM call to generate poem"""

        msg = llm.invoke(f"Write a poem about {state['topic']}")
        return {"poem": msg.content}


    def route(state: State):
        """Route to the routing function"""
        # Route logic ...
        # This could be an LLM call or some other logic
        # For this example, we'll just check if the topic is "cats"
        if state["topic"] == "cats":
            return "joke"
        elif state["topic"] == "dogs":
            return "story"
        else:
            return "poem"


    # Build workflow
    routing_builder = StateGraph(State)

    # Add nodes
    routing_builder.add_node("call_llm_1", call_llm_1)
    routing_builder.add_node("call_llm_2", call_llm_2)
    routing_builder.add_node("call_llm_3", call_llm_3)

    # Add edges to connect nodes
    routing_builder.add_conditional_edges(
        START,
        route,
        {
            "joke": "call_llm_1",
            "story": "call_llm_2",
            "poem": "call_llm_3",
        },
    )
    routing_builder.add_edge("call_llm_1", END)
    routing_builder.add_edge("call_llm_2", END)
    routing_builder.add_edge("call_llm_3", END)
    routing_workflow = routing_builder.compile()

    # Show workflow
    display(Image(routing_workflow.get_graph().draw_mermaid_png()))

    # Invoke
    state = routing_workflow.invoke({"topic": "cats"})
    print(state["joke"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/443f721d-7208-4122-b258-a7df427a1999/r

    **Resources:**

    **Documentation**

    See our documentation on routing [here](https://langchain-ai.github.io/langgraph/how-tos/branching/).

    **LangChain Academy**

    See our lesson on routing [here](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/router.ipynb).
    :::

    :::js
    ```typescript
    // Graph state
    const State = z.object({
      topic: z.string(),
      joke: z.string().optional(),
      story: z.string().optional(),
      poem: z.string().optional(),
    });

    // Nodes
    const callLlm1 = async (state: z.infer<typeof State>) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a joke about ${state.topic}`);
      return { joke: msg.content };
    };

    const callLlm2 = async (state: z.infer<typeof State>) => {
      // Second LLM call to generate story
      const msg = await llm.invoke(`Write a story about ${state.topic}`);
      return { story: msg.content };
    };

    const callLlm3 = async (state: z.infer<typeof State>) => {
      // Third LLM call to generate poem
      const msg = await llm.invoke(`Write a poem about ${state.topic}`);
      return { poem: msg.content };
    };

    const route = (state: z.infer<typeof State>) => {
      // Route logic ...
      // This could be an LLM call or some other logic
      // For this example, we'll just check if the topic is "cats"
      if (state.topic === "cats") {
        return "call_llm_1";
      } else if (state.topic === "dogs") {
        return "call_llm_2";
      } else {
        return "call_llm_3";
      }
    };

    // Build workflow
    const routingBuilder = new StateGraph(State)
      .addNode("call_llm_1", callLlm1)
      .addNode("call_llm_2", callLlm2)
      .addNode("call_llm_3", callLlm3)
      .addConditionalEdges(
        START,
        route,
        {
          "call_llm_1": "call_llm_1",
          "call_llm_2": "call_llm_2",
          "call_llm_3": "call_llm_3",
        }
      )
      .addEdge("call_llm_1", END)
      .addEdge("call_llm_2", END)
      .addEdge("call_llm_3", END);

    const routingWorkflow = routingBuilder.compile();

    // Invoke
    const state = await routingWorkflow.invoke({ topic: "cats" });
    console.log(state.joke);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    @task
    def call_llm_1(topic: str):
        """First LLM call to generate initial joke"""
        msg = llm.invoke(f"Write a joke about {topic}")
        return msg.content


    @task
    def call_llm_2(topic: str):
        """Second LLM call to generate story"""
        msg = llm.invoke(f"Write a story about {topic}")
        return msg.content


    @task
    def call_llm_3(topic):
        """Third LLM call to generate poem"""
        msg = llm.invoke(f"Write a poem about {topic}")
        return msg.content


    @entrypoint()
    def routing_workflow(topic: str):
        if topic == "cats":
            return call_llm_1(topic).result()
        elif topic == "dogs":
            return call_llm_2(topic).result()
        else:
            return call_llm_3(topic).result()

    # Invoke
    for step in routing_workflow.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/33924fca-8926-4074-9467-9c869ea07817/r
    :::

    :::js
    ```typescript
    const callLlm1 = task("call_llm_1", async (topic: string) => {
      // First LLM call to generate initial joke
      const msg = await llm.invoke(`Write a joke about ${topic}`);
      return msg.content;
    });

    const callLlm2 = task("call_llm_2", async (topic: string) => {
      // Second LLM call to generate story
      const msg = await llm.invoke(`Write a story about ${topic}`);
      return msg.content;
    });

    const callLlm3 = task("call_llm_3", async (topic: string) => {
      // Third LLM call to generate poem
      const msg = await llm.invoke(`Write a poem about ${topic}`);
      return msg.content;
    });

    const routingWorkflow = entrypoint("routingWorkflow", async (topic: string) => {
      if (topic === "cats") {
        return await callLlm1(topic);
      } else if (topic === "dogs") {
        return await callLlm2(topic);
      } else {
        return await callLlm3(topic);
      }
    });

    // Invoke
    const stream = await routingWorkflow.stream("cats", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## 协调者-工人 (Orchestrator-Worker)

在协调者-工人工作流中，中央 LLM 将任务动态划分较小的子任务，将它们分发给工作智能体，并集成它们的结果。

正如 Anthropic 博客文章 `Building Effective Agents` 中所述：

> 在协调者-工人工作流中，中央 LLM 将任务动态分解，将其发送给工作智能体，并合成其结果。当您无法预测所需的子任务数量（例如，在您正在编码的示例中，要更改的文件数可能因任务而异）时，此工作流程非常适合。从概念上讲，这也是我们所说的“并行化”工作流的更高级版本。

> 何时使用此工作流：当子任务无法预先定义（例如，从不同的来源，编码）时，此工作流程非常适合。

![orchestrator-worker.png](./workflows/img/orchestrator-worker.png)

=== "Graph API"

    :::python
    ```python
    import operator
    from typing import Annotated

    # Schema for structured output
    class Section(BaseModel):
        name: str = Field(description="Name for this section of the report.")
        description: str = Field(
            description="Brief overview of the main topics and concepts to be covered in this section."
        )


    class Sections(BaseModel):
        sections: list[Section] = Field(description="Sections of the report.")


    # Graph state
    class State(TypedDict):
        topic: str
        sections: list[Section]
        completed_sections: Annotated[list, operator.add]
        final_report: str
        
    class WorkerState(TypedDict):
        section: Section
        completed_sections: Annotated[list, operator.add]


    # Nodes
    def orchestrator(state: State):
        """Orchestrator that finds sections"""

        # Augment the LLM with schema for structured output
        structured_llm = llm.with_structured_output(Sections)
        sections = structured_llm.invoke(f"Write a report about {state['topic']}")
        return {"sections": sections.sections}


    def llm_call(state: WorkerState):
        """Worker that writes a section"""

        section = state["section"]
        msg = llm.invoke(f"Write a section of the report about {section.name}. \n\n {section.description}")
        return {"completed_sections": [msg.content]}


    def synthesizer(state: State):
        """Synthesizes the completed sections into a final report"""

        completed_sections = state["completed_sections"]
        combined_report = "\n\n".join(completed_sections)
        return {"final_report": combined_report}


    # Sending the task to the worker
    from langgraph.constants import Send

    def assign_workers(state: State):
        """Assigns the task to the worker"""

        return [Send("llm_call", {"section": s}) for s in state["sections"]]


    # Build workflow
    orchestrator_worker_builder = StateGraph(State)

    # Add nodes
    orchestrator_worker_builder.add_node("orchestrator", orchestrator)
    orchestrator_worker_builder.add_node("llm_call", llm_call)
    orchestrator_worker_builder.add_node("synthesizer", synthesizer)

    # Add edges to connect nodes
    orchestrator_worker_builder.add_edge(START, "orchestrator")
    orchestrator_worker_builder.add_conditional_edges(
        "orchestrator", assign_workers, ["llm_call"]
    )
    orchestrator_worker_builder.add_edge("llm_call", "synthesizer")
    orchestrator_worker_builder.add_edge("synthesizer", END)
    orchestrator_worker = orchestrator_worker_builder.compile()

    # Show workflow
    display(Image(orchestrator_worker.get_graph().draw_mermaid_png()))

    # Invoke
    state = orchestrator_worker.invoke({"topic": "cats"})
    print(state["final_report"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/4b0870cb-7c6d-4171-a477-9be740e55729/r

    **Resources:**

    **Documentation**

    See our documentation on map-reduce [here](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/).
    :::

    :::js
    ```typescript
    import { Annotation, Send } from "@langchain/langgraph";

    // Schema for structured output
    const Section = z.object({
      name: z.string().describe("Name for this section of the report."),
      description: z.string().describe("Brief overview of the main topics and concepts to be covered in this section."),
    });

    const Sections = z.object({
      sections: z.array(Section).describe("Sections of the report."),
    });

    // Graph state
    const State = Annotation.Root({
      topic: Annotation<string>,
      sections: Annotation<z.infer<typeof Section>[]>,
      completed_sections: Annotation<string[]>({
        reducer: (a, b) => a.concat(b),
        default: () => [],
      }),
      final_report: Annotation<string>,
    });

    // Nodes
    const orchestrator = async (state: typeof State.State) => {
      // Orchestrator that finds sections
      // Augment the LLM with schema for structured output
      const structuredLlm = llm.withStructuredOutput(Sections);
      const sections = await structuredLlm.invoke(`Write a report about ${state.topic}`);
      return { sections: sections.sections };
    };

    const llmCall = async (state: typeof State.State & { section: z.infer<typeof Section> }) => {
      // Worker that writes a section
      const section = state.section;
      const msg = await llm.invoke(`Write a section of the report about ${section.name}. \n\n ${section.description}`);
      return { completed_sections: [msg.content] };
    };

    const synthesizer = (state: typeof State.State) => {
      // Synthesizes the completed sections into a final report
      const completedSections = state.completed_sections;
      const combinedReport = completedSections.join("\n\n");
      return { final_report: combinedReport };
    };

    const assignWorkers = (state: typeof State.State) => {
      // Assigns the task to the worker
      return state.sections.map((section) => new Send("llm_call", { section }));
    };

    // Build workflow
    const orchestratorWorkerBuilder = new StateGraph(State)
      .addNode("orchestrator", orchestrator)
      .addNode("llm_call", llmCall)
      .addNode("synthesizer", synthesizer)
      .addEdge(START, "orchestrator")
      .addConditionalEdges(
        "orchestrator",
        assignWorkers,
        ["llm_call"]
      )
      .addEdge("llm_call", "synthesizer")
      .addEdge("synthesizer", END);

    const orchestratorWorker = orchestratorWorkerBuilder.compile();

    // Invoke
    const state = await orchestratorWorker.invoke({ topic: "cats" });
    console.log(state.final_report);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    # Schema for structured output
    class Section(BaseModel):
        name: str = Field(description="Name for this section of the report.")
        description: str = Field(
            description="Brief overview of the main topics and concepts to be covered in this section.",
        )


    class Sections(BaseModel):
        sections: list[Section] = Field(description="Sections of the report.")


    @task
    def orchestrator(topic: str):
        """Orchestrator that finds sections"""
        # Augment the LLM with schema for structured output
        structured_llm = llm.with_structured_output(Sections)
        sections = structured_llm.invoke(f"Write a report about {topic}")
        return sections


    @task
    def llm_call(section: Section):
        """Worker that writes a section"""
        msg = llm.invoke(
            f"Write a section of the report about {section.name}. \n\n {section.description}"
        )
        return msg.content


    @task
    def synthesizer(completed_sections: list[str]):
        """Synthesizes the completed sections into a final report"""
        combined_report = "\n\n".join(completed_sections)
        return combined_report


    @entrypoint()
    def orchestrator_worker(topic: str):
        sections = orchestrator(topic).result()
        section_futures = [llm_call(s) for s in sections.sections]
        completed_sections = [f.result() for f in section_futures]
        return synthesizer(completed_sections).result()


    # Invoke
    for step in orchestrator_worker.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/25696d3f-5858-45be-a4fa-15349e5d7967/r
    :::

    :::js
    ```typescript
    // Schema for structured output
    const Section = z.object({
      name: z.string().describe("Name for this section of the report."),
      description: z.string().describe("Brief overview of the main topics and concepts to be covered in this section."),
    });

    const Sections = z.object({
      sections: z.array(Section).describe("Sections of the report."),
    });

    const orchestrator = task("orchestrator", async (topic: string) => {
      // Orchestrator that finds sections
      // Augment the LLM with schema for structured output
      const structuredLlm = llm.withStructuredOutput(Sections);
      return await structuredLlm.invoke(`Write a report about ${topic}`);
    });

    const llmCall = task("llm_call", async (section: z.infer<typeof Section>) => {
      // Worker that writes a section
      const msg = await llm.invoke(`Write a section of the report about ${section.name}. \n\n ${section.description}`);
      return msg.content;
    });

    const synthesizer = task("synthesizer", (completedSections: string[]) => {
      // Synthesizes the completed sections into a final report
      return completedSections.join("\n\n");
    });

    const orchestratorWorker = entrypoint("orchestratorWorker", async (topic: string) => {
      const sections = await orchestrator(topic);
      const sectionFutures = sections.sections.map((section) => llmCall(section));
      const completedSections = await Promise.all(sectionFutures);
      return await synthesizer(completedSections);
    });

    // Invoke
    const stream = await orchestratorWorker.stream("cats", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::
## 评估者-优化器 (Evaluator-optimizer)

在评估者-优化器工作流中，一个 LLM 调用生成响应，而另一个用作批评者，提供反馈和评估分数。

正如 Anthropic 博客文章 `Building Effective Agents` 中所述：

> 在评估者-优化器工作流中，一个 LLM 调用生成响应，而另一个充当批评者，给予反馈和评估分数。此工作流特别适用于具有明确评估标准的情况，其迭代优化提供了可衡量的价值。两个 LLM 共同运行，直到输出达到所需的质量标准。

> 何时使用此工作流：当有明确的评估标准，并且迭代优化具有可衡量的价值时，此工作流尤其有效。适用于需要高质量输出的复杂任务，例如编码或创意写作。

![evaluator-optimizer.png](./workflows/img/evaluator-optimizer.png)

=== "Graph API"

    :::python
    ```python
    # Graph state
    class State(TypedDict):
        joke: str
        topic: str
        feedback: str
        funny_or_not: str


    # Schema for structured output to use in evaluation
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="Decide if the joke is funny or not.",
        )
        feedback: str = Field(
            description="If the joke is not funny, provide feedback on how to improve it.",
        )


    # Augment the LLM with schema for structured output
    evaluator = llm.with_structured_output(Feedback)


    # Nodes
    def llm_call_generator(state: State):
        """LLM generates a joke"""

        if state.get("feedback"):
            msg = llm.invoke(
                f"Write a joke about {state['topic']} but take into account the feedback: {state['feedback']}"
            )
        else:
            msg = llm.invoke(f"Write a joke about {state['topic']}")
        return {"joke": msg.content}


    def llm_call_evaluator(state: State):
        """LLM evaluates the joke"""

        grade = evaluator.invoke(f"Grade the joke {state['joke']}")
        return {"funny_or_not": grade.grade, "feedback": grade.feedback}


    # Conditional edge function to route back to joke generator or end
    def route_joke(state: State):
        """Route back to joke generator or end based upon feedback from the evaluator"""

        if state["funny_or_not"] == "funny":
            return "Accepted"
        elif state["funny_or_not"] == "not funny":
            return "Rejected + Feedback"


    # Build workflow
    optimizer_builder = StateGraph(State)

    # Add nodes
    optimizer_builder.add_node("llm_call_generator", llm_call_generator)
    optimizer_builder.add_node("llm_call_evaluator", llm_call_evaluator)

    # Add edges to connect nodes
    optimizer_builder.add_edge(START, "llm_call_generator")
    optimizer_builder.add_edge("llm_call_generator", "llm_call_evaluator")
    optimizer_builder.add_conditional_edges(
        "llm_call_evaluator",
        route_joke,
        {
            "Accepted": END,
            "Rejected + Feedback": "llm_call_generator",
        },
    )

    # Compile the workflow
    optimizer_workflow = optimizer_builder.compile()

    # Show workflow
    display(Image(optimizer_workflow.get_graph().draw_mermaid_png()))

    # Invoke
    state = optimizer_workflow.invoke({"topic": "Cats"})
    print(state["joke"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/86ab3e60-2000-4bff-b988-9b89a3269789/r

    **Resources:**

    **Examples**

    [Here](https://github.com/langchain-ai/local-deep-researcher) is an assistant that uses evaluator-optimizer to improve a report. See our video [here](https://www.youtube.com/watch?v=XGuTzHoqlj8).

    [Here](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag_local/) is a RAG workflow that grades answers for hallucinations or errors. See our video [here](https://www.youtube.com/watch?v=bq1Plo2RhYI).
    :::

    :::js
    ```typescript
    // Graph state
    const State = z.object({
      joke: z.string().optional(),
      topic: z.string(),
      feedback: z.string().optional(),
      funny_or_not: z.string().optional(),
    });

    // Schema for structured output to use in evaluation
    const Feedback = z.object({
      grade: z.enum(["funny", "not funny"]).describe("Decide if the joke is funny or not."),
      feedback: z.string().describe("If the joke is not funny, provide feedback on how to improve it."),
    });

    // Augment the LLM with schema for structured output
    const evaluator = llm.withStructuredOutput(Feedback);

    // Nodes
    const llmCallGenerator = async (state: z.infer<typeof State>) => {
      // LLM generates a joke
      let msg;
      if (state.feedback) {
        msg = await llm.invoke(
          `Write a joke about ${state.topic} but take into account the feedback: ${state.feedback}`
        );
      } else {
        msg = await llm.invoke(`Write a joke about ${state.topic}`);
      }
      return { joke: msg.content };
    };

    const llmCallEvaluator = async (state: z.infer<typeof State>) => {
      // LLM evaluates the joke
      const grade = await evaluator.invoke(`Grade the joke ${state.joke}`);
      return { funny_or_not: grade.grade, feedback: grade.feedback };
    };

    // Conditional edge function to route back to joke generator or end
    const routeJoke = (state: z.infer<typeof State>) => {
      // Route back to joke generator or end based upon feedback from the evaluator
      if (state.funny_or_not === "funny") {
        return "Accepted";
      } else if (state.funny_or_not === "not funny") {
        return "Rejected + Feedback";
      }
    };

    // Build workflow
    const optimizerBuilder = new StateGraph(State)
      .addNode("llm_call_generator", llmCallGenerator)
      .addNode("llm_call_evaluator", llmCallEvaluator)
      .addEdge(START, "llm_call_generator")
      .addEdge("llm_call_generator", "llm_call_evaluator")
      .addConditionalEdges(
        "llm_call_evaluator",
        routeJoke,
        {
          "Accepted": END,
          "Rejected + Feedback": "llm_call_generator",
        }
      );

    // Compile the workflow
    const optimizerWorkflow = optimizerBuilder.compile();

    // Invoke
    const state = await optimizerWorkflow.invoke({ topic: "Cats" });
    console.log(state.joke);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    # Schema for structured output to use in evaluation
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="Decide if the joke is funny or not.",
        )
        feedback: str = Field(
            description="If the joke is not funny, provide feedback on how to improve it.",
        )


    # Augment the LLM with schema for structured output
    evaluator = llm.with_structured_output(Feedback)


    # Nodes
    @task
    def llm_call_generator(topic: str, feedback: Feedback):
        """LLM generates a joke"""
        if feedback:
            msg = llm.invoke(
                f"Write a joke about {topic} but take into account the feedback: {feedback}"
            )
        else:
            msg = llm.invoke(f"Write a joke about {topic}")
        return msg.content


    @task
    def llm_call_evaluator(joke: str):
        """LLM evaluates the joke"""
        feedback = evaluator.invoke(f"Grade the joke {joke}")
        return feedback


    @entrypoint()
    def optimizer_workflow(topic: str):
        feedback = None
        while True:
            joke = llm_call_generator(topic, feedback).result()
            feedback = llm_call_evaluator(joke).result()
            if feedback.grade == "funny":
                break

        return joke

    # Invoke
    for step in optimizer_workflow.stream("Cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/f66830be-4339-4a6b-8a93-389ce5ae27b4/r
    :::

    :::js
    ```typescript
    // Schema for structured output to use in evaluation
    const Feedback = z.object({
      grade: z.enum(["funny", "not funny"]).describe("Decide if the joke is funny or not."),
      feedback: z.string().describe("If the joke is not funny, provide feedback on how to improve it."),
    });

    // Augment the LLM with schema for structured output
    const evaluator = llm.withStructuredOutput(Feedback);

    // Nodes
    const llmCallGenerator = task("llm_call_generator", async (topic: string, feedback?: string) => {
      // LLM generates a joke
      if (feedback) {
        const msg = await llm.invoke(
          `Write a joke about ${topic} but take into account the feedback: ${feedback}`
        );
        return msg.content;
      } else {
        const msg = await llm.invoke(`Write a joke about ${topic}`);
        return msg.content;
      }
    });

    const llmCallEvaluator = task("llm_call_evaluator", async (joke: string) => {
      // LLM evaluates the joke
      const feedback = await evaluator.invoke(`Grade the joke ${joke}`);
      return feedback;
    });

    const optimizerWorkflow = entrypoint("optimizerWorkflow", async (topic: string) => {
      let feedback;
      while (true) {
        const joke = await llmCallGenerator(topic, feedback?.feedback);
        feedback = await llmCallEvaluator(joke);
        if (feedback.grade === "funny") {
          return joke;
        }
      }
    });

    // Invoke
    const stream = await optimizerWorkflow.stream("Cats", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## 智能体 (Agent)

智能体通常实现为在循环中根据环境反馈执行动作（通过工具调用）的 LLM。正如 Anthropic 博客 `Building Effective Agents` 中所述：

> 智能体可以处理复杂的任务，其实施通常很简单。它们通常只是在循环中根据环境反馈使用工具的 LLM。因此，清晰并深思熟虑地设计工具集及其文档至关重要。

> 何时使用智能体：智能体可用于难以或无法预测所需步骤数，且无法硬编码固定路径的开放式问题。LLM 可能会运行多轮，您必须对其决策有一定程度的信任。智能体的自主性使其成为在受信任环境中扩展任务的理想选择。

![agent.png](./workflows/img/agent.png)

:::python

```python
from langchain_core.tools import tool


# Define tools
@tool
def multiply(a: int, b: int) -> int:
    """Multiply a and b.

    Args:
        a: first int
        b: second int
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """Adds a and b.

    Args:
        a: first int
        b: second int
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """Divide a and b.

    Args:
        a: first int
        b: second int
    """
    return a / b


# Augment the LLM with tools
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";

// Define tools
const multiply = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "Multiply a and b.",
    schema: z.object({
      a: z.number().describe("first int"),
      b: z.number().describe("second int"),
    }),
  }
);

const add = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a + b;
  },
  {
    name: "add",
    description: "Adds a and b.",
    schema: z.object({
      a: z.number().describe("first int"),
      b: z.number().describe("second int"),
    }),
  }
);

const divide = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a / b;
  },
  {
    name: "divide",
    description: "Divide a and b.",
    schema: z.object({
      a: z.number().describe("first int"),
      b: z.number().describe("second int"),
    }),
  }
);

// Augment the LLM with tools
const tools = [add, multiply, divide];
const toolsByName = Object.fromEntries(tools.map((tool) => [tool.name, tool]));
const llmWithTools = llm.bindTools(tools);
```

:::

=== "Graph API"

    :::python
    ```python
    from langgraph.graph import MessagesState
    from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage


    # Nodes
    def llm_call(state: MessagesState):
        """LLM decides whether to call a tool or not"""

        return {
            "messages": [
                llm_with_tools.invoke(
                    [
                        SystemMessage(
                            content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                        )
                    ]
                    + state["messages"]
                )
            ]
        }


    def tool_node(state: dict):
        """Performs the tool call"""

        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}


    # Conditional edge function to route to the tool node or end based upon whether the LLM made a tool call
    def should_continue(state: MessagesState) -> Literal["Action", END]:
        """Decide if we should continue the loop or stop based upon whether the LLM made a tool call"""

        messages = state["messages"]
        last_message = messages[-1]
        # If the LLM makes a tool call, then perform an action
        if last_message.tool_calls:
            return "Action"
        # Otherwise, we stop (reply to the user)
        return END


    # Build workflow
    agent_builder = StateGraph(MessagesState)

    # Add nodes
    agent_builder.add_node("llm_call", llm_call)
    agent_builder.add_node("environment", tool_node)

    # Add edges to connect nodes
    agent_builder.add_edge(START, "llm_call")
    agent_builder.add_conditional_edges(
        "llm_call",
        should_continue,
        {
            # Name returned by should_continue : Name of next node to visit
            "Action": "environment",
            END: END,
        },
    )
    agent_builder.add_edge("environment", "llm_call")

    # Compile the agent
    agent = agent_builder.compile()

    # Show the agent
    display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

    # Invoke
    messages = [HumanMessage(content="Add 3 and 4.")]
    messages = agent.invoke({"messages": messages})
    for m in messages["messages"]:
        m.pretty_print()
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/051f0391-6761-4f8c-a53b-22231b016690/r

    **Resources:**

    **LangChain Academy**

    See our lesson on agents [here](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/agent.ipynb).

    **Examples**

    [Here](https://github.com/langchain-ai/memory-agent) is a project that uses a tool calling agent to create / store long-term memories.
    :::

    :::js
    ```typescript
    import { MessagesZodState, ToolNode } from "@langchain/langgraph/prebuilt";
    import { SystemMessage, HumanMessage, ToolMessage, isAIMessage } from "@langchain/core/messages";

    // Nodes
    const llmCall = async (state: z.infer<typeof MessagesZodState>) => {
      // LLM decides whether to call a tool or not
      const response = await llmWithTools.invoke([
        new SystemMessage(
          "You are a helpful assistant tasked with performing arithmetic on a set of inputs."
        ),
        ...state.messages,
      ]);
      return { messages: [response] };
    };

    const toolNode = new ToolNode(tools);

    // Conditional edge function to route to the tool node or end
    const shouldContinue = (state: z.infer<typeof MessagesZodState>) => {
      // Decide if we should continue the loop or stop
      const messages = state.messages;
      const lastMessage = messages[messages.length - 1];
      // If the LLM makes a tool call, then perform an action
      if (isAIMessage(lastMessage) && lastMessage.tool_calls?.length) {
        return "Action";
      }
      // Otherwise, we stop (reply to the user)
      return END;
    };

    // Build workflow
    const agentBuilder = new StateGraph(MessagesZodState)
      .addNode("llm_call", llmCall)
      .addNode("environment", toolNode)
      .addEdge(START, "llm_call")
      .addConditionalEdges(
        "llm_call",
        shouldContinue,
        {
          "Action": "environment",
          [END]: END,
        }
      )
      .addEdge("environment", "llm_call");

    // Compile the agent
    const agent = agentBuilder.compile();

    // Invoke
    const messages = [new HumanMessage("Add 3 and 4.")];
    const result = await agent.invoke({ messages });
    for (const m of result.messages) {
      console.log(`${m.getType()}: ${m.content}`);
    }
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from langgraph.graph import add_messages
    from langchain_core.messages import (
        SystemMessage,
        HumanMessage,
        BaseMessage,
        ToolCall,
    )


    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM decides whether to call a tool or not"""
        return llm_with_tools.invoke(
            [
                SystemMessage(
                    content="You are a helpful assistant tasked with performing arithmetic on a set of inputs."
                )
            ]
            + messages
        )


    @task
    def call_tool(tool_call: ToolCall):
        """Performs the tool call"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)


    @entrypoint()
    def agent(messages: list[BaseMessage]):
        llm_response = call_llm(messages).result()

        while True:
            if not llm_response.tool_calls:
                break

            # Execute tools
            tool_result_futures = [
                call_tool(tool_call) for tool_call in llm_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [llm_response, *tool_results])
            llm_response = call_llm(messages).result()

        messages = add_messages(messages, llm_response)
        return messages

    # Invoke
    messages = [HumanMessage(content="Add 3 and 4.")]
    for chunk in agent.stream(messages, stream_mode="updates"):
        print(chunk)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/42ae8bf9-3935-4504-a081-8ddbcbfc8b2e/r
    :::

    :::js
    ```typescript
    import { addMessages } from "@langchain/langgraph";
    import {
      SystemMessage,
      HumanMessage,
      BaseMessage,
      ToolCall,
    } from "@langchain/core/messages";

    const callLlm = task("call_llm", async (messages: BaseMessage[]) => {
      // LLM decides whether to call a tool or not
      return await llmWithTools.invoke([
        new SystemMessage(
          "You are a helpful assistant tasked with performing arithmetic on a set of inputs."
        ),
        ...messages,
      ]);
    });

    const callTool = task("call_tool", async (toolCall: ToolCall) => {
      // Performs the tool call
      const tool = toolsByName[toolCall.name];
      return await tool.invoke(toolCall);
    });

    const agent = entrypoint("agent", async (messages: BaseMessage[]) => {
      let currentMessages = messages;
      let llmResponse = await callLlm(currentMessages);

      while (true) {
        if (!llmResponse.tool_calls?.length) {
          break;
        }

        // Execute tools
        const toolResults = await Promise.all(
          llmResponse.tool_calls.map((toolCall) => callTool(toolCall))
        );

        // Append to message list
        currentMessages = addMessages(currentMessages, [
          llmResponse,
          ...toolResults,
        ]);

        // Call model again
        llmResponse = await callLlm(currentMessages);
      }

      return llmResponse;
    });

    // Invoke
    const messages = [new HumanMessage("Add 3 and 4.")];
    const stream = await agent.stream(messages, { streamMode: "updates" });
    for await (const chunk of stream) {
      console.log(chunk);
      console.log("\n");
    }
    ```
    :::

#### 预构建 (Pre-built)

:::python
LangGraph 还提供了一种 **预构建的方法** 来创建如上定义的智能体（使用 @[`create_react_agent`][create_react_agent] 函数）：

https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/

```python
from langgraph.prebuilt import create_react_agent

# Pass in:
# (1) the augmented LLM with tools
# (2) the tools list (which is used to create the tool node)
pre_built_agent = create_react_agent(llm, tools=tools)

# Show the agent
display(Image(pre_built_agent.get_graph().draw_mermaid_png()))

# Invoke
messages = [HumanMessage(content="Add 3 and 4.")]
messages = pre_built_agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

**LangSmith Trace**

https://smith.langchain.com/public/abab6a44-29f6-4b97-8164-af77413e494d/r
:::

:::js
LangGraph 还提供了一种 **预构建的方法** 来创建如上定义的智能体（使用 @[`createReactAgent`][create_react_agent] 函数）：

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";

# Pass in:
# (1) the augmented LLM with tools
# (2) the tools list (which is used to create the tool node)
const preBuiltAgent = createReactAgent({ llm, tools });

// Invoke
const messages = [new HumanMessage("Add 3 and 4.")];
const result = await preBuiltAgent.invoke({ messages });
for (const m of result.messages) {
  console.log(`${m.getType()}: ${m.content}`);
}
```

:::

## LangGraph 提供了什么 (What LangGraph provides)

通过在 LangGraph 中构建上述每个工作流程，我们可以获得一些好处：

### 持久性：人在回路 (Persistence: Human-in-the-Loop)

LangGraph 持久性层支持操作的中断和批准（例如，人在回路）。请参阅 [LangChain Academy 的第 3 单元](https://github.com/langchain-ai/langchain-academy/tree/main/module-3)。

### 持久性：记忆 (Persistence: Memory)

LangGraph 持久性层支持会话（短期）记忆和长期记忆。请参阅 LangChain Academy 的 [第 2 单元](https://github.com/langchain-ai/langchain-academy/tree/main/module-2) [和第 5 单元](https://github.com/langchain-ai/langchain-academy/tree/main/module-5)。

### 流式传输 (Streaming)

LangGraph 提供了多种流式传输工作流/智能体输出或中间状态的方法。请参阅 [LangChain Academy 的第 3 单元](https://github.com/langchain-ai/langchain-academy/blob/main/module-3/streaming-interruption.ipynb)。

### 部署 (Deployment)

LangGraph 提供了部署、可观察性和评估的简单入口。请参阅 LangChain Academy 的 [第 6 单元](https://github.com/langchain-ai/langchain-academy/tree/main/module-6)。
