# 第六章：工具系统 (Tool.ts + tools.ts)

## 6.1 概述

工具系统是 Claude Code **Agentic 能力**的核心。它让 AI 能够执行实际操作而非仅仅生成文本。

**核心文件**：
- src/Tool.ts — 工具接口定义
- src/tools.ts — 工具注册表

**工具实现目录**：`src/tools/<ToolName>/`

## 6.2 工具接口

### 6.2.1 Tool 类型定义

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 基础信息
  name: string
  aliases?: string[]
  searchHint?: string  // 关键词提示，用于 ToolSearch

  // 输入输出
  inputSchema: Input
  outputSchema?: z.ZodType<unknown>
  inputJSONSchema?: ToolInputJSONSchema

  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  description(
    input: z.infer<Input>,
    options: {...}
  ): Promise<string>

  prompt(options: {...}): Promise<string>

  // 工具能力声明
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  isEnabled(): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // 权限
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext
  ): Promise<PermissionResult>

  // UI 渲染
  renderToolResultMessage(content: Output, ...): React.ReactNode
  renderToolUseMessage(input: Partial<Input>, ...): React.ReactNode
  renderToolUseProgressMessage?(...): React.ReactNode
  renderToolUseRejectedMessage?(...): React.ReactNode
  renderToolUseErrorMessage?(...): React.ReactNode

  // 元数据
  maxResultSizeChars: number
  isMcp?: boolean
  isLsp?: boolean
  shouldDefer?: boolean
  alwaysLoad?: boolean
}
```

### 6.2.2 工具能力声明

```typescript
// 1. 是否并发安全
isConcurrencySafe(input: ReadInput): boolean {
  // Bash: 并发执行可能有问题
  return false
}

// 2. 是否只读操作
isReadOnly(input: ReadInput): boolean {
  return true  // 只读取文件，不修改
}

// 3. 是否破坏性操作
isDestructive?(input: WriteInput): boolean {
  return input.overwrite === true  // 覆盖文件是危险的
}

// 4. 中断行为
interruptBehavior?(): 'cancel' | 'block' {
  return 'block'  // 新消息等待当前工具完成
}
```

## 6.3 工具注册表

### 6.3.1 getTools 函数

```typescript
// tools.ts
export function getTools(): Tools {
  return [
    // 文件操作
    BashTool,
    FileEditTool,
    FileReadTool,
    FileWriteTool,
    GlobTool,
    NotebookEditTool,

    // 网络
    WebFetchTool,
    WebSearchTool,

    // 子代理
    AgentTool,

    // 任务
    TaskStopTool,
    TaskOutputTool,
    TodoWriteTool,

    // 定时
    CronCreateTool,
    CronDeleteTool,
    CronListTool,

    // 其他
    AskUserQuestionTool,
    BriefTool,
    EnterPlanModeTool,
    ExitPlanModeV2Tool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    // ... 更多
  ]
}
```

### 6.3.2 工具分类

| 类别 | 工具 | 功能 |
|------|------|------|
| **文件读写** | FileReadTool, FileWriteTool, FileEditTool | 文件 CRUD |
| **代码搜索** | GrepTool, GlobTool | 代码定位 |
| **Shell 执行** | BashTool | 命令行操作 |
| **网络** | WebFetchTool, WebSearchTool | Web 访问 |
| **子代理** | AgentTool | 会话派生 |
| **任务管理** | TodoWriteTool, TaskCreateTool 等 | 任务跟踪 |
| **定时任务** | CronCreateTool, CronDeleteTool, CronListTool | 定时执行 |
| **其他** | AskUserQuestionTool, SkillTool 等 | 特殊功能 |

## 6.4 工具构建器

### 6.4.1 buildTool 工厂函数

```typescript
// Tool.ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input, _ctx) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**设计意图**：提供合理的默认值，减少工具实现者的样板代码。

### 6.4.2 工具定义示例

```typescript
// 简化版的 ReadTool 定义
const ReadToolDef = {
  name: 'Read',
  inputSchema: z.object({
    file_path: z.string(),
    limit: z.number().optional(),
    offset: z.number().optional(),
  }),

  isReadOnly: () => true,
  isConcurrencySafe: () => true,

  async call(args, context, canUseTool) {
    // 读取文件逻辑
    const content = await readFile(args.file_path)
    return { data: { content, path: args.file_path } }
  },

  async description(input) {
    return `Read the contents of ${input.file_path}`
  },
}

export const ReadTool = buildTool(ReadToolDef)
```

