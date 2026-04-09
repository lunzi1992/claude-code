# 第九章：Ink 终端 UI 系统

## 9.1 概述

Claude Code 使用 **Ink** 作为终端 UI 框架。Ink 是 React 的终端渲染实现，允许使用 React 组件构建 TUI。

**核心文件**：

- src/ink.ts — Ink 入口
- src/ink/reconciler.ts — 自定义 React 调和器
- src/components/ — UI 组件

## 9.2 Ink vs 传统 TUI

| 方案              | 编程模型   | 生态          | 学习曲线           |
| ----------------- | ---------- | ------------- | ------------------ |
| **Ink**     | React 组件 | React 生态    | 低（React 开发者） |
| **Blessed** | 命令式     | ClojureScript | 中                 |
| **Chalk**   | CSS-like   | 无            | 低                 |
| **ncurses** | 过程式     | C             | 高                 |

## 9.3 核心组件

### 9.3.1 App 组件

```typescript
// src/components/App.tsx
export function App() {
  return (
    <Box flexDirection="column">
      <Messages />
      <PromptInput />
      <StatusBar />
    </Box>
  )
}
```

### 9.3.2 Messages 组件

```typescript
// src/components/Messages.tsx
export function Messages({ messages }: { messages: Message[] }) {
  return (
    <Box flexDirection="column">
      {messages.map(msg => (
        <MessageRow key={msg.uuid} message={msg} />
      ))}
    </Box>
  )
}

export function MessageRow({ message }: { message: Message }) {
  if (message.type === 'user') {
    return <UserMessage message={message} />
  }
  if (message.type === 'assistant') {
    return <AssistantMessage message={message} />
  }
  return <SystemMessage message={message} />
}
```

## 9.4 Ink 核心原语

### 9.4.1 Box

```typescript
// Box = div
<Box flexDirection="column" padding={1}>
  <Text>Hello</Text>
  <Text>World</Text>
</Box>
```

### 9.4.2 Text

```typescript
// Text = span
<Text color="green" bold>
  Success!
</Text>
<Text color="red" dim>
  Error message
</Text>
```

### 9.4.3 自定义渲染

```typescript
// Ink reconciler 允许自定义渲染
import { render } from '../ink/reconciler'

const tree = render(<App />)

while (true) {
  const { output } = treecontinue
  process.stdout.write(output)
}
```

## 9.5 Hooks

### 9.5.1 useInput

```typescript
// 捕获用户输入
function PromptInput() {
  const [value, setValue] = useState('')

  useInput((input, key) => {
    if (key.return) {
      submit(value)
      setValue('')
    } else if (key.backspace) {
      setValue(prev => prev.slice(0, -1))
    } else if (input) {
      setValue(prev => prev + input)
    }
  })

  return <Text>{value}</Text>
}
```

### 9.5.2 useTerminalSize

```typescript
// 获取终端大小
function MyComponent() {
  const { columns, rows } = useTerminalSize()

  return (
    <Text>
      Terminal: {columns}x{rows}
    </Text>
  )
}
```

## 9.6 虚拟列表

```typescript
// 大量消息时的虚拟化渲染
export function VirtualMessageList({ messages }: Props) {
  const { rows } = useTerminalSize()

  // 只渲染可见行
  const visibleMessages = useMemo(() => {
    return calculateVisibleMessages(messages, rows)
  }, [messages, rows])

  return (
    <Box>
      {visibleMessages.map(msg => (
        <MessageRow key={msg.uuid} message={msg} />
      ))}
    </Box>
  )
}
```

## 9.7 ANSI 颜色支持

```typescript
// Ink 自动处理 ANSI 转义序列
<Text color="cyan">$ </Text>
<Text color="yellow">warning: </Text>
<Text color="red">error: </Text>

// 支持 256 色
<Text color="#ff6600">Orange</Text>
```

## 9.8 自定义 Reconciler

Claude Code 使用自定义的 React Reconciler 来适配终端渲染：

```typescript
// ink/reconciler.ts
export const reconciler = {
  // 挂载组件
  mountRoot(element, context) {
    // 创建根节点
    const node = createNode(element.type)
    // 渲染子树
    renderSubtree(element, node, context)
    // 输出 ANSI
    return toANSI(node)
  },

  // 更新组件
  updateInstance(element, newElement, context) {
    // 差异计算
    const diff = diffElements(element, newElement)
    // 应用变更
    applyChanges(diff, context)
    // 输出更新
    return toANSI(diff)
  },
}
```

## 9.9 与 React 的差异

| 特性     | Web React   | Ink            |
| -------- | ----------- | -------------- |
| 渲染目标 | DOM         | ANSI 转义序列  |
| 布局     | CSS Flexbox | Yoga Layout    |
| 事件     | DOM Events  | Terminal Input |
| 样式     | CSS         | 对象字面量     |
| 更新     | Virtual DOM | 直接更新       |

## 9.10 主题系统

```typescript
// src/utils/theme.ts
type Theme = {
  colors: {
    text: string
    background: string
    primary: string
    secondary: string
    error: string
    warning: string
  }
}

const themes: Record<ThemeName, Theme> = {
  light: {
    colors: { text: 'black', background: 'white', ... }
  },
  dark: {
    colors: { text: 'white', background: 'black', ... }
  }
}
```

## 9.11 组件渲染流程

````mermaid
flowchart TD

    classDef jsx fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#172554,rx:10,ry:10
    classDef reconciler fill:#f3e8ff,stroke:#6b21a8,stroke-width:3px,color:#3b0764,rx:12,ry:12
    classDef tree fill:#ecfdf5,stroke:#0f766e,stroke-width:2px,color:#134e4a,rx:10,ry:10
    classDef output fill:#fefce8,stroke:#854d0e,stroke-width:2px,color:#451a03,rx:10,ry:10

    JSX["**JSX 组件**<br/>(React Components)"]:::jsx
    R1["React.createElement()"]:::reconciler
    R2["构建 Virtual DOM 树"]:::reconciler
    R3["diff (虚拟 DOM 对比)"]:::reconciler
    R4["转换为 Ink 节点"]:::reconciler

    BOX["<Box>"]:::tree
    TEXT1["Text: 'Hello'"]:::tree
    TEXT2["Text: '$' <span style='color:cyan'>cyan</span>"]:::tree
    TEXT3["Text: 'command'"]:::tree

    ANSI["**ANSI 转义序列**<br/>```\\x1b[36m$\\x1b[0m command```"]:::output
    TERMINAL["**终端最终输出**<br/>$ command"]:::output

    JSX --> R1
    R1 --> R2 --> R3 --> R4
    R4 --> BOX
    BOX --> TEXT1
    BOX --> TEXT2
    BOX --> TEXT3
    TEXT3 --> ANSI
    ANSI --> TERMINAL
````

## 9.12 总结

| 设计点                      | 实现         | 价值             |
| --------------------------- | ------------ | ---------------- |
| **React 模型**        | Ink          | React 开发者友好 |
| **Yoga 布局**         | C 实现       | 高性能           |
| **虚拟列表**          | 可视区域计算 | 处理大量消息     |
| **主题系统**          | ThemeName    | 暗/亮模式        |
| **自定义 Reconciler** | 终端特化     | 完整控制         |
