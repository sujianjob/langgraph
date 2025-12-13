# 如何部署到 Cloud SaaS (How to Deploy to Cloud SaaS)

在部署之前，请查看 [Cloud SaaS 的概念指南](../../concepts/langgraph_cloud.md) 部署选项。

## 先决条件 (Prerequisites)

1. LangGraph Platform 应用程序从 GitHub 存储库部署。配置 LangGraph Platform 应用程序并将其上传到 GitHub 存储库，以便将其部署到 LangGraph Platform。
2. [验证 LangGraph API 是否在本地运行](../../tutorials/langgraph-platform/local-server.md)。如果 API 未成功运行（即 `langgraph dev`），则部署到 LangGraph Platform 也将失败。

## 创建新部署 (Create New Deployment)

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
2. 在右上角，选择 `+ New Deployment` 以创建新部署。
3. 在 `Create New Deployment` 面板中，填写必填字段。
    1. `Deployment details`（部署详情）
        1. 选择 `Import from GitHub` 并按照 GitHub OAuth 工作流程安装并授权 LangChain 的 `hosted-langserve` GitHub 应用程序访问所选存储库。安装完成后，返回 `Create New Deployment` 面板并从下拉菜单中选择要部署的 GitHub 存储库。**注意**：安装 LangChain 的 `hosted-langserve` GitHub 应用程序的 GitHub 用户必须是组织或帐户的 [所有者](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners)。
        1. 指定部署的名称。
        1. 指定所需的 `Git Branch`（Git 分支）。部署链接到一个分支。创建新修订版本时，将部署链接分支的代码。可以稍后在 [部署设置](#deployment-settings) 中更新该分支。
        1. 指定 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 的完整路径，包括文件名。例如，如果文件 `langgraph.json` 位于存储库的根目录中，只需指定 `langgraph.json`。
        1. 选中/取消选中复选框以 `Automatically update deployment on push to branch`（推送到分支时自动更新部署）。如果选中，则在将更改推送到指定的 `Git Branch` 时将自动更新部署。可以稍后在 [部署设置](#deployment-settings) 中启用/禁用此设置。
    1. 选择所需的 `Deployment Type`（部署类型）。
        1. `Development`（开发）部署适用于非生产用例，并配置了最少的资源。
        1. `Production`（生产）部署每秒最多可服务 500 个请求，并配置了具有自动备份功能的高可用性存储。
    1. 确定部署是否应 `Shareable through LangGraph Studio`（通过 LangGraph Studio 共享）。
        1. 如果未选中，则只有使用工作区的有效 LangSmith API 密钥才能访问部署。
        1. 如果选中，任何 LangSmith 用户都可以通过 LangGraph Studio 访问部署。将提供部署的 LangGraph Studio 直接 URL 以与其他 LangSmith 用户共享。
    1. 指定 `Environment Variables`（环境变量）和密钥。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。
        1. API 密钥（例如 `OPENAI_API_KEY`）等敏感值应指定为密钥。
        1. 也可以指定其他非密钥环境变量。
    1. 将自动创建一个与部署同名的新 LangSmith `Tracing Project`（跟踪项目）。
4. 在右上角，选择 `Submit`（提交）。几秒钟后，`Deployment` 视图出现，新部署将排队等待预配。

## 创建新修订版本 (Create New Revision)

[创建新部署](#create-new-deployment) 时，默认情况下会创建一个新修订版本。可以创建后续修订版本以部署新的代码更改。

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
2. 选择要为其创建新修订版本的现有部署。
3. 在 `Deployment` 视图中，在右上角，选择 `+ New Revision`（新修订版本）。
4. 在 `New Revision` 模态框中，填写必填字段。
    1. 指定 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 的完整路径，包括文件名。例如，如果文件 `langgraph.json` 位于存储库的根目录中，只需指定 `langgraph.json`。
    1. 确定部署是否应 `Shareable through LangGraph Studio`（通过 LangGraph Studio 共享）。
        1. 如果未选中，则只有使用工作区的有效 LangSmith API 密钥才能访问部署。
        1. 如果选中，任何 LangSmith 用户都可以通过 LangGraph Studio 访问部署。将提供部署的 LangGraph Studio 直接 URL 以与其他 LangSmith 用户共享。
    1. 指定 `Environment Variables`（环境变量）和密钥。现有的密钥和环境变量已预填充。请参阅 [环境变量参考](../reference/env_var.md) 以配置修订版本的其他变量。
        1. 添加新的密钥或环境变量。
        1. 删除现有的密钥或环境变量。
        1. 更新现有密钥或环境变量的值。
5. 选择 `Submit`（提交）。几秒钟后，`New Revision` 模态框将关闭，新修订版本将排队等待部署。

## 查看构建和服务器日志 (View Build and Server Logs)

构建和服务器日志可用于每个修订版本。

从 `LangGraph Platform` 视图开始...

1. 从 `Revisions` 表中选择所需的修订版本。面板从右侧滑出，默认情况下选中 `Build` 选项卡，该选项卡显示修订版本的构建日志。
2. 在面板中，选择 `Server` 选项卡以查看修订版本的服务器日志。服务器日志仅在部署修订版本后才可用。
3. 在 `Server` 选项卡中，根据需要调整日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 7 days`（过去 7 天）。

## 查看部署指标 (View Deployment Metrics)

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
2. 选择要监控的现有部署。
3. 选择 `Monitoring`（监控）选项卡以查看部署指标。查看 [所有可用指标](../../concepts/langgraph_control_plane.md#monitoring) 的列表。
4. 在 `Monitoring` 选项卡中，根据需要使用日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 15 minutes`（过去 15 分钟）。

## 中断修订版本 (Interrupt Revision)

中断修订版本将停止修订版本的部署。

!!! warning "Undefined Behavior"
    中断的修订版本具有未定义的行为。只有当您需要部署新修订版本并且已经有一个修订版本“卡在”进度中时，这主要才有用。将来，可能会删除此功能。

从 `LangGraph Platform` 视图开始...

1. 从 `Revisions` 表中选择所需修订版本的行右侧的菜单图标（三个点）。
2. 从菜单中选择 `Interrupt`（中断）。
3. 将出现一个模态框。查看确认消息。选择 `Interrupt revision`（中断修订版本）。

## 删除部署 (Delete Deployment)

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
2. 选择所需部署的行右侧的菜单图标（三个点），然后选择 `Delete`（删除）。
3. 将出现 `Confirmation`（确认）模态框。选择 `Delete`（删除）。

## 部署设置 (Deployment Settings)

从 `LangGraph Platform` 视图开始...

1. 在右上角，选择齿轮图标 (`Deployment Settings`)。
2. 将 `Git Branch` 更新为所需的分支。
3. 选中/取消选中复选框以 `Automatically update deployment on push to branch`。
4. 分支创建/删除和标签创建/删除事件不会触发更新。只有推送到现有分支才会触发更新。
5. 连续推送到分支将使后续更新排队。构建完成后，最近的提交将开始构建，其他排队的构建将被跳过。

## 添加或删除 GitHub 存储库 (Add or Remove GitHub Repositories)

安装并授权 LangChain 的 `hosted-langserve` GitHub 应用程序后，可以修改应用程序的存储库访问权限以添加新存储库或删除现有存储库。如果创建了新存储库，可能需要显式添加它。

1. 从 GitHub 个人资料中，导航到 `Settings` > `Applications` > `hosted-langserve` > 单击 `Configure`。
2. 在 `Repository access` 下，选择 `All repositories` 或 `Only select repositories`。如果选择了 `Only select repositories`，则必须显式添加新存储库。
3. 单击 `Save`。
4. 创建新部署时，下拉菜单中的 GitHub 存储库列表将更新以反映存储库访问权限更改。

## 白名单 IP 地址 (Whitelisting IP Addresses)

2025 年 1 月 6 日之后创建的 `LangGraph Platform` 部署的所有流量都将通过 NAT 网关。
此 NAT 网关将拥有多个静态 IP 地址，具体取决于您部署的区域。请参考下表以获取要列入白名单的 IP 地址列表：

| US             | EU              |
|----------------|-----------------|
| 35.197.29.146  | 34.90.213.236   |
| 34.145.102.123 | 34.13.244.114   |
| 34.169.45.153  | 34.32.180.189   |
| 34.82.222.17   | 34.34.69.108    |
| 35.227.171.135 | 34.32.145.240   | 
| 34.169.88.30   | 34.90.157.44    |
| 34.19.93.202   | 34.141.242.180  |
| 34.19.34.50    | 34.32.141.108   |
