# 迭代提示词 (Iterate on prompts)

## 概述 (Overview)

LangGraph Studio 支持两种修改图中的提示词的方法：直接节点编辑和 LangSmith Playground 界面。

## 直接节点编辑 (Direct Node Editing)

Studio 允许您直接从图界面编辑各个节点内使用的提示词。

!!! info "Prerequisites"

    - [Assistants overview](../../concepts/assistants.md)

### 图配置 (Graph Configuration)

定义您的 [配置](https://langchain-ai.github.io/langgraph/how-tos/configuration/)，使用 `langgraph_nodes` 和 `langgraph_type` 键指定提示词字段及其关联节点。

#### 配置参考 (Configuration Reference)

##### `langgraph_nodes`

- **描述**: 指定配置字段与图的哪些节点相关联。
- **值类型**: 字符串数组，其中每个字符串是图中节点的名称。
- **使用上下文**: 包含在 Pydantic 模型的 `json_schema_extra` 字典中，或 dataclasses 的 `metadata["json_schema_extra"]` 字典中。
- **示例**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={"langgraph_nodes": ["call_model", "other_node"]},
  )
  ```

##### `langgraph_type`

- **描述**: 指定配置字段的类型，这决定了它在 UI 中的处理方式。
- **值类型**: 字符串
- **支持的值**:
  - `"prompt"`: 指示该字段包含应在 UI 中特殊处理的提示词文本。
- **使用上下文**: 包含在 Pydantic 模型的 `json_schema_extra` 字典中，或 dataclasses 的 `metadata["json_schema_extra"]` 字典中。
- **示例**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={
          "langgraph_nodes": ["call_model"],
          "langgraph_type": "prompt",
      },
  )
  ```

#### 示例配置 (Example Configuration)

```python
## Using Pydantic
from pydantic import BaseModel, Field
from typing import Annotated, Literal

class Configuration(BaseModel):
    """The configuration for the agent."""

    system_prompt: str = Field(
        default="You are a helpful AI assistant.",
        description="The system prompt to use for the agent's interactions. "
        "This prompt sets the context and behavior for the agent.",
        json_schema_extra={
            "langgraph_nodes": ["call_model"],
            "langgraph_type": "prompt",
        },
    )

    model: Annotated[
        Literal[
            "anthropic/claude-3-7-sonnet-latest",
            "anthropic/claude-3-5-haiku-latest",
            "openai/o1",
            "openai/gpt-4o-mini",
            "openai/o1-mini",
            "openai/o3-mini",
        ],
        {"__template_metadata__": {"kind": "llm"}},
    ] = Field(
        default="openai/gpt-4o-mini",
        description="The name of the language model to use for the agent's main interactions. "
        "Should be in the form: provider/model-name.",
        json_schema_extra={"langgraph_nodes": ["call_model"]},
    )

## Using Dataclasses
from dataclasses import dataclass, field

@dataclass(kw_only=True)
class Configuration:
    """The configuration for the agent."""

    system_prompt: str = field(
        default="You are a helpful AI assistant.",
        metadata={
            "description": "The system prompt to use for the agent's interactions. "
            "This prompt sets the context and behavior for the agent.",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

    model: Annotated[str, {"__template_metadata__": {"kind": "llm"}}] = field(
        default="anthropic/claude-3-5-sonnet-20240620",
        metadata={
            "description": "The name of the language model to use for the agent's main interactions. "
            "Should be in the form: provider/model-name.",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

```

### 在 UI 中编辑提示词 (Editing prompts in UI)

1. 在具有关联配置字段的节点上找到齿轮图标
2. 单击以打开配置模态框
3. 编辑值
4. 保存以更新当前助手版本或创建一个新版本

## LangSmith Playground

[LangSmith Playground](https://docs.smith.langchain.com/prompt_engineering/how_to_guides#playground) 界面允许测试单个 LLM 调用，而无需运行完整的图：

1. 选择一个线程
2. 单击节点上的 "View LLM Runs"。这将列出节点内进行的所有 LLM 调用（如果有）。
3. 选择一个 LLM 运行以在 Playground 中打开
4. 修改提示词并测试不同的模型和工具设置
5. 将更新后的提示词复制回您的图

有关高级 Playground 功能，请单击右上角的展开按钮。
