# 如何添加自定义生命周期事件 (How to add custom lifespan events)

将代理部署到 LangGraph 平台时，您通常需要在服务器启动时初始化资源（如数据库连接），并确保在服务器关闭时正确关闭这些资源。生命周期事件允许您挂钩到服务器的启动和关闭序列，以处理这些关键的设置和清理任务。

这与 [添加自定义路由](./custom_routes.md) 的工作方式相同。您只需要提供自己的 [`Starlette`](https://www.starlette.io/applications/) 应用程序（包括 [`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/) 和其他兼容的应用程序）。

下面是一个使用 FastAPI 的示例。

???+ note "仅限 Python (Python only)"

    我们目前仅支持 `langgraph-api>=0.0.26` 的 Python 部署中的自定义生命周期事件。

## 创建应用 (Create app)

从 **现有** 的 LangGraph 平台应用程序开始，将以下生命周期代码添加到您的 `webapp.py` 文件中。如果您是从头开始，可以使用 CLI 从模板创建一个新应用程序。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

拥有 LangGraph 项目后，添加以下应用程序代码：

```python
# ./src/agent/webapp.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

@asynccontextmanager
async def lifespan(app: FastAPI):
    # for example...
    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
    # Create reusable session factory
    async_session = sessionmaker(engine, class_=AsyncSession)
    # Store in app state
    app.state.db_session = async_session
    yield
    # Clean up connections
    await engine.dispose()

# highlight-next-line
app = FastAPI(lifespan=lifespan)

# ... can add custom routes if needed.
```

## 配置 `langgraph.json` (Configure `langgraph.json`)

将以下内容添加到您的 `langgraph.json` 配置文件中。确保路径指向您在上面创建的 `webapp.py` 文件。

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

您应该会看到服务器启动时打印的启动消息，以及使用 `Ctrl+C` 停止服务器时打印的清理消息。

## 部署 (Deploying)

您可以将应用程序原样部署到 LangGraph 平台或您的自托管平台。

## 后续步骤 (Next steps)

既然您已将生命周期事件添加到部署中，您可以使用类似的技术添加 [自定义路由](./custom_routes.md) 或 [自定义中间件](./custom_middleware.md) 以进一步自定义服务器的行为。
