---
title: 概述
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 使用预构建组件开发代理 (Agent)

LangGraph 提供了用于构建基于代理的应用程序的低级原语和高级预构建组件。本节重点介绍预构建的、开箱即用的组件，旨在帮助您快速可靠地构建代理系统——无需从头开始实现编排、内存或人工反馈处理。

## 什么是代理 (Agent)？

一个 **代理 (Agent)** 由三个组件组成：**大型语言模型 (LLM)**、它可以使用的**工具 (Tools)** 集合，以及提供指令的**提示 (Prompt)**。

LLM 在一个循环中运行。在每次迭代中，它选择要调用的工具，提供输入，接收结果（观察），并使用该观察来通知下一个动作。循环一直持续到满足停止条件——通常是当代理收集了足够的信息来响应用户时。

<figure markdown="1">
![image](./assets/agent.png){: style="max-height:400px"}
<figcaption>代理循环：LLM 选择工具并使用其输出来满足用户请求。</figcaption>
</figure>

## 关键特性

LangGraph 包含构建健壮的、生产级代理系统所需的几个关键功能：

- [**内存集成**](../how-tos/memory/add-memory.md)：对 _短期_（基于会话）和 _长期_（跨会话持久化）内存的原生支持，使聊天机器人和助手具有状态行为。
- [**人机协同控制 (Human-in-the-loop)**](../concepts/human_in_the_loop.md)：执行可以 _无限期_ 暂停以等待人工反馈——这与仅限于实时交互的基于 websocket 的解决方案不同。这使得可以在工作流的任何点进行异步批准、更正或干预。
- [**流式支持**](../how-tos/streaming.md)：实时流式传输代理状态、模型 token、工具输出或组合流。
- [**部署工具**](../tutorials/langgraph-platform/local-server.md)：包括免基础设施的部署工具。[**LangGraph Platform**](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) 支持测试、调试和部署。
  - **[Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)**：用于检查和调试工作流的可视化 IDE。
  - 支持多种用于生产环境的[**部署选项**](https://langchain-ai.github.io/langgraph/concepts/deployment_options.md)。

## 高级构建块

LangGraph 附带了一组实现常见代理行为和工作流的预构建组件。这些抽象构建在 LangGraph 框架之上，提供了一条通往生产的更快路径，同时为高级定制保持灵活性。

使用 LangGraph 进行代理开发使您可以专注于应用程序的逻辑和行为，而不是构建和维护状态、内存和人工反馈的支持基础设施。

:::python

## 包生态系统

高级组件被组织成几个包，每个包都有特定的关注点。

| 包 (Package) | 描述 | 安装 |
| ------------------------------------------ | ---------------------------------------------------------------------------------------- | --------------------------------------- |
| `langgraph-prebuilt` (part of `langgraph`) | 用于 [**创建代理**](./agents.md) 的预构建组件 | `pip install -U langgraph langchain` |
| `langgraph-supervisor` | 用于构建 [**主管 (supervisor)**](./multi-agent.md#supervisor) 代理的工具 | `pip install -U langgraph-supervisor` |
| `langgraph-swarm` | 用于构建 [**swarm**](./multi-agent.md#swarm) 多代理系统的工具 | `pip install -U langgraph-swarm` |
| `langchain-mcp-adapters` | 用于工具和资源集成的 [**MCP 服务器**](./mcp.md) 接口 | `pip install -U langchain-mcp-adapters` |
| `langmem` | 代理内存管理：[**短期和长期**](../how-tos/memory/add-memory.md) | `pip install -U langmem` |
| `agentevals` | 用于 [**评估代理性能**](./evals.md) 的实用程序 | `pip install -U agentevals` |

## 可视化代理图 (Agent Graph)

使用以下工具可视化由 @[`create_react_agent`][create_react_agent] 生成的图，并查看相应代码的大纲。这将允许您探索由以下各项定义及其存在的代理基础设施：

- [`tools`](../how-tos/tool-calling.md)：代理可以用来执行任务的工具列表（函数、API 或其他可调用对象）。
- [`pre_model_hook`](../how-tos/create-react-agent-manage-message-history.ipynb)：在调用模型之前调用的函数。它可用于压缩消息或执行其他预处理任务。
- `post_model_hook`：在调用模型之后调用的函数。它可用于实现护栏、人机协同流程或其他后处理任务。
- [`response_format`](../agents/agents.md#6-configure-structured-output)：用于约束最终输出类型的结构，例如 `pydantic` `BaseModel`。

<div class="agent-layout">
  <div class="agent-graph-features-container">
    <div class="agent-graph-features">
      <h3 class="agent-section-title">特性 (Features)</h3>
      <label><input type="checkbox" id="tools" checked> <code>tools</code></label>
      <label><input type="checkbox" id="pre_model_hook"> <code>pre_model_hook</code></label>
      <label><input type="checkbox" id="post_model_hook"> <code>post_model_hook</code></label>
      <label><input type="checkbox" id="response_format"> <code>response_format</code></label>
    </div>
  </div>

  <div class="agent-graph-container">
    <h3 class="agent-section-title">图 (Graph)</h3>
    <img id="agent-graph-img" src="../assets/react_agent_graphs/0001.svg" alt="graph image" style="max-width: 100%;"/>
  </div>
</div>

由于脚本和交互式内容在 Markdown 翻译中较难完全复现且可能依赖特定环境，上述可视化工具的交互部分保留原样。

下面的代码片段展示了如何使用 @[`create_react_agent`][create_react_agent] 创建上述代理（和底层图）：

<div class="language-python">
  <pre><code id="agent-code" class="language-python"></code></pre>
</div>

<script>
function getCheckedValue(id) {
  return document.getElementById(id).checked ? "1" : "0";
}

function getKey() {
  return [
    getCheckedValue("response_format"),
    getCheckedValue("post_model_hook"),
    getCheckedValue("pre_model_hook"),
    getCheckedValue("tools")
  ].join("");
}

function generateCodeSnippet({ tools, pre, post, response }) {
  const lines = [
    "from langgraph.prebuilt import create_react_agent",
    "from langchain_openai import ChatOpenAI"
  ];

  if (response) lines.push("from pydantic import BaseModel");

  lines.push("", 'model = ChatOpenAI("o4-mini")', "");

  if (tools) {
    lines.push(
      "def tool() -> None:",
      '    """Testing tool."""',
      "    ...",
      ""
    );
  }

  if (pre) {
    lines.push(
      "def pre_model_hook() -> None:",
      '    """Pre-model hook."""',
      "    ...",
      ""
    );
  }

  if (post) {
    lines.push(
      "def post_model_hook() -> None:",
      '    """Post-model hook."""',
      "    ...",
      ""
    );
  }

  if (response) {
    lines.push(
      "class ResponseFormat(BaseModel):",
      '    """Response format for the agent."""',
      "    result: str",
      ""
    );
  }

  lines.push("agent = create_react_agent(");
  lines.push("    model,");

  if (tools) lines.push("    tools=[tool],");
  if (pre) lines.push("    pre_model_hook=pre_model_hook,");
  if (post) lines.push("    post_model_hook=post_model_hook,");
  if (response) lines.push("    response_format=ResponseFormat,");

  lines.push(")", "", "# Visualize the graph", "# For Jupyter or GUI environments:", "agent.get_graph().draw_mermaid_png()", "", "# To save PNG to file:", "png_data = agent.get_graph().draw_mermaid_png()", "with open(\"graph.png\", \"wb\") as f:", "    f.write(png_data)", "", "# For terminal/ASCII output:", "agent.get_graph().draw_ascii()");

  return lines.join("\n");
}

async function render() {
  const key = getKey();
  document.getElementById("agent-graph-img").src = `../assets/react_agent_graphs/${key}.svg`;

  const state = {
    tools: document.getElementById("tools").checked,
    pre: document.getElementById("pre_model_hook").checked,
    post: document.getElementById("post_model_hook").checked,
    response: document.getElementById("response_format").checked
  };

  document.getElementById("agent-code").textContent = generateCodeSnippet(state);
}

function initializeWidget() {
  render(); // no need for `await` here
  document.querySelectorAll(".agent-graph-features input").forEach((input) => {
    input.addEventListener("change", render);
  });
}

// Init for both full reload and SPA nav (used by MkDocs Material)
window.addEventListener("DOMContentLoaded", initializeWidget);
document$.subscribe(initializeWidget);
</script>

:::

:::js

## 包生态系统

高级组件被组织成几个包，每个包都有特定的关注点。

| 包 (Package) | 描述 | 安装 |
| ------------------------ | --------------------------------------------------------------------------- | -------------------------------------------------- |
| `langgraph` | 用于 [**创建代理**](./agents.md) 的预构建组件 | `npm install @langchain/langgraph @langchain/core` |
| `langgraph-supervisor` | 用于构建 [**主管 (supervisor)**](./multi-agent.md#supervisor) 代理的工具 | `npm install @langchain/langgraph-supervisor` |
| `langgraph-swarm` | 用于构建 [**swarm**](./multi-agent.md#swarm) 多代理系统的工具 | `npm install @langchain/langgraph-swarm` |
| `langchain-mcp-adapters` | 用于工具和资源集成的 [**MCP 服务器**](./mcp.md) 接口 | `npm install @langchain/mcp-adapters` |
| `agentevals` | 用于 [**评估代理性能**](./evals.md) 的实用程序 | `npm install agentevals` |

## 可视化代理图 (Agent Graph)

使用以下工具可视化由 @[`createReactAgent`][create_react_agent] 生成的图，并查看相应代码的大纲。这将允许您探索由以下各项定义及其存在的代理基础设施：

- [`tools`](./tools.md)：代理可以用来执行任务的工具列表（函数、API 或其他可调用对象）。
- `preModelHook`：在调用模型之前调用的函数。它可用于压缩消息或执行其他预处理任务。
- `postModelHook`：在调用模型之后调用的函数。它可用于实现护栏、人机协同流程或其他后处理任务。
- [`responseFormat`](./agents.md#6-configure-structured-output)：用于约束最终输出类型的数据结构（通过 Zod schemas）。

<div class="agent-layout">
  <div class="agent-graph-features-container">
    <div class="agent-graph-features">
      <h3 class="agent-section-title">特性 (Features)</h3>
      <label><input type="checkbox" id="tools" checked> <code>tools</code></label>
      <label><input type="checkbox" id="preModelHook"> <code>preModelHook</code></label>
      <label><input type="checkbox" id="postModelHook"> <code>postModelHook</code></label>
      <label><input type="checkbox" id="responseFormat"> <code>responseFormat</code></label>
    </div>
  </div>

  <div class="agent-graph-container">
    <h3 class="agent-section-title">图 (Graph)</h3>
    <img id="agent-graph-img" src="../assets/react_agent_graphs/0001.svg" alt="graph image" style="max-width: 100%;"/>
  </div>
</div>

下面的代码片段展示了如何使用 @[`createReactAgent`][create_react_agent] 创建上述代理（和底层图）：

<div class="language-typescript">
  <pre><code id="agent-code" class="language-typescript"></code></pre>
</div>

<script>
function getCheckedValue(id) {
  return document.getElementById(id).checked ? "1" : "0";
}

function getKey() {
  return [
    getCheckedValue("responseFormat"),
    getCheckedValue("postModelHook"),
    getCheckedValue("preModelHook"),
    getCheckedValue("tools")
  ].join("");
}

function dedent(strings, ...values) {
  const str = String.raw({ raw: strings }, ...values)
  const [space] = str.split("\n").filter(Boolean).at(0).match(/^(\s*)/)
  const spaceLen = space.length
  return str.split("\n").map(line => line.slice(spaceLen)).join("\n").trim()
}

Object.assign(dedent, {
  offset: (size) => (strings, ...values) => {
    return dedent(strings, ...values).split("\n").map(line => " ".repeat(size) + line).join("\n")
  }
})




function generateCodeSnippet({ tools, pre, post, response }) {
  const lines = []

  lines.push(dedent`
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatOpenAI } from "@langchain/openai";
  `)

  if (tools) lines.push(`import { tool } from "@langchain/core/tools";`);  
  if (response || tools) lines.push(`import { z } from "zod";`);

  lines.push("", dedent`
    const agent = createReactAgent({
      llm: new ChatOpenAI({ model: "o4-mini" }),
  `)

  if (tools) {
    lines.push(dedent.offset(2)`
      tools: [
        tool(() => "Sample tool output", {
          name: "sampleTool",
          schema: z.object({}),
        }),
      ],
    `)
  }

  if (pre) {
    lines.push(dedent.offset(2)`
      preModelHook: (state) => ({ llmInputMessages: state.messages }),
    `)
  }

  if (post) {
    lines.push(dedent.offset(2)`
      postModelHook: (state) => state,
    `)
  }

  if (response) {
    lines.push(dedent.offset(2)`
      responseFormat: z.object({ result: z.string() }),
    `)
  }

  lines.push(`});`);

  return lines.join("\n");
}

function render() {
  const key = getKey();
  document.getElementById("agent-graph-img").src = `../assets/react_agent_graphs/${key}.svg`;

  const state = {
    tools: document.getElementById("tools").checked,
    pre: document.getElementById("preModelHook").checked,
    post: document.getElementById("postModelHook").checked,
    response: document.getElementById("responseFormat").checked
  };

  document.getElementById("agent-code").textContent = generateCodeSnippet(state);
}

function initializeWidget() {
  render(); // no need for `await` here
  document.querySelectorAll(".agent-graph-features input").forEach((input) => {
    input.addEventListener("change", render);
  });
}

// Init for both full reload and SPA nav (used by MkDocs Material)
window.addEventListener("DOMContentLoaded", initializeWidget);
document$.subscribe(initializeWidget);
</script>

:::
