---
search:
  boost: 2
---

# LangGraph 平台计划 (LangGraph Platform Plans)


## 概述 (Overview)
LangGraph Platform 是用于在生产环境中部署代理应用程序的解决方案。
有三种不同的计划可供使用。

- **Developer**: 所有 [LangSmith](https://smith.langchain.com/) 用户都可以访问此计划。您只需创建一个 LangSmith 帐户即可注册此计划。这使您可以访问 [本地部署](./deployment_options.md#free-deployment) 选项。
- **Plus**: 所有拥有 [Plus 帐户](https://docs.smith.langchain.com/administration/pricing) 的 [LangSmith](https://smith.langchain.com/) 用户都可以访问此计划。您只需将 LangSmith 帐户升级到 Plus 计划类型即可注册此计划。这使您可以访问 [云端](./deployment_options.md#cloud-saas) 部署选项。
- **Enterprise**: 这与 LangSmith 计划是分开的。您可以联系我们的 [销售团队](https://www.langchain.com/contact-sales) 注册此计划。这使您可以访问所有 [部署选项](./deployment_options.md)。


## 计划详情 (Plan Details)

|                                                                  | Developer                                   | Plus                                                  | Enterprise                                          |
|------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------|-----------------------------------------------------|
| 部署选项 (Deployment Options)                                    | 本地 (Local)                          | Cloud SaaS                                         | <ul><li>Cloud SaaS</li><li>自托管数据平面 (Self-Hosted Data Plane)</li><li>自托管控制平面 (Self-Hosted Control Plane)</li><li>独立容器 (Standalone Container)</li></ul> |
| 使用 (Usage)                                                     | 免费 (Free) | 参见 [定价](https://www.langchain.com/langgraph-platform-pricing) | 自定义 (Custom)                                              |
| 用于检索和更新状态及对话历史记录的 API (APIs for retrieving and updating state and conversational history) | ✅                                           | ✅                                                     | ✅                                                   |
| 用于检索和更新长期记忆的 API (APIs for retrieving and updating long-term memory)                | ✅                                           | ✅                                                     | ✅                                                   |
| 水平可扩展的任务队列和服务器 (Horizontally scalable task queues and servers)                    | ✅                                           | ✅                                                     | ✅                                                   |
| 输出和中间步骤的实时流式传输 (Real-time streaming of outputs and intermediate steps)            | ✅                                           | ✅                                                     | ✅                                                   |
| Assistants API（LangGraph 应用程序的可配置模板） (Assistants API (configurable templates for LangGraph apps))       | ✅                                           | ✅                                                     | ✅                                                   |
| Cron 调度 (Cron scheduling)                                      | --                                          | ✅                                                     | ✅                                                   |
| 用于原型的 LangGraph Studio (LangGraph Studio for prototyping)                                 | 	✅                                         | ✅                                                    | ✅                                                  |
| 调用 LangGraph API 的身份验证和授权 (Authentication & authorization to call the LangGraph APIs)        | --                                          | 即将推出! (Coming Soon!)                                          | 即将推出! (Coming Soon!)                                        |
| 智能缓存以减少对 LLM API 的流量 (Smart caching to reduce traffic to LLM API)                       | --                                          | 即将推出! (Coming Soon!)                                          | 即将推出! (Coming Soon!)                                        |
| 状态的发布/订阅 API (Publish/subscribe API for state)                                  | --                                          | 即将推出! (Coming Soon!)                                          | 即将推出! (Coming Soon!)                                        |
| 调度优先级 (Scheduling prioritization)                                        | --                                          | 即将推出! (Coming Soon!)                                          | 即将推出! (Coming Soon!)                                        |

有关定价信息，请参阅 [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)。

## 相关内容 (Related)

有关更多信息，请参阅：

* [部署选项概念指南](./deployment_options.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
* [LangSmith 计划](https://docs.smith.langchain.com/administration/pricing)
