## 前言

如今随着 GPT 私有化的不断实现、Deepseek 等更容易部署的模型发布，各大企业都开始投入到 AI 私有化、与自身平台应用的集成进程中，在前端领域的你也应该熟悉和掌握 Gpt Web/客户端的开发技术，以能够快速搭建定制化的企业级 AI 平台应用。

## 市面上的 Gpt UI 框架

目前市面上对于私有部署 Gpt UI 的框架很多，例如 Dify、Fastgpt 等一键部署前后端的，缺点是太重，逻辑，定制起来比较麻烦。本文介绍一个轻量级的用于快速搭建 Gpt 聊天窗口界面的 UI 组件库 `assistant-ui`

## assistant-ui 理念

assistant-ui 官网的介绍它是一个 Typescript 的 React 组件库。
经过我几天的 demo 编写体验，感受是它将很多逻辑都封装为一个个模块化的 JSX 标签，例如输入框、编辑按钮、甚至将 IF 条件语句也给用户封装了 JSX 组件，这些组件内有数据流逻辑，每一个组件底层都有数据流操作耦合逻辑，所以它不是一个纯粹的 UI 组件库。
这些 JSX 组件自带了 Tailwind 的样式风格，用这些组件搭建 UI 界面就像堆积木一样，需要什么 UI 模块，堆在一个页面中即可。
同时它将 Gpt 的数据流逻辑进行了抽象，对外暴露了自定义接口，可以接入私有的三方 gpt api，可以兼容任何格式的 gpt 返回结构。

### 开发者无感知 Gpt 数据流

gpt 返回的数据是流式的，需要将流式的数据片段式地填充到页面上展示，这部分逻辑 assistant-ui 已经内置封装在组件中，你只需要按格式返回给 assistant-ui 即可，

如下所示，实现模型适配器：

```ts
const MyModelAdapter: ChatModelAdapter = {
  async *run({ messages, abortSignal, context }) {
    const stream = await streamRequest({ messages, abortSignal, context });
 
    let text = "";
    for await (const part of stream) {
      text += part.content
 
      yield {
        content: [{ type: "text", text }],
      };
    }
  },
};
```

也可以直接实现为：

```ts
const MyModelAdapter: ChatModelAdapter = {
  async *run({ messages, abortSignal, context }) {
    const stream = await streamRequest({ messages, abortSignal, context });
    yield* stream
  },
};
```

当然 `streamRequest` 也需要定义为 Generator 生成器函数，返回复合要求的数据结构，例如实现 GPT 流式请求：

```ts
export async function* streamRequest(): AsyncGenerator<ChatModelRunResult> {
  const result = await fetch(url, options)
  const reader = result.body.getReader()

   while (true) {
      const { done, value } = await reader.read()
      if(done) {
        break
      }
      yield { status: 'running', content: value }
   }
}
```

### 组件堆砌的开发方式

assistant-ui 提供了和逻辑层耦合的 UI 组件，所以几乎无需编写逻辑，只需要页面添加对应组件即可，例如：

```tsx
const MyGptUI = () => {
  return (
    <MyRuntimeProvider>
      <ThreadListPrimitive.New />  {/* 新建对话 */}
      <ThreadListPrimitive.Items />  {/* 对话列表 */}
      <ThreadPrimitive.Viewport>  {/* 对话消息窗口 */}
        <ThreadPrimitive.Suggestion
          prompt="如何用 Typescript 实现 Helloworld？"
          method="replace"
          autoSend
        /> {/* 预设推荐问句 */}
        <ThreadPrimitive.Messages /> {/* 消息列表 */}
        <ThreadPrimitive.ScrollToBottom /> {/* 滚到窗口底部按钮 */}
        <ComposerPrimitive.Input /> {/* 输入框 */}
        <ComposerPrimitive.Send /> {/* 发送按钮 */}
      </ThreadPrimitive.Viewport>
    </MyRuntimeProvider>
  )
}
```

这些例如 `New` 、`Messages`、`Send` 组件内部都有数据流监听和处理逻辑，不是纯的 UI 视图组件。

## assistant-ui 的基础组件

在私有定制化需求中，我们通常需要调整 UI 界面的样式、布局，下面将介绍各个组件的使用方法。

### 聊天线程列表

gpt 界面上通常左侧为不同主题/上下文的聊天列表，通俗来讲就是左侧是聊天室列表，每个聊天室保存了 gpt 对话的上下文，在 `assistant-ui` 框架中称为聊天线程。

1. 新建聊天线程

要开始一个 gpt 对话窗口，首先创建一个聊天线程，

