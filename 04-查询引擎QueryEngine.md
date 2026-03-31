# 第4章：查询引擎（QueryEngine）

## 概述

QueryEngine 是 Claude Code 的核心组件，负责管理整个查询生命周期和会话状态。它将原本在 `ask()` 函数中的核心逻辑提取为一个独立的类，可同时用于 headless/SDK 路径和 REPL 交互模式。

每个会话对应一个 QueryEngine 实例，每次 `submitMessage()` 调用都会在同一会话中启动一个新的轮次（turn）。消息状态、文件缓存、使用量统计等在轮次之间持久保存。

## 4.1 引擎架构设计

### 4.1.1 核心类结构

```
┌─────────────────────────────────────────────────────────────────┐
│                        QueryEngine                               │
├─────────────────────────────────────────────────────────────────┤
│ - config: QueryEngineConfig                                      │
│ - mutableMessages: Message[]                                     │
│ - abortController: AbortController                               │
│ - permissionDenials: SDKPermissionDenial[]                       │
│ - totalUsage: NonNullableUsage                                   │
│ - readFileState: FileStateCache                                  │
│ - discoveredSkillNames: Set<string>                              │
│ - loadedNestedMemoryPaths: Set<string>                           │
├─────────────────────────────────────────────────────────────────┤
│ + submitMessage(prompt, options): AsyncGenerator<SDKMessage>     │
│ + interrupt(): void                                              │
│ + getMessages(): readonly Message[]                              │
│ + getReadFileState(): FileStateCache                             │
│ + getSessionId(): string                                         │
│ + setModel(model: string): void                                  │
└─────────────────────────────────────────────────────────────────┘
          │
          │ uses
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         query.ts                                 │
├─────────────────────────────────────────────────────────────────┤
│ query(params: QueryParams): AsyncGenerator<Message>              │
│ queryLoop(params, consumedCommandUuids): AsyncGenerator          │
└─────────────────────────────────────────────────────────────────┘
          │
          │ calls
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  services/api/claude.ts                          │
├─────────────────────────────────────────────────────────────────┤
│ queryModelWithStreaming(): AsyncGenerator                        │
│ withRetry(): AsyncGenerator<SystemAPIErrorMessage, T>            │
└─────────────────────────────────────────────────────────────────┘
```

### 4.1.2 QueryEngineConfig 配置结构

```typescript
type QueryEngineConfig = {
  // 基础配置
  cwd: string                          // 工作目录
  tools: Tools                         // 可用工具列表
  commands: Command[]                  // 斜杠命令列表
  mcpClients: MCPServerConnection[]    // MCP 服务器连接
  agents: AgentDefinition[]            // Agent 定义

  // 权限与状态
  canUseTool: CanUseToolFn             // 工具权限检查函数
  getAppState: () => AppState          // 获取应用状态
  setAppState: (f: (prev: AppState) => AppState) => void

  // 消息与缓存
  initialMessages?: Message[]          // 初始消息列表
  readFileCache: FileStateCache        // 文件状态缓存

  // 提示词配置
  customSystemPrompt?: string          // 自定义系统提示词
  appendSystemPrompt?: string          // 追加系统提示词

  // 模型配置
  userSpecifiedModel?: string          // 用户指定模型
  fallbackModel?: string               // 回退模型
  thinkingConfig?: ThinkingConfig      // 思考模式配置

  // 限制配置
  maxTurns?: number                    // 最大轮次
  maxBudgetUsd?: number                // 预算上限（美元）
  taskBudget?: { total: number }       // 任务预算

  // 输出配置
  jsonSchema?: Record<string, unknown> // JSON Schema 输出格式
  verbose?: boolean                    // 详细模式

  // SDK 特定配置
  replayUserMessages?: boolean         // 重放用户消息
  handleElicitation?: ToolUseContext['handleElicitation']
  includePartialMessages?: boolean     // 包含部分消息
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission
  snipReplay?: (yieldedSystemMsg, store) => {...} | undefined
}
```

### 4.1.3 设计原则

1. **单例会话**：每个 QueryEngine 实例对应一个完整的对话会话
2. **状态持久化**：消息、文件缓存、使用量在轮次间保持
3. **可中断性**：通过 AbortController 支持用户中断
4. **流式处理**：使用 AsyncGenerator 实现流式响应

