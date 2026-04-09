# 第五章：会话引擎 (QueryEngine.ts)

## 5.1 概述

**文件位置**：src/QueryEngine.ts

**代码规模**：\~1300 行

**核心职责**：

1. 管理对话会话的完整生命周期
2. 处理用户消息提交
3. 协调 query() 执行
4. 管理文件历史快照
5. 会话归因（Attribution）
6. 压缩触发协调

## 5.2 与 query.ts 的区别

| 模块                     | 职责                  | 层级       |
| ------------------------ | --------------------- | ---------- |
| **query.ts**       | Agentic Loop 核心逻辑 | 工具执行层 |
| **QueryEngine.ts** | 会话状态编排          | 应用编排层 |

```mermaid
flowchart TD

    %% ==================== 样式定义 ====================
    classDef qe fill:#dbeafe,stroke:#1e40af,stroke-width:3px,color:#172554,rx:12,ry:12
    classDef query fill:#f3e8ff,stroke:#6b21a8,stroke-width:3px,color:#3b0764,rx:12,ry:12
    classDef node fill:#ecfdf5,stroke:#0f766e,stroke-width:2px,color:#134e4a,rx:8,ry:8

    %% ==================== QueryEngine 子图 ====================
    subgraph QE ["QueryEngine (会话管理层)"]
        QE1["管理 mutableMessages 状态"]
        QE2["处理用户输入<br/>（submitMessage）"]
        QE3["调用 query() 执行循环"]
        QE4["协调上下文压缩<br/>与归因"]
        QE5["管理 Turn 生命周期"]
    
        QE1 --> QE2 --> QE3 --> QE4 --> QE5
    end

    %% ==================== query() Agentic Loop 子图 ====================
    subgraph Q ["query() (核心 Agentic Loop)"]
        Q1["实现 Agentic Loop<br/>（while / for await）"]
        Q2["API 调用 + 工具执行<br/>（callModel + runTools）"]
        Q3["Token 预算管理<br/>与上下文构建"]
        Q4["流式事件处理<br/>（yield StreamEvent）"]
    
        Q1 --> Q2 --> Q3 --> Q4
    end

    %% ==================== 流程连接 ====================
    QE3 -->|"submitMessage()"| Q1
    Q4 -->|"yield events<br/>（实时更新 UI）"| QE

    %% ==================== Agentic 循环反馈（虚线） ====================
    Q2 -.->|"工具结果回传<br/>继续下一轮循环"| Q2

    %% ==================== 应用样式 ====================
    class QE qe
    class Q query
    class QE1,QE2,QE3,QE4,QE5,Q1,Q2,Q3,Q4 node
```

## 5.3 核心类结构

```typescript
export class QueryEngine {
  // 配置
  private config: QueryEngineConfig

  // 会话状态
  private mutableMessages: Message[]      // 可变的消息历史
  private abortController: AbortController // 中断控制器
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录

  // 统计
  private totalUsage: NonNullableUsage    // API 使用统计

  // 文件状态
  private readFileState: FileStateCache   // 文件读取缓存

  // 技能发现
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()

  // 核心方法
  async submitMessage(userMessage: Message): Promise<SubmitMessageResult>
  private handleQueryEvents(event: StreamEvent | Message): Promise<void>
  private afterQueryLoopCleanup(): Promise<void>
}
```

