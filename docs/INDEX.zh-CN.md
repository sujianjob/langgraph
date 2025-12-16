# LangGraph 中文文档索引

本文档整理了最新提交中包含的 LangGraph 中文文档及其用途说明。

## 📁 根目录文档

| 文件路径 | 说明 |
| :--- | :--- |
| [CONTRIBUTING.zh-CN.md](../.github/CONTRIBUTING.zh-CN.md) | 贡献指南：指导开发者如何为项目做出贡献 |
| [AGENTS.zh-CN.md](../AGENTS.zh-CN.md) | 智能体相关说明：关于 LangGraph 智能体的顶层介绍 |
| [CLAUDE.zh-CN.md](../CLAUDE.zh-CN.md) | Claude 相关说明：特定于 Claude 模型的集成或使用说明 |
| [README.zh-CN.md](../README.zh-CN.md) | 项目自述文件：包含项目简介、安装和快速开始 |
| [docs/README.zh-CN.md](README.zh-CN.md) | 文档概览：文档结构、构建过程和本地开发指南 |
| [docs/_scripts/js_translation/codeblocks/README.zh-CN.md](_scripts/js_translation/codeblocks/README.zh-CN.md) | JS 翻译脚本说明：关于代码块翻译工具的说明 |
| [docs/docs/llms-txt-overview.zh-CN.md](docs/llms-txt-overview.zh-CN.md) | LLMs.txt 概览：关于 LLM 上下文文件的说明 |

## 🤖 智能体 (Agents)
位于 `docs/docs/agents/`

| 文件名 | 说明 |
| :--- | :--- |
| [overview.zh-CN.md](docs/agents/overview.zh-CN.md) | 智能体概览：LangGraph 智能体系统的总览 |
| [agents.zh-CN.md](docs/agents/agents.zh-CN.md) | 智能体：详细介绍智能体的概念和实现 |
| [context.zh-CN.md](docs/agents/context.zh-CN.md) | 上下文管理：如何在智能体中管理上下文 |
| [evals.zh-CN.md](docs/agents/evals.zh-CN.md) | 评估：智能体性能的评估方法 |
| [mcp.zh-CN.md](docs/agents/mcp.zh-CN.md) | MCP 协议：模型上下文协议 (Model Context Protocol) 相关文档 |
| [models.zh-CN.md](docs/agents/models.zh-CN.md) | 模型：支持的模型及其配置 |
| [multi-agent.zh-CN.md](docs/agents/multi-agent.zh-CN.md) | 多智能体系统：构建多智能体协作应用 |
| [prebuilt.zh-CN.md](docs/agents/prebuilt.zh-CN.md) | 预构建智能体：使用开箱即用的智能体 |
| [run_agents.zh-CN.md](docs/agents/run_agents.zh-CN.md) | 运行智能体：执行和管理智能体的指南 |
| [ui.zh-CN.md](docs/agents/ui.zh-CN.md) | 用户界面：智能体交互界面的相关说明 |

## ☁️ 云端与部署 (Cloud & Deployment)
位于 `docs/docs/cloud/`

### 概念 (Concepts)
| 文件名 | 说明 |
| :--- | :--- |
| [concepts/cron_jobs.zh-CN.md](docs/cloud/concepts/cron_jobs.zh-CN.md) | 定时任务：云端定时任务的概念 |
| [concepts/data_storage_and_privacy.zh-CN.md](docs/cloud/concepts/data_storage_and_privacy.zh-CN.md) | 数据存储与隐私：云端数据处理策略 |
| [concepts/webhooks.zh-CN.md](docs/cloud/concepts/webhooks.zh-CN.md) | Webhooks：云端 Webhooks 机制 |

