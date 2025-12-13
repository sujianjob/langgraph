---
search:
  boost: 2
---

# 记忆 (Memory)

[记忆 (Memory)](../how-tos/memory/add-memory.md) 是一个用于记住先前交互信息的系统。对于人工智能代理来说，记忆至关重要，因为它使它们能够记住先前的交互、从反馈中学习并适应用户的偏好。随着代理处理越来越多具有大量用户交互的复杂任务，这种能力对于效率和用户满意度都变得至关重要。

本概念指南根据回忆范围涵盖了两种类型的记忆：

- [短期记忆 (Short-term memory)](#short-term-memory)，或 [线程 (thread)](persistence.md#threads) 范围的记忆，通过维护会话中的消息历史记录来跟踪正在进行的对话。LangGraph 将短期记忆作为代理 [状态 (state)](low_level.md#state) 的一部分进行管理。状态使用 [检查点 (checkpointer)](persistence.md#checkpoints) 持久化到数据库，以便随时恢复线程。短期记忆在图被调用或步骤完成时更新，并且在每个步骤开始时读取状态。

- [长期记忆 (Long-term memory)](#long-term-memory) 跨会话存储特定于用户或应用程序级别的数据，并在对话线程 *之间* 共享。它可以 *随时* 在 *任何线程* 中被回忆。记忆可以作用于任何自定义命名空间，而不仅仅是在单个线程 ID 内。LangGraph 提供 [存储 (stores)](persistence.md#memory-store) ([参考文档](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore)) 让您保存和回忆长期记忆。

![](img/memory/short-vs-long.png)

## 短期记忆 (Short-term memory)

[短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory) 让您的应用程序记住单个 [线程](persistence.md#threads) 或对话中的先前交互。[线程](persistence.md#threads) 组织会话中的多个交互，类似于电子邮件将消息分组在单个对话中的方式。

LangGraph 将短期记忆作为代理状态的一部分进行管理，通过线程范围的检查点进行持久化。此状态通常可以包括对话历史记录以及其他有状态数据，例如上传的文件、检索到的文档或生成的工件。通过将这些存储在图的状态中，机器人可以访问给定对话的完整上下文，同时保持不同线程之间的分离。

### 管理短期记忆 (Manage short-term memory)

对话历史记录是短期记忆最常见的形式，而长对话对今天的 LLM 构成了挑战。完整的历史记录可能无法容纳在 LLM 的上下文窗口中，从而导致不可恢复的错误。即使您的 LLM 支持完整的上下文长度，大多数 LLM 在长上下文中的表现仍然很差。它们会被陈旧或离题的内容“分散注意力”，同时遭受更慢的响应时间和更高的成本。

聊天模型使用消息接受上下文，其中包括开发人员提供的指令（系统消息）和用户输入（人类消息）。在聊天应用程序中，消息在人类输入和模型响应之间交替，导致消息列表随着时间的推移而变长。因为上下文窗口是有限的，而且包含大量令牌的消息列表可能代价高昂，所以许多应用程序可以受益于使用手动删除或忘记陈旧信息的技术。

![](img/memory/filter.png)

有关管理消息的常用技术的更多信息，请参阅 [添加和管理记忆](../how-tos/memory/add-memory.md#manage-short-term-memory) 指南。

## 长期记忆 (Long-term memory)

LangGraph 中的 [长期记忆](../how-tos/memory/add-memory.md#add-long-term-memory) 允许系统跨不同的对话或会话保留信息。与 **线程范围** 的短期记忆不同，长期记忆保存在自定义“命名空间”中。

长期记忆是一个复杂的挑战，没有放之四海而皆准的解决方案。但是，由于以下问题提供了一个框架来帮助您浏览不同的技术：

- [记忆的类型是什么？(#memory-types) 人类使用记忆来记住事实 ([语义记忆](#semantic-memory))、经历 ([情景记忆](#episodic-memory)) 和规则 ([程序记忆](#procedural-memory))。AI 代理可以以相同的方式使用记忆。例如，AI 代理可以使用记忆来记住有关用户的特定事实以完成任务。

- [您想何时更新记忆？](#writing-memories) 记忆可以作为代理应用程序逻辑的一部分进行更新（例如，“在热路径上”）。在这种情况下，代理通常决定在响应用户之前记住事实。或者，记忆可以作为后台任务更新（在后台/异步运行并生成记忆的逻辑）。我们在 [下面的部分](#writing-memories) 中解释了这些方法之间的权衡。

### 记忆类型 (Memory types)

不同的应用程序需要各种类型的记忆。虽然这个类比并不完美，但检查 [人类记忆类型](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev) 会很有启发性。一些研究（例如，[CoALA 论文](https://arxiv.org/pdf/2309.02427)）甚至将这些人类记忆类型映射到 AI 代理中使用的那些。

| 记忆类型 (Memory Type) | 存储内容 (What is Stored) | 人类示例 (Human Example) | 代理示例 (Agent Example) |
|-------------|----------------|---------------|---------------|
| [语义 (Semantic)](#semantic-memory) | 事实 (Facts) | 我在学校学到的东西 | 关于用户的事实 |
| [情景 (Episodic)](#episodic-memory) | 经历 (Experiences) | 我做过的事情 | 过去的代理操作 |
| [程序 (Procedural)](#procedural-memory) | 指令 (Instructions) | 本能或运动技能 | 代理系统提示 |

#### 语义记忆 (Semantic memory)

[语义记忆](https://en.wikipedia.org/wiki/Semantic_memory)，无论是在人类还是 AI 代理中，都涉及保留特定的事实和概念。在人类中，它可以包括在学校学到的信息以及对概念及其关系的理解。对于 AI 代理，语义记忆通常用于通过记住过去交互中的事实或概念来个性化应用程序。

!!! note

    语义记忆不同于“语义搜索”，后者是一种使用“含义”（通常作为嵌入）查找相似内容的技术。语义记忆是一个心理学术语，指的是存储事实和知识，而语义搜索是一种基于含义而不是精确匹配来检索信息的方法。

##### 资料 (Profile)

语义记忆可以通过不同的方式进行管理。例如，记忆可以是关于用户、组织或其他实体（包括代理本身）的范围明确且具体的单个、持续更新的“资料”。资料通常只是一个 JSON 文档，其中包含您选择用来表示您的域的各种键值对。

在记住资料时，您需要确保每次都在 **更新** 资料。因此，您需要传入先前的资料并 [要求模型生成新的资料](https://github.com/langchain-ai/memory-template)（或一些 [JSON补丁](https://github.com/hinthornw/trustcall) 以应用于旧资料）。随着资料变大，这可能会变得容易出错，并且可能受益于将资料拆分为多个文档或在生成文档时进行 **严格** 解码以确保记忆模式保持有效。

![](img/memory/update-profile.png)

##### 集合 (Collection)

或者，记忆可以是随时间不断更新和扩展的文档集。每个单独的记忆可以范围更窄且更容易生成，这意味着您不太可能随着时间的推移 **丢失** 信息。对于 LLM 来说，为新信息生成 *新* 对象比将新信息与现有资料协调起来更容易。因此，文档集往往会导致 [下游更高的召回率](https://en.wikipedia.org/wiki/Precision_and_recall)。

但是，这将一些复杂性转移到了记忆更新上。模型现在必须 *删除* 或 *更新* 列表中的现有项目，这可能很棘手。此外，一些模型可能默认过度插入，而其他模型可能默认过度更新。请参阅 [Trustcall](https://github.com/hinthornw/trustcall) 包以了解管理此问题的一种方法，并考虑评估（例如，使用像 [LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation) 这样的工具）以帮助您调整行为。

使用文档集还将复杂性转移到了对列表的记忆 **搜索** 上。`Store` 目前支持 [语义搜索](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query) 和 [按内容过滤](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter)。

最后，使用记忆集合可能会使向模型提供全面的上下文变得具有挑战性。虽然个别记忆可能遵循特定的模式，但这种结构可能无法捕获完整的上下文或记忆之间的关系。因此，在使用这些记忆生成响应时，模型可能缺乏在统一资料方法中更容易获得的重要上下文信息。

![](img/memory/update-list.png)

无论采用哪种记忆管理方法，中心点是代理将使用语义记忆来 [这地 (ground) 其响应](https://python.langchain.com/docs/concepts/rag/)，这通常会导致更加个性化和相关的交互。

#### 情景记忆 (Episodic memory)

[情景记忆](https://en.wikipedia.org/wiki/Episodic_memory)，无论是在人类还是 AI 代理中，都涉及回忆过去的事件或行动。[CoALA 论文](https://arxiv.org/pdf/2309.02427) 对此进行了很好的构建：事实可以写入语义记忆，而 *经历* 可以写入情景记忆。对于 AI 代理，情景记忆通常用于帮助代理记住如何完成任务。

:::python
在实践中，情景记忆通常通过 [少样本示例提示 (few-shot example prompting)](https://python.langchain.com/docs/concepts/few_shot_prompting/) 来实现，其中代理从过去的序列中学习以此正确执行任务。有时“展示”比“讲述”更容易，LLM 从示例中学习得很好。少样本学习让您可以通过使用输入-输出示例更新提示来说明预期行为，从而 ["编程"](https://x.com/karpathy/status/1627366413840322562) 您的 LLM。虽可以使用各种 [最佳实践](https://python.langchain.com/docs/concepts/#1-generating-examples) 来生成少样本示例，但挑战往往在于根据用户输入选择最相关的示例。
:::

:::js
在实践中，情景记忆通常通过少样本示例提示来实现，其中代理从过去的序列中学习以此正确执行任务。有时“展示”比“讲述”更容易，LLM 从示例中学习得很好。少样本学习让您可以通过使用输入-输出示例更新提示来说明预期行为，从而 ["编程"](https://x.com/karpathy/status/1627366413840322562) 您的 LLM。虽然可以使用各种最佳实践来生成少样本示例，但挑战往往在于根据用户输入选择最相关的示例。
:::

:::python
请注意，记忆 [存储](persistence.md#memory-store) 只是将数据存储为少样本示例的一种方式。如果您希望有更多的开发人员参与，或者将少样本更紧密地绑定到您的评估工具，您也可以使用 [LangSmith 数据集](https://docs.smith.langchain.com/evaluation/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) 来存储您的数据。然后可以使用开箱即用的动态少样本示例选择器来实现这一目标。LangSmith 将为您索引数据集，并能够根据关键字相似性（[使用类似 BM25 的算法](https://docs.smith.langchain.com/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) 进行基于关键字的相似性）检索与用户输入最相关的少样本示例。

请参阅此操作指南 [视频](https://www.youtube.com/watch?v=37VaU7e7t5o) 以了解 LangSmith 中动态少样本示例选择的用法示例。此外，请参阅此 [博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/) 展示少样本提示以提高工具调用性能，以及此 [博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/) 使用少样本示例使 LLM 与人类偏好保持一致。
:::

:::js
请注意，记忆 [存储](persistence.md#memory-store) 只是将数据存储为少样本示例的一种方式。如果您希望有更多的开发人员参与，或者将少样本更紧密地绑定到您的评估工具，您也可以使用 LangSmith 数据集来存储您的数据。然后可以使用开箱即用的动态少样本示例选择器来实现这一目标。LangSmith 将为您索引数据集，并能够根据关键字相似性检索与用户输入最相关的少样本示例。

请参阅此操作指南 [视频](https://www.youtube.com/watch?v=37VaU7e7t5o) 以了解 LangSmith 中动态少样本示例选择的用法示例。此外，请参阅此 [博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/) 展示少样本提示以提高工具调用性能，以及此 [博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/) 使用少样本示例使 LLM 与人类偏好保持一致。
:::

#### 程序记忆 (Procedural memory)

[程序记忆](https://en.wikipedia.org/wiki/Procedural_memory)，无论是在人类还是 AI 代理中，都涉及记住用于执行任务的规则。在人类中，程序记忆就像是如何执行任务的内在知识，例如通过基本的运动技能和平衡骑自行车。另一方面，情景记忆涉及回忆特定的经历，例如您第一次成功骑没有辅助轮的自行车或通过风景优美的路线进行难忘的骑行。对于 AI 代理，程序记忆是模型权重、代理代码和代理提示的组合，它们共同决定了代理的功能。

在实践中，代理修改其模型权重或重写其代码相当罕见。然而，代理修改自己的提示更为常见。

完善代理指令的一种有效方法是通过 ["反思 (Reflection)"](https://blog.langchain.dev/reflection-agents/) 或元提示。这涉及用当前的指令（例如，系统提示）以及最近的对话或明确的用户反馈来提示代理。然后代理根据此输入完善其自己的指令。此方法对于难以预先指定指令的任务特别有用，因为它允许代理从交互中学习和适应。

例如，我们使用外部反馈和提示重写构建了一个 [推文生成器](https://www.youtube.com/watch?v=Vn8A3BxfplE)，为 Twitter 生成高质量的论文摘要。在这种情况下，具体的摘要提示很难 *先验地* 指定，但用户很容易批评生成的推文并提供有关如何改进摘要过程的反馈。

下面的伪代码显示了如何使用 LangGraph 记忆 [存储](persistence.md#memory-store) 实现这一点，使用存储来保存提示，使用 `update_instructions` 节点获取当前提示（以及从 `state["messages"]` 中捕获的与用户的对话反馈），更新提示，并将新提示保存回存储。然后，`call_model` 从存储中获取更新的提示并使用它来生成响应。

:::python
```python
# Node that *uses* the instructions
def call_model(state: State, store: BaseStore):
    namespace = ("agent_instructions", )
    instructions = store.get(namespace, key="agent_a")[0]
    # Application logic
    prompt = prompt_template.format(instructions=instructions.value["instructions"])
    ...

# Node that updates instructions
def update_instructions(state: State, store: BaseStore):
    namespace = ("instructions",)
    current_instructions = store.search(namespace)[0]
    # Memory logic
    prompt = prompt_template.format(instructions=current_instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```
:::

:::js
```typescript
// Node that *uses* the instructions
const callModel = async (state: State, store: BaseStore) => {
    const namespace = ["agent_instructions"];
    const instructions = await store.get(namespace, "agent_a");
    // Application logic
    const prompt = promptTemplate.format({ 
        instructions: instructions[0].value.instructions 
    });
    // ...
};

// Node that updates instructions
const updateInstructions = async (state: State, store: BaseStore) => {
    const namespace = ["instructions"];
    const currentInstructions = await store.search(namespace);
    // Memory logic
    const prompt = promptTemplate.format({ 
        instructions: currentInstructions[0].value.instructions, 
        conversation: state.messages 
    });
    const output = await llm.invoke(prompt);
    const newInstructions = output.new_instructions;
    await store.put(["agent_instructions"], "agent_a", { 
        instructions: newInstructions 
    });
    // ...
};
```
:::

![](img/memory/update-instructions.png)

### 写入记忆 (Writing memories)

代理写入记忆主要有两种方法：["在热路径中"](#in-the-hot-path) 和 ["在后台"](#in-the-background)。

![](img/memory/hot_path_vs_background.png)

#### 在热路径中 (In the hot path)

在运行时创建记忆既有优点也有挑战。积极的一面是，这种方法允许实时更新，使新记忆立即可用于后续交互。它还可以实现透明度，因为可以在创建和存储记忆时通知用户。

然而，这种方法也带来了挑战。如果代理需要一个新工具来决定要提交什么到记忆，它可能会增加复杂性。此外，推理要保存什么到记忆的过程会影响代理的延迟。最后，代理必须在记忆创建和其他职责之间进行多任务处理，这可能会影响创建的记忆的数量和质量。

作为一个例子，ChatGPT 使用 [save_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/) 工具将记忆作为内容字符串进行更新插入，并在每个用户消息中决定是否以及如何使用此工具。请参阅我们的 [memory-agent](https://github.com/langchain-ai/memory-agent) 模板作为参考实现。

#### 在后台 (In the background)

将记忆创建作为单独的后台任务提供了几个优点。它消除了主应用程序中的延迟，将应用程序逻辑与记忆管理分离，并允许代理更专注于完成任务。这种方法还提供了在时间上安排记忆创建的灵活性，以避免重复工作。

然而，这种方法也有其自身的挑战。确定记忆写入的频率变得至关重要，因为不频繁的更新可能会让其他线程没有新的上下文。决定何时触发记忆形成也很重要。常见的策略包括在设定的时间段后安排（如果发生新事件则重新安排）、使用 cron 计划，或允许用户或应用程序逻辑手动触发。

请参阅我们的 [memory-service](https://github.com/langchain-ai/memory-template) 模板作为参考实现。

### 记忆存储 (Memory storage)

LangGraph 将长期记忆作为 JSON 文档存储在 [存储](persistence.md#memory-store) 中。每个记忆都在自定义的 `namespace`（类似于文件夹）和独特的 `key`（类似于文件名）下组织。命名空间通常包括用户或组织 ID 或其他便于组织信息的标签。这种结构实现了记忆的分层组织。然后通过内容过滤器支持跨命名空间搜索。

:::python
```python
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# get the "memory" by ID
item = store.get(namespace, "a-memory")
# search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```
:::

:::js
```typescript
import { InMemoryStore } from "@langchain/langgraph";

const embed = (texts: string[]): number[][] => {
    // Replace with an actual embedding function or LangChain embeddings object
    return texts.map(() => [1.0, 2.0]);
};

// InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
const store = new InMemoryStore({ index: { embed, dims: 2 } });
const userId = "my-user";
const applicationContext = "chitchat";
const namespace = [userId, applicationContext];

await store.put(
    namespace,
    "a-memory",
    {
        rules: [
            "User likes short, direct language",
            "User only speaks English & TypeScript",
        ],
        "my-key": "my-value",
    }
);

// get the "memory" by ID
const item = await store.get(namespace, "a-memory");

// search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
const items = await store.search(
    namespace, 
    { 
        filter: { "my-key": "my-value" }, 
        query: "language preferences" 
    }
);
```
:::

有关记忆存储的更多信息，请参阅 [持久化](persistence.md#memory-store) 指南。
