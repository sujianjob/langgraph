# 添加自定义认证 (Add custom authentication)

!!! tip "前提条件"

    本指南假设您熟悉以下概念：
    
      *  [**认证与访问控制**](../../concepts/auth.md)
      *  [**LangGraph Platform**](../../concepts/langgraph_platform.md)
    
    有关更具指导性的演练，请参阅 [**设置自定义认证**](../../tutorials/auth/getting_started.md) 教程。

???+ note "按部署类型支持"

    **托管 LangGraph Platform** 的所有部署以及 **Enterprise** 自托管计划均支持自定义认证。

本指南展示了如何向您的 LangGraph Platform 应用程序添加自定义认证。本指南适用于 LangGraph Platform 和自托管部署。它不适用于在您自己的自定义服务器中隔离使用 LangGraph 开源库的情况。

!!! note

    **托管 LangGraph Platform** 的所有部署以及 **Enterprise** 自托管计划均支持自定义认证。

## 向您的部署添加自定义认证 (Add custom authentication to your deployment)

要在您的部署中利用自定义认证并访问用户级元数据，请设置自定义认证，以便通过自定义认证处理程序自动填充 `config["configurable"]["langgraph_auth_user"]` 对象。然后，您可以在图中使用 `langgraph_auth_user` 键访问此对象，以 [允许代理代表用户执行经过身份验证的操作](#enable-agent-authentication)。

:::python

1.  实现认证：

    !!! note

        如果没有自定义 `@auth.authenticate` 处理程序，LangGraph 仅看到 API 密钥所有者（通常是开发人员），因此请求不会限定于单个最终用户。要传播自定义令牌，您必须实现自己的处理程序。

    ```python
    from langgraph_sdk import Auth
    import requests

    auth = Auth()

    def is_valid_key(api_key: str) -> bool:
        is_valid = # your API key validation logic
        return is_valid

    @auth.authenticate # (1)!
    async def authenticate(headers: dict) -> Auth.types.MinimalUserDict:
        api_key = headers.get("x-api-key")
        if not api_key or not is_valid_key(api_key):
            raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid API key")

        # Fetch user-specific tokens from your secret store
        user_tokens = await fetch_user_tokens(api_key)

        return { # (2)!
            "identity": api_key,  #  fetch user ID from LangSmith
            "github_token" : user_tokens.github_token
            "jira_token" : user_tokens.jira_token
            # ... custom fields/secrets here
        }
    ```

    1. 此处理程序接收请求（标头等），验证用户，并返回至少包含 identity 字段的字典。
    2. 您可以添加所需的任何自定义字段（例如 OAuth 令牌、角色、组织 ID 等）。

2.  在您的 `langgraph.json` 中，添加认证文件的路径：

    ```json hl_lines="7-9"
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./agent.py:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.py:my_auth"
      }
    }
    ```

3.  在服务器中设置认证后，请求必须根据您选择的方案包含所需的授权信息。假设您使用的是 JWT 令牌认证，您可以使用以下任何方法访问您的部署：

    === "Python Client"

        ```python
        from langgraph_sdk import get_client

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        client = get_client(
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await client.threads.search()
        ```

    === "Python RemoteGraph"

        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```
        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## 启用代理认证 (Enable agent authentication)

[认证](#add-custom-authentication-to-your-deployment) 后，平台会创建一个特殊的配置对象 (`config`) 并传递给 LangGraph Platform 部署。此对象包含当前用户的信息，包括您从 `@auth.authenticate` 处理程序返回的任何自定义字段。

要允许代理代表用户执行经过身份验证的操作，请在图中使用 `langgraph_auth_user` 键访问此对象：

```python
def my_node(state, config):
    user_config = config["configurable"].get("langgraph_auth_user")
    # token was resolved during the @auth.authenticate function
    token = user_config.get("github_token","")
    ...
```

!!! note

    从安全的密钥存储库获取用户凭据。不建议将密钥存储在图状态中。

### 授权 Studio 用户 (Authorizing a Studio user)

默认情况下，如果您对资源添加自定义授权，这也将应用于从 Studio 进行的交互。如果您愿意，可以通过检查 [is_studio_user()](../../reference/functions/sdk_auth.isStudioUser.html) 来以不同方式处理已登录的 Studio 用户。

!!! note
    `is_studio_user` 是在 langgraph-sdk 0.1.73 版本中添加的。如果您使用的是旧版本，仍然可以检查 `isinstance(ctx.user, StudioUser)`。

```python
from langgraph_sdk.auth import is_studio_user, Auth
auth = Auth()

# ... Setup authenticate, etc.

@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,
    value: dict  # The payload being sent to this access method
) -> dict:  # Returns a filter dict that restricts access to resources
    if is_studio_user(ctx.user):
        return {}

    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    return filters