## 5.4 配置结构

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
  handleElicitation?: ToolUseContext['handleElicitation']
  includePartialMessages?: boolean
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission
  snipReplay?: (yieldedSystemMsg: Message, store: Message[]) => ...
}
```

## 5.5 submitMessage 流程

```typescript
async submitMessage(userMessage: Message): Promise<SubmitMessageResult> {
  // 1. 追加用户消息
  this.mutableMessages.push(userMessage)

  // 2. 初始化或恢复状态
  if (!this.abortController) {
    this.abortController = new AbortController()
  }

  // 3. 检查 turn 限制
  const currentTurn = this.getCurrentTurn()
  if (this.config.maxTurns && currentTurn >= this.config.maxTurns) {
    return { reason: 'max_turns' }
  }

  // 4. 构建查询参数
  const queryParams = this.buildQueryParams(userMessage)

  // 5. 调用 query() 并处理事件
  for await (const event of query(queryParams)) {
    await this.handleQueryEvents(event)
  }

  // 6. 后处理
  await this.afterQueryLoopCleanup()

  return { reason: 'done' }
}
```

## 5.6 事件处理

### 5.6.1 消息事件

```typescript
private async handleQueryEvents(event: StreamEvent | Message): Promise<void> {
  if (isTombstoneMessage(event)) {
    // 墓碑消息 - 删除孤儿消息
    this.removeMessage(event.message.uuid)
  } else if (isToolUseSummaryMessage(event)) {
    // 工具使用摘要
    this.handleToolUseSummary(event)
  } else if (isCompactBoundaryMessage(event)) {
    // 压缩边界
    this.handleCompactBoundary(event)
  } else if (isProgressMessage(event)) {
    // 进度消息 - 更新 UI
    this.updateProgress(event)
  } else if (isAssistantMessage(event)) {
    // 助手消息
    this.handleAssistantMessage(event)
  } else if (isUserMessage(event)) {
    // 用户消息（通常是工具结果）
    this.handleUserMessage(event)
  }
}
```

### 5.6.2 消息添加

```typescript
private addMessage(msg: Message): void {
  // 检查重复
  if (this.mutableMessages.some(m => m.uuid === msg.uuid)) {
    return
  }

  // 添加消息
  this.mutableMessages.push(msg)

  // 更新 AppState
  this.config.setAppState(prev => ({
    ...prev,
    messages: [...this.mutableMessages],
  }))
}
```

## 5.7 Turn 管理

### 5.7.1 Turn 概念

一个 **Turn** 是指用户发送一条消息，AI 响应并可能执行多轮工具调用的完整过程：

```mermaid
sequenceDiagram
    participant U as 用户
    participant System as 系统 (REPL + QueryEngine)
    participant AI as Claude (LLM)
    participant Tool as 工具层 (Read/Edit/Bash...)

    Note over U,Tool: Turn 1

    U->>System: "帮我修一下这个 bug"
    System->>AI: 发送消息 + 上下文
    AI->>Tool: Read(file.ts)
    Tool-->>AI: 文件内容
    AI->>Tool: Edit(file.ts)
    Tool-->>AI: 编辑结果
    AI->>Tool: Bash("npm test")
    Tool-->>AI: 测试输出
    AI-->>System: 最终修复结果
    System-->>U: 显示修复说明和测试结果

    Note over U,Tool: Turn 2

    U->>System: "测试通过了，很好"
    System->>AI: 发送用户反馈
    AI-->>System: "很高兴帮到你！"
    System-->>U: 显示回复
```

### 5.7.2 Turn 状态追踪

```typescript
// QueryEngine 内部追踪
private currentTurnId: string
private turnCounter: number = 0
private turnsSinceLastCompact: number = 0

// 每个 turn 开始时
startTurn() {
  this.turnId = generateUUID()
  this.turnCounter++
  this.turnsSinceLastCompact++
}

// turn 结束时
endTurn() {
  // 检查是否需要压缩
  if (this.turnsSinceLastCompact >= AUTO_COMPACT_THRESHOLD) {
    this.triggerCompaction()
  }
}
```

## 5.8 文件历史快照

### 5.8.1 为什么要快照

Claude Code 需要追踪对话期间文件的变化，以便：

1. 支持 `/diff` 命令查看变更
2. 支持 `/clear` 清除历史但保留文件状态
3. 支持会话恢复

### 5.8.2 快照机制

```typescript
private async updateFileHistory(change: FileChange): Promise<void> {
  const snapshot = fileHistoryMakeSnapshot(
    this.config.readFileCache,
    change
  )

  // 保存快照
  await saveFileHistorySnapshot(this.sessionId, snapshot)

  // 更新状态
  this.config.setAppState(prev => ({
    ...prev,
    fileHistoryState: {
      snapshots: [...prev.fileHistoryState.snapshots, snapshot],
      currentIndex: prev.fileHistoryState.snapshots.length,
    },
  }))
}
```

### 5.8.3 文件变更类型

```typescript
type FileChange =
  | { type: 'edit'; path: string; before: string; after: string }
  | { type: 'create'; path: string; content: string }
  | { type: 'delete'; path: string }
  | { type: 'read'; path: string; content: string }
