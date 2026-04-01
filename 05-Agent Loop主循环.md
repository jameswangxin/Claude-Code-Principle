# 第19章：Agent Loop 主循环

> 本章深入分析 Claude Code 的核心执行循环 —— Agent Loop，它是整个系统的心脏，负责协调对话、工具执行和状态管理。

## 19.1 Agent Loop 概述

### 19.1.1 什么是 Agent Loop

Agent Loop 是 Claude Code 的核心执行循环，它实现了 AI 助手的"思考-行动-观察"循环模式。每次用户输入后，Agent Loop 会：

1. **思考**：调用 Claude API 生成响应
2. **行动**：执行工具调用（如读写文件、运行命令）
3. **观察**：收集工具结果，决定是否继续

这个循环会持续进行，直到任务完成或用户中断。

### 19.1.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Loop 核心职责                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │   对话管理    │    │   工具执行    │    │   状态管理    │         │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤         │
│  │ 消息累积     │    │ 工具调用     │    │ 上下文压缩   │         │
│  │ 多轮对话     │    │ 并发执行     │    │ Token 预算   │         │
│  │ 历史管理     │    │ 权限检查     │    │ 恢复机制     │         │
│  └──────────────┘    └──────────────┘    └──────────────┘         │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    循环控制                               │     │
│  │  Continue（继续）←─────────────────→ Terminal（终止）     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.1.3 与 QueryEngine 的关系

Agent Loop 位于 `src/query.ts` 中的 `queryLoop` 函数，它是 QueryEngine 的核心执行逻辑：

```
用户输入 → query() → queryLoop() → [循环执行] → 结果输出
                           │
                           ├── API 调用
                           ├── 工具执行
                           ├── 上下文压缩
                           └── 状态更新
```

---

## 19.2 queryLoop 函数分析

### 19.2.1 函数签名

```typescript
// src/query.ts
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  // ...
}
```

**关键特点**：
- 使用 `AsyncGenerator` 实现流式输出
- 通过 `yield` 发送事件和消息
- 通过 `return` 终止循环（返回 `Terminal`）

### 19.2.2 状态管理结构

```typescript
type State = {
  messages: Message[]              // 对话历史
  toolUseContext: ToolUseContext   // 工具执行上下文
  maxOutputTokensOverride: number | undefined
  autoCompactTracking: AutoCompactTracking | undefined
  stopHookActive: boolean | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  turnCount: number                // 当前轮次
  pendingToolUseSummary: ToolUseSummaryMessage | undefined
  transition: 'continue' | 'terminal' | undefined
}
```

### 19.2.3 循环结构

```typescript
// 主循环：无限循环直到显式退出
while (true) {
  // 1. 解构状态
  let { toolUseContext } = state
  const { messages, turnCount, ... } = state

  // 2. 准备消息
  let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

  // 3. 上下文压缩检查
  const { compactionResult } = await deps.autocompact(...)

  // 4. API 调用 + 工具执行
  for await (const message of queryModelWithStreaming(...)) {
    // 处理流式响应
    if (message.type === 'assistant') {
      // 收集助手消息
    }
    if (toolUse) {
      // 执行工具
    }
  }

  // 5. 决定是否继续
  if (needsFollowUp) {
    state = { ...state, messages: [...messages, ...newMessages] }
    continue  // 继续下一轮
  } else {
    return { type: 'terminal' }  // 终止循环
  }
}
```

---

## 19.3 单次迭代流程

### 19.3.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    queryLoop 单次迭代流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ 1. 解构状态      │                                               │
│  │   messages      │                                               │
│  │   toolUseContext│                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ 2. 消息准备      │                                               │
│  │   预算检查      │                                               │
│  │   Snip 压缩     │                                               │
│  │   Microcompact  │                                               │
│  │   Context Collapse│                                             │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ 3. 自动压缩      │                                               │
│  │   Token 检查    │                                               │
│  │   压缩决策      │                                               │
│  │   生成摘要      │                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ 4. API 调用      │                                               │
│  │   构建请求      │                                               │
│  │   流式响应      │                                               │
│  │   内容块处理    │                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ 5. 工具执行      │                                               │
│  │   解析 tool_use │                                               │
│  │   权限检查      │                                               │
│  │   并发执行      │                                               │
│  │   收集结果      │                                               │
│  └────────┬────────┘                                               │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                               │
│  │ 6. 循环决策      │                                               │
│  │   needsFollowUp?│                                               │
│  │   ├─ Yes → continue                                             │
│  │   └─ No  → terminal                                             │
│  └─────────────────┘                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 19.3.2 消息准备阶段

