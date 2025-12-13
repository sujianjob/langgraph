---
search:
  boost: 2
---

# LangGraph 数据平面 (LangGraph Data Plane)

术语“数据平面”被广泛用于指代 [LangGraph Server](./langgraph_server.md)（部署）、每个服务器的相应基础设施以及持续轮询来自 [LangGraph 控制平面](./langgraph_control_plane.md) 的更新的“侦听器”应用程序。

## 服务器基础设施 (Server Infrastructure)

除了 [LangGraph Server](./langgraph_server.md) 本身之外，每个服务器的以下基础设施组件也包含在“数据平面”的广义定义中：

- Postgres
- Redis
- 机密存储 (Secrets store)
- 自动缩放器 (Autoscalers)

## “侦听器”应用程序 ("Listener" Application)

数据平面“侦听器”应用程序定期调用 [控制平面 APIs](../concepts/langgraph_control_plane.md#control-plane-api) 以：

- 确定是否应创建新部署。
- 确定是否应更新现有部署（即新修订版）。
- 确定是否应删除现有部署。

换句话说，数据平面“侦听器”读取控制平面的最新状态（期望状态）并采取行动协调未完成的部署（当前状态）以匹配最新状态。

## Postgres

Postgres 是 LangGraph Server 中所有用户、运行和长期记忆数据的持久层。这既存储检查点（在此处查看更多信息 [here](./persistence.md)）、服务器资源（线程、运行、助手和 cron），也存储保存在长期记忆存储中的项目（在此处查看更多信息 [here](./persistence.md#memory-store)）。

## Redis

Redis 用于每个 LangGraph Server，作为服务器和队列工作者通信的方式，以及存储临时元数据。Redis 中不存储用户或运行数据。

### 通信 (Communication)

LangGraph Server 中的所有运行都由作为每个部署一部分的后台工作者池执行。为了为这些运行启用某些功能（例如取消和输出流式传输），我们需要一个通道在服务器和处理特定运行的工作者之间进行双向通信。我们使用 Redis 来组织该通信。

1. Redis 列表用作一种机制，一旦创建新运行就唤醒工作者。此列表中仅存储一个哨兵值，没有实际的运行信息。然后，工作者从 Postgres 检索运行信息。
2. Redis 字符串和 Redis PubSub 通道的组合用于服务器将运行取消请求传达给适当的工作者。
3. Redis PubSub 通道用于工作者在处理运行时广播来自代理的流式输出。服务器中的任何打开的 `/stream` 请求都将订阅该通道，并在事件到达时将其转发到响应。任何时候都不会在 Redis 中存储任何事件。

### 临时元数据 (Ephemeral metadata)

LangGraph Server 中的运行可能会因特定故障而重试（目前仅针对运行期间遇到的瞬态 Postgres 错误）。为了限制重试次数（目前限制为每次运行 3 次尝试），我们在 Redis 字符串被拾取时记录尝试次数。除了其 ID 之外，这不包含任何特定于运行的信息，并且会在短暂延迟后过期。

## 数据平面功能 (Data Plane Features)

本节描述数据平面的各种功能。

### 数据区域 (Data Region)

!!! info "Only for Cloud SaaS"
    数据区域仅适用于 [Cloud SaaS](../concepts/langgraph_cloud.md) 部署。

可以在 2 个数据区域中创建部署：US 和 EU

部署的数据区域由创建部署的 LangSmith 组织的数据区域暗示。部署和部署的基础数据库无法在数据区域之间迁移。

### 自动缩放 (Autoscaling)

[`Production` 类型](../concepts/langgraph_control_plane.md#deployment-types) 部署自动扩展到 10 个容器。缩放基于 3 个指标：

1. CPU 利用率
1. 内存利用率
1. 挂起（进行中）[运行](./assistants.md#execution) 的数量

对于 CPU 利用率，自动缩放器目标是 75% 利用率。这意味着自动缩放器将增加或减少容器数量，以确 CPU 利用率处于或接近 75%。对于内存利用率，自动缩放器目标也是 75% 利用率。

对于挂起运行的数量，自动缩放器目标是 10 个挂起运行。例如，如果当前容器数量为 1，但挂起运行数量为 20，则自动缩放器会将部署扩展到 2 个容器（20 个挂起运行 / 2 个容器 = 每个容器 10 个挂起运行）。

每个指标都是独立计算的，自动缩放器将根据导致最大容器数量的指标确定缩放操作。

缩减操作会延迟 30 分钟，然后再采取任何行动。换句话说，如果自动缩放器决定缩减部署，它将首先等待 30 分钟再缩减。30 分钟后，重新计算指标，如果重新计算的指标导致的容器数量少于当前数量，则部署将缩减。否则，部署保持扩展状态。这个“冷却”期确保部署不会过于频繁地缩放。

### 静态 IP 地址 (Static IP Addresses)

!!! info "Only for Cloud SaaS"
静态 IP 地址仅适用于 [Cloud SaaS](../concepts/langgraph_cloud.md) 部署。

2025 年 1 月 6 日之后创建的部署的所有流量都将通过 NAT 网关。根据数据区域，此 NAT 网关将具有多个静态 IP 地址。请参阅下表以获取静态 IP 地址列表：

| US             | EU             |
| -------------- | -------------- |
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  |
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

### 自定义 Postgres (Custom Postgres)

!!! info
自定义 Postgres 实例仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Postgres 实例代替 [控制平面自动创建的实例](./langgraph_control_plane.md#database-provisioning)。指定 [`POSTGRES_URI_CUSTOM`](../cloud/reference/env_var.md#postgres_uri_custom) 环境变量以使用自定义 Postgres 实例。

多个部署可以共享同一个 Postgres 实例。例如，对于 `Deplyment A`，`POSTGRES_URI_CUSTOM` 可以设置为 `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>`，对于 `Deployment B`，`POSTGRES_URI_CUSTOM` 可以设置为 `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`。`<database_name_1>` 和 `database_name_2` 是同一实例中的不同数据库，但 `<hostname_1>` 是共享的。**同一数据库不能用于单独的部署**。

### 自定义 Redis (Custom Redis)

!!! info
自定义 Redis 实例仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_control_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Redis 实例代替控制平面自动创建的实例。指定 [REDIS_URI_CUSTOM](../cloud/reference/env_var.md#redis_uri_custom) 环境变量以使用自定义 Redis 实例。

多个部署可以共享同一个 Redis 实例。例如，对于 `Deployment A`，`REDIS_URI_CUSTOM` 可以设置为 `redis://<hostname_1>:<port>/1`，对于 `Deployment B`，`REDIS_URI_CUSTOM` 可以设置为 `redis://<hostname_1>:<port>/2`。`1` 和 `2` 是同一实例中的不同数据库编号，但 `<hostname_1>` 是共享的。**同一数据库编号不能用于单独的部署**。

### LangSmith 跟踪 (LangSmith Tracing)

LangGraph Server 自动配置为向 LangSmith 发送跟踪。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS                               | Self-Hosted Data Plane                                      | Self-Hosted Control Plane                                          | Standalone Container                                                                         |
| ---------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| 必需 (Required)<br><br>跟踪到 LangSmith SaaS。 | 可选 (Optional)<br><br>禁用跟踪或跟踪到 LangSmith SaaS。 | 可选 (Optional)<br><br>禁用跟踪或跟踪到自托管 LangSmith。 | 可选 (Optional)<br><br>禁用跟踪、跟踪到 LangSmith SaaS 或跟踪到自托管 LangSmith。 |

### 遥测 (Telemetry)

LangGraph Server 自动配置为报告用于计费目的的遥测元数据。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS                        | Self-Hosted Data Plane            | Self-Hosted Control Plane                                                                                                           | Standalone Container                                                                                                                |
| --------------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 遥测发送到 LangSmith SaaS。 | 遥测发送到 LangSmith SaaS。 | 针对气隙许可证密钥的自我报告使用情况（审计）。<br><br>针对 LangGraph Platform 许可证密钥的遥测发送到 LangSmith SaaS。 | 针对气隙许可证密钥的自我报告使用情况（审计）。<br><br>针对 LangGraph Platform 许可证密钥的遥测发送到 LangSmith SaaS。 |

### 许可 (Licensing)

LangGraph Server 自动配置为执行许可证密钥验证。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS                                          | Self-Hosted Data Plane                              | Self-Hosted Control Plane                                                                  | Standalone Container                                                                       |
| --------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| 根据 LangSmith SaaS 验证 LangSmith API 密钥。 | 根据 LangSmith SaaS 验证 LangSmith API 密钥。 | 根据 LangSmith SaaS 验证气隙许可证密钥或 LangGraph Platform 许可证密钥。 | 根据 LangSmith SaaS 验证气隙许可证密钥或 LangGraph Platform 许可证密钥。 |
