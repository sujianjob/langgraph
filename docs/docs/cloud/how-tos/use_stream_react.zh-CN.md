# 如何将 LangGraph 集成到您的 React 应用程序中 (How to integrate LangGraph into your React application)

!!! info "Prerequisites"

    - [LangGraph Platform](../../concepts/langgraph_platform.md)
    - [LangGraph Server](../../concepts/langgraph_server.md)

`useStream()` React hook 提供了一种将 LangGraph 无缝集成到 React 应用程序中的方法。它处理所有流式传输、状态管理和分支逻辑的复杂性，让您专注于构建出色的聊天体验。

主要功能：

- 消息流式传输：处理消息块流以形成完整的消息
- 自动状态管理：用于消息、中断、加载状态和错误
- 对话分支：从聊天记录中的任何点创建备用对话路径
- UI 无关设计：自带组件和样式

让我们探索如何在您的 React 应用程序中使用 `useStream()`。

`useStream()` 为创建定制的聊天体验提供了坚实的基础。对于预构建的聊天组件和界面，我们还建议查看 [CopilotKit](https://docs.copilotkit.ai/coagents/quickstart/langgraph) 和 [assistant-ui](https://www.assistant-ui.com/docs/runtimes/langgraph)。

## 安装 (Installation)

```bash
npm install @langchain/langgraph-sdk @langchain/core
```

## 示例 (Example)

```tsx
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import type { Message } from "@langchain/langgraph-sdk";

export default function App() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [{ type: "human", content: message }] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button keytype="submit">Send</button>
        )}
      </form>
    </div>
  );
}
```

## 自定义您的 UI (Customizing Your UI)

`useStream()` hook 在后台处理所有复杂的状态管理，为您提供简单的界面来构建您的 UI。这是开箱即用的功能：

- 线程状态管理
- 加载和错误状态
- 中断
- 消息处理和更新
- 分支支持

以下是如何有效使用这些功能的一些示例：

### 加载状态 (Loading States)

`isLoading` 属性告诉您流何时处于活动状态，使您能够：

- 显示加载指示器
- 在处理过程中禁用输入字段
- 显示取消按钮

```tsx
export default function App() {
  const { isLoading, stop } = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <form>
      {isLoading && (
        <button key="stop" type="button" onClick={() => stop()}>
          Stop
        </button>
      )}
    </form>
  );
}
```

### 页面刷新后恢复流 (Resume a stream after page refresh)

`useStream()` hook 可以通过设置 `reconnectOnMount: true` 在挂载时自动恢复正在进行的运行。这对于在页面刷新后继续流式传输非常有用，确保存机期间生成的任何消息和事件都不会丢失。

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: true,
});
```

默认情况下，创建的运行的 ID 存储在 `window.sessionStorage` 中，可以通过在 `reconnectOnMount` 中传递自定义存储来交换。该存储用于持久化线程的正在进行的运行 ID（在 `lg:stream:${threadId}` 键下）。

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: () => window.localStorage,
});
```

您还可以通过使用运行回调来持久化运行元数据，并使用 `joinStream` 函数来恢复流，从而手动管理恢复过程。确保在创建运行时传递 `streamResumable: true`；否则可能会丢失某些事件。

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useCallback, useState, useEffect, useRef } from "react";

export default function App() {
  const [threadId, onThreadId] = useSearchParam("threadId");

  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",

    threadId,
    onThreadId,

    onCreated: (run) => {
      window.sessionStorage.setItem(`resume:${run.thread_id}`, run.run_id);
    },
    onFinish: (_, run) => {
      window.sessionStorage.removeItem(`resume:${run?.thread_id}`);
    },
  });

  // Ensure that we only join the stream once per thread.
  const joinedThreadId = useRef<string | null>(null);
  useEffect(() => {
    if (!threadId) return;

    const resume = window.sessionStorage.getItem(`resume:${threadId}`);
    if (resume && joinedThreadId.current !== threadId) {
      thread.joinStream(resume);
      joinedThreadId.current = threadId;
    }
  }, [threadId]);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const message = new FormData(form).get("message") as string;
        thread.submit(
          { messages: [{ type: "human", content: message }] },
          { streamResumable: true }
        );
      }}
    >
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>
      <input type="text" name="message" />
      <button type="submit">Send</button>
    </form>
  );
}

// Utility method to retrieve and persist data in URL as search param
function useSearchParam(key: string) {
  const [value, setValue] = useState<string | null>(() => {
    const params = new URLSearchParams(window.location.search);
    return params.get(key) ?? null;
  });

  const update = useCallback(
    (value: string | null) => {
      setValue(value);

      const url = new URL(window.location.href);
      if (value == null) {
        url.searchParams.delete(key);
      } else {
        url.searchParams.set(key, value);
      }

      window.history.pushState({}, "", url.toString());
    },
    [key]
  );

  return [value, update] as const;
}
```

### 线程管理 (Thread Management)

使用内置的线程管理来跟踪对话。您可以访问当前线程 ID 并在创建新线程时获得通知：

```tsx
const [threadId, setThreadId] = useState<string | null>(null);