```typescript
// 1. 获取压缩边界后的消息
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

// 2. 应用工具结果预算
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(...) : undefined,
)

// 3. 应用 Snip 压缩（轻量级历史裁剪）
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}

// 4. 应用 Microcompact（缓存优化）
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)

// 5. 应用 Context Collapse（上下文折叠）
if (feature('CONTEXT_COLLAPSE')) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
  messagesForQuery = collapseResult.messages
}
```

### 19.3.3 API 调用阶段

```typescript
for await (const message of queryModelWithStreaming({
  messages: normalizeMessagesForAPI(messagesForQuery),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  maxOutputTokens,
  model: currentModel,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  abortController: toolUseContext.abortController,
})) {
  // 处理不同类型的消息
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    yield yieldMessage  // 发送给 UI
  }

  // 收集 tool_use 块
  if (block.type === 'tool_use') {
    toolUseBlocks.push(block)
    needsFollowUp = true
  }
}
```

---

## 19.4 多轮对话机制

### 19.4.1 Continue 模式（循环继续）

当满足以下条件时，循环继续：

```typescript
// 条件1：有工具需要执行
if (toolUseBlocks.length > 0) {
  needsFollowUp = true
}

// 条件2：工具执行后需要后续
if (toolResults.length > 0) {
  needsFollowUp = true
}

// 继续循环
if (needsFollowUp) {
  state = {
    ...state,
    messages: [...messages, ...assistantMessages, ...toolResults],
    turnCount: turnCount + 1,
  }
  continue  // 回到 while(true) 开头
}
```

### 19.4.2 Terminal 模式（循环终止）

当满足以下条件时，循环终止：

```typescript
// 条件1：没有工具调用
if (toolUseBlocks.length === 0) {
  needsFollowUp = false
}

// 条件2：达到最大轮次
if (turnCount >= maxTurns) {
  needsFollowUp = false
}

// 条件3：用户中断
if (toolUseContext.abortController.signal.aborted) {
  return { type: 'terminal', reason: 'aborted' }
}

// 终止循环
if (!needsFollowUp) {
  return { type: 'terminal' }
}
```

### 19.4.3 消息累积策略

```typescript
// 每轮迭代后，新消息被追加到历史
state = {
  ...state,
  messages: [
    ...messages,           // 之前的历史
    ...assistantMessages,  // 本轮助手消息
    ...toolResults,        // 本轮工具结果
  ],
}
```

---

## 19.5 工具执行流程

### 19.5.1 StreamingToolExecutor

```typescript
class StreamingToolExecutor {
  private tools: Tool[]
  private canUseTool: CanUseToolFn
  private context: ToolUseContext
  private pendingExecutions: Map<string, Promise<ToolResult>>

  // 并发执行工具
  async executeAll(toolUseBlocks: ToolUseBlock[]): Promise<ToolResult[]> {
    const results = await Promise.all(
      toolUseBlocks.map(block => this.executeOne(block))
    )
    return results
  }

  // 执行单个工具
  private async executeOne(block: ToolUseBlock): Promise<ToolResult> {
    const tool = findToolByName(this.tools, block.name)

    // 权限检查
    const permission = await this.canUseTool(tool, block.input)
    if (permission === 'deny') {
      return { type: 'error', message: 'Permission denied' }
    }

    // 执行工具
    const result = await tool.call(block.input, this.context)
    return result
  }
}
```

### 19.5.2 并发执行规则

```typescript
// 并发安全检查
const canExecuteInParallel = (tool: Tool) => {
  // 只读工具可以并发
  if (tool.isReadOnly) return true

  // 文件操作需要检查路径冲突
  if (tool.name === 'FileEditTool') {
    return !hasPathConflict(tool.input.file_path)
  }

  // Bash 命令需要检查资源竞争
  if (tool.name === 'BashTool') {
    return !hasResourceConflict(tool.input.command)
  }

  // 默认串行执行
  return false
}
```

### 19.5.3 权限检查集成

