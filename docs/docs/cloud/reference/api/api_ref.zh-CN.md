# LangGraph Server API 参考 (LangGraph Server API Reference)

LangGraph Server API 参考在每个部署的 `/docs` 端点（例如 `http://localhost:8124/docs`）中可用。

单击 <a href="/langgraph/cloud/reference/api/api_ref.html" target="_blank">此处</a> 查看 API 参考。

## 身份验证 (Authentication)

对于 LangGraph Platform 的部署，需要进行身份验证。在对 LangGraph Server 的每个请求中传递 `X-Api-Key` 标头。标头的值应设置为部署 LangGraph Server 的组织的有效 LangSmith API 密钥。

示例 `curl` 命令：
```shell
curl --request POST \
  --url http://localhost:8124/assistants/search \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "metadata": {},
  "limit": 10,
  "offset": 0
}'
```