```

## 5.9 会话归因 (Attribution)

### 5.9.1 归因目的

归因系统追踪**每行代码是谁写的**：

- 用户直接编写
- AI 根据用户指示编写
- AI 自主编写（无明确指示）

### 5.9.2 归因状态

```typescript
type AttributionState = {
  // 当前 turn 的归因
  currentTurn: {
    userLines: number
    aiLines: number
    aiAutonomousLines: number
  }

  // 累计归因
  total: {
    userLines: number
    aiLines: number
    aiAutonomousLines: number
  }
}
```

### 5.9.3 归因计算

```typescript
private computeAttribution(): AttributionState {
  // 分析 diff 和变更
  const diffs = extractDiffs(this.mutableMessages)

  let userLines = 0
  let aiLines = 0
  let aiAutonomousLines = 0

  for (const diff of diffs) {
    if (diff.source === 'user') {
      userLines += diff.linesAdded
    } else if (diff.source === 'ai') {
      if (diff.isAutonomous) {
        aiAutonomousLines += diff.linesAdded
      } else {
        aiLines += diff.linesAdded
      }
    }
  }

  return {
    currentTurn: { userLines, aiLines, aiAutonomousLines },
    total:累加(this.totalAttribution, { userLines, aiLines, aiAutonomousLines }),
  }
}
```

## 5.10 压缩协调

### 5.10.1 压缩触发

QueryEngine 协调多种压缩机制：

```typescript
private async checkCompaction(): Promise<void> {
  // 1. 检查 token 预算
  const { isAtBlockingLimit, shouldAutoCompact } = calculateTokenWarningState(
    this.estimateTokenCount(),
    this.config.model
  )

  if (isAtBlockingLimit) {
    // 需要压缩才能继续
    await this.triggerCompaction('blocking')
  } else if (shouldAutoCompact) {
    // 可以压缩（预防性）
    await this.triggerCompaction('preventive')
  }
}
```

### 5.10.2 压缩类型

| 类型                    | 触发条件           | 说明                 |
| ----------------------- | ------------------ | -------------------- |
| **auto-compact**  | token 超过阈值     | 完整压缩，生成摘要   |
| **micro-compact** | 缓存编辑           | 只压缩缓存相关的编辑 |
| **snip**          | HISTORY\_SNIP flag | 裁剪历史片段         |

## 5.11 权限拒绝追踪

```typescript
// 追踪权限拒绝以实现降级
private permissionDenials: SDKPermissionDenial[] = []

