# 基于数据集运行实验 (Run experiments over a dataset)

LangGraph Studio 支持评估，允许您在预定义的 LangSmith 数据集上运行您的助手。这使您能够了解您的应用程序在各种输入下的表现，将结果与参考输出进行比较，并使用 [评估器 (evaluators)](../../../agents/evals.md) 对结果进行评分。

本指南向您展示如何从 Studio 端到端地运行实验。

---

## 先决条件 (Prerequisites)

在运行实验之前，请确保您具备以下条件：

1.  **LangSmith 数据集**: 您的数据集应包含您要测试的输入，以及用于比较的可选参考输出。

    - 输入的模式必须与助手所需的输入模式匹配。有关模式的更多信息，请参阅 [此处](../../../concepts/low_level.md#schema)。
    - 有关创建数据集的更多信息，请参阅 [如何管理数据集](https://docs.smith.langchain.com/evaluation/how_to_guides/manage_datasets_in_application#set-up-your-dataset)。

2.  **(可选) 评估器**: 您可以将评估器（例如，LLM-as-a-Judge、启发式方法或自定义函数）附加到 LangSmith 中的数据集。这些评估器将在图处理完所有输入后自动运行。

    - 要了解更多信息，请阅读 [评估概念](https://docs.smith.langchain.com/evaluation/concepts#evaluators)。

3.  **正在运行的应用程序**: 实验可以针对以下对象运行：
    - 部署在 [LangGraph Platform](../../quick_start.md) 上的应用程序。
    - 通过 [langgraph-cli](../../../tutorials/langgraph-platform/local-server.md) 启动的本地运行的应用程序。

---

## 分步指南 (Step-by-step guide)

### 1. 启动实验

单击 Studio 页面右上角的 **Run experiment**（运行实验）按钮。

### 2. 选择您的数据集

在出现的模态框中，选择要用于实验的数据集（或特定数据集拆分），然后单击 **Start**（开始）。

### 3. 监控进度

数据集中的所有输入现在将针对当前活动的助手运行。通过右上角的徽章监控实验进度。

您可以在实验在后台运行时继续在 Studio 中工作。随时单击箭头图标按钮以导航到 LangSmith 并查看详细的实验结果。

---

## 故障排除 (Troubleshooting)

### "Run experiment" 按钮被禁用

如果 "Run experiment" 按钮被禁用，请检查以下内容：

- **已部署的应用程序**: 如果您的应用程序部署在 LangGraph Platform 上，您可能需要创建一个新版本以启用此功能。
- **本地开发服务器**: 如果您在本地运行应用程序，请确保已升级到最新版本的 `langgraph-cli` (`pip install -U langgraph-cli`)。此外，请确保通过在项目的 `.env` 文件中设置 `LANGSMITH_API_KEY` 来启用跟踪。

### 缺少评估器结果

运行实验时，任何附加的评估器都会被安排在队列中执行。如果您没有立即看到结果，这可能意味着它们仍在等待中。
