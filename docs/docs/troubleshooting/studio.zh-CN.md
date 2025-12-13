# LangGraph Studio 故障排除 (LangGraph Studio Troubleshooting)

## :fontawesome-brands-safari:{ .safari } Safari 连接问题

Safari 阻止本地主机上的纯 HTTP 流量。使用 `langgraph dev` 运行 Studio 时，您可能会看到 "Failed to load assistants"（加载助手失败）错误。

### 解决方案 1：使用 Cloudflare Tunnel

:::python

```shell
pip install -U langgraph-cli>=0.2.6
langgraph dev --tunnel
```

:::

:::js

```shell
npx @langchain/langgraph-cli dev
```

:::

该命令以这种格式输出 URL：

```shell
https://smith.langchain.com/studio/?baseUrl=https://hamilton-praise-heart-costumes.trycloudflare.com
```

在 Safari 中使用此 URL 加载 Studio。在这里，`baseUrl` 参数指定您的代理服务器端点。

### 解决方案 2：使用 Chromium 浏览器

Chrome 和其他 Chromium 浏览器允许本地主机上的 HTTP。使用 `langgraph dev` 而无需其他配置。

## :fontawesome-brands-brave:{ .brave } Brave 连接问题

启用 Brave Shields 时，Brave 会阻止本地主机上的纯 HTTP 流量。使用 `langgraph dev` 运行 Studio 时，您可能会看到 "Failed to load assistants"（加载助手失败）错误。

### 解决方案 1：禁用 Brave Shields

使用 URL 栏中的 Brave 图标禁用 LangSmith 的 Brave Shields。

![Brave Shields](./img/brave-shields.png)

### 解决方案 2：使用 Cloudflare Tunnel

:::python

```shell
pip install -U langgraph-cli>=0.2.6
langgraph dev --tunnel
```

:::

:::js

```shell
npx @langchain/langgraph-cli dev
```

:::

该命令以这种格式输出 URL：

```shell
https://smith.langchain.com/studio/?baseUrl=https://hamilton-praise-heart-costumes.trycloudflare.com
```

在 Brave 中使用此 URL 加载 Studio。在这里，`baseUrl` 参数指定您的代理服务器端点。

## 图边缘问题 (Graph Edge Issues)

:::python
未定义的条件边缘可能会在您的图中显示意外的连接。这是因为如果没有适当的定义，LangGraph Studio 会假设条件边缘可以访问所有其他节点。为了解决这个问题，请使用以下方法之一明确定义路由路径：

### 解决方案 1：路径映射 (Path Map)

定义路由器输出和目标节点之间的映射：

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

### 解决方案 2：路由器类型定义 (Python) (Router Type Definition)

使用 Python 的 `Literal` 类型指定可能的路由目的地：

```python
def routing_function(state: GraphState) -> Literal["node_b","node_c"]:
    if state['some_condition'] == True:
        return "node_b"
    else:
        return "node_c"
```

:::

:::js
未定义的条件边缘可能会在您的图中显示意外的连接。这是因为如果没有适当的定义，LangGraph Studio 会假设条件边缘可以访问所有其他节点。
为了解决这个问题，请明确定义路由器输出和目标节点之间的映射：

```typescript
graph.addConditionalEdges("node_a", routingFunction, {
  true: "node_b",
  false: "node_c",
});
```

:::
