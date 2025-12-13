# 如何部署自托管数据平面 (How to Deploy Self-Hosted Data Plane)

在部署之前，请查看 [自托管数据平面的概念指南](../../concepts/langgraph_self_hosted_data_plane.md) 部署选项。

!!! info "Important"
    自托管数据平面部署选项需要 [企业版](../../concepts/plans.md) 计划。

## 先决条件 (Prerequisites)

1. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) [在本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
2. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 构建 Docker 镜像（即 `langgraph build`）并将其推送到您的 Kubernetes 集群或 Amazon ECS 集群可以访问的注册表。

## Kubernetes

### 先决条件 (Prerequisites)
1. 集群上已安装 `KEDA`。

        helm repo add kedacore https://kedacore.github.io/charts
        helm install keda kedacore/keda --namespace keda --create-namespace

2. 集群上已安装有效的 `Ingress` 控制器。
3. 您的集群中有用于多次部署的闲置空间。建议使用 `Cluster-Autoscaler` 自动配置新节点。
4. 您需要启用两个控制平面 URL 的出口。侦听器轮询这些端点以进行部署：

        https://api.host.langchain.com
        https://api.smith.langchain.com

### 设置 (Setup)

1. 您向我们提供您的 LangSmith 组织 ID。我们将为您的组织启用自托管数据平面。
2. 我们为您提供一个 [Helm chart](https://github.com/langchain-ai/helm/tree/main/charts/langgraph-dataplane)，您运行该 chart 来设置您的 Kubernetes 集群。此 chart 包含一些重要组件。
    1. `langgraph-listener`：这是一个侦听 LangChain [控制平面](../../concepts/langgraph_control_plane.md) 以获取部署更改并创建/更新下游 CRD 的服务。
    1. `LangGraphPlatform CRD`：用于 LangGraph Platform 部署的 CRD。这包含用于管理 LangGraph 平台部署实例的规范。
    1. `langgraph-platform-operator`：此操作员处理对您的 LangGraph Platform CRD 的更改。
3. 配置您的 `langgraph-dataplane-values.yaml` 文件。

        config:
          langsmithApiKey: "" # API Key of your Workspace
          langsmithWorkspaceId: "" # Workspace ID
          hostBackendUrl: "https://api.host.langchain.com" # Only override this if on EU
          smithBackendUrl: "https://api.smith.langchain.com" # Only override this if on EU

4. 部署 `langgraph-dataplane` Helm chart。

        helm repo add langchain https://langchain-ai.github.io/helm/
        helm repo update
        helm upgrade -i langgraph-dataplane langchain/langgraph-dataplane --values langgraph-dataplane-values.yaml

5. 如果成功，您将在命名空间中看到两个服务启动。

        NAME                                          READY   STATUS              RESTARTS   AGE
        langgraph-dataplane-listener-7fccd788-wn2dx   0/1     Running             0          9s
        langgraph-dataplane-redis-0                   0/1     ContainerCreating   0          9s

6. 您从 [控制平面 UI](../../concepts/langgraph_control_plane.md#control-plane-ui) 创建部署。

## Amazon ECS

即将推出！