```tsx
import { Button } from 'antd'
import { DeleteOutlined, PlusOutlined } from '@ant-design/icons'
import { ThreadListPrimitive } from '@assistant-ui/react'

// 点击创建新聊天
const ThreadListNew: FC = () => {
  return (
    <ThreadListPrimitive.New asChild>
      <Button type="ghost">
        <PlusOutlined />
        开启新对话
      </Button>
    </ThreadListPrimitive.New>
  )
}
```

这段代码中使用了 antd 的按钮组件实现了创建聊天按钮，使用 `ThreadListPrimitive.New` 组件包了一层，这样在点击按钮时，就会自动初始化一个聊天线程。

下面是 `ThreadListPrimitive.New` 组件实现的简要原理：

```tsx
import { useAssistantRuntime } from '@assistant-ui/react'
import { Primitive } from "@radix-ui/react-primitive";

export const ThreadListPrimitiveNew = () => {
  const runtime = useAssistantRuntime()
  return (
    <Primitive.button
      onClick={() => {
        runtime.switchToNewThread()
      }}
    />
  )
}
```

调用 `switchToNewThread` 方法即可创建聊天线程。

`useAssistantRuntime` 是 `assistant-ui` 提供的一个 Hook Api，可获取到机器人的 API，进行聊天线程创建、切换等操作。

2. 切换到另一个聊天线程

创建了多个聊天线程时，需要切换时可使用 `Trigger` 组件：
 
```tsx
import { Button } from 'antd'
import { ThreadListItemPrimitive } from '@assistant-ui/react'

const ThreadListItem: FC = () => {
  return (
    <ThreadListItemPrimitive.Trigger>
      <Button>对话1</Button>
    </ThreadListItemPrimitive.Trigger>
  )
}
```

点击对话1按钮时，就会切换到对话1的聊天线程。

下面是 `Trigger` 组件实现原理：

```tsx
import { useThreadListItemRuntime } from '@assistant-ui/react'

const Trigger = ({ children }) => {
  const runtime = useThreadListItemRuntime()
  return (
    <div
      onClick={() => {
        runtime.switchTo()
      }}
    >
      {children}
    </div>
  )
}
```

点击 `Trigger` 组件时，调用了 `useThreadListItemRuntime` Hook API 的 `switchTo` 方法，即可自动切换到该线程。

3. 聊天线程的操作

assistant-ui 提供了聊天线程归档、删除等操作，例如拓展上文的 ThreadListItem 实现：

```tsx
import { Button, Space } from 'antd'
import { ThreadListItemPrimitive } from '@assistant-ui/react'

const ThreadListItem: FC = () => {
  return (
    <Space>
      <ThreadListItemPrimitive.Trigger>
        <Button>对话1</Button>
      </ThreadListItemPrimitive.Trigger>
      <ThreadListItemPrimitive.Archive>
        归档 Icon
      </ThreadListItemPrimitive.Archive>
      <ThreadListItemPrimitive.Delete>
        删除 Icon
      </ThreadListItemPrimitive.Delete>
    </Space>
  )
}
```

归档和删除也都是通过 `useThreadListItemRuntime` Hook 实现的：

```ts
const useThreadListItemHook = () => {
  const runtime = useThreadListItemRuntime();

  const delete = () => {
    runtime.delete();
  };
  const archive = () => {
    runtime.archive();
  };
  const unarchive = () => {
    runtime.unarchive();
  };

  return { delete, archive, unarchive }
};
```

### 聊天线程

1. 预设问答

可以使用 assistant-ui 提供的 `ThreadPrimitive.Suggestion` 组件，预设几个问句卡片，当用户点击卡片时，会自动提交发送并开始对话

```tsx
import { Space } from 'antd'
import { ThreadPrimitive } from '@assistant-ui/react'

const ThreadWelcomeSuggestions: FC = () => {
  return (
    <Space>
      <ThreadPrimitive.Suggestion
        prompt="如何用 Typescript 实现 Helloworld？"
        method="replace"
        autoSend
      >
        如何用 Typescript 实现 Helloworld？
      </ThreadPrimitive.Suggestion>
      <ThreadPrimitive.Suggestion
        prompt="物联网是什么？"
        method="replace"
        autoSend
      >
        物联网是什么？
      </ThreadPrimitive.Suggestion>
    </Space>
  )
}
```

`Suggestion` 组件的实现原理，通过 `useThreadRuntime` Hook 可以获取到输入框 composer 实例，可快速将文本输入到对话输入框，或通过 append 方法直接对话获取回答：

```tsx
import { useThreadRuntime } from '@assistant-ui/react'

const Suggestion = ({ children, autoSend }) => {
  const threadRuntime = useThreadRuntime()
  return (
    <div
      onClick={() => {
        if(autoSend) {
          threadRuntime.append(prompt);
        } else {
          threadRuntime.composer.setText(prompt)
        }
      }}
    >
      {children}
    </div>
  )
}
```

2. 聊天上下文

