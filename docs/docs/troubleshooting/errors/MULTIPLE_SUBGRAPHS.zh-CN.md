# 多个子图 (MULTIPLE_SUBGRAPHS)

您在一个启用了检查点 (checkpointing) 的单个 LangGraph 节点中多次调用子图。

由于对子图检查点命名空间工作方式的内部限制，目前不允许这样做。

## 故障排除 (Troubleshooting)

以下方法可能有助于解决此错误：

:::python

- 如果您不需要从子图中中断/恢复，请在编译时传递 `checkpointer=False`，如下所示：`.compile(checkpointer=False)`
  :::

:::js

- 如果您不需要从子图中中断/恢复，请在编译时传递 `checkpointer: false`，如下所示：`.compile({ checkpointer: false })`
  :::

- 不要在一个节点中强制多次调用图，而是使用 [`Send`](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) API。