recordDenial(denial: SDKPermissionDenial) {
  this.permissionDenials.push({
    ...denial,
    timestamp: Date.now(),
  })

  // 如果拒绝次数过多，切换到提示模式
  if (this.permissionDenials.length >= DENIAL_THRESHOLD) {
    this.config.setAppState(prev => ({
      ...prev,
      toolPermissionContext: {
        ...prev.toolPermissionContext,
        mode: 'auto',  // 降级到自动模式
      },
    }))
  }
}
```

## 5.12 子代理上下文

### 5.12.1 上下文创建

```typescript
createSubagentContext(agentId: AgentId): ToolUseContext {
  // 从父上下文克隆，但覆盖特定字段
  return {
    ...this.baseToolUseContext,

    // 子代理特有
    agentId,
    agentType: 'fork',  // 或 'async', 'background'

    // 隔离的消息历史
    messages: [],

    // 独立的 abort controller
    abortController: new AbortController(),

    // 共享的内容替换状态
    contentReplacementState: this.baseToolUseContext.contentReplacementState,
  }
}
```

### 5.12.2 上下文差异

| 字段                    | 主会话    | 子代理         |
| ----------------------- | --------- | -------------- |
| messages                | 完整历史  | 空（独立历史） |
| abortController         | 共享      | 独立           |
| agentId                 | undefined | 生成新 ID      |
| contentReplacementState | 共享      | 克隆           |

## 5.13 会话恢复

### 5.13.1 保存状态

```typescript
async saveSession(): Promise<SessionSnapshot> {
  return {
    sessionId: this.sessionId,
    conversationId: this.conversationId,
    messages: this.mutableMessages,
    fileHistory: this.fileHistorySnapshots,
    attribution: this.computeAttribution(),
    totalUsage: this.totalUsage,
    createdAt: this.createdAt,
    lastActivity: Date.now(),
  }
}
```

### 5.13.2 恢复状态

```typescript
async restoreSession(snapshot: SessionSnapshot): Promise<void> {
  this.sessionId = snapshot.sessionId
  this.conversationId = snapshot.conversationId
  this.mutableMessages = snapshot.messages
  this.fileHistorySnapshots = snapshot.fileHistory
  this.totalUsage = snapshot.totalUsage

  // 重建 AppState
  this.config.setAppState(prev => ({
    ...prev,
    messages: this.mutableMessages,
    fileHistoryState: this.rebuildFileHistoryState(),
  }))
}
```

## 5.14 与其他模块的交互

```mermaid
flowchart TD

    %% ==================== 样式定义 ====================
    classDef ui fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#172554,rx:10,ry:10
    classDef qe fill:#fef3c7,stroke:#b45309,stroke-width:2px,color:#451a03,rx:10,ry:10
    classDef core fill:#ecfdf5,stroke:#0f766e,stroke-width:2px,color:#134e4a,rx:10,ry:10
    classDef state fill:#f3e8ff,stroke:#6b21a8,stroke-width:2px,color:#3b0764,rx:10,ry:10
    classDef tool fill:#fee2e9,stroke:#9f1239,stroke-width:2px,color:#431407,rx:10,ry:10
    classDef api fill:#ecfeff,stroke:#155e75,stroke-width:2px,color:#164e63,rx:10,ry:10
    classDef file fill:#fefce8,stroke:#854d0e,stroke-width:2px,color:#451a03,rx:10,ry:10

    %% ==================== 各层子图 ====================
    subgraph UI ["REPL.tsx (表示层)"]
        UI1["UI 渲染 (Ink)"]
        UI2["用户输入处理"]
        UI1 --> UI2
    end

    subgraph QE ["QueryEngine (应用层)"]
        QE1["管理 mutableMessages 状态"]
        QE2["调用 query() 执行循环"]
        QE3["协调压缩和归因"]
        QE1 --> QE2 --> QE3
    end

    subgraph CORE ["query.ts (核心层)"]
        CORE1["Agentic Loop 核心循环"]
        CORE2["工具执行调度"]
        CORE1 --> CORE2
    end

    subgraph STATE ["AppState (Zustand)"]
        STATE1["全局状态存储"]
        STATE2["Zustand Store"]
        STATE1 --> STATE2
    end

    subgraph TOOL ["Tool.ts (工具层)"]
        TOOL1["工具接口定义 & 实现"]
    end

    subgraph API ["services/api (服务层)"]
        API1["多 Provider API 调用"]
    end

    subgraph FILE ["FileHistory"]
        FILE1["快照管理 & 历史记录"]
    end

    %% ==================== 连接关系（从上到下逻辑流） ====================
    UI --> QE
    QE --> CORE
    QE --> STATE
    CORE --> TOOL
    CORE --> API
    API --> FILE
    STATE <--> QE
    STATE <--> UI

    %% ==================== 应用样式 ====================
    class UI ui
    class QE qe
    class CORE core
    class STATE state
    class TOOL tool
    class API api
    class FILE file
    class UI1,UI2,QE1,QE2,QE3,CORE1,CORE2,STATE1,STATE2,TOOL1,API1,FILE1 node
```

## 5.15 总结

QueryEngine 的核心设计要点：

| 设计点               | 实现                    | 价值                   |
| -------------------- | ----------------------- | ---------------------- |
| **状态封装**   | mutableMessages 私有    | 数据封装，防止意外修改 |
| **事件驱动**   | handleQueryEvents       | 统一处理各类事件       |
| **Turn 追踪**  | turnCounter + turnId    | 精确控制对话轮次       |
| **文件快照**   | fileHistoryMakeSnapshot | 支持 diff 和回滚       |
| **归因系统**   | computeAttribution      | 代码所有权追踪         |
| **压缩协调**   | checkCompaction         | Token 预算控制         |
| **权限降级**   | permissionDenials 追踪  | 优雅降级               |
| **子代理隔离** | createSubagentContext   | 独立又共享状态         |