Messages 组件用来展示聊天消息上下文，其中必须实现的有以下 3 个组件：

1. UserMessage: 用户消息
2. EditComposer: 用户消息可编辑
3. AssistantMessage: AI回复消息

```tsx
import { ThreadPrimitive } from '@assistant-ui/react'

const App = () => {
  return (
    <div>
      {/* Left 省略 线程列表... */}
      <ThreadPrimitive.Messages
        components={{
          UserMessage: UserMessage,
          EditComposer: EditComposer,
          AssistantMessage: AssistantMessage,
        }}
      />
      {/* Bottom 省略 输入框... */}
    </div>
  )
}

```

2.1 用户聊天消息展示

用来展示用户输入的消息、复制按钮、分支切换按钮等

```tsx
import { Space } from 'antd'
import { MessagePrimitive, ActionBarPrimitive } from '@assistant-ui/react'

const UserMessage: FC = () => {
  return (
    <>
      <MessagePrimitive.Content />
      <ActionBarPrimitive.Edit>编辑Icon</ActionBarPrimitive.Edit>
    </>
  )
}
```

使用 `ActionBarPrimitive.Edit` 组件可实现一个编辑按钮，点击后会使用户消息变成输入框，再次输入后修改重新进入问答。

`Edit` 组件的实现原理：

```tsx
import { useMessageRuntime } from '@assistant-ui/react'

const Edit = ({ children }) => {
  const messageRuntime = useMessageRuntime()

  return (
    <div
      onClick={() => {
        messageRuntime.composer.beginEdit()
      }}
    >
      {children}
    </div>
  )
}
```

通过 `useMessageRuntime` Hook 可获取到 composer 输入框实例，点击后触发编辑。
 
工具栏的组件还有以下几种：
1. 编辑 ActionBarPrimitive.Edit
2. 复制 ActionBarPrimitive.Copy
3. 刷新 ActionBarPrimitive.Reload
5. 阅读 ActionBarPrimitive.Speak

2.2 分支选择

使用 BranchPickerPrimitive 组件可切换不同的问句和答句

```tsx
import { BranchPickerPrimitive } from '@assistant-ui/react'

const BranchPicker = ({ className }) => {
  return (
    <BranchPickerPrimitive.Root
      hideWhenSingleBranch
    >
      <BranchPickerPrimitive.Previous asChild>
        <LeftOutlined />
      </BranchPickerPrimitive.Previous>
      <span>
        <BranchPickerPrimitive.Number /> / <BranchPickerPrimitive.Count />
      </span>
      <BranchPickerPrimitive.Next asChild>
        <RightOutlined />
      </BranchPickerPrimitive.Next>
    </BranchPickerPrimitive.Root>
  )
}
```

2.3 AI 回复消息展示

AI回复展示消息的组件和用户消息组件相同，只是在 `ThreadPrimitive.Messages` 中设置到 `AssistantMessage` 即可

3. 条件语句

ThreadPrimitive.If 组件有很多功能，用来条件渲染当前是否是对话中、复制完成、播放中、聊天是否为空等，例如在用户没有对话记录时，可渲染一个占位提示；

```tsx
import { Alert } from 'antd'
import { ThreadPrimitive } from '@assistant-ui/react'

<ThreadPrimitive.If empty={true}>
  {/* 没有数据时的占位 */}
  <Alert
    type="warning"
    message="我是AI，可以回答你的问题，请在下方输入框输入你的需求～"
  />
</ThreadPrimitive.If>
```

4. 滚动容器

聊天记录要随着消息不断填充，滚动条自动滚到底部，可以使用 assistant-ui 的 Viewport 组件自动完成:

```tsx
import { ThreadPrimitive } from '@assistant-ui/react'

const App = () => {
  return (
    <div>
      {/* Left 省略 线程列表... */}
      <ThreadPrimitive.Viewport style={{ height: '100%', overflowY: 'scroll' }}>
        <ThreadPrimitive.Messages
          components={{
            UserMessage: UserMessage,
            EditComposer: EditComposer,
            AssistantMessage: AssistantMessage,
          }}
        />
        {/* Bottom 省略 输入框... */}
        <div style={{ position: 'sticky', bottom: 0 }}>
          <ThreadPrimitive.ScrollToBottom>滚到底部</ThreadPrimitive.ScrollToBottom>
          <Composer>
        </div>
      </ThreadPrimitive.Viewport>
    </div>
  )
}
```

`ThreadPrimitive.ScrollToBottom` 组件可实现点击后自动滚到底部

5. 输入框

```tsx
import { Space } from 'antd'
import { ComposerPrimitive } from '@assistant-ui/react'

// 底部的输入框
const Composer: FC = () => {
  return (
    <ComposerPrimitive.Root>
      <Space>
        <ComposerPrimitive.Input rows={1} autoFocus placeholder="Write a message..." />
        <ComposerAction />
      </Space>
    </ComposerPrimitive.Root>
  )
}
```