### 部署 (Deployment)
| 文件名 | 说明 |
| :--- | :--- |
| [deployment/cloud.zh-CN.md](docs/cloud/deployment/cloud.zh-CN.md) | LangGraph Cloud：云服务部署概览 |
| [deployment/custom_docker.zh-CN.md](docs/cloud/deployment/custom_docker.zh-CN.md) | 自定义 Docker：使用自定义容器镜像部署 |
| [deployment/egress.zh-CN.md](docs/cloud/deployment/egress.zh-CN.md) | 出站流量：网络出站规则配置 |
| [deployment/graph_rebuild.zh-CN.md](docs/cloud/deployment/graph_rebuild.zh-CN.md) | 图重建：关于图的重新构建机制 |
| [deployment/self_hosted_control_plane.zh-CN.md](docs/cloud/deployment/self_hosted_control_plane.zh-CN.md) | 自托管控制平面：部署自己的控制平面 |
| [deployment/self_hosted_data_plane.zh-CN.md](docs/cloud/deployment/self_hosted_data_plane.zh-CN.md) | 自托管数据平面：部署自己的数据平面 |
| [deployment/semantic_search.zh-CN.md](docs/cloud/deployment/semantic_search.zh-CN.md) | 语义搜索：部署语义搜索功能 |
| [deployment/setup.zh-CN.md](docs/cloud/deployment/setup.zh-CN.md) | 设置指南：部署环境的初始化设置 |
| [deployment/setup_javascript.zh-CN.md](docs/cloud/deployment/setup_javascript.zh-CN.md) | JS 环境设置：JavaScript 项目的部署设置 |
| [deployment/setup_pyproject.zh-CN.md](docs/cloud/deployment/setup_pyproject.zh-CN.md) | Python 环境设置：Python 项目的部署设置 |
| [deployment/standalone_container.zh-CN.md](docs/cloud/deployment/standalone_container.zh-CN.md) | 独立容器：单容器部署方案 |

### 操作指南 (How-tos)
包含 `docs/docs/cloud/how-tos/` 下的各类云端操作指南：

*   **配置与管理**: [configuration_cloud](docs/cloud/how-tos/configuration_cloud.zh-CN.md) (云配置), [configurable_headers](docs/cloud/how-tos/configurable_headers.zh-CN.md) (配置头信息), `environment_variables` (环境变量 - 见 Reference)
*   **开发调试**: [clone_traces_studio](docs/cloud/how-tos/clone_traces_studio.zh-CN.md) (克隆 Trace 到 Studio), [invoke_studio](docs/cloud/how-tos/invoke_studio.zh-CN.md) (在 Studio 中调用), [iterate_graph_studio](docs/cloud/how-tos/iterate_graph_studio.zh-CN.md) (在 Studio 中迭代图), [studio/quick_start](docs/cloud/how-tos/studio/quick_start.zh-CN.md) (Studio 快速开始)
*   **运行控制**: [background_run](docs/cloud/how-tos/background_run.zh-CN.md) (后台运行), [cron_jobs](docs/cloud/how-tos/cron_jobs.zh-CN.md) (定时任务), [enqueue_concurrent](docs/cloud/how-tos/enqueue_concurrent.zh-CN.md) (排队并发), [interrupt_concurrent](docs/cloud/how-tos/interrupt_concurrent.zh-CN.md) (中断并发), [reject_concurrent](docs/cloud/how-tos/reject_concurrent.zh-CN.md) (拒绝并发), [rollback_concurrent](docs/cloud/how-tos/rollback_concurrent.zh-CN.md) (回滚并发)
*   **流式与交互**: [streaming](docs/cloud/how-tos/streaming.zh-CN.md) (流式传输), [use_stream_react](docs/cloud/how-tos/use_stream_react.zh-CN.md) (在 React 中使用流), [generative_ui_react](docs/cloud/how-tos/generative_ui_react.zh-CN.md) (React 生成式 UI)
*   **人机交互 (HITL)**: [add-human-in-the-loop](docs/cloud/how-tos/add-human-in-the-loop.zh-CN.md) (添加 HITL), [human_in_the_loop_time_travel](docs/cloud/how-tos/human_in_the_loop_time_travel.zh-CN.md) (HITL 时间旅行)
*   **高级功能**: [datasets_studio](docs/cloud/how-tos/datasets_studio.zh-CN.md) (数据集管理), [run_evals](docs/cloud/how-tos/studio/run_evals.zh-CN.md) (运行评估), [threads_studio](docs/cloud/how-tos/threads_studio.zh-CN.md) (线程管理), [webhooks](docs/cloud/how-tos/webhooks.zh-CN.md) (使用 Webhooks), [stateless_runs](docs/cloud/how-tos/stateless_runs.zh-CN.md) (无状态运行), [same-thread](docs/cloud/how-tos/same-thread.zh-CN.md) (同线程运行)

