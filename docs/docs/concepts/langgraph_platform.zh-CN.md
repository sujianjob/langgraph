---
search:
  boost: 2
---

# LangGraph 平台 (LangGraph Platform)

使用 **LangGraph 平台** —— 专为长时间运行的代理工作流构建的平台，来开发、部署、扩展和管理代理。

!!! tip "Get started with LangGraph Platform"

    查看 [LangGraph 平台快速入门](../tutorials/langgraph-platform/local-server.md)，了解有关如何使用 LangGraph 平台在本地运行 LangGraph 应用程序的说明。

## 为什么使用 LangGraph 平台? (Why use LangGraph Platform?)

<div align="center"><iframe width="560" height="315" src="https://www.youtube.com/embed/pfAQxBS5z88?si=XGS6Chydn6lhSO1S" title="What is LangGraph Platform?" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

LangGraph 平台使您可以轻松地在生产环境中运行您的代理 —— 无论它是使用 LangGraph 还是其他框架构建的 —— 这样您就可以专注于应用程序逻辑，而不是基础设施。一键部署即可获得实时端点，并使用我们强大的 API 和内置任务队列来处理生产规模。

- **[流式传输支持 (Streaming Support)](../cloud/how-tos/streaming.md)**: 随着代理变得越来越复杂，它们通常受益于将令牌输出和中间状态流式传输回用户。如果没有这一点，用户将在长时间运行的操作中等待且没有任何反馈。LangGraph Server 提供了针对各种应用需求优化的多种流模式。

- **[后台运行 (Background Runs)](../cloud/how-tos/background_run.md)**: 对于处理时间较长（例如数小时）的代理，保持打开连接可能是不切实际的。LangGraph Server 支持在后台启动代理运行，并提供轮询端点和 Webhook 来有效地监控运行状态。
 
- **支持长时间运行 (Support for long runs)**: 常规服务器设置在处理需要很长时间才能完成的请求时经常会遇到超时或中断。LangGraph Server 的 API 通过发送定期心跳信号为这些任务提供了强大的支持，从而防止在长时间过程中意外关闭连接。

- **处理爆发性 (Handling Burstiness)**: 某些应用程序，尤其是那些具有实时用户交互的应用程序，可能会遇到“爆发性”请求负载，即大量请求同时到达服务器。LangGraph Server 包含一个任务队列，即使在重负载下也能确保一致地处理请求而不会丢失。

- **[双重文本 (Double-texting)](../cloud/how-tos/interrupt_concurrent.md)**: 在用户驱动的应用程序中，用户快速发送多条消息是很常见的。如果处理不当，这种“双重发短信”可能会破坏代理流程。LangGraph Server 提供内置策略来解决和管理此类交互。

- **[检查点和内存管理 (Checkpointers and memory management)](persistence.md#checkpoints)**: 对于需要持久性（例如对话记忆）的代理，部署健壮的存储解决方案可能很复杂。LangGraph 平台包含优化的 [检查点](persistence.md#checkpoints) 和 [记忆存储](persistence.md#memory-store)，无需自定义解决方案即可跨会话管理状态。

- **[人在回路支持 (Human-in-the-loop support)](../cloud/how-tos/human_in_the_loop_breakpoint.md)**: 在许多应用程序中，用户需要一种干预代理流程的方法。LangGraph Server 为人在回路场景提供专用端点，简化了将手动监督集成到代理工作流中的过程。

- **[LangGraph Studio](./langgraph_studio.md)**: 启用实现 LangGraph Server API 协议的代理系统的可视化、交互和调试。Studio 还与 LangSmith 集成，以支持跟踪、评估和提示工程。

- **[部署 (Deployment)](./deployment_options.md)**: 可以在 LangGraph 平台上使用四种方式进行部署：[Cloud SaaS](../concepts/langgraph_cloud.md)、[自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md)、[自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 和 [独立容器](../concepts/langgraph_standalone_container.md)。
