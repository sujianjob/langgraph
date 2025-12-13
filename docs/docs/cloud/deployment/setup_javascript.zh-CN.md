# 如何设置 LangGraph.js 应用程序 (How to Set Up a LangGraph.js Application)

必须使用 [LangGraph 配置文件](../reference/cli.md#configuration-file) 配置 [LangGraph.js](https://langchain-ai.github.io/langgraphjs/) 应用程序，以便将其部署到 LangGraph Platform（或进行自托管）。本操作指南讨论了使用 `package.json` 指定项目依赖项来设置 LangGraph.js 应用程序以进行部署的基本步骤。

本演练基于 [此存储库](https://github.com/langchain-ai/langgraphjs-studio-starter)，您可以试用它以了解有关如何设置 LangGraph 应用程序以进行部署的更多信息。

最终的存储库结构将如下所示：

```bash
my-app/
├── src # all project code lies within here
│   ├── utils # optional utilities for your graph
│   │   ├── tools.ts # tools for your graph
│   │   ├── nodes.ts # node functions for you graph
│   │   └── state.ts # state definition of your graph
│   └── agent.ts # code for constructing your graph
├── package.json # package dependencies
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

每一步之后，都会提供一个示例文件目录来演示如何组织代码。

## 指定依赖项 (Specify Dependencies)

可以在 `package.json` 中指定依赖项。如果未创建这些文件，则可以在 [LangGraph 配置文件](#create-langgraph-api-config) 中稍后指定依赖项。

示例 `package.json` 文件：

```json
{
  "name": "langgraphjs-studio-starter",
  "packageManager": "yarn@1.22.22",
  "dependencies": {
    "@langchain/community": "^0.2.31",
    "@langchain/core": "^0.2.31",
    "@langchain/langgraph": "^0.2.0",
    "@langchain/openai": "^0.2.8"
  }
}
```

部署应用程序时，将使用您选择的包管理器安装依赖项，前提是它们符合下面列出的兼容版本范围：

```
"@langchain/core": "^0.3.42",
"@langchain/langgraph": "^0.2.57",
"@langchain/langgraph-checkpoint": "~0.0.16",
```

示例文件目录：

```bash
my-app/
└── package.json # package dependencies
```

## 指定环境变量 (Specify Environment Variables)

可以选择在文件（例如 `.env`）中指定环境变量。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
TAVILY_API_KEY=key_2
```

示例文件目录：

```bash
my-app/
├── package.json
└── .env # environment variables
```

## 定义图 (Define Graphs)

实现您的图！图可以在单个文件或多个文件中定义。记下要包含在 LangGraph 应用程序中的每个已编译图的变量名称。这些变量名称稍后将在创建 [LangGraph 配置文件](../reference/cli.md#configuration-file) 时使用。

这是 `agent.ts` 的示例：

```ts
import type { AIMessage } from "@langchain/core/messages";
import { TavilySearchResults } from "@langchain/community/tools/tavily_search";
import { ChatOpenAI } from "@langchain/openai";

import { MessagesAnnotation, StateGraph } from "@langchain/langgraph";
import { ToolNode } from "@langchain/langgraph/prebuilt";

const tools = [new TavilySearchResults({ maxResults: 3 })];

// Define the function that calls the model
async function callModel(state: typeof MessagesAnnotation.State) {
  /**
   * Call the LLM powering our agent.
   * Feel free to customize the prompt, model, and other logic!
   */
  const model = new ChatOpenAI({
    model: "gpt-4o",
  }).bindTools(tools);

  const response = await model.invoke([
    {
      role: "system",
      content: `You are a helpful assistant. The current date is ${new Date().getTime()}.`,
    },
    ...state.messages,
  ]);

  // MessagesAnnotation supports returning a single message or array of messages
  return { messages: response };
}

// Define the function that determines whether to continue or not
function routeModelOutput(state: typeof MessagesAnnotation.State) {
  const messages = state.messages;
  const lastMessage: AIMessage = messages[messages.length - 1];
  // If the LLM is invoking tools, route there.
  if ((lastMessage?.tool_calls?.length ?? 0) > 0) {
    return "tools";
  }
  // Otherwise end the graph.
  return "__end__";
}

// Define a new graph.
// See https://langchain-ai.github.io/langgraphjs/how-tos/define-state/#getting-started for
// more on defining custom graph states.
const workflow = new StateGraph(MessagesAnnotation)
  // Define the two nodes we will cycle between
  .addNode("callModel", callModel)
  .addNode("tools", new ToolNode(tools))
  // Set the entrypoint as `callModel`
  // This means that this node is the first one called
  .addEdge("__start__", "callModel")
  .addConditionalEdges(
    // First, we define the edges' source node. We use `callModel`.
    // This means these are the edges taken after the `callModel` node is called.
    "callModel",
    // Next, we pass in the function that will determine the sink node(s), which
    // will be called after the source node is called.
    routeModelOutput,
    // List of the possible destinations the conditional edge can route to.
    // Required for conditional edges to properly render the graph in Studio
    ["tools", "__end__"]
  )
  // This means that after `tools` is called, `callModel` node is called next.
  .addEdge("tools", "callModel");

// Finally, we compile it!
// This compiles it into a graph you can invoke and deploy.
export const graph = workflow.compile();
```

示例文件目录：

```bash
my-app/
├── src # all project code lies within here
│   ├── utils # optional utilities for your graph
│   │   ├── tools.ts # tools for your graph
│   │   ├── nodes.ts # node functions for you graph
│   │   └── state.ts # state definition of your graph
│   └── agent.ts # code for constructing your graph
├── package.json # package dependencies
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

## 创建 LangGraph API 配置 (Create LangGraph API Config)

创建一个名为 `langgraph.json` 的 [LangGraph 配置文件](../reference/cli.md#configuration-file)。有关配置文件的 JSON 对象中每个键的详细说明，请参阅 [LangGraph 配置文件参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "node_version": "20",
  "dockerfile_lines": [],
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.ts:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名称出现在顶级 `graphs` 键中每个子键值的末尾（即 `:<variable_name>`）。

!!! info "Configuration Location"

    LangGraph 配置文件必须放置在包含已编译图及相关依赖项的 TypeScript 文件的同一级别或更高级别的目录中。

## 下一步 (Next)

设置项目并将其放置在 GitHub 存储库中后，就可以 [部署您的应用程序](./cloud.md) 了。
