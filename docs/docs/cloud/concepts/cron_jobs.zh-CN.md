## Cron 作业 (Cron jobs)

在许多情况下，按计划运行助手很有用。

例如，假设您正在构建一个每天运行并发送当天新闻的电子邮件摘要的助手。您可以使用 Cron 作业在每天晚上 8:00 运行助手。

LangGraph Platform 支持 Cron 作业，它们按用户定义的计划运行。用户指定计划、助手和一些输入。之后，在指定的计划中，服务器将：

- 使用指定的助手创建一个新线程
- 将指定的输入发送到该线程

请注意，这每次都会向线程发送相同的输入。请参阅 [操作指南](../../cloud/how-tos/cron_jobs.md) 以创建 Cron 作业。

LangGraph Platform API 提供了几个用于创建和管理 Cron 作业的端点。有关更多详细信息，请参阅 [API 参考](../../cloud/reference/api/api_ref.html#tag/runscreate/POST/threads/{thread_id}/runs/crons)。