## 4.2 请求处理流程

### 4.2.1 整体流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         submitMessage() 入口                              │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. 初始化阶段                                                             │
│    - 设置工作目录 (setCwd)                                                │
│    - 检查会话持久化设置                                                    │
│    - 包装 canUseTool 以追踪权限拒绝                                        │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 2. 系统提示词构建                                                         │
│    - fetchSystemPromptParts() 获取默认提示词                              │
│    - 合并自定义/追加提示词                                                 │
│    - 加载 memory-mechanics 提示词（如适用）                                │
│    - 注册结构化输出强制钩子                                                │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 3. 用户输入处理                                                           │
│    - processUserInput() 处理斜杠命令                                      │
│    - 推送新消息到 mutableMessages                                         │
│    - 持久化用户消息到会话存储                                              │
│    - 更新工具权限上下文                                                    │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 4. 技能/插件加载                                                          │
│    - getSlashCommandToolSkills() 获取技能                                 │
│    - loadAllPluginsCacheOnly() 加载插件                                   │
│    - 生成 system_init 消息                                                │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 5. 查询循环 (query())                                                     │
│    - 进入主查询循环                                                       │
│    - 处理 API 响应和工具调用                                               │
│    - 流式 yield 消息                                                      │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 6. 结果生成                                                               │
│    - 检查成功/失败状态                                                     │
│    - 生成 result 消息                                                     │
│    - 刷新会话存储                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2.2 消息处理状态机

```
                    ┌─────────────┐
                    │   START     │
                    └─────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  shouldQuery?         │
              │  (检查是否需要API调用) │
              └───────────────────────┘
                    /           \
                  Yes            No
                  /               \
                 ▼                 ▼
        ┌────────────┐    ┌─────────────────┐
        │ query()    │    │ 返回本地命令结果  │
        │ 循环       │    │ yield result     │
        └────────────┘    └─────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 消息类型分发        │
    └─────────────────────┘
          │    │    │
          ▼    ▼    ▼
     ┌────┐┌────┐┌──────────┐
     │user││asst││system/   │
     │    ││    ││attachment│
     └────┘└────┘└──────────┘
          │    │    │
          ▼    ▼    ▼
     ┌─────────────────────┐
     │ normalizeMessage()  │
     │ 转换为 SDKMessage   │
     └─────────────────────┘
              │
              ▼
        ┌─────────────┐
        │ yield SDK   │
        │ Message     │
        └─────────────┘
```

### 4.2.3 核心代码路径

**submitMessage 方法的核心逻辑**：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
  // 1. 初始化
  this.discoveredSkillNames.clear()
  setCwd(cwd)
  const startTime = Date.now()

  // 2. 构建 canUseTool 包装器（追踪权限拒绝）
  const wrappedCanUseTool: CanUseToolFn = async (...) => {
    const result = await canUseTool(...)
    if (result.behavior !== 'allow') {
      this.permissionDenials.push({...})
    }
    return result
  }

  // 3. 获取系统提示词
  const { defaultSystemPrompt, userContext, systemContext } =
    await fetchSystemPromptParts({...})

  // 4. 处理用户输入
  const { messages: messagesFromUserInput, shouldQuery, ... } =
    await processUserInput({...})

  // 5. 加载技能和插件
  const [skills, { enabled: enabledPlugins }] = await Promise.all([...])

  // 6. 进入查询循环
  for await (const message of query({...})) {
    // 处理各类消息并 yield
    switch (message.type) {
      case 'assistant': ...
      case 'user': ...
      case 'progress': ...
      case 'stream_event': ...
      case 'attachment': ...
      case 'system': ...
    }
  }

  // 7. 生成最终结果
  yield { type: 'result', subtype: 'success', ... }
}
```

## 4.3 工具调用机制

### 4.3.1 工具调用架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           API 响应                                        │
│                    (包含 tool_use blocks)                                 │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     StreamingToolExecutor                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ - tools: TrackedTool[]                                               │  │
│  │ - toolUseContext: ToolUseContext                                     │  │
│  │ - siblingAbortController: AbortController                            │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │ + addTool(block, assistantMessage): void                             │  │
│  │ + getCompletedResults(): Generator<MessageUpdate>                    │  │
│  │ + getRemainingResults(): AsyncGenerator<MessageUpdate>               │  │
│  │ + discard(): void                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
            │ Concurrent  │ │ Concurrent  │ │ Non-Concurrent│
            │ Safe Tool   │ │ Safe Tool   │ │ Tool         │
            │ (Read)      │ │ Tool (Glob) │ │ (Bash/Edit)  │
            └─────────────┘ └─────────────┘ └─────────────┘
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        runToolUse()                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 1. 权限检查 (canUseTool)                                             │  │
│  │ 2. 输入验证 (validateInput)                                          │  │
│  │ 3. 执行工具 (tool.call)                                              │  │
│  │ 4. 生成 tool_result 消息                                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.3.2 工具并发控制

StreamingToolExecutor 实现了智能的并发控制策略：

```typescript
class StreamingToolExecutor {
  private tools: TrackedTool[] = []