6. 发送和取消问答

```tsx
import { ComposerPrimitive, ThreadPrimitive } from '@assistant-ui/react'

// 底部的发送/停止对话消息按钮
const ComposerAction: FC = () => {
  return (
    <>
      <ThreadPrimitive.If running={false}>
        <ComposerPrimitive.Send asChild>
          发送
        </ComposerPrimitive.Send>
      </ThreadPrimitive.If>
      <ThreadPrimitive.If running>
        <ComposerPrimitive.Cancel asChild>
          停止
        </ComposerPrimitive.Cancel>
      </ThreadPrimitive.If>
    </>
  )
}
```

### Gpt 接口接入

```tsx
export function MyRuntimeProvider({ children }: MyRuntimeProviderProps) {
  // 使用自定义的 ai 接口请求
  const runtime = useLocalRuntime(MyModelAdapterStream)

  return (
    <AssistantRuntimeProvider runtime={runtime}>
      {children}
    </AssistantRuntimeProvider>
  )
}

const MyApp = () => {
  return (
    <MyRuntimeProvider>
      <ThreadList />
      <Thread />
    </MyRuntimeProvider>
  )
}
```

#### 流式对接 gpt 响应数据流

```ts
import { streamRequest } from '@/utils/streamRequest'
import { ChatModelAdapter } from '@assistant-ui/react'

export const MyModelAdapterStream: ChatModelAdapter = {
  async *run({ messages, abortSignal }) {
    // messages 保存了当前对话的上下文，messages[messages.length - 1] 就是最后一次即当前询问的内容
    let inputText = messages[messages.length - 1].content[0].text

    const stream = streamRequest('/openapi/v1/app/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${GPT_TOKEN}`,
        },
        body: JSON.stringify({ query: inputText, stream: true }), // 处理参数
        signal: abortSignal, // 点击取消发送时取消问答
      },
    )

    yield* stream
  },
}
```

streamRequest 流式处理：

```ts
import { ChatModelRunResult } from '@assistant-ui/react'
import { parseStreamData } from './parseStreamData'

export async function* streamRequest(
  url: string,
  options: RequestInit,
): AsyncGenerator<ChatModelRunResult> {
  const result = await fetch(url, options)
  const reader = result.body.getReader()

  let currentContent = ''
  let reasonContent = ''
  let chunks = ''

  while (true) {
    try {
      const { done, value } = await reader.read()
      // 流读取完成
      if (done) {
        yield {
          status: { type: 'complete', reason: 'stop' },
          content: [ { text: currentContent, type: 'text' } ]
        }
        break
      }

      // 处理接收到的数据片段
      const text = new TextDecoder().decode(value, { stream: true })

      chunks += text
      const result = parseStreamData(chunks)
      for (const parserRes of result) {
        if (parserRes.event === 'answer') {
          const { content, reasoning_content } = parserRes.data.delta
          currentContent += content || ''
          reasonContent += reasoning_content || ''

          yield {
            status: { type: 'running' },
            content: [{ text: currentContent, type: 'text' }],
          }
        }
      }
      chunks = ''
    } catch (error) {
      // 点击了取消
      if (error?.name === 'AbortError') {
        options.signal && options.signal.throwIfAborted()
        yield {
          status: { type: 'incomplete', reason: 'error' },
          content: [ { text: reasonContent, type: 'reasoning' } ],
        }
        break
      }
    }
  }
}
```


数据解析：

```ts
export function parseStreamData(content: string) {
  // 按行分割数据
  const chunks = content.split('\n\n').filter(Boolean)
  const result: { event: string; id: string; data: any }[] = []
  // 遍历行以解析事件类型、ID和数据

  for (const chunk of chunks) {
    let event: string
    let id: string
    let data: string
    const lines = chunk.split('\n')

    for (const line of lines) {
      if (line.startsWith('event:')) {
        event = line.replace('event:', '').trim()
      }

      if (line.startsWith('id:')) {
        id = line.replace('id:', '').trim()
      }

      if (line.startsWith('data:')) {
        try {
          data = JSON.parse(line.replace('data:', '').trim())
        } catch (error) {}
      }
    }

    result.push({ event, id, data })
  }

  return result
}
```

### 技术应用和前景

未来各种web、原生应用上会有 AI 集中爆发式应用，同时企业会积累自己的私有知识库，搭配自己私有化的 Gpt 客户端，实现企业级 AI Web、App应用，对于前端领域来说，掌握和了解 Gpt 客户端开发的技术链路非常有必要。使用 assistant-ui 可快速实现搭建企业级私有的 Gpt Web 应用，快速实现各种定制化功能。