const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",

  threadId: threadId,
  onThreadId: setThreadId,
});
```

我们建议将 `threadId` 存储在 URL 的查询参数中，以便用户在页面刷新后恢复对话。

### 消息处理 (Messages Handling)

`useStream()` hook 将跟踪从服务器接收的消息块，并将它们连接在一起以形成完整的消息。完整的消息块可以通过 `messages` 属性检索。

默认情况下，`messagesKey` 设置为 `messages`，这会将新的消息块附加到 `values["messages"]`。如果您将消息存储在不同的键中，可以更改 `messagesKey` 的值。

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";

export default function HomePage() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
}
```

在底层，`useStream()` hook 将使用 `streamMode: "messages-tuple"` 从图节点内的任何 LangChain 聊天模型调用接收消息流（即单个 LLM 令牌）。在 [流式传输](../how-tos/streaming.md#messages) 指南中了解有关消息流的更多信息。

### 中断 (Interrupts)

`useStream()` hook 公开了 `interrupt` 属性，该属性将填充来自线程的最后一次中断。您可以使用中断来：

- 在执行节点之前呈现确认 UI
- 等待人工输入，允许代理向用户提出澄清问题

在 [如何处理中断](../../how-tos/human_in_the_loop/wait-user-input.ipynb) 指南中了解有关中断的更多信息。

```tsx
const thread = useStream<{ messages: Message[] }, { InterruptType: string }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

if (thread.interrupt) {
  return (
    <div>
      Interrupted! {thread.interrupt.value}
      <button
        type="button"
        onClick={() => {
          // `resume` can be any value that the agent accepts
          thread.submit(undefined, { command: { resume: true } });
        }}
      >
        Resume
      </button>
    </div>
  );
}
```

### 分支 (Branching)

对于每条消息，您可以使用 `getMessagesMetadata()` 获取首次看到该消息的第一个检查点。然后，您可以从首次看到的检查点之前的检查点创建一个新的运行，以在线程中创建一个新分支。

可以通过以下方式创建分支：

1. 编辑以前的用户消息。
2. 请求重新生成以前的助手消息。

```tsx
"use client";

import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useState } from "react";

function BranchSwitcher({
  branch,
  branchOptions,
  onSelect,
}: {
  branch: string | undefined;
  branchOptions: string[] | undefined;
  onSelect: (branch: string) => void;
}) {
  if (!branchOptions || !branch) return null;
  const index = branchOptions.indexOf(branch);

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={() => {
          const prevBranch = branchOptions[index - 1];
          if (!prevBranch) return;
          onSelect(prevBranch);
        }}
      >
        Prev
      </button>
      <span>
        {index + 1} / {branchOptions.length}
      </span>
      <button
        type="button"
        onClick={() => {
          const nextBranch = branchOptions[index + 1];
          if (!nextBranch) return;
          onSelect(nextBranch);
        }}
      >
        Next
      </button>
    </div>
  );
}

function EditMessage({
  message,
  onEdit,
}: {
  message: Message;
  onEdit: (message: Message) => void;
}) {
  const [editing, setEditing] = useState(false);

  if (!editing) {
    return (
      <button type="button" onClick={() => setEditing(true)}>
        Edit
      </button>
    );
  }

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const content = new FormData(form).get("content") as string;

        form.reset();
        onEdit({ type: "human", content });
        setEditing(false);
      }}
    >
      <input name="content" defaultValue={message.content as string} />
      <button type="submit">Save</button>
    </form>
  );
}

export default function App() {
  const thread = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => {
          const meta = thread.getMessagesMetadata(message);
          const parentCheckpoint = meta?.firstSeenState?.parent_checkpoint;

          return (
            <div key={message.id}>
              <div>{message.content as string}</div>

              {message.type === "human" && (
                <EditMessage
                  message={message}
                  onEdit={(message) =>
                    thread.submit(
                      { messages: [message] },
                      { checkpoint: parentCheckpoint }
                    )
                  }
                />
              )}

              {message.type === "ai" && (
                <button
                  type="button"
                  onClick={() =>
                    thread.submit(undefined, { checkpoint: parentCheckpoint })
                  }
                >
                  <span>Regenerate</span>
                </button>
              )}

              <BranchSwitcher
                branch={meta?.branch}
                branchOptions={meta?.branchOptions}
                onSelect={(branch) => thread.setBranch(branch)}
              />
            </div>
          );
        })}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [message] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button key="submit" type="submit">
            Send
          </button>
        )}
      </form>
    </div>
  );
}
```

对于高级用例，您可以使用 `experimental_branchTree` 属性来获取线程的树形表示，该树形表示可用于为非基于消息的图呈现分支控件。

### 乐观更新 (Optimistic Updates)

您可以在向代理执行网络请求之前乐观地更新客户端状态，从而允许您向用户提供即时反馈，例如在代理看到请求之前立即显示用户消息。

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

const handleSubmit = (text: string) => {
  const newMessage = { type: "human" as const, content: text };

  stream.submit(
    { messages: [newMessage] },
    {
      optimisticValues(prev) {
        const prevMessages = prev.messages ?? [];
        const newMessages = [...prevMessages, newMessage];
        return { ...prev, messages: newMessages };
      },
    }
  );
};
```

### 缓存线程显示 (Cached Thread Display)

使用 `initialValues` 选项可在从服务器加载历史记录时立即显示缓存的线程数据。这通过在导航到现有线程时立即显示缓存数据来改善用户体验。

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";

const CachedThreadExample = ({ threadId, cachedThreadData }) => {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    // Show cached data immediately while history loads
    initialValues: cachedThreadData?.values,
    messagesKey: "messages",
  });

  return (
    <div>
      {stream.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
};
```

### 乐观线程创建 (Optimistic Thread Creation)

在 `submit` 函数中使用 `threadId` 选项以启用乐观 UI 模式，在此模式下，您需要在实际创建线程之前知道线程 ID。

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";

const OptimisticThreadExample = () => {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [optimisticThreadId] = useState(() => crypto.randomUUID());

  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    onThreadId: setThreadId, // (3) Updated after thread has been created.
    messagesKey: "messages",
  });

  const handleSubmit = (text: string) => {
    // (1) Perform a soft navigation to /threads/${optimisticThreadId}
    // without waiting for thread creation.
    window.history.pushState({}, "", `/threads/${optimisticThreadId}`);

    // (2) Submit message to create thread with the predetermined ID.
    stream.submit(
      { messages: [{ type: "human", content: text }] },
      { threadId: optimisticThreadId }
    );
  };

  return (
    <div>
      <p>Thread ID: {threadId ?? optimisticThreadId}</p>
      {/* Rest of component */}
    </div>
  );
};
```

### TypeScript

`useStream()` hook 对用 TypeScript 编写的应用程序很友好，您可以为状态指定类型以获得更好的类型安全性和 IDE 支持。

```tsx
// Define your types
type State = {
  messages: Message[];
  context?: Record<string, unknown>;
};