  // 检查工具是否可以执行
  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executingTools = this.tools.filter(t => t.status === 'executing')
    return (
      executingTools.length === 0 ||
      // 并发安全的工具可以与其他并发安全工具同时执行
      (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
    )
  }

  // 处理队列
  private async processQueue(): Promise<void> {
    for (const tool of this.tools) {
      if (tool.status !== 'queued') continue

      if (this.canExecuteTool(tool.isConcurrencySafe)) {
        await this.executeTool(tool)
      } else {
        // 非并发工具必须等待，保持顺序
        if (!tool.isConcurrencySafe) break
      }
    }
  }
}
```

**工具状态转换**：

```
  queued ──────► executing ──────► completed ──────► yielded
     │               │                  │
     │               │                  │
     └───────────────┴──────────────────┘
              (可被中断)
                    │
                    ▼
              synthetic error
```

### 4.3.3 工具定义结构

```typescript
type Tool<Input = AnyObject, Output = unknown, P = ToolProgressData> = {
  // 基础属性
  name: string
  aliases?: string[]
  searchHint?: string

  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // Schema 定义
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>

  // 行为控制
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // 权限检查
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>

  // 结果处理
  maxResultSizeChars: number
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  renderToolResultMessage?(...): React.ReactNode
  renderToolUseMessage(input, options): React.ReactNode

  // 其他方法
  userFacingName(input): string
  getToolUseSummary?(input): string | null
  getActivityDescription?(input): string | null
  toAutoClassifierInput(input): unknown
}
```

### 4.3.4 工具执行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     runToolUse() 执行流程                            │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
               ┌────────────────────────────────┐
               │ 1. 解析和验证输入               │
               │ inputSchema.safeParse(input)   │
               └────────────────────────────────┘
                                 │
                     ┌───────────┴───────────┐
                     ▼                       ▼
              解析成功                   解析失败
                 │                          │
                 ▼                          ▼
    ┌────────────────────────┐   ┌──────────────────┐
    │ 2. 检查工具是否启用     │   │ 返回错误结果      │
    │ tool.isEnabled()       │   └──────────────────┘
    └────────────────────────┘
                 │
                 ▼
    ┌────────────────────────┐
    │ 3. 权限检查             │
    │ canUseTool()           │
    └────────────────────────┘
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
    allow              deny/cancel
       │                   │
       ▼                   ▼
┌────────────────┐  ┌──────────────────┐
│ 4. 执行工具     │  │ 返回拒绝结果      │
│ tool.call()    │  └──────────────────┘
└────────────────┘
       │
       ▼
┌────────────────────────┐
│ 5. 构建工具结果消息     │
│ createUserMessage()    │
│ type: 'tool_result'    │
└────────────────────────┘
       │
       ▼
┌────────────────────────┐
│ 6. 处理进度回调         │
│ onProgress(data)       │
└────────────────────────┘
```

## 4.4 流式响应处理