### 参考 (Reference)
| 文件名 | 说明 |
| :--- | :--- |
| [api/api_ref.zh-CN.md](docs/cloud/reference/api/api_ref.zh-CN.md) | API 参考：云端 API 文档 |
| [api/api_ref_control_plane.zh-CN.md](docs/cloud/reference/api/api_ref_control_plane.zh-CN.md) | 控制平面 API：控制平面特定 API |
| [cli.zh-CN.md](docs/cloud/reference/cli.zh-CN.md) | CLI 参考：命令行工具参考手册 |
| [env_var.zh-CN.md](docs/cloud/reference/env_var.zh-CN.md) | 环境变量：配置环境变量参考 |
| [langgraph_server_changelog.zh-CN.md](docs/cloud/reference/langgraph_server_changelog.zh-CN.md) | 服务器更新日志：LangGraph Server 版本记录 |
| [sdk/js_ts_sdk_ref.zh-CN.md](docs/cloud/reference/sdk/js_ts_sdk_ref.zh-CN.md) | JS/TS SDK 参考：JavaScript SDK 文档 |
| [sdk/python_sdk_ref.zh-CN.md](docs/cloud/reference/sdk/python_sdk_ref.zh-CN.md) | Python SDK 参考：Python SDK 文档 |

## 💡 核心概念 (Concepts)
位于 `docs/docs/concepts/`