## 6.5 核心工具解析

### 6.5.1 BashTool

**功能**：在终端执行 Shell 命令

```typescript
class BashTool {
  name = 'Bash'
  maxResultSizeChars = 100_000

  async call(args, context, canUseTool) {
    // 1. 权限检查
    const permission = await canUseTool({
      tool: 'Bash',
      input: args,
      context,
    })

    if (permission.behavior === 'deny') {
      return { data: { denied: true, reason: permission.reason } }
    }

    // 2. 执行命令
    const result = await executeBash({
      command: args.command,
      cwd: context.options.cwd,
      timeout: args.timeout,
      environment: args.environment,
    })

    // 3. 返回结果
    return {
      data: {
        stdout: result.stdout,
        stderr: result.stderr,
        exitCode: result.exitCode,
      }
    }
  }
}
```

**安全机制**：
- 沙箱执行（可选）
- 路径验证
- 超时控制
- 环境变量过滤

### 6.5.2 FileEditTool

**功能**：通过字符串替换编辑文件

```typescript
class FileEditTool {
  name = 'Edit'

  async call(args, context, canUseTool) {
    // 1. 读取原文件
    const originalContent = await readFile(args.file_path)

    // 2. 验证替换目标存在
    if (!originalContent.includes(args.old_string)) {
      throw new Error(`old_string not found in file`)
    }

    // 3. 执行替换
    const newContent = originalContent.replace(
      args.old_string,
      args.new_string
    )

    // 4. 写入文件
    await writeFile(args.file_path, newContent)

    // 5. 记录 diff 用于 UI
    return {
      data: {
        path: args.file_path,
        diff: {
          before: args.old_string,
          after: args.new_string,
        }
      }
    }
  }
}
```

**特点**：
- 精确字符串匹配（不会误改其他内容）
- 自动创建备份
- Diff 追踪用于 UI 显示

### 6.5.3 AgentTool

**功能**：派生新的子代理会话

```typescript
class AgentTool {
  name = 'Agent'

  async call(args, context, canUseTool) {
    // 创建子代理
    const subagent = await createSubagent({
      type: args.agentType || 'fork',  // fork, async, background
      prompt: args.prompt,
      tools: resolveAgentTools(args.agentType),
      parentContext: context,
    })

    // 如果是同步等待结果
    if (args.mode === 'wait') {
      const result = await subagent.waitForCompletion()
      return { data: result }
    }

    // 如果是异步立即返回
    return {
      data: {
        taskId: subagent.id,
        status: 'running',
      }
    }
  }
}
```

**子代理类型**：

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| `fork` | 从当前状态分叉，独立执行 | 并行研究 |
| `async` | 异步执行，结果稍后获取 | 后台任务 |
| `background` | 完全后台运行 | 长时任务 |
| `remote` | 远程机器执行 | 资源密集型 |

### 6.5.4 WebFetchTool

**功能**：获取网页内容并转换为 Markdown

```typescript
class WebFetchTool {
  name = 'WebFetch'

  async call(args, context, canUseTool) {
    // 1. 发送 HTTP 请求
    const response = await fetch(args.url, {
      headers: args.headers,
    })

    // 2. 获取 HTML 内容
    const html = await response.text()

    // 3. 转换为 Markdown
    const markdown = await turndown(html)

    // 4. 可选：AI 摘要
    const summary = args.max_length
      ? await summarize(markdown, args.max_length)
      : markdown

    return {
      data: {
        content: summary,
        url: args.url,
        title: extractTitle(html),
      }
    }
  }
}
```

## 6.6 工具执行上下文

### 6.6.1 ToolUseContext

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]           // 可用斜杠命令
    debug: boolean                // 调试模式
    mainLoopModel: string         // 当前模型
    tools: Tools                  // 可用工具列表
    verbose: boolean              // 详细输出
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  setToolJSX?: SetToolJSXFn       // 设置工具 UI
  addNotification?: (notif: Notification) => void
  sendOSNotification?: (opts) => void
  // ... 更多
}
```

## 6.7 工具与 UI

### 6.7.1 渲染接口

每个工具可以定义多个渲染方法：

```typescript
// 工具输入渲染（显示 AI 要执行的操作）
renderToolUseMessage(
  input: Partial<Input>,
  options: { theme: ThemeName; verbose: boolean; commands?: Command[] }
): React.ReactNode