### 4.4.1 流式响应架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      API Streaming Response                               │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    queryModelWithStreaming()                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 事件类型处理：                                                        │  │
│  │ - message_start: 初始化消息，捕获 usage                              │  │
│  │ - content_block_start: 创建内容块（text/tool_use/thinking）          │  │
│  │ - content_block_delta: 增量更新内容                                  │  │
│  │ - content_block_stop: 完成内容块                                     │  │
│  │ - message_delta: 更新 stop_reason, usage                            │  │
│  │ - message_stop: 消息完成                                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      流式事件 yield                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ yield {                                                              │  │
│  │   type: 'stream_event',                                              │  │
│  │   event: BetaRawMessageStreamEvent                                   │  │
│  │ }                                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     query() 消费流式事件                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ - 累积 assistantMessages                                             │  │
│  │ - 收集 toolUseBlocks                                                 │  │
│  │ - StreamingToolExecutor 并行执行工具                                 │  │
│  │ - yield 完成的消息                                                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.4.2 流式事件处理

```typescript
// 流式事件处理核心逻辑
for await (const part of stream) {
  switch (part.type) {
    case 'message_start':
      partialMessage = part.message
      usage = updateUsage(usage, part.message?.usage)
      break

    case 'content_block_start':
      switch (part.content_block.type) {
        case 'tool_use':
          contentBlocks[part.index] = {
            ...part.content_block,
            input: '',  // 将通过 delta 累积
          }
          break
        case 'text':
          contentBlocks[part.index] = {
            ...part.content_block,
            text: '',
          }
          break
        case 'thinking':
          contentBlocks[part.index] = {
            ...part.content_block,
            thinking: '',
            signature: '',
          }
          break
      }
      break

    case 'content_block_delta':
      const contentBlock = contentBlocks[part.index]
      const delta = part.delta

      switch (delta.type) {
        case 'input_json_delta':
          // 累积工具输入 JSON
          if (contentBlock.type === 'tool_use') {
            contentBlock.input += delta.partial_json
          }
          break
        case 'text_delta':
          contentBlock.text += delta.text
          break
        case 'thinking_delta':
          contentBlock.thinking += delta.thinking
          break
        case 'signature_delta':
          contentBlock.signature = delta.signature
          break
      }
      break

    case 'message_delta':
      stopReason = part.delta.stop_reason
      usage = updateUsage(usage, part.usage)
      break
  }
}
```

### 4.4.3 Usage 统计累积

```typescript
// 消息级别的 usage 追踪
let currentMessageUsage: NonNullableUsage = EMPTY_USAGE

// message_start: 初始化当前消息 usage
if (message.event.type === 'message_start') {
  currentMessageUsage = EMPTY_USAGE
  currentMessageUsage = updateUsage(
    currentMessageUsage,
    message.event.message.usage
  )
}

// message_delta: 更新 usage
if (message.event.type === 'message_delta') {
  currentMessageUsage = updateUsage(
    currentMessageUsage,
    message.event.usage
  )
  // 捕获 stop_reason
  if (message.event.delta.stop_reason != null) {
    lastStopReason = message.event.delta.stop_reason
  }
}

// message_stop: 累积到总 usage
if (message.event.type === 'message_stop') {
  this.totalUsage = accumulateUsage(
    this.totalUsage,
    currentMessageUsage
  )
}
```

### 4.4.4 流式响应时序图

```
Client              API                    QueryEngine           StreamingToolExecutor
  │                  │                          │                        │
  │   POST /messages │                          │                        │
  │─────────────────►│                          │                        │
  │                  │                          │                        │
  │  message_start   │                          │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │  content_block_start (text)                 │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │  text_delta      │                          │                        │
  │◄─────────────────│                          │                        │
  │  ...             │                          │                        │
  │                  │                          │                        │
  │  content_block_start (tool_use)             │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │  input_json_delta│                          │                        │
  │◄─────────────────│                          │                        │
  │  ...             │                          │                        │
  │                  │                          │                        │
  │  content_block_stop                         │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │                  │     yield assistant      │                        │
  │                  │     with tool_use        │                        │
  │                  │◄─────────────────────────│                        │
  │                  │                          │                        │
  │                  │                          │    addTool()           │
  │                  │                          │───────────────────────►│
  │                  │                          │                        │
  │                  │                          │    executeTool() async │
  │                  │                          │───────────────────────►│
  │                  │                          │                        │
  │  message_delta   │                          │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │  message_stop    │                          │                        │
  │◄─────────────────│                          │                        │
  │                  │                          │                        │
  │                  │                          │    getCompletedResults │
  │                  │                          │◄───────────────────────│
  │                  │                          │                        │
  │                  │     yield tool_result    │                        │
  │                  │◄─────────────────────────│                        │
  │                  │                          │                        │
```

