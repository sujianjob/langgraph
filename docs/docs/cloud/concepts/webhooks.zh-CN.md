# Webhooks

Webhooks 启用从 LangGraph Platform 应用程序到外部服务的事件驱动通信。例如，您可能希望在对 LangGraph Platform 的 API 调用运行完成后向单独的服务发布更新。

许多 LangGraph Platform 端点接受 `webhook` 参数。如果可以接受 POST 请求的端点指定了此参数，LangGraph Platform 将在运行完成时发送请求。

有关更多详细信息，请参阅相应的 [操作指南](../../cloud/how-tos/webhooks.md)。
