# 如何部署自托管控制平面 (How to Deploy Self-Hosted Control Plane)

在部署之前，请查看 [自托管控制平面的概念指南](../../concepts/langgraph_self_hosted_control_plane.md) 部署选项。

!!! info "Important"
    自托管控制平面部署选项需要 [企业版](../../concepts/plans.md) 计划。

## 先决条件 (Prerequisites)

1. 您正在使用 Kubernetes。
2. 您已经部署了自托管 LangSmith。
3. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) [在本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
4. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 构建 Docker 镜像（即 `langgraph build`）并将其推送到您的 Kubernetes 集群可以访问的注册表。
5. 集群上已安装 `KEDA`。

         helm repo add kedacore https://kedacore.github.io/charts 
         helm install keda kedacore/keda --namespace keda --create-namespace
6. Ingress 配置
    1. 您必须为您的 LangSmith 实例设置 ingress。所有代理都将部署为此 ingress 后面的 Kubernetes 服务。
    1. 您可以使用此指南为您的实例 [设置 ingress](https://docs.smith.langchain.com/self_hosting/configuration/ingress)。
7. 您的集群中有用于多次部署的闲置空间。建议使用 `Cluster-Autoscaler` 自动配置新节点。
8. 您的集群上有有效的动态 PV 预配器或可用的 PV。您可以通过运行以下命令进行验证：

        kubectl get storageclass

9. 从您的网络出口到 `https://beacon.langchain.com`。如果不在气隙 (air-gapped) 模式下运行，这是许可证验证和使用报告所必需的。有关更多详细信息，请参阅 [出口文档](../../cloud/deployment/egress.md)。

## 设置 (Setup)

1. 作为配置自托管 LangSmith 实例的一部分，您启用了 `langgraphPlatform` 选项。这将配置一些关键资源。
    1. `listener`：这是一个侦听 [控制平面](../../concepts/langgraph_control_plane.md) 以获取部署更改并创建/更新下游 CRD 的服务。
    1. `LangGraphPlatform CRD`：用于 LangGraph Platform 部署的 CRD。这包含用于管理 LangGraph 平台部署实例的规范。
    1. `operator`：此操作员处理对您的 LangGraph Platform CRD 的更改。
    1. `host-backend`：这是 [控制平面](../../concepts/langgraph_control_plane.md)。
2. chart 将使用两个额外的镜像。使用最新版本中指定的镜像。

        hostBackendImage:
          repository: "docker.io/langchain/hosted-langserve-backend"
          pullPolicy: IfNotPresent
        operatorImage:
          repository: "docker.io/langchain/langgraph-operator"
          pullPolicy: IfNotPresent

3. 在 langsmith 的配置文件（通常是 `langsmith_config.yaml`）中，启用 `langgraphPlatform` 选项。请注意，您还必须具有有效的 ingress 设置：

        config:
          langgraphPlatform:
            enabled: true
            langgraphPlatformLicenseKey: "YOUR_LANGGRAPH_PLATFORM_LICENSE_KEY"
4. 在您的 `values.yaml` 文件中，配置 `hostBackendImage` 和 `operatorImage` 选项（如果您需要镜像）

5. 您还可以通过 [在此处](https://github.com/langchain-ai/helm/blob/main/charts/langsmith/values.yaml#L898) 覆盖基本模板来为您的代理配置基本模板。
6. 您从 [控制平面 UI](../../concepts/langgraph_control_plane.md#control-plane-ui) 创建部署。
