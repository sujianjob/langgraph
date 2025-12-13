# 概述 (Overview)

LangGraph 专为想要构建强大且适应性强的 AI 代理的开发人员而构建。开发人员选择 LangGraph 的原因：

- **可靠性和可控性**：通过审核检查和人机回圈批准来引导代理操作。LangGraph 为长时间运行的工作流保留上下文，使您的代理保持在正轨上。
- **低级和可扩展**：使用完全描述性的低级原语构建自定义代理，无需限制自定义的严格抽象。设计可扩展的多代理系统，每个代理都服务于针对您的用例量身定制的特定角色。
- **一流的流式传输支持**：通过逐个令牌的流式传输和中间步骤的流式传输，LangGraph 让用户可以清晰地看到代理推理和行动在实时展开。

## 学习 LangGraph 基础知识 (Learn LangGraph basics)

要熟悉 LangGraph 的关键概念和功能，请完成以下 LangGraph 基础教程系列：

1. [构建基本聊天机器人](../tutorials/get-started/1-build-basic-chatbot.md)
2. [添加工具](../tutorials/get-started/2-add-tools.md)
3. [添加记忆](../tutorials/get-started/3-add-memory.md)
4. [添加人机回圈控制](../tutorials/get-started/4-human-in-the-loop.md)
5. [自定义状态](../tutorials/get-started/5-customize-state.md)
6. [时间旅行](../tutorials/get-started/6-time-travel.md)

在完成本系列教程后，您将在 LangGraph 中构建一个支持聊天机器人，它可以：

* ✅ 通过搜索网络 **回答常见问题**
* ✅ 在调用之间 **保持对话状态**
* ✅ 将 **复杂查询路由** 给人工进行审查
* ✅ **使用自定义状态** 来控制其行为
* ✅ **倒带和探索** 替代对话路径
