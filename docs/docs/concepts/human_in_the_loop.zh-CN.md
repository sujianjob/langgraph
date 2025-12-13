---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - overview
hide:
  - tags
---

# 人机回圈 (Human-in-the-loop)

要审查、编辑和批准代理或工作流中的工具调用，请 [使用 LangGraph 的人机回圈功能](../how-tos/human_in_the_loop/add-human-in-the-loop.md) 以在工作流的任何点启用人工干预。这在大型语言模型 (LLM) 驱动的应用程序中特别有用，因为模型输出可能需要验证、更正或额外的上下文。

<figure markdown="1">
![image](../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
</figure>

!!! tip

    有关如何使用人机回圈的信息，请参阅 [启用人工干预](../how-tos/human_in_the_loop/add-human-in-the-loop.md) 和 [使用 Server API 的人机回圈](../cloud/how-tos/add-human-in-the-loop.md)。

## 关键功能 (Key capabilities)

* **持久执行状态**: 中断使用 LangGraph 的 [持久化](./persistence.md) 层（保存图状态）无限期暂停图执行，直到您恢复。这是可能的，因为 LangGraph 在每个步骤后都会对图状态进行检查点检查，这允许系统持久保存执行上下文并在以后恢复工作流，从中断的地方继续。这支持没有时间限制的异步人工审查或输入。

    有两种暂停图的方法：

    - [动态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt): 使用 `interrupt` 根据图的当前状态从特定节点内暂停图。
    - [静态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#debug-with-interrupts): 使用 `interrupt_before` 和 `interrupt_after` 在预定义点（节点执行之前或之后）暂停图。

    <figure markdown="1">
    ![image](./img/breakpoints.png){: style="max-height:400px"}
    <figcaption>一个由 3 个顺序步骤组成的示例图，在 step_3 之前有一个断点。</figcaption> </figure>

* **灵活的集成点**: 人机回圈逻辑可以在工作流的任何点引入。这允许有针对性的人工参与，例如批准 API 调用、纠正输出或指导对话。

## 模式 (Patterns)

您可以使用 `interrupt` 和 `Command` 实现四种典型的设计模式：

- [批准或拒绝](../how-tos/human_in_the_loop/add-human-in-the-loop.md#approve-or-reject): 在关键步骤（例如 API 调用）之前暂停图，以审查和批准操作。如果操作被拒绝，您可以阻止图执行该步骤，并可能采取替代操作。这种模式通常涉及根据人类的输入路由图。
- [编辑图状态](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-and-edit-state): 暂停图以审查和编辑图状态。这对纠正错误或使用附加信息更新状态很有用。这种模式通常涉及使用人类的输入更新状态。
- [审查工具调用](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-tool-calls): 在工具执行之前暂停图，以审查和编辑 LLM 请求的工具调用。
- [验证人工输入](../how-tos/human_in_the_loop/add-human-in-the-loop.md#validate-human-input): 暂停图以在继续下一步之前验证人工输入。
