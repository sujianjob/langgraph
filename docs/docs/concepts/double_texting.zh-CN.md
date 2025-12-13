---
search:
  boost: 2
---

# 双重文本 (Double Texting)

!!! info "Prerequisites"
    - [LangGraph Server](./langgraph_server.md)

很多时候，用户可能会以意想不到的方式与您的图进行交互。
例如，用户可能会发送一条消息，然后在图完成运行之前发送第二条消息。
更一般地说，用户可能会在第一次运行完成之前第二次调用图。
我们称之为“双重文本 (double texting)”。

目前，LangGraph 仅将其作为 [LangGraph Platform](langgraph_platform.md) 的一部分来解决，而不是在开源中。
这样做的原因是，为了处理这个问题，我们需要知道图是如何部署的，并且由于 LangGraph Platform 处理部署，因此逻辑需要存在于那里。
如果您不想使用 LangGraph Platform，我们在下面详细描述了我们已实施的选项。

![](img/double_texting.png)

## 拒绝 (Reject)

这是最简单的选项，它只会拒绝任何后续运行，并且不允许双重文本。
有关配置拒绝双重文本选项的信息，请参阅 [操作指南](../cloud/how-tos/reject_concurrent.md)。

## 入队 (Enqueue)

这是一个相对简单的选项，它继续第一次运行直到完成整个运行，然后将新输入作为单独的运行发送。
有关配置入队双重文本选项的信息，请参阅 [操作指南](../cloud/how-tos/enqueue_concurrent.md)。

## 中断 (Interrupt)

此选项中断当前执行，但保存直到该点为止所做的所有工作。
然后它插入用户输入并从那里继续。

如果启用此选项，您的图应该能够处理可能出现的奇怪边缘情况。
例如，您可能调用了一个工具，但尚未从运行该工具获得结果。
您可能需要删除该工具调用，以免出现悬空的工具调用。

有关配置中断双重文本选项的信息，请参阅 [操作指南](../cloud/how-tos/interrupt_concurrent.md)。

## 回滚 (Rollback)

此选项中断当前执行并回滚直到该点为止所做的所有工作，包括原始运行输入。然后它发送新的用户输入，基本上就像它是原始输入一样。

有关配置回滚双重文本选项的信息，请参阅 [操作指南](../cloud/how-tos/rollback_concurrent.md)。