| 文件名 | 说明 |
| :--- | :--- |
| [agentic_concepts.zh-CN.md](docs/concepts/agentic_concepts.zh-CN.md) | 代理概念：智能体系统的核心理念 |
| [application_structure.zh-CN.md](docs/concepts/application_structure.zh-CN.md) | 应用结构：LangGraph 应用的架构 |
| [assistants.zh-CN.md](docs/concepts/assistants.zh-CN.md) | 助手：助手 (Assistants) 的概念 |
| [auth.zh-CN.md](docs/concepts/auth.zh-CN.md) | 认证：安全与认证机制 |
| [deployment_options.zh-CN.md](docs/concepts/deployment_options.zh-CN.md) | 部署选项：各种部署方式的对比 |
| [double_texting.zh-CN.md](docs/concepts/double_texting.zh-CN.md) | 双重消息处理：处理用户连续发送消息的机制 |
| [durable_execution.zh-CN.md](docs/concepts/durable_execution.zh-CN.md) | 持久执行：保证任务可靠执行的机制 |
| [faq.zh-CN.md](docs/concepts/faq.zh-CN.md) | 常见问题：FAQ |
| [functional_api.zh-CN.md](docs/concepts/functional_api.zh-CN.md) | 函数式 API：Functional API 的使用概念 |
| [human_in_the_loop.zh-CN.md](docs/concepts/human_in_the_loop.zh-CN.md) | 人机交互：HITL 核心概念 |
| [langgraph_cli.zh-CN.md](docs/concepts/langgraph_cli.zh-CN.md) | LangGraph CLI：命令行工具概念 |
| [langgraph_cloud.zh-CN.md](docs/concepts/langgraph_cloud.zh-CN.md) | LangGraph Cloud：云服务概念 |
| [langgraph_components.zh-CN.md](docs/concepts/langgraph_components.zh-CN.md) | 组件：LangGraph 的主要组件 |
| [langgraph_control_plane.zh-CN.md](docs/concepts/langgraph_control_plane.zh-CN.md) | 控制平面：架构中的控制平面 |
| [langgraph_data_plane.zh-CN.md](docs/concepts/langgraph_data_plane.zh-CN.md) | 数据平面：架构中的数据平面 |
| [langgraph_platform.zh-CN.md](docs/concepts/langgraph_platform.zh-CN.md) | 平台：LangGraph Platform 概览 |
| [langgraph_studio.zh-CN.md](docs/concepts/langgraph_studio.zh-CN.md) | Studio：可视化开发环境 Studio 概念 |
| [low_level.zh-CN.md](docs/concepts/low_level.zh-CN.md) | 底层机制：框架的底层工作原理 |
| [memory.zh-CN.md](docs/concepts/memory.zh-CN.md) | 记忆/存储：状态记忆与检查点 |
| [multi_agent.zh-CN.md](docs/concepts/multi_agent.zh-CN.md) | 多智能体：多智能体协作概念 |
| [persistence.zh-CN.md](docs/concepts/persistence.zh-CN.md) | 持久化：状态持久化机制 |
| [plans.zh-CN.md](docs/concepts/plans.zh-CN.md) | 计划：关于订阅计划或执行计划的概念 |
| [pregel.zh-CN.md](docs/concepts/pregel.zh-CN.md) | Pregel：基于 Pregel 的图计算模型 |
| [scalability_and_resilience.zh-CN.md](docs/concepts/scalability_and_resilience.zh-CN.md) | 扩展性与弹性：系统的高可用设计 |
| [sdk.zh-CN.md](docs/concepts/sdk.zh-CN.md) | SDK：软件开发工具包概览 |
| [server-mcp.zh-CN.md](docs/concepts/server-mcp.zh-CN.md) | Server MCP：服务器端的模型上下文协议 |
| [streaming.zh-CN.md](docs/concepts/streaming.zh-CN.md) | 流式传输：流式输出的概念 |
| [subgraphs.zh-CN.md](docs/concepts/subgraphs.zh-CN.md) | 子图：图的嵌套与复用 |
| [template_applications.zh-CN.md](docs/concepts/template_applications.zh-CN.md) | 模板应用：应用模板的使用 |
| [time-travel.zh-CN.md](docs/concepts/time-travel.zh-CN.md) | 时间旅行：调试与回溯功能 |
| [tools.zh-CN.md](docs/concepts/tools.zh-CN.md) | 工具：智能体可使用的工具 |
| [tracing.zh-CN.md](docs/concepts/tracing.zh-CN.md) | 追踪：执行链路追踪 |
| [why-langgraph.zh-CN.md](docs/concepts/why-langgraph.zh-CN.md) | 为什么选择 LangGraph：框架的设计哲学 |

## 📘 操作指南 (How-tos)
位于 `docs/docs/how-tos/`

