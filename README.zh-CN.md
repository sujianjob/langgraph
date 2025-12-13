<picture class="github-only">
  <source media="(prefers-color-scheme: light)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg">
  <source media="(prefers-color-scheme: dark)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_light.svg">
  <img alt="LangGraph Logo" src="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg" width="80%">
</picture>

<div>
<br>
</div>

[![Version](https://img.shields.io/pypi/v/langgraph.svg)](https://pypi.org/project/langgraph/)
[![Downloads](https://static.pepy.tech/badge/langgraph/month)](https://pepy.tech/project/langgraph)
[![Open Issues](https://img.shields.io/github/issues-raw/langchain-ai/langgraph)](https://github.com/langchain-ai/langgraph/issues)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://docs.langchain.com/oss/python/langgraph/overview)

受到 Klarna、Replit、Elastic 等塑造智能体未来的公司信赖 —— LangGraph 是一个低级编排框架，用于构建、管理和部署长时间运行的有状态智能体。

## 快速开始

安装 LangGraph：

```
pip install -U langgraph
```

创建一个简单的工作流：

```python
from langgraph.graph import START, StateGraph
from typing_extensions import TypedDict


class State(TypedDict):
    text: str


def node_a(state: State) -> dict:
    return {"text": state["text"] + "a"}


def node_b(state: State) -> dict:
    return {"text": state["text"] + "b"}


graph = StateGraph(State)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")

print(graph.compile().invoke({"text": ""}))
# {'text': 'ab'}
```

通过 [LangGraph 快速入门](https://docs.langchain.com/oss/python/langgraph/quickstart) 开始使用。

要使用基于 LangGraph 构建的 LangChain `create_agent` 快速构建智能体，请参阅 [LangChain 智能体文档](https://docs.langchain.com/oss/python/langchain/agents)。

## 核心优势

LangGraph 为*任何*长时间运行的有状态工作流或智能体提供低级支持基础设施。LangGraph 不会抽象提示词或架构，并提供以下核心优势：

- [持久执行](https://docs.langchain.com/oss/python/langgraph/durable-execution)：构建能够从故障中恢复并可以长时间运行的智能体，自动从中断处继续执行。
- [人机协同](https://docs.langchain.com/oss/python/langgraph/interrupts)：通过在执行过程中的任何时刻检查和修改智能体状态，无缝集成人工监督。
- [全面的记忆](https://docs.langchain.com/oss/python/langgraph/memory)：创建真正有状态的智能体，既具有用于持续推理的短期工作记忆，也具有跨会话的长期持久记忆。
- [使用 LangSmith 调试](http://www.langchain.com/langsmith)：通过可视化工具深入了解复杂的智能体行为，这些工具可以跟踪执行路径、捕获状态转换并提供详细的运行时指标。
- [生产就绪的部署](https://docs.langchain.com/langsmith/app-development)：通过专为处理有状态、长时间运行工作流独特挑战而设计的可扩展基础设施，自信地部署复杂的智能体系统。

## LangGraph 生态系统

虽然 LangGraph 可以独立使用，但它也与任何 LangChain 产品无缝集成，为开发者提供构建智能体的完整工具套件。为了改进您的 LLM 应用开发，将 LangGraph 与以下工具配对使用：

- [LangSmith](http://www.langchain.com/langsmith) — 用于智能体评估和可观测性。调试性能不佳的 LLM 应用运行，评估智能体轨迹，在生产中获得可见性，并随时间改进性能。
- [LangSmith 部署](https://docs.langchain.com/langsmith/deployments) — 使用专为长时间运行的有状态工作流而构建的部署平台，轻松部署和扩展智能体。在团队间发现、重用、配置和共享智能体 —— 并通过 [LangGraph Studio](https://docs.langchain.com/oss/python/langgraph/studio) 中的可视化原型快速迭代。
- [LangChain](https://docs.langchain.com/oss/python/langchain/overview) — 提供集成和可组合组件，以简化 LLM 应用开发。

> [!NOTE]
> 正在寻找 LangGraph 的 JS 版本？请参阅 [JS 仓库](https://github.com/langchain-ai/langgraphjs) 和 [JS 文档](https://docs.langchain.com/oss/javascript/langgraph/overview)。

## 其他资源

- [指南](https://docs.langchain.com/oss/python/langgraph/guides)：关于流式处理、添加记忆和持久化、设计模式（例如分支、子图等）等主题的快速、可操作的代码片段。
- [参考](https://reference.langchain.com/python/langgraph/)：关于核心类、方法、如何使用图和检查点 API 以及更高级别的预构建组件的详细参考。
- [示例](https://docs.langchain.com/oss/python/langgraph/agentic-rag)：开始使用 LangGraph 的指导性示例。
- [LangChain 论坛](https://forum.langchain.com/)：与社区联系并分享您所有的技术问题、想法和反馈。
- [LangChain 学院](https://academy.langchain.com/courses/intro-to-langgraph)：在我们的免费结构化课程中学习 LangGraph 的基础知识。
- [案例研究](https://www.langchain.com/built-with-langgraph)：了解行业领导者如何使用 LangGraph 大规模部署 AI 应用。

## 致谢

LangGraph 受到 [Pregel](https://research.google/pubs/pub37252/) 和 [Apache Beam](https://beam.apache.org/) 的启发。公共接口借鉴了 [NetworkX](https://networkx.org/documentation/latest/) 的灵感。LangGraph 由 LangChain 的创建者 LangChain Inc 构建，但可以在不使用 LangChain 的情况下使用。