## 4.5 错误处理与重试策略

### 4.5.1 错误分类体系

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           错误类型层次                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  APIError                                                                │
│  ├── 400 Bad Request                                                     │
│  │   ├── prompt_too_long (413/400)                                      │
│  │   ├── invalid_model_name                                             │
│  │   ├── tool_use_mismatch                                              │
│  │   ├── duplicate_tool_use_id                                          │
│  │   ├── image_too_large                                                │
│  │   └── pdf_too_large                                                  │
│  │                                                                       │
│  ├── 401 Unauthorized                                                    │
│  │   └── authentication_failed                                          │
│  │                                                                       │
│  ├── 403 Forbidden                                                       │
│  │   ├── token_revoked                                                  │
│  │   └── oauth_org_not_allowed                                          │
│  │                                                                       │
│  ├── 404 Not Found                                                       │
│  │   └── model_not_found                                                │
│  │                                                                       │
│  ├── 408 Request Timeout                                                 │
│  │                                                                       │
│  ├── 409 Conflict                                                        │
│  │                                                                       │
│  ├── 413 Request Too Large                                               │
│  │                                                                       │
│  ├── 429 Rate Limit                                                      │
│  │   └── rate_limit                                                     │
│  │                                                                       │
│  └── 529 Overloaded                                                      │
│      └── server_overload                                                │
│                                                                          │
│  APIConnectionError                                                      │
│  ├── timeout                                                             │
│  ├── ECONNRESET                                                          │
│  └── EPIPE                                                               │
│                                                                          │
│  APIUserAbortError                                                       │
│  └── user_interrupted                                                   │
│                                                                          │
│  ImageSizeError / ImageResizeError                                       │
│                                                                          │
│  FallbackTriggeredError                                                  │
│  └── model_fallback                                                     │
│                                                                          │
│  CannotRetryError                                                        │
│  └── retry_exhausted                                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.5.2 重试策略

```typescript
// 重试配置
const DEFAULT_MAX_RETRIES = 10
const BASE_DELAY_MS = 500
const MAX_529_RETRIES = 3

// 前台查询源（支持 529 重试）
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'compact',
  // ...
])

// 重试延迟计算
function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  // 优先使用 Retry-After 头
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  // 指数退避 + 抖动
  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

### 4.5.3 withRetry 重试流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          withRetry() 流程                                 │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │ attempt = 1..maxRetries+1 │
                    └───────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │ 检查 signal.aborted?      │
                    └───────────────────────────┘
                          /           \
                        Yes            No
                        /               \
                       ▼                 ▼
            ┌──────────────┐    ┌─────────────────┐
            │ throw        │    │ 执行 operation  │
            │ APIUserAbort │    └─────────────────┘
            └──────────────┘            │
                               ┌────────┴────────┐
                               ▼                 ▼
                            成功              失败
                               │                 │
                               ▼                 ▼
                    ┌──────────────┐  ┌─────────────────────┐
                    │ return T     │  │ 分类错误类型        │
                    └──────────────┘  └─────────────────────┘
                                                │
                              ┌─────────────────┼─────────────────┐
                              ▼                 ▼                 ▼
                        529 错误         429 限流          其他错误
                              │                 │                 │
                              ▼                 ▼                 ▼
                    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
                    │ 检查重试次数 │  │ Fast Mode    │  │ shouldRetry? │
                    │ <= MAX_529   │  │ 处理         │  │              │
                    └──────────────┘  └──────────────┘  └──────────────┘
                              │                 │                 │
                              └─────────────────┼─────────────────┘
                                                ▼
                              ┌─────────────────────────────┐
                              │ 计算 delayMs                │
                              │ yield api_retry 消息        │
                              │ await sleep(delayMs)        │
                              └─────────────────────────────┘
                                                │
                                                ▼
                                         continue
```

### 4.5.4 错误恢复策略