| 文件名 | 说明 |
| :--- | :--- |
| [auth/custom_auth.zh-CN.md](docs/how-tos/auth/custom_auth.zh-CN.md) | 自定义认证：实现自定义认证流 |
| [auth/openapi_security.zh-CN.md](docs/how-tos/auth/openapi_security.zh-CN.md) | OpenAPI 安全：OpenAPI 安全配置 |
| [autogen-integration.zh-CN.md](docs/how-tos/autogen-integration.zh-CN.md) | AutoGen 集成：与 AutoGen 框架集成 |
| [enable-tracing.zh-CN.md](docs/how-tos/enable-tracing.zh-CN.md) | 开启追踪：配置链路追踪 |
| [graph-api.zh-CN.md](docs/how-tos/graph-api.zh-CN.md) | 图 API：使用 Graph API |
| [http/custom_lifespan.zh-CN.md](docs/how-tos/http/custom_lifespan.zh-CN.md) | HTTP 生命周期：自定义 HTTP 生命周期事件 |
| [http/custom_middleware.zh-CN.md](docs/how-tos/http/custom_middleware.zh-CN.md) | HTTP 中间件：自定义 HTTP 中间件 |
| [http/custom_routes.zh-CN.md](docs/how-tos/http/custom_routes.zh-CN.md) | HTTP 路由：自定义 HTTP 路由 |
| [human_in_the_loop/add-human-in-the-loop.zh-CN.md](docs/how-tos/human_in_the_loop/add-human-in-the-loop.zh-CN.md) | 添加 HITL：实现人机交互流程 |
| [human_in_the_loop/time-travel.zh-CN.md](docs/how-tos/human_in_the_loop/time-travel.zh-CN.md) | HITL 时间旅行：在 HITL 中使用时间旅行 |
| [memory/add-memory.zh-CN.md](docs/how-tos/memory/add-memory.zh-CN.md) | 添加记忆：为图添加持久化记忆 |
| [multi_agent.zh-CN.md](docs/how-tos/multi_agent.zh-CN.md) | 多智能体指南：实现多智能体模式 |
| [run-id-langsmith.zh-CN.md](docs/how-tos/run-id-langsmith.zh-CN.md) | LangSmith 运行 ID：关联运行 ID 到 LangSmith |
| [streaming.zh-CN.md](docs/how-tos/streaming.zh-CN.md) | 流式指南：实现流式输出 |
| [subgraph.zh-CN.md](docs/how-tos/subgraph.zh-CN.md) | 子图指南：使用子图组织逻辑 |
| [tool-calling.zh-CN.md](docs/how-tos/tool-calling.zh-CN.md) | 工具调用：实现工具调用功能 |
| [ttl/configure_ttl.zh-CN.md](docs/how-tos/ttl/configure_ttl.zh-CN.md) | 配置 TTL：设置状态生存时间 |
| [use-functional-api.zh-CN.md](docs/how-tos/use-functional-api.zh-CN.md) | 使用 Functional API：函数式编程风格指南 |
| [use-remote-graph.zh-CN.md](docs/how-tos/use-remote-graph.zh-CN.md) | 使用远程图：连接和使用远程部署的图 |

## 🛠️ 故障排除 (Troubleshooting)
位于 `docs/docs/troubleshooting/`

| 文件名 | 说明 |
| :--- | :--- |
| [studio.zh-CN.md](docs/troubleshooting/studio.zh-CN.md) | Studio 故障排除：解决使用 LangGraph Studio 遇到的问题 |
| [errors/index.zh-CN.md](docs/troubleshooting/errors/index.zh-CN.md) | 错误索引：常见错误列表 |
| [errors/GRAPH_RECURSION_LIMIT.zh-CN.md](docs/troubleshooting/errors/GRAPH_RECURSION_LIMIT.zh-CN.md) | 错误：图递归限制 |
| [errors/INVALID_CHAT_HISTORY.zh-CN.md](docs/troubleshooting/errors/INVALID_CHAT_HISTORY.zh-CN.md) | 错误：无效的聊天记录 |
| [errors/INVALID_CONCURRENT_GRAPH_UPDATE.zh-CN.md](docs/troubleshooting/errors/INVALID_CONCURRENT_GRAPH_UPDATE.zh-CN.md) | 错误：无效的并发图更新 |
| [errors/INVALID_GRAPH_NODE_RETURN_VALUE.zh-CN.md](docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE.zh-CN.md) | 错误：无效的图节点返回值 |
| [errors/INVALID_LICENSE.zh-CN.md](docs/troubleshooting/errors/INVALID_LICENSE.zh-CN.md) | 错误：无效的许可证 |
| [errors/MULTIPLE_SUBGRAPHS.zh-CN.md](docs/troubleshooting/errors/MULTIPLE_SUBGRAPHS.zh-CN.md) | 错误：多个子图冲突 |

## 📚 教程 (Tutorials)
位于 `docs/docs/tutorials/`

| 文件名 | 说明 |
| :--- | :--- |
| [multi_agent/agent_supervisor.zh-CN.md](docs/tutorials/multi_agent/agent_supervisor.zh-CN.md) | 代理监督者：构建 Supervisor 模式的多智能体系统 |
| [workflows.zh-CN.md](docs/tutorials/workflows.zh-CN.md) | 工作流：构建基本工作流教程 |