// Use them with the hook
const thread = useStream<State>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

您还可以根据需要为不同场景指定类型，例如：

- `ConfigurableType`: `config.configurable` 属性的类型 (通常是: `Record<string, unknown>`)
- `InterruptType`: 中断值的类型 - 例如 `interrupt(...)` 函数的内容 (默认: `unknown`)
- `CustomEventType`: 自定义事件的类型 (默认: `unknown`)
- `UpdateType`: 提交函数的类型 (默认: `Partial<State>`)

```tsx
const thread = useStream<
  State,
  {
    UpdateType: {
      messages: Message[] | Message;
      context?: Record<string, unknown>;
    };
    InterruptType: string;
    CustomEventType: {
      type: "progress" | "debug";
      payload: unknown;
    };
    ConfigurableType: {
      model: string;
    };
  }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

如果您正在使用 LangGraph.js，您还可以重用图的注释类型。但是，请确保仅导入注释模式的类型，以避免导入整个 LangGraph.js 运行时（即通过 `import type { ... }` 指令）。

```tsx
import {
  Annotation,
  MessagesAnnotation,
  type StateType,
  type UpdateType,
} from "@langchain/langgraph/web";

const AgentState = Annotation.Root({
  ...MessagesAnnotation.spec,
  context: Annotation<string>(),
});

const thread = useStream<
  StateType<typeof AgentState.spec>,
  { UpdateType: UpdateType<typeof AgentState.spec> }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

## 事件处理 (Event Handling)

`useStream()` hook 提供了几个回调选项来帮助您响应不同的事件：

- `onError`: 发生错误时调用。
- `onFinish`: 流式传输完成时调用。
- `onUpdateEvent`: 收到更新事件时调用。
- `onCustomEvent`: 收到自定义事件时调用。请参阅 [流式传输](../../how-tos/streaming.md#stream-custom-data) 指南以了解如何流式传输自定义事件。
- `onMetadataEvent`: 收到元数据事件时调用，其中包含运行 ID 和线程 ID。

## 学习更多 (Learn More)

- [JS/TS SDK 参考](../reference/sdk/js_ts_sdk_ref.md)
