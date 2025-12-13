# 将节点添加到数据集 (Add node to dataset)

本指南展示了如何将线程日志中节点的示例添加到 [LangSmith 数据集](https://docs.smith.langchain.com/evaluation/how_to_guides#dataset-management)。这对于评估代理的单个步骤非常有用。

1. 选择一个线程。
2. 单击 `Add to Dataset`（添加到数据集）按钮。
3. 选择要将其输入/输出添加到数据集的节点。
4. 对于每个选定的节点，选择要在其中创建示例的目标数据集。默认情况下，将选择特定助手和节点的数据集。如果此数据集尚不存在，则会创建它。
5. 在将示例添加到数据集之前，根据需要编辑示例的输入/输出。
6. 选择页面底部的 "Add to dataset"（添加到数据集），将所有选定的节点添加到各自的数据集。

有关如何评估中间步骤的更多详细信息，请参阅 [评估中间步骤](https://docs.smith.langchain.com/evaluation/how_to_guides/langgraph#evaluating-intermediate-steps)。