```typescript
// 在工具执行前进行权限检查
const permissionResult = await hasPermissionsToUseTool({
  tool,
  input: block.input,
  toolUseContext,
  mode: permissionMode,
})

switch (permissionResult.decision) {
  case 'allow':
    // 直接执行
    break
  case 'deny':
    // 返回错误
    return { type: 'error', message: permissionResult.reason }
  case 'ask':
    // 弹出权限请求对话框
    const userResponse = await showPermissionDialog(permissionResult)
    if (userResponse === 'deny') {
      return { type: 'error', message: 'User denied' }
    }
    break
}
```

---

## 19.6 上下文管理

### 19.6.1 Token 预算

```typescript
// 计算当前 Token 使用量
const currentTokens = tokenCountWithEstimation(messagesForQuery)

// 检查是否达到限制
const { isAtBlockingLimit, isAtWarningLimit } = calculateTokenWarningState(
  currentTokens,
  mainLoopModel,
)

if (isAtBlockingLimit) {
  // 触发强制压缩或拒绝请求
  throw new PromptTooLongError()
}

if (isAtWarningLimit) {
  // 显示警告，建议压缩
  yield { type: 'token_warning', currentTokens }
}
```

### 19.6.2 压缩触发条件

```typescript
// 自动压缩触发检查
const shouldCompact = () => {
  // 条件1：Token 使用超过阈值
  if (currentTokens > AUTO_COMPACT_THRESHOLD) return true

  // 条件2：消息数量过多
  if (messages.length > MAX_MESSAGE_COUNT) return true

  // 条件3：用户配置允许
  if (!isAutoCompactEnabled()) return false

  return currentTokens > WARNING_THRESHOLD
}
```

### 19.6.3 边界消息处理

```typescript
// 压缩后生成边界消息
const boundaryMessage: TombstoneMessage = {
  type: 'tombstone',
  message: {
    type: 'compact_boundary',
    summary: 'Context compressed',
    tokensBefore: preCompactTokenCount,
    tokensAfter: postCompactTokenCount,
  },
}

yield boundaryMessage

// 后续迭代只处理边界后的消息
messagesForQuery = getMessagesAfterCompactBoundary(messages)
```

---

## 19.7 错误处理与恢复

### 19.7.1 重试机制

```typescript
// API 调用重试
const withRetry = async <T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> => {
  let lastError: Error

  for (let attempt = 0; attempt < options.maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error

      // 判断是否可重试
      if (!isRetryableError(error)) {
        throw error
      }

      // 指数退避
      const delay = Math.min(
        options.baseDelay * Math.pow(2, attempt),
        options.maxDelay
      )
      await sleep(delay)
    }
  }

  throw lastError
}
```

### 19.7.2 错误传播

```typescript
// 错误在循环中的传播
try {
  for await (const message of queryModelWithStreaming(...)) {
    yield message
  }
} catch (error) {
  // 1. 记录错误
  logError(error)

  // 2. 生成错误消息
  const errorMessage = createAssistantAPIErrorMessage(error)

  // 3. 决定是否重试或终止
  if (isRecoverableError(error)) {
    // 恢复后继续
    state = { ...state, maxOutputTokensRecoveryCount: recoveryCount + 1 }
    continue
  } else {
    // 终止并传播错误
    throw error
  }
}
```

### 19.7.3 中断处理

```typescript
// 监听中断信号
const abortController = toolUseContext.abortController

// 在关键点检查中断
if (abortController.signal.aborted) {
  // 清理资源
  await cleanup()

  // 生成中断消息
  const interruptionMessage = createUserInterruptionMessage()

  // 终止循环
  return { type: 'terminal', reason: 'interrupted', message: interruptionMessage }
}
```

---

## 19.8 总结

Agent Loop 是 Claude Code 的核心执行引擎，它通过以下机制实现了强大的 AI 助手能力：

| 机制 | 描述 |
|------|------|
| **循环控制** | Continue/Terminal 双模式，灵活控制执行流程 |
| **状态管理** | 不可变状态 + 结构更新，保证一致性 |
| **工具执行** | 并发执行 + 权限检查，安全高效 |
| **上下文管理** | 多层压缩策略，智能 Token 预算 |
| **错误恢复** | 重试 + 恢复 + 中断，健壮可靠 |

理解 Agent Loop 是理解 Claude Code 整体架构的关键，它连接了 QueryEngine、工具系统、状态管理和上下文管理等所有核心模块。

---

*本章基于 `src/query.ts` 源代码分析编写*