```typescript
// 错误恢复策略映射
const errorRecoveryStrategies = {
  // Prompt 太长：尝试上下文压缩
  'prompt_too_long': {
    strategy: 'reactive_compact',
    maxAttempts: 1,
  },

  // 模型过载：指数退避重试
  'server_overload': {
    strategy: 'exponential_backoff',
    maxAttempts: MAX_529_RETRIES,
  },

  // 限流：等待 Retry-After 或退避
  'rate_limit': {
    strategy: 'respect_retry_after',
    fallbackModel: true,
  },

  // 认证失败：刷新 token 重试
  'authentication_failed': {
    strategy: 'refresh_auth',
    maxAttempts: 2,
  },

  // 输出 token 超限：增加 max_tokens 重试
  'max_output_tokens': {
    strategy: 'escalate_tokens',
    maxAttempts: 3,
  },

  // 连接错误：立即重试
  'connection_error': {
    strategy: 'immediate_retry',
    maxAttempts: DEFAULT_MAX_RETRIES,
  },
}
```

### 4.5.5 错误消息生成

```typescript
// 错误消息生成流程
function getAssistantMessageFromError(
  error: unknown,
  model: string,
): AssistantMessage {
  // 超时错误
  if (error instanceof APIConnectionTimeoutError) {
    return createAssistantAPIErrorMessage({
      content: API_TIMEOUT_ERROR_MESSAGE,
      error: 'unknown',
    })
  }

  // 限流错误
  if (error instanceof APIError && error.status === 429) {
    const rateLimitType = error.headers?.get('anthropic-ratelimit-...')
    // ... 构建详细的限流消息
    return createAssistantAPIErrorMessage({
      content: getRateLimitErrorMessage(limits, model),
      error: 'rate_limit',
    })
  }

  // Prompt 太长
  if (error.message.toLowerCase().includes('prompt is too long')) {
    return createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
      error: 'invalid_request',
      errorDetails: error.message,  // 保留原始 token 计数
    })
  }

  // ... 其他错误类型处理
}
```

### 4.5.6 中断处理

```typescript
// 中断处理流程
if (toolUseContext.abortController.signal.aborted) {
  // 1. 消费剩余工具结果（生成合成错误）
  if (streamingToolExecutor) {
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) {
        yield update.message
      }
    }
  } else {
    // 2. 为未完成的工具生成中断消息
    yield* yieldMissingToolResultBlocks(
      assistantMessages,
      'Interrupted by user'
    )
  }

  // 3. 清理资源（如 computer use）
  if (feature('CHICAGO_MCP') && !toolUseContext.agentId) {
    await cleanupComputerUseAfterTurn(toolUseContext)
  }

  // 4. 生成中断消息（除非是 submit-interrupt）
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: false })
  }

  return { reason: 'aborted_streaming' }
}
```

### 4.5.7 结果状态判断

```typescript
// 成功结果判断
function isResultSuccessful(
  message: Message | undefined,
  stopReason: string | null = null,
): message is Message {
  if (!message) return false

  // Assistant 消息：最后一个内容块是 text/thinking
  if (message.type === 'assistant') {
    const lastContent = last(message.message.content)
    return (
      lastContent?.type === 'text' ||
      lastContent?.type === 'thinking' ||
      lastContent?.type === 'redacted_thinking'
    )
  }

  // User 消息：所有内容块都是 tool_result
  if (message.type === 'user') {
    const content = message.message.content
    if (
      Array.isArray(content) &&
      content.length > 0 &&
      content.every(block => block.type === 'tool_result')
    ) {
      return true
    }
  }

  // API 完成（end_turn）但无内容
  return stopReason === 'end_turn'
}
```

## 4.6 总结

QueryEngine 作为 Claude Code 的核心查询引擎，承担了以下关键职责：

1. **会话管理**：维护消息历史、文件状态缓存、使用量统计等会话状态
2. **请求编排**：协调系统提示词构建、用户输入处理、API 调用等流程
3. **工具执行**：通过 StreamingToolExecutor 实现并发的工具调用管理
4. **流式处理**：处理 API 流式响应，支持实时输出和进度更新
5. **错误恢复**：实现多层次的错误处理和重试策略

其设计体现了以下工程实践：

- **职责分离**：QueryEngine 专注于会话管理，query() 专注于查询循环
- **流式优先**：使用 AsyncGenerator 实现非阻塞的流式处理
- **弹性设计**：多层错误恢复机制确保系统稳定性
- **可扩展性**：通过配置对象支持多种使用场景（SDK、REPL、headless）
