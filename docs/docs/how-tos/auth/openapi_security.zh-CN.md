# 如何将 LangGraph API 添加到 OpenAPI 安全白名单 (How to add LangGraph API to OpenAPI security allowlist)

LangGraph 随带一个预构建的 API 服务器。当您 [部署 LangGraph 应用程序](../../concepts/langgraph_platform.md) 时，我们提供了一个 OpenAPI 规范 (URL 为 `/openapi.json`) 和包含所有 API 路由的 Swagger UI (URL 为 `/docs`)。

要为这些端点配置安全方案，您可以修改应用程序中的 `langgraph.json` 文件。

默认情况下，`langgraph.json` 中的 `auth` 部分可能包含如下配置：

```json title="langgraph.json"
{
  "auth": {
    "path": "src.security:auth"
  }
}
```

其中 `src.security:auth` 是指向 [`langgraph_sdk.Auth`](https://langchain-ai.github.io/langgraph/reference/sdk/#langgraph_sdk.auth.Auth) 实例的路径，该实例包含针对通过 API 服务器发送的请求的认证和授权规则。

但是，您可能不希望将安全性应用于诸如 `/docs` 和 `/openapi.json` 之类的端点。为了实现这一点，您可以在 `langgraph.json` 中配置 `openapi` 部分。

```json title="langgraph.json"
{
  "auth": {
      "path": "src.security:auth",
      // highlight-next-line
      "openapi": {
          // highlight-next-line
          "security": [
            // highlight-next-line
            {
               // highlight-next-line
                "AuthBearer": []
                // highlight-next-line
            }
            // highlight-next-line
        ]
      }
  }
}
```

安全块中指定的键必须与 `auth` 模块中的安全方案名称匹配。`auth` 模块可以定义如下：

=== "Python"

    ```python title="src/security.py"
    from langgraph_sdk import Auth

    auth = Auth()

    # The name of the security scheme here must match the key in langgraph.json
    @auth.security_scheme("AuthBearer")
    def auth_scheme():
        return {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT"
        }
    ```

=== "Javascript"

    ```javascript title="src/security.ts"
    import { Auth } from "@langchain/langgraph-sdk/auth";

    // The name of the security scheme here must match the key in langgraph.json
    export const auth = new Auth().useSecurityScheme("AuthBearer", {
      type: "http",
      scheme: "bearer",
      bearerFormat: "JWT",
    });
    ```

此配置将确保 `/docs` 和 `/openapi.json` 端点对公众可访问，并且在请求这些端点时，安全方案包含在 OpenAPI 规范中。
