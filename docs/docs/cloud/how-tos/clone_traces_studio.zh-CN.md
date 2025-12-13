# 调试 LangSmith 跟踪 (Debug LangSmith traces)

本指南介绍如何在 LangGraph Studio 中打开 LangSmith 跟踪以进行交互式调查和调试。

## 打开已部署的线程 (Open deployed threads)

1. 打开 LangSmith 跟踪，选择根运行。
2. 单击 "Run in Studio"（在 Studio 中运行）。

这将打开连接到相关 LangGraph Platform 部署并选择了跟踪的父线程的 LangGraph Studio。

## 使用远程跟踪测试本地代理 (Testing local agents with remote traces)

本节介绍如何针对来自 LangSmith 的远程跟踪测试本地代理。这使您能够使用生产跟踪作为本地测试的输入，从而允许您在开发环境中调试和验证代理修改。

### 要求 (Requirements)

- LangSmith 跟踪的线程
- 本地运行的代理。请参阅 [此处](../how-tos/studio/quick_start.md#local-development-server) 获取设置说明。

!!! info "Local agent requirements"

    - langgraph>=0.3.18
    - langgraph-api>=0.0.32
    - 包含与远程跟踪中相同的一组节点

### 克隆线程 (Cloning Thread)

1. 打开 LangSmith 跟踪，选择根运行。
2. 单击 "Run in Studio" 旁边的下拉菜单。
3. 输入本地代理的 URL。
4. 选择 "Clone thread locally"（本地克隆线程）。
5. 如果存在多个图，请选择目标图。

将在您的本地代理中创建一个新线程，线程历史记录从远程线程推断和复制，并且您将导航到本地运行的应用程序的 LangGraph Studio。
