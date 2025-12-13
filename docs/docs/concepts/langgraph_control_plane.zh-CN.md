---
search:
  boost: 2
---

# LangGraph 控制平面 (LangGraph Control Plane)

术语“控制平面”被广泛用于指代用户创建和更新 [LangGraph Server](./langgraph_server.md)（部署）的控制平面 UI 以及支持 UI 体验的控制平面 API。

当用户通过控制平面 UI 进行更新时，更新存储在控制平面状态中。[LangGraph 数据平面](./langgraph_data_plane.md)“侦听器”应用程序通过调用控制平面 API 轮询这些更新。

## 控制平面 UI (Control Plane UI)

在控制平面 UI 中，您可以：

- 查看未完成的部署列表。
- 查看单个部署的详细信息。
- 创建新部署。
- 更新部署。
- 更新部署的环境变量。
- 查看部署的构建和服务器日志。
- 查看部署指标，例如 CPU 和内存使用情况。
- 删除部署。

控制平面 UI 嵌入在 [LangSmith](https://docs.smith.langchain.com/langgraph_cloud) 中。

## 控制平面 API (Control Plane API)

本节描述控制平面 API 的数据模型。API 用于创建、更新和删除部署。有关更多详细信息，请参阅 [控制平面 API 参考](../cloud/reference/api/api_ref_control_plane.md)。

### 部署 (Deployment)

部署是 LangGraph Server 的实例。单个部署可以有许多修订版。

### 修订版 (Revision)

修订版是部署的迭代。这也是创建新部署时，会自动创建初始修订版。要为部署部署代码更改或更新机密，必须创建新的修订版。

## 控制平面功能 (Control Plane Features)

本节描述控制平面的各种功能。

### 部署类型 (Deployment Types)

为简单起见，控制平面提供两种具有不同资源分配的部署类型：`Development` 和 `Production`。

| **部署类型 (Deployment Type)** | **CPU/内存**  | **缩放 (Scaling)**       | **数据库 (Database)**                                                                     |
| ------------------- | --------------- | ----------------- | -------------------------------------------------------------------------------- |
| Development         | 1 CPU, 1 GB RAM | 最多 1 个副本   | 10 GB 磁盘，无备份                                                           |
| Production          | 2 CPU, 2 GB RAM | 最多 10 个副本 | 自动缩放磁盘，自动备份，高可用（多区域配置） |

CPU 和内存资源是每个副本的。

!!! warning "Immutable Deployment Type"

    创建部署后，无法更改部署类型。

!!! info "Self-Hosted Deployment"
[自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署的资源可以完全自定义。部署类型仅适用于 [Cloud SaaS](../concepts/langgraph_cloud.md) 部署。

#### Production

`Production` 类型部署适用于“生产”工作负载。例如，为关键路径中的面向客户的应用程序选择 `Production`。

`Production` 类型部署的资源可以根据用例和容量限制逐个手动增加。联系 support@langchain.dev 请求增加资源。

#### Development

`Development` 类型部署适用于开发和测试。例如，为内部测试环境选择 `Development`。`Development` 类型部署不适用于“生产”工作负载。

!!! danger "Preemptible Compute Infrastructure"
`Development` 类型部署（API 服务器、队列服务器和数据库）是在抢占式计算基础设施上配置的。这意味着计算基础设施 **可能随时终止，恕不另行通知**。这可能会导致间歇性...

    - Redis 连接超时/错误
    - Postgres 连接超时/错误
    - 后台运行失败或重试

    这种行为是预期的。抢占式计算基础设施 **显着降低了配置 `Development` 类型部署的成本**。按照设计，LangGraph Server 具有容错能力。实现将自动尝试从 Redis/Postgres 连接错误中恢复并重试失败的后台运行。

    `Production` 类型部署是在持久计算基础设施上配置的，而不是抢占式计算基础设施。

`Development` 类型部署的数据库磁盘大小可以根据用例和容量限制逐个手动增加。对于大多数用例，应配置 [TTLs](../how-tos/ttl/configure_ttl.md) 以管理磁盘使用率。联系 support@langchain.dev 请求增加资源。

### 数据库配置 (Database Provisioning)

控制平面和 [LangGraph 数据平面](./langgraph_data_plane.md)“侦听器”应用程序协调为每个部署自动创建 Postgres 数据库。数据库充当部署的 [持久层](../concepts/persistence.md)。

实现 LangGraph 应用程序时，开发人员无需配置 [检查点](../concepts/persistence.md#checkpointer-libraries)。相反，会为图自动配置检查点。为图配置的任何检查点都将被自动配置的检查点替换。

无法直接访问数据库。对数据库的所有访问都通过 [LangGraph Server](../concepts/langgraph_server.md) 进行。

在删除部署之前，永远不会删除数据库。

!!! info
可以为 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署配置自定义 Postgres 实例。

### 异步部署 (Asynchronous Deployment)

部署和修订版的基础设施是异步配置和部署的。它们不会在提交后立即部署。目前，部署可能需要几分钟。

- 创建新部署时，会为部署创建以个新数据库。数据库创建是一个一次性步骤。此步骤会导致部署初始修订版的部署时间更长。
- 为部署创建后续修订版时，没有数据库创建步骤。后续修订版的部署时间比初始修订版的部署时间快得多。
- 每个修订版的部署过程包含一个构建步骤，这可能需要几分钟。

控制平面和 [LangGraph 数据平面](./langgraph_data_plane.md)“侦听器”应用程序协调以实现异步部署。

### 监控 (Monitoring)

部署准备就绪后，控制平面会监控部署并记录各种指标，例如：

- 部署的 CPU 和内存使用情况。
- 容器重启次数。
- 副本数（这将随 [自动缩放](../concepts/langgraph_data_plane.md#autoscaling) 增加）。
- [Postgres](../concepts/langgraph_data_plane.md#postgres) CPU、内存使用情况和磁盘使用情况。
- [LangGraph Server 队列](../concepts/langgraph_server.md#persistence-and-task-queue) 挂起/活动运行计数。
- [LangGraph Server API](../concepts/langgraph_server.md) 成功响应计数、错误响应计数和延迟。

这些指标在控制平面 UI 中显示为图表。

### LangSmith 集成 (LangSmith Integration)

会自动为每个部署创建一个 [LangSmith](https://docs.smith.langchain.com/) 跟踪项目和 LangSmith API 密钥。部署使用 API 密钥自动将跟踪发送到 LangSmith。

- 跟踪项目与部署具有相同的名称。
- API 密钥的描述为 `LangGraph Platform: <deployment_name>`。
- API 密钥永远不会泄露，也无法手动删除。
- 创建部署时，不需要指定 `LANGCHAIN_TRACING` 和 `LANGSMITH_API_KEY`/`LANGCHAIN_API_KEY` 环境变量；它们由控制平面自动设置。

删除部署时，不会删除跟踪和跟踪项目。但是，当删除部署时，API 将被删除。
