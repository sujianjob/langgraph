## 组件 (Components)

LangGraph Platform 由协同工作的组件组成，以支持 LangGraph 应用程序的开发、部署、调试和监控：

- [LangGraph Server](./langgraph_server.md): 服务器定义了一个固有的 API 和架构，结合了部署代理应用程序的最佳实践，使您能够专注于构建代理逻辑，而不是开发服务器基础设施。
- [LangGraph CLI](./langgraph_cli.md): LangGraph CLI 是一个命令行界面，有助于与本地 LangGraph 进行交互
- [LangGraph Studio](./langgraph_studio.md): LangGraph Studio 是一个专门的 IDE，可以连接到 LangGraph Server 以在本地启用应用程序的可视化、交互和调试。
- [Python/JS SDK](./sdk.md): Python/JS SDK 提供了一种以编程方式与部署的 LangGraph 应用程序交互的方法。
- [远程图 (Remote Graph)](../how-tos/use-remote-graph.md): RemoteGraph 允许您与任何部署的 LangGraph 应用程序交互，就像它在本地运行一样。
- [LangGraph 控制平面](./langgraph_control_plane.md): LangGraph 控制平面是指控制平面 UI，用户在其中创建和更新 LangGraph Server，以及支持 UI 体验的控制平面 API。
- [LangGraph 数据平面](./langgraph_data_plane.md): LangGraph 数据平面是指 LangGraph Server、每个服务器的相应基础设施以及持续轮询来自 LangGraph 控制平面的更新的“侦听器”应用程序。

![LangGraph components](img/lg_platform.png)
