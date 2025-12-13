---
search:
  boost: 2
---

# 常见问题 (FAQ)

常见问题及其解答！

## 我需要使用 LangChain 才能使用 LangGraph 吗？(Do I need to use LangChain to use LangGraph? What’s the difference?)

不需要。LangGraph 是一个用于复杂代理系统的编排框架，比 LangChain 代理更底层且更可控。LangChain 提供了一个标准接口来与模型和其他组件进行交互，这对于简单的链和检索流程非常有用。

## LangGraph 与其他代理框架有何不同？(How is LangGraph different from other agent frameworks?)

其他代理框架可能适用于简单的通用任务，但在处理复杂任务时往往力不从心。LangGraph 提供了一个更具表现力的框架来处理您的独特任务，而不会将您限制在一个黑盒认知架构中。

## LangGraph 会影响我的应用程序性能吗？(Does LangGraph impact the performance of my app?)

LangGraph 不会给您的代码增加任何开销，并且是专门为流式工作流设计的。

## LangGraph 是开源的吗？它是免费的吗？(Is LangGraph open source? Is it free?)

是的。LangGraph 是一个 MIT 许可的开源库，并且可以免费使用。

## LangGraph 和 LangGraph Platform 有什么不同？(How are LangGraph and LangGraph Platform different?)

LangGraph 是一个有状态的编排框架，为代理工作流带来了额外的控制。LangGraph Platform 是一个用于部署和扩展 LangGraph 应用程序的服务，具有用于构建代理 UX 的固有 API，以及一个集成的开发人员工作室。

| 特性 (Features)     | LangGraph (开源)                                          | LangGraph Platform                                                                                     |
| ------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| 描述 (Description)  | 用于代理应用程序的有状态编排框架                          | 用于部署 LangGraph 应用程序的可扩展基础设施                                                            |
| SDKs                | Python 和 JavaScript                                      | Python 和 JavaScript                                                                                  |
| HTTP APIs           | 无 (None)                                                 | 是 (Yes) - 适于检索和更新状态或长期记忆，或创建可配置的助手                                       |
| 流式传输 (Streaming)| 基础 (Basic)                                              | 针对逐个令牌消息的专用模式                                                                             |
| 检查点 (Checkpointer)| 社区贡献 (Community contributed)                          | 开箱即用支持 (Supported out-of-the-box)                                                                |
| 持久层 (Persistence Layer)| 自管 (Self-managed)                                 | 具有高效存储的托管 Postgres                                                                            |
| 部署 (Deployment)   | 自管 (Self-managed)                                       | • Cloud SaaS <br> • 免费自托管 <br> • 企业版 (付费自托管)                                              |
| 可扩展性 (Scalability)| 自管 (Self-managed)                                     | 任务队列和服务器的自动缩放                                                                             |
| 容错 (Fault-tolerance)| 自管 (Self-managed)                                     | 自动重试                                                                                               |
| 并发控制 (Concurrency Control)| 简单线程 (Simple threading)                     | 支持双重文本 (Supports double-texting)                                                                 |
| 调度 (Scheduling)   | 无 (None)                                                 | Cron 调度                                                                                              |
| 监控 (Monitoring)   | 无 (None)                                                 | 与 LangSmith 集成以实现可观测性                                                                         |
| IDE 集成 (IDE integration)| LangGraph Studio                                    | LangGraph Studio                                                                                       |

## LangGraph Platform 是开源的吗？(Is LangGraph Platform open source?)

不。LangGraph Platform 是专有软件。

LangGraph Platform 有一个免费的自托管版本，可以访问基本功能。Cloud SaaS 部署选项和自托管部署选项是付费服务。[联系我们的销售团队](https://www.langchain.com/contact-sales) 了解更多信息。

有关更多信息，请参阅我们的 [LangGraph Platform 定价页面](https://www.langchain.com/pricing-langgraph-platform)。

## LangGraph 是否适用于不支持工具调用的 LLM？(Does LangGraph work with LLMs that don't support tool calling?)

是的！您可以将 LangGraph 与任何 LLM 一起使用。我们使用支持工具调用的 LLM 的主要原因是，这通常是让 LLM 做出关于做什么的决定的最方便的方式。即便您的 LLM 不支持工具调用，您仍然可以使用它 —— 您只需要编写一点逻辑将原始 LLM 字符串响应转换为关于做什么的决定。

## LangGraph 是否适用于 OSS LLM？(Does LangGraph work with OSS LLMs?)

是的！LangGraph 对底层使用什么 LLM 完全不敏感。我们在大多数教程中使用闭源 LLM 的主要原因是它们无缝支持工具调用，而 OSS LLM 通常不支持。但工具调用不是必须的（参见 [本节](#does-langgraph-work-with-llms-that-dont-support-tool-calling)），因此您完全可以将 LangGraph 与 OSS LLM 一起使用。

## 我可以在不登录 LangSmith 的情况下使用 LangGraph Studio 吗？(Can I use LangGraph Studio without logging in to LangSmith)

是的！您可以使用 [LangGraph Server 的开发版本](../tutorials/langgraph-platform/local-server.md) 在本地运行后端。
这将连接到作为 LangSmith 一部分托管的 studio 前端。
如果您设置环境变量 `LANGSMITH_TRACING=false`，则不会向 LangSmith 发送任何跟踪。

## LangGraph Platform 使用情况的“已执行节点”是什么意思？(What does "nodes executed" mean for LangGraph Platform usage?)

**已执行节点 (Nodes Executed)** 是在应用程序调用期间被调用并成功完成的 LangGraph 应用程序中节点的总数。如果图中的节点在执行期间未被调用或以错误状态结束，则这些节点将不会被计数。如果一个节点被调用并成功完成多次，则每次出现都会被计数。
