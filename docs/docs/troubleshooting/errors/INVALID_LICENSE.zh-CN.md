# 无效的许可证 (INVALID_LICENSE)

当尝试启动自托管 LangGraph Platform 服务器时，如果许可证验证失败，将会引发此错误。此错误特定于 LangGraph Platform，与开源库无关。

## 何时发生 (When This Occurs)

如果在没有有效企业许可证或 API 密钥的情况下运行 LangGraph Platform 的自托管部署，则会发生此错误。

## 故障排除 (Troubleshooting)

### 确认部署类型

首先，确认所需的部署模式。

#### 本地开发 (For Local Development)

如果您只是在本地开发，可以通过运行 `langgraph dev` 来使用轻量级的内存服务器。
有关更多信息，请参阅 [本地服务器](../../tutorials/langgraph-platform/local-server.md) 文档。

#### 托管 LangGraph Platform (For Managed LangGraph Platform)

如果您想要一个快速的托管环境，请考虑 [Cloud SaaS](../../concepts/langgraph_cloud.md) 部署选项。这不需要额外的许可证密钥。

#### 独立容器 (For Standalone Container)

对于自托管，请设置 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量。如果您对企业许可证密钥感兴趣，请联系 LangChain 支持团队。

有关部署选项及其功能的更多信息，请参阅 [部署选项](../../concepts/deployment_options.md) 文档。

### 确认凭据

如果您已确认要自托管 LangGraph Platform，请验证您的凭据。

#### 独立容器 (For Standalone Container)

1. 确认您已在部署环境或 `.env` 文件中提供了有效的 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量
2. 确认密钥仍然有效且未过期