```

仅当您希望允许开发人员访问部署在托管 LangGraph Platform SaaS 上的图时，才应使用此功能。

:::

:::js

1.  实现认证：

    !!! note

        如果没有自定义 `authenticate` 处理程序，LangGraph 仅看到 API 密钥所有者（通常是开发人员），因此请求不会限定于单个最终用户。要传播自定义令牌，您必须实现自己的处理程序。

    ```typescript
    import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

    const auth = new Auth()
      .authenticate(async (request) => {
        const authorization = request.headers.get("Authorization");
        const token = authorization?.split(" ")[1]; // "Bearer <token>"
        if (!token) {
          throw new HTTPException(401, "No token provided");
        }
        try {
          const user = await verifyToken(token);
          return user;
        } catch (error) {
          throw new HTTPException(401, "Invalid token");
        }
      })
      // Add authorization rules to actually control access to resources
      .on("*", async ({ user, value }) => {
        const filters = { owner: user.identity };
        const metadata = value.metadata ?? {};
        metadata.update(filters);
        return filters;
      })
      // Assumes you organize information in store like (user_id, resource_type, resource_id)
      .on("store", async ({ user, value }) => {
        const namespace = value.namespace;
        if (namespace[0] !== user.identity) {
          throw new HTTPException(403, "Not authorized");
        }
      });
    ```

    1. 此处理程序接收请求（标头等），验证用户，并返回一个至少包含 identity 字段的对象。
    2. 您可以添加所需的任何自定义字段（例如 OAuth 令牌、角色、组织 ID 等）。

2.  在您的 `langgraph.json` 中，添加认证文件的路径：

    ```json hl_lines="7-9"
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./agent.ts:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.ts:my_auth"
      }
    }
    ```

3.  在服务器中设置认证后，请求必须根据您选择的方案包含所需的授权信息。假设您使用的是 JWT 令牌认证，您可以使用以下任何方法访问您的部署：

    === "SDK Client"

        ```javascript
        import { Client } from "@langchain/langgraph-sdk";

        const my_token = "your-token"; // In practice, you would generate a signed token with your auth provider
        const client = new Client({
          apiUrl: "http://localhost:2024",
          defaultHeaders: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await client.threads.search();
        ```

    === "RemoteGraph"

        ```javascript
        import { RemoteGraph } from "@langchain/langgraph/remote";

        const my_token = "your-token"; // In practice, you would generate a signed token with your auth provider
        const remoteGraph = new RemoteGraph({
        graphId: "agent",
          url: "http://localhost:2024",
          headers: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await remoteGraph.invoke(...);
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## 启用代理认证 (Enable agent authentication)

[认证](#add-custom-authentication-to-your-deployment) 后，平台会创建一个特殊的配置对象 (`config`) 并传递给 LangGraph Platform 部署。此对象包含当前用户的信息，包括您从 `authenticate` 处理程序返回的任何自定义字段。

要允许代理代表用户执行经过身份验证的操作，请在图中使用 `langgraph_auth_user` 键访问此对象：

```ts
async function myNode(state, config) {
  const userConfig = config["configurable"]["langgraph_auth_user"];
  // token was resolved during the authenticate function
  const token = userConfig["github_token"];
  ...
}
```

!!! note

    从安全的密钥存储库获取用户凭据。不建议将密钥存储在图状态中。

:::

## 了解更多 (Learn more)

- [认证与访问控制](../../concepts/auth.md)
- [LangGraph Platform](../../concepts/langgraph_platform.md)
- [设置自定义认证教程](../../tutorials/auth/getting_started.md)
