# 可配置标头 (Configurable Headers)

LangGraph 允许运行时配置以动态修改代理行为和权限。使用 [LangGraph Platform](../quick_start.md) 时，您可以在请求体 (`config`) 或特定请求标头中传递此配置。这使得能够根据用户身份或其他请求数据进行调整。

为了保护隐私，请通过 `langgraph.json` 文件中的 `http.configurable_headers` 部分控制哪些标头传递给运行时配置。

以下是自定义包含和排除的标头的方法：

```json
{
  "http": {
    "configurable_headers": {
      "include": ["x-user-id", "x-organization-id", "my-prefix-*"],
      "exclude": ["authorization", "x-api-key"]
    }
  }
}
```

`include` 和 `exclude` 列表接受确切的标头名称或使用 `*` 匹配任意数量字符的模式。为了您的安全，不支持其他正则表达式模式。

## 在图中使用 (Using within your graph)

您可以使用任何节点的 `config` 参数在图中访问包含的标头。

```python
def my_node(state, config):
  organization_id = config["configurable"].get("x-organization-id")
  ...
```

或者通过从上下文获取（在工具或和其他嵌套函数中有用）。

```python
from langgraph.config import get_config

def search_everything(query: str):
  organization_id = get_config()["configurable"].get("x-organization-id")
  ...
```

您甚至可以使用它来动态编译图。

```python
# my_graph.py.
import contextlib

@contextlib.asynccontextmanager
async def generate_agent(config):
  organization_id = config["configurable"].get("x-organization-id")
  if organization_id == "org1":
    graph = ...
    yield graph
  else:
    graph = ...
    yield graph

```

```json
{
  "graphs": {"agent": "my_grph.py:generate_agent"}
}
```

### 选择退出可配置标头 (Opt-out of configurable headers)

如果您想选择退出可配置标头，只需在 `exclude` 列表中设置通配符模式即可：

```json
{
  "http": {
    "configurable_headers": {
      "exclude": ["*"]
    }
  }
}
```

这将排除所有标头被添加到运行的配置中。

请注意，排除优先于包含。