// 工具结果渲染（显示执行结果）
renderToolResultMessage(
  content: Output,
  progressMessagesForMessage: ProgressMessage<P>[],
  options: {...}
): React.ReactNode

// 进度渲染（执行中显示）
renderToolUseProgressMessage?(...): React.ReactNode

// 拒绝渲染（权限被拒绝时）
renderToolUseRejectedMessage?(...): React.ReactNode

// 错误渲染（执行出错时）
renderToolUseErrorMessage?(...): React.ReactNode
```

## 6.8 工具搜索 (ToolSearch)

### 6.8.1 延迟加载

```typescript
// 工具可以标记为延迟加载
const WebSearchTool = {
  name: 'WebSearch',
  shouldDefer: true,  // 不在初始工具列表中
  // ...
}
```

### 6.8.2 ToolSearch 流程

```
1. AI 决定需要搜索工具
2. 调用 ToolSearch("web search")
3. ToolSearch 返回匹配的延迟工具
4. AI 决定使用该工具
5. 工具被加载并执行
```

## 6.9 工具权限

### 6.9.1 权限检查流程

```typescript
async function executeToolWithPermission(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  canUseTool: CanUseToolFn
) {
  // 1. 验证输入
  const validation = await tool.validateInput?.(input, context)
  if (!validation.result) {
    throw new Error(validation.message)
  }

  // 2. 检查权限
  const permission = await canUseTool({
    tool: tool.name,
    input,
    context,
  })

  if (permission.behavior === 'deny') {
    return {
      data: { denied: true, reason: permission.reason }
    }
  }

  // 3. 执行工具
  return await tool.call(input, context, canUseTool)
}
```

### 6.9.2 权限模式

| 模式 | 说明 |
|------|------|
| `auto` | 自动允许/拒绝 |
| `manual` | 每次都询问 |
| `plan` | 计划模式下更宽松 |
| `yolo` | 无确认直接执行 |

## 6.10 MCP 工具集成

### 6.10.1 MCP 工具包装

```typescript
// 将 MCP 工具适配到 Tool 接口
class MCPToolAdapter implements Tool {
  constructor(
    private serverName: string,
    private mcpTool: MCPTool
  ) {}

  get name() {
    return `mcp__${this.serverName}__${this.mcpTool.name}`
  }

  async call(args, context, canUseTool) {
    // 调用 MCP 服务器
    const result = await this.mcpTool.call(args)
    return { data: result }
  }

  // ...
}
```

### 6.10.2 工具注册

```typescript
// 在 tools.ts 中
export async function getTools(): Promise<Tools> {
  const builtInTools = [
    BashTool,
    FileEditTool,
    // ...
  ]

  // 加载 MCP 工具
  const mcpTools = await loadMcpTools(mcpClients)

  // 合并
  return [...builtInTools, ...mcpTools]
}
```

## 6.11 工具执行编排

### 6.11.1 串行执行

```typescript
// 一个接一个执行
for (const toolCall of toolCalls) {
  const result = await executeTool(toolCall)
  messages.push(createToolResultMessage(result))
}
```

### 6.11.2 并行执行

```typescript
// 多个并发安全的工具并行执行
const concurrencySafeCalls = toolCalls.filter(
  t => t.tool.isConcurrencySafe(t.input)
)

const results = await Promise.all(
  concurrencySafeCalls.map(call => executeTool(call))
)
```

### 6.11.3 流式执行

```typescript
// StreamingToolExecutor
class StreamingToolExecutor {
  addTool(toolBlock: ToolUseBlock, assistantMessage: AssistantMessage) {
    // 立即开始执行
    this.executeTool(toolBlock).then(result => {
      this.completedResults.push({ toolBlock, result })
    })
  }

  getCompletedResults() {
    return this.completedResults.splice(0)
  }
}
```

## 6.12 总结

| 设计点 | 实现 | 价值 |
|--------|------|------|
| **统一接口** | Tool type | 所有工具一致性 |
| **能力声明** | isReadOnly/isConcurrencySafe | 安全和并行优化 |
| **工厂模式** | buildTool | 减少样板代码 |
| **分层渲染** | renderToolXXX | 灵活的 UI |
| **权限抽象** | canUseTool 回调 | 集中权限管理 |
| **MCP 适配** | MCPToolAdapter | 扩展性 |
| **流式执行** | StreamingToolExecutor | 减少等待 |
