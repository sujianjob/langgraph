# 如何为 LangGraph 应用程序添加 TTL (How to add TTLs to your LangGraph application)

!!! tip "先决条件 (Prerequisites)"

    本指南假设您熟悉 [LangGraph Platform](../../concepts/langgraph_platform.md)、[持久性 (Persistence)](../../concepts/persistence.md) 和 [跨线程持久性 (Cross-thread persistence)](../../concepts/persistence.md#memory-store) 的概念。

???+ note "仅限 LangGraph 平台 (LangGraph platform only)"
    
    TTL 仅支持 LangGraph 平台部署。本指南不适用于 LangGraph OSS。

LangGraph 平台持久保存 [检查点 (checkpoints)](../../concepts/persistence.md#checkpoints)（线程状态）和 [跨线程记忆 (cross-thread memories)](../../concepts/persistence.md#memory-store)（存储项）。在 `langgraph.json` 中配置生存时间 (TTL) 策略，以自动管理此数据的生命周期，防止无限期累积。

## 配置检查点 TTL (Configuring Checkpoint TTL)

检查点捕获对话线程的状态。设置 TTL 可确保自动删除旧的检查点和线程。

向您的 `langgraph.json` 文件添加 `checkpointer.ttl` 配置：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200 
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200 
    }
  }
}
```
:::

*   `strategy`：指定过期时采取的操作。目前，仅支持 `"delete"`，它会在过期时删除线程中的所有检查点。
*   `sweep_interval_minutes`：定义系统检查过期检查点的频率（以分钟为单位）。
*   `default_ttl`：设置检查点的默认寿命（以分钟为单位）（例如，43200 分钟 = 30 天）。

## 配置存储项 TTL (Configuring Store Item TTL)

存储项允许跨线程数据持久性。为存储项配置 TTL 有助于通过删除陈旧数据来管理内存。

向您的 `langgraph.json` 文件添加 `store.ttl` 配置：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

*   `refresh_on_read`：（可选，默认为 `true`）如果为 `true`，通过 `get` 或 `search` 访问项目会重置其过期计时器。如果为 `false`，TTL 仅在 `put` 时刷新。
*   `sweep_interval_minutes`：（可选）定义系统检查过期项目的频率（以分钟为单位）。如果省略，则不进行清理。
*   `default_ttl`：（可选）设置存储项的默认寿命（以分钟为单位）（例如，10080 分钟 = 7 天）。如果省略，项目默认不会过期。

## 组合 TTL 配置 (Combining TTL Configurations)

您可以在同一个 `langgraph.json` 文件中为检查点和存储项配置 TTL，以便为每种数据类型设置不同的策略。示例如下：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

## 运行时覆盖 (Runtime Overrides)

`langgraph.json` 中的默认 `store.ttl` 设置可以在运行时通过在 SDK 方法调用（如 `get`、`put` 和 `search`）中提供特定的 TTL 值来覆盖。

## 部署流程 (Deployment Process)

在 `langgraph.json` 中配置 TTL 后，部署或重新启动您的 LangGraph 应用程序以使更改生效。使用 `langgraph dev` 进行本地开发，或使用 `langgraph up` 进行 Docker 部署。

有关其他可配置选项的更多详细信息，请参阅 @[langgraph.json CLI 参考][langgraph.json]。
