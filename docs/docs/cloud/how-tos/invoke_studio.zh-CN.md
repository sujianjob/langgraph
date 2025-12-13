# 运行应用程序 (Run application)

!!! info "Prerequisites"
    - [运行代理](../../agents/run_agents.md#running-agents)

本指南展示了如何向您的应用程序提交 [运行](../../concepts/assistants.md#execution)。

## 图模式 (Graph mode)

### 指定输入 (Specify input)
首先在页面左侧图表界面下方的 "Input"（输入）部分定义图形的输入。

Studio 将尝试根据图定义的 [状态模式](../../concepts/low_level.md/#schema) 为您的输入渲染表单。要禁用此功能，请单击 "View Raw" 按钮，这将为您呈现一个 JSON 编辑器。

单击 "Input" 部分顶部的向上/向下箭头可切换并使用以前提交的输入。

### 运行设置 (Run settings)

#### 助手 (Assistant)

要指定用于运行的 [助手](../../concepts/assistants.md)，请单击左下角的设置按钮。如果当前选择了助手，该按钮还将列出助手名称。如果没有选择助手，它将显示 "Manage Assistants"（管理助手）。

选择要运行的助手，然后单击模式顶部的 "Active" 切换开关以将其激活。有关管理助手的更多信息，[请参阅此处](./studio/manage_assistants.md)。

#### 流式传输 (Streaming)
单击 "Submit"（提交）旁边的下拉菜单，然后单击切换开关以启用/禁用流式传输。

#### 断点 (Breakpoints)
要使用断点运行图，请单击 "Interrupt"（中断）按钮。选择一个节点，以及是否在该节点执行之前和/或之后暂停。单击线程日志中的 "Continue"（继续）以恢复执行。


有关断点的更多信息，请参阅 [此处](../../concepts/human_in_the_loop.md)。

### 提交运行 (Submit run)

要使用指定的输入和运行设置提交运行，请单击 "Submit"（提交）按钮。这将向现有的选定 [线程](../../concepts/persistence.md#threads) 添加一个 [运行](../../concepts/assistants.md#execution)。如果当前未选择线程，将创建一个新线程。

要取消正在进行的运行，请单击 "Cancel"（取消）按钮。


## 聊天模式 (Chat mode)
在对话面板的底部指定聊天应用程序的输入。单击 "Send message"（发送消息）按钮将输入作为 Human 消息提交，并流式传回响应。

要取消正在进行的运行，请单击 "Cancel"（取消）按钮。单击 "Show tool calls"（显示工具调用）切换开关以在对话中隐藏/显示工具调用。

## 了解更多 (Learn more)

要从现有线程中的特定检查点运行应用程序，请参阅 [本指南](./threads_studio.md#edit-thread-history)。
