# 工具 (Tools)

许多 AI 应用程序通过自然语言与用户交互。然而，某些用例要求模型直接与外部系统（如 API、数据库或文件系统）交互，并使用结构化输入。在这些场景中，[工具调用](../how-tos/tool-calling.md) 使模型能够生成符合指定输入模式的请求。

:::python
**工具** 封装了一个可调用函数及其输入模式。这些可以传递给兼容的 [聊天模型](https://python.langchain.com/docs/concepts/chat_models)，允许模型决定是否调用工具以及使用什么参数。
:::

:::js
**工具** 封装了一个可调用函数及其输入模式。这些可以传递给兼容的 [聊天模型](https://js.langchain.com/docs/concepts/chat_models)，允许模型决定是否调用工具以及使用什么参数。
:::

## 工具调用

![模型调用工具的图示](./img/tool_call.png)

工具调用通常是 **有条件** 的。根据用户输入和可用工具，模型可能会选择发出工具调用请求。此请求在 `AIMessage` 对象中返回，其中包含指定工具名称和输入参数的 `tool_calls` 字段：

:::python

```python
llm_with_tools.invoke("What is 2 multiplied by 3?")
# -> AIMessage(tool_calls=[{'name': 'multiply', 'args': {'a': 2, 'b': 3}, ...}])
```

```
AIMessage(
  tool_calls=[
    ToolCall(name="multiply", args={"a": 2, "b": 3}),
    ...
  ]
)
```

:::

:::js

```typescript
await llmWithTools.invoke("What is 2 multiplied by 3?");
```

```
AIMessage {
  tool_calls: [
    ToolCall {
      name: "multiply",
      args: { a: 2, b: 3 },
      ...
    },
    ...
  ]
}
```

:::

如果输入与任何工具无关，模型仅返回自然语言消息：

:::python

```python
llm_with_tools.invoke("Hello world!")  # -> AIMessage(content="Hello!")
```

:::

:::js

```typescript
await llmWithTools.invoke("Hello world!"); // { content: "Hello!" }
```

:::

重要的是，模型并不执行工具——它只是生成请求。一个单独的执行器（如运行时或代理）负责处理工具调用并返回结果。

有关更多详细信息，请参阅 [工具调用指南](../how-tos/tool-calling.md)。

## 预构建工具

LangChain 提供了针对常见外部系统（包括 API、数据库、文件系统和 Web 数据）的预构建工具集成。

:::python
浏览 [集成目录](https://python.langchain.com/docs/integrations/tools/) 以获取可用工具。
:::

:::js
浏览 [集成目录](https://js.langchain.com/docs/integrations/tools/) 以获取可用工具。
:::

常见类别：

- **搜索**: Bing, SerpAPI, Tavily
- **代码执行**: Python REPL, Node.js REPL
- **数据库**: SQL, MongoDB, Redis
- **Web 数据**: 抓取和浏览
- **API**: OpenWeatherMap, NewsAPI 等

## 自定义工具

:::python
您可以使用 `@tool` 装饰器或普通 Python 函数定义自定义工具。例如：

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

:::

:::js
您可以使用 `tool` 函数定义自定义工具。例如：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);
```

:::

有关更多详细信息，请参阅 [工具调用指南](../how-tos/tool-calling.md)。

## 工具执行

虽然模型决定何时调用工具，但工具调用的执行必须由运行时组件处理。

LangGraph 为此提供了预构建组件：

:::python

- @[`ToolNode`][ToolNode]: 一个执行工具的预构建节点。
- @[`create_react_agent`][create_react_agent]: 构建一个自动管理工具调用的完整代理。
:::

:::js

- @[ToolNode]: 一个执行工具的预构建节点。
- @[`createReactAgent`][create_react_agent]: 构建一个自动管理工具调用的完整代理。
:::
