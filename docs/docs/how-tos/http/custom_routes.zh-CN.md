# 如何添加自定义路由 (How to add custom routes)

在将代理部署到 LangGraph 平台时，您的服务器会自动公开用于创建运行和线程、与长期记忆存储交互、管理可配置助手以及其他核心功能的路由（[查看所有默认 API 端点](../../cloud/reference/api/api_ref.md)）。

您可以通过提供自己的 [`Starlette`](https://www.starlette.io/applications/) 应用程序（包括 [`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/) 和其他兼容的应用程序）来添加自定义路由。您可以通过在 `langgraph.json` 配置文件中提供应用程序的路径来让 LangGraph 平台感知到这一点。

定义自定义应用程序对象允许您添加任何您想要的路由，因此您可以做任何事情，从添加 `/login` 端点到编写整个全栈 Web 应用程序，所有这些都部署在单个 LangGraph 服务器中。

下面是一个使用 FastAPI 的示例。

## 创建应用 (Create app)

从 **现有** 的 LangGraph 平台应用程序开始，将以下自定义路由代码添加到您的 webapp 文件中。如果您是从头开始，可以使用 CLI 从模板创建一个新应用程序。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

拥有 LangGraph 项目后，添加以下应用程序代码：

```python
# ./src/agent/webapp.py
from fastapi import FastAPI

# highlight-next-line
app = FastAPI()


@app.get("/hello")
def read_root():
    return {"Hello": "World"}

```

## 配置 `langgraph.json` (Configure `langgraph.json`)

将以下内容添加到您的 `langgraph.json` 配置文件中。确保路径指向您在上面创建的 `webapp.py` 文件中的 FastAPI 应用程序实例 `app`。

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // Other configuration options like auth, store, etc.
}
```

## 启动服务器 (Start server)

在本地测试服务器：

```bash
langgraph dev --no-browser
```

如果您在浏览器中导航到 `localhost:2024/hello`（`2024` 是默认开发端口），您应该会看到 `/hello` 端点返回 `{"Hello": "World"}`。

!!! note "覆盖默认端点 (Shadowing default endpoints)"

    您在应用程序中创建的路由优先于系统默认路由，这意味着您可以覆盖和重新定义任何默认端点的行为。

## 部署 (Deploying)

您可以将此应用程序原样部署到 LangGraph 平台或您的自托管平台。

## 后续步骤 (Next steps)

既然您已将自定义路由添加到部署中，您可以使用相同的技术进一步自定义服务器的行为，例如定义自定义 [自定义中间件](./custom_middleware.md) 和 [自定义生命周期事件](./custom_lifespan.md)。
