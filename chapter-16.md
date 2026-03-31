# 第16章：权限系统

## 概述

权限系统是Claude Code安全架构的核心组件，负责控制和管理所有工具的执行权限。该系统采用多层防御策略，结合静态规则配置、动态权限检查、AI分类器自动审批以及沙盒隔离执行，确保AI代理在执行敏感操作时遵循最小权限原则。

权限系统设计遵循以下核心理念：

1. **默认拒绝**：未经明确授权的工具操作默认需要用户确认
2. **分层授权**：支持用户级、项目级、策略级等多层权限配置
3. **灵活模式**：提供多种权限模式适应不同使用场景
4. **可审计性**：所有权限决策都有完整的日志记录和原因追踪

## 16.1 权限模型设计

### 16.1.1 Permission类型定义

权限系统的类型定义位于`src/types/permissions.ts`，采用纯类型定义文件设计以避免循环依赖。

#### 权限模式（PermissionMode）

```typescript
// src/types/permissions.ts

// 外部可配置的权限模式
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',      // 自动接受文件编辑操作
  'bypassPermissions', // 绕过所有权限检查（危险模式）
  'default',          // 默认模式，需要交互确认
  'dontAsk',          // 不询问，自动拒绝未授权操作
  'plan',             // 计划模式，用于任务规划
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]

// 内部权限模式（包含不可直接配置的模式）
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode
```

各权限模式的行为特性：

| 模式 | 行为描述 | 适用场景 |
|------|----------|----------|
| `default` | 未授权操作弹出确认对话框 | 日常交互使用 |
| `acceptEdits` | 自动允许文件编辑，其他操作正常检查 | 批量代码修改 |
| `bypassPermissions` | 绕过所有权限检查 | 受信任的自动化脚本 |
| `dontAsk` | 自动拒绝未授权操作 | 非交互式会话 |
| `plan` | 计划模式，支持任务分解和规划 | 复杂任务设计 |
| `auto` | AI分类器自动决策 | 自动化工作流 |

#### 权限行为（PermissionBehavior）

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

三种基本行为：
- **allow**：允许执行，无需进一步确认
- **deny**：拒绝执行，返回错误信息
- **ask**：需要用户交互确认

#### 权限规则（PermissionRule）

```typescript
// 规则来源类型
export type PermissionRuleSource =
  | 'userSettings'    // 用户全局设置
  | 'projectSettings' // 项目级设置
  | 'localSettings'   // 本地工作区设置
  | 'flagSettings'    // 功能标志设置
  | 'policySettings'  // 策略设置（企业管控）
  | 'cliArg'          // 命令行参数
  | 'command'         // 会话内命令
  | 'session'         // 会话级临时规则

// 规则值定义
export type PermissionRuleValue = {
  toolName: string      // 工具名称，如 "Bash"、"Edit"
  ruleContent?: string  // 规则内容，如 "npm:*" 表示npm相关命令
}

// 完整规则定义
export type PermissionRule = {
  source: PermissionRuleSource       // 规则来源
  ruleBehavior: PermissionBehavior   // 规则行为
  ruleValue: PermissionRuleValue     // 规则值
}
```

规则示例：
- `Bash(npm:*)`：匹配所有npm命令
- `Bash(git:*)`：匹配所有git命令
- `Edit`：匹配整个Edit工具
- `mcp__server1`：匹配整个MCP服务器的所有工具

### 16.1.2 权限决策结果

权限检查的结果通过`PermissionDecision`类型表示：

```typescript
// 允许决策
export type PermissionAllowDecision<Input = { [key: string]: unknown }> = {
  behavior: 'allow'
  updatedInput?: Input           // 可能被修改的输入
  userModified?: boolean         // 用户是否修改了输入
  decisionReason?: PermissionDecisionReason  // 决策原因
  toolUseID?: string
  acceptFeedback?: string        // 用户反馈
  contentBlocks?: ContentBlockParam[]  // 附加内容块
}

// 询问决策
export type PermissionAskDecision<Input = { [key: string]: unknown }> = {
  behavior: 'ask'
  message: string                // 提示消息
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]  // 权限更新建议
  blockedPath?: string           // 被阻止的路径
  metadata?: PermissionMetadata
  pendingClassifierCheck?: PendingClassifierCheck  // 待执行的分类器检查
  contentBlocks?: ContentBlockParam[]
}

// 拒绝决策
export type PermissionDenyDecision = {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason
  toolUseID?: string
}

// 联合类型
export type PermissionDecision<Input = { [key: string]: unknown }> =
  | PermissionAllowDecision<Input>
  | PermissionAskDecision<Input>
  | PermissionDenyDecision
```

### 16.1.3 决策原因追踪

每个权限决策都包含决策原因，支持完整的审计追踪：

```typescript
export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }  // 由规则触发
  | { type: 'mode'; mode: PermissionMode }  // 由模式触发
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }  // 子命令结果
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }  // 钩子触发
  | { type: 'asyncAgent'; reason: string }  // 异步代理
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }  // AI分类器
  | { type: 'workingDir'; reason: string }  // 工作目录限制
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }  // 安全检查
  | { type: 'other'; reason: string }
```

### 16.1.4 权限上下文

工具权限检查所需的完整上下文：

```typescript
export type ToolPermissionContext = {
  readonly mode: PermissionMode  // 当前权限模式
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource  // 允许规则
  readonly alwaysDenyRules: ToolPermissionRulesBySource   // 拒绝规则
  readonly alwaysAskRules: ToolPermissionRulesBySource    // 询问规则
  readonly isBypassPermissionsModeAvailable: boolean      // 是否可用绕过模式
  readonly strippedDangerousRules?: ToolPermissionRulesBySource  // 被剥离的危险规则
  readonly shouldAvoidPermissionPrompts?: boolean         // 是否避免弹窗
  readonly awaitAutomatedChecksBeforeDialog?: boolean     // 是否等待自动检查
  readonly prePlanMode?: PermissionMode  // 计划模式前的模式
}
```

## 16.2 权限检查机制

### 16.2.1 核心检查流程

权限检查的核心函数是`hasPermissionsToUseTool`，位于`src/utils/permissions/permissions.ts`：

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool,
  input,
  context,
  assistantMessage,
  toolUseID,
): Promise<PermissionDecision> => {
  const result = await hasPermissionsToUseToolInner(tool, input, context)

  // 成功的工具使用重置连续拒绝计数
  if (result.behavior === 'allow') {
    // ... 处理拒绝计数重置
    return result
  }

  // 应用dontAsk模式转换：将'ask'转换为'deny'
  if (result.behavior === 'ask') {
    const appState = context.getAppState()

    if (appState.toolPermissionContext.mode === 'dontAsk') {
      return {
        behavior: 'deny',
        decisionReason: { type: 'mode', mode: 'dontAsk' },
        message: DONT_ASK_REJECT_MESSAGE(tool.name),
      }
    }

    // 应用auto模式：使用AI分类器代替用户确认
    if (appState.toolPermissionContext.mode === 'auto') {
      // ... AI分类器处理逻辑
    }

    // 后台代理无法显示权限弹窗时的处理
    if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
      // 先运行钩子，再自动拒绝
      const hookDecision = await runPermissionRequestHooksForHeadlessAgent(...)
      if (hookDecision) return hookDecision
      return {
        behavior: 'deny',
        decisionReason: { type: 'asyncAgent', reason: '...' },
        message: AUTO_REJECT_MESSAGE(tool.name),
      }
    }
  }

  return result
}
```

### 16.2.2 内部检查步骤

`hasPermissionsToUseToolInner`实现了详细的检查流程：

```typescript
async function hasPermissionsToUseToolInner(
  tool: Tool,
  input: { [key: string]: unknown },
  context: ToolUseContext,
): Promise<PermissionDecision> {
  // 步骤1：检查工具是否被拒绝
  // 1a. 整个工具被拒绝
  const denyRule = getDenyRuleForTool(appState.toolPermissionContext, tool)
  if (denyRule) {
    return {
      behavior: 'deny',
      decisionReason: { type: 'rule', rule: denyRule },
      message: `Permission to use ${tool.name} has been denied.`,
    }
  }

  // 1b. 检查是否需要总是询问
  const askRule = getAskRuleForTool(appState.toolPermissionContext, tool)
  if (askRule) {
    // 沙盒自动允许检查...
    return {
      behavior: 'ask',
      decisionReason: { type: 'rule', rule: askRule },
      message: createPermissionRequestMessage(tool.name),
    }
  }

  // 1c. 调用工具自身的权限检查
  let toolPermissionResult = await tool.checkPermissions(parsedInput, context)

  // 1d. 工具实现拒绝权限
  if (toolPermissionResult?.behavior === 'deny') {
    return toolPermissionResult
  }

  // 1e. 工具需要用户交互（即使绕过模式也需要）
  if (tool.requiresUserInteraction?.() && toolPermissionResult?.behavior === 'ask') {
    return toolPermissionResult
  }

  // 1f. 内容特定的ask规则优先级高于bypassPermissions模式
  if (toolPermissionResult?.behavior === 'ask' &&
      toolPermissionResult.decisionReason?.type === 'rule' &&
      toolPermissionResult.decisionReason.rule.ruleBehavior === 'ask') {
    return toolPermissionResult
  }

  // 1g. 安全检查（如.git/、.claude/目录）绕过免疫
  if (toolPermissionResult?.behavior === 'ask' &&
      toolPermissionResult.decisionReason?.type === 'safetyCheck') {
    return toolPermissionResult
  }

  // 步骤2：检查模式是否允许工具运行
  // 2a. bypassPermissions模式
  const shouldBypassPermissions =
    appState.toolPermissionContext.mode === 'bypassPermissions' ||
    (appState.toolPermissionContext.mode === 'plan' &&
     appState.toolPermissionContext.isBypassPermissionsModeAvailable)
  if (shouldBypassPermissions) {
    return { behavior: 'allow', ... }
  }

  // 2b. 整个工具被允许
  const alwaysAllowedRule = toolAlwaysAllowedRule(appState.toolPermissionContext, tool)
  if (alwaysAllowedRule) {
    return { behavior: 'allow', ... }
  }

  // 步骤3：将"passthrough"转换为"ask"
  const result = toolPermissionResult.behavior === 'passthrough'
    ? { ...toolPermissionResult, behavior: 'ask' as const, ... }
    : toolPermissionResult

  return result
}
```

### 16.2.3 规则匹配机制

规则匹配支持多种匹配模式：

```typescript
// src/utils/permissions/permissions.ts

// 检查整个工具是否匹配规则
function toolMatchesRule(
  tool: Pick<Tool, 'name' | 'mcpInfo'>,
  rule: PermissionRule,
): boolean {
  // 规则不能有内容才能匹配整个工具
  if (rule.ruleValue.ruleContent !== undefined) {
    return false
  }

  const nameForRuleMatch = getToolNameForPermissionCheck(tool)

  // 直接工具名匹配
  if (rule.ruleValue.toolName === nameForRuleMatch) {
    return true
  }

  // MCP服务器级别权限：规则"mcp__server1"匹配工具"mcp__server1__tool1"
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForRuleMatch)

  return (
    ruleInfo !== null &&
    toolInfo !== null &&
    (ruleInfo.toolName === undefined || ruleInfo.toolName === '*') &&
    ruleInfo.serverName === toolInfo.serverName
  )
}
```

### 16.2.4 toolPermission Hooks

Hooks机制允许在权限检查过程中注入自定义逻辑：

```typescript
// src/hooks/toolPermission/handlers/coordinatorHandler.ts

async function handleCoordinatorPermission(
  params: CoordinatorPermissionParams,
): Promise<PermissionDecision | null> {
  const { ctx, updatedInput, suggestions, permissionMode } = params

  try {
    // 1. 首先尝试权限钩子（快速、本地）
    const hookResult = await ctx.runHooks(
      permissionMode,
      suggestions,
      updatedInput,
    )
    if (hookResult) return hookResult

    // 2. 尝试分类器（慢速、推理 - 仅bash）
    const classifierResult = feature('BASH_CLASSIFIER')
      ? await ctx.tryClassifier?.(params.pendingClassifierCheck, updatedInput)
      : null
    if (classifierResult) {
      return classifierResult
    }
  } catch (error) {
    // 自动检查失败时降级到对话框
    logError(error)
  }

  // 3. 两者都未解决 - 降级到对话框
  return null
}
```

## 16.3 用户授权流程

### 16.3.1 交互式权限处理

交互式处理是主要的用户授权流程：

```typescript
// src/hooks/toolPermission/handlers/interactiveHandler.ts

function handleInteractivePermission(
  params: InteractivePermissionParams,
  resolve: (decision: PermissionDecision) => void,
): void {
  const { ctx, description, result, bridgeCallbacks, channelCallbacks } = params
  const { resolve: resolveOnce, isResolved, claim } = createResolveOnce(resolve)
  let userInteracted = false

  // 将权限请求推入队列
  ctx.pushToQueue({
    assistantMessage: ctx.assistantMessage,
    tool: ctx.tool,
    description,
    input: displayInput,
    toolUseContext: ctx.toolUseContext,
    toolUseID: ctx.toolUseID,
    permissionResult: result,
    permissionPromptStartTimeMs,
    onUserInteraction() {
      userInteracted = true
      clearClassifierChecking(ctx.toolUseID)
    },
    onAbort() {
      if (!claim()) return
      ctx.logCancelled()
      resolveOnce(ctx.cancelAndAbort(undefined, true))
    },
    async onAllow(updatedInput, permissionUpdates, feedback, contentBlocks) {
      if (!claim()) return
      resolveOnce(
        await ctx.handleUserAllow(
          updatedInput,
          permissionUpdates,
          feedback,
          permissionPromptStartTimeMs,
          contentBlocks,
          result.decisionReason,
        ),
      )
    },
    onReject(feedback, contentBlocks) {
      if (!claim()) return
      resolveOnce(ctx.cancelAndAbort(feedback, undefined, contentBlocks))
    },
    async recheckPermission() {
      const freshResult = await hasPermissionsToUseTool(...)
      if (freshResult.behavior === 'allow') {
        if (!claim()) return
        resolveOnce(ctx.buildAllow(freshResult.updatedInput ?? ctx.input))
      }
    },
  })

  // 竞态：CCR远程权限响应
  if (bridgeCallbacks && bridgeRequestId) {
    bridgeCallbacks.sendRequest(...)
    const unsubscribe = bridgeCallbacks.onResponse(bridgeRequestId, response => {
      if (!claim()) return
      // 处理远程响应...
    })
  }

  // 竞态：Channel权限中继（Telegram、iMessage等）
  if (channelCallbacks && !ctx.tool.requiresUserInteraction?.()) {
    // 发送权限提示到各渠道...
  }

  // 异步执行权限钩子
  void (async () => {
    const hookDecision = await ctx.runHooks(...)
    if (!hookDecision || !claim()) return
    resolveOnce(hookDecision)
  })()

  // 异步执行bash分类器检查
  if (result.pendingClassifierCheck && ctx.tool.name === BASH_TOOL_NAME) {
    setClassifierChecking(ctx.toolUseID)
    void executeAsyncClassifierCheck(
      result.pendingClassifierCheck,
      ctx.toolUseContext.abortController.signal,
      {
        shouldContinue: () => !isResolved() && !userInteracted,
        onAllow: decisionReason => {
          if (!claim()) return
          ctx.updateQueueItem({
            classifierCheckInProgress: false,
            classifierAutoApproved: true,
          })
          resolveOnce(ctx.buildAllow(ctx.input, { decisionReason }))
        },
      },
    )
  }
}
```

### 16.3.2 权限请求队列

权限请求通过React状态管理的队列系统处理：

```typescript
// src/hooks/toolPermission/PermissionContext.ts

type PermissionQueueOps = {
  push(item: ToolUseConfirm): void
  remove(toolUseID: string): void
  update(toolUseID: string, patch: Partial<ToolUseConfirm>): void
}

function createPermissionQueueOps(
  setToolUseConfirmQueue: React.Dispatch<React.SetStateAction<ToolUseConfirm[]>>,
): PermissionQueueOps {
  return {
    push(item: ToolUseConfirm) {
      setToolUseConfirmQueue(queue => [...queue, item])
    },
    remove(toolUseID: string) {
      setToolUseConfirmQueue(queue =>
        queue.filter(item => item.toolUseID !== toolUseID),
      )
    },
    update(toolUseID: string, patch: Partial<ToolUseConfirm>) {
      setToolUseConfirmQueue(queue =>
        queue.map(item =>
          item.toolUseID === toolUseID ? { ...item, ...patch } : item,
        ),
      )
    },
  }
}
```

### 16.3.3 Swarm Worker权限处理

分布式代理的权限通过Leader节点转发处理：

```typescript
// src/hooks/toolPermission/handlers/swarmWorkerHandler.ts

async function handleSwarmWorkerPermission(
  params: SwarmWorkerPermissionParams,
): Promise<PermissionDecision | null> {
  if (!isAgentSwarmsEnabled() || !isSwarmWorker()) {
    return null
  }

  const { ctx, description, updatedInput, suggestions } = params

  // 对于bash命令，先尝试分类器自动批准
  const classifierResult = feature('BASH_CLASSIFIER')
    ? await ctx.tryClassifier?.(params.pendingClassifierCheck, updatedInput)
    : null
  if (classifierResult) {
    return classifierResult
  }

  // 通过mailbox转发权限请求到Leader
  const decision = await new Promise<PermissionDecision>(resolve => {
    const { resolve: resolveOnce, claim } = createResolveOnce(resolve)

    // 创建权限请求
    const request = createPermissionRequest({
      toolName: ctx.tool.name,
      toolUseId: ctx.toolUseID,
      input: ctx.input,
      description,
      permissionSuggestions: suggestions,
    })

    // 注册回调（在发送请求前避免竞态）
    registerPermissionCallback({
      requestId: request.id,
      toolUseId: ctx.toolUseID,
      async onAllow(allowedInput, permissionUpdates, feedback, contentBlocks) {
        if (!claim()) return
        clearPendingRequest()
        resolveOnce(await ctx.handleUserAllow(...))
      },
      onReject(feedback, contentBlocks) {
        if (!claim()) return
        clearPendingRequest()
        resolveOnce(ctx.cancelAndAbort(feedback, undefined, contentBlocks))
      },
    })

    // 发送请求到Leader
    void sendPermissionRequestViaMailbox(request)

    // 显示等待指示器
    ctx.toolUseContext.setAppState(prev => ({
      ...prev,
      pendingWorkerRequest: {
        toolName: ctx.tool.name,
        toolUseId: ctx.toolUseID,
        description,
      },
    }))
  })

  return decision
}
```

### 16.3.4 权限更新持久化

用户批准时可选择持久化权限规则：

```typescript
// src/hooks/toolPermission/PermissionContext.ts

async persistPermissions(updates: PermissionUpdate[]): Promise<boolean> {
  if (updates.length === 0) return false
  persistPermissionUpdates(updates)
  const appState = toolUseContext.getAppState()
  setToolPermissionContext(
    applyPermissionUpdates(appState.toolPermissionContext, updates),
  )
  return updates.some(update => supportsPersistence(update.destination))
}
```

权限更新类型定义：

```typescript
// src/types/permissions.ts

export type PermissionUpdateDestination =
  | 'userSettings'    // 持久化到用户设置
  | 'projectSettings' // 持久化到项目设置
  | 'localSettings'   // 持久化到本地设置
  | 'session'         // 仅会话期间有效
  | 'cliArg'          // 命令行参数

export type PermissionUpdate =
  | {
      type: 'addRules'
      destination: PermissionUpdateDestination
      rules: PermissionRuleValue[]
      behavior: PermissionBehavior
    }
  | {
      type: 'replaceRules'
      destination: PermissionUpdateDestination
      rules: PermissionRuleValue[]
      behavior: PermissionBehavior
    }
  | {
      type: 'removeRules'
      destination: PermissionUpdateDestination
      rules: PermissionRuleValue[]
      behavior: PermissionBehavior
    }
  | {
      type: 'setMode'
      destination: PermissionUpdateDestination
      mode: ExternalPermissionMode
    }
  | {
      type: 'addDirectories'
      destination: PermissionUpdateDestination
      directories: string[]
    }
  | {
      type: 'removeDirectories'
      destination: PermissionUpdateDestination
      directories: string[]
    }
```

### 16.3.5 权限决策日志

所有权限决策都通过统一的日志系统记录：

```typescript
// src/hooks/toolPermission/permissionLogging.ts

type PermissionDecisionArgs =
  | { decision: 'accept'; source: PermissionApprovalSource | 'config' }
  | { decision: 'reject'; source: PermissionRejectionSource | 'config' }

function logPermissionDecision(
  ctx: PermissionLogContext,
  args: PermissionDecisionArgs,
  permissionPromptStartTimeMs?: number,
): void {
  const { tool, input, toolUseContext, messageId, toolUseID } = ctx
  const { decision, source } = args

  const waiting_for_user_permission_ms =
    permissionPromptStartTimeMs !== undefined
      ? Date.now() - permissionPromptStartTimeMs
      : undefined

  // 记录分析事件
  if (args.decision === 'accept') {
    logApprovalEvent(tool, messageId, args.source, waiting_for_user_permission_ms)
  } else {
    logRejectionEvent(tool, messageId, args.source, waiting_for_user_permission_ms)
  }

  // 跟踪代码编辑工具指标
  if (isCodeEditingTool(tool.name)) {
    void buildCodeEditToolAttributes(tool, input, decision, sourceString)
      .then(attributes => getCodeEditToolDecisionCounter()?.add(1, attributes))
  }

  // 持久化决策到上下文
  if (!toolUseContext.toolDecisions) {
    toolUseContext.toolDecisions = new Map()
  }
  toolUseContext.toolDecisions.set(toolUseID, {
    source: sourceString,
    decision,
    timestamp: Date.now(),
  })

  // OTel遥测
  void logOTelEvent('tool_decision', {
    decision,
    source: sourceString,
    tool_name: sanitizeToolNameForAnalytics(tool.name),
  })
}
```

## 16.4 沙盒执行环境

### 16.4.1 沙盒架构概述

Claude Code的沙盒系统基于`@anthropic-ai/sandbox-runtime`构建，提供了OS级别的隔离执行环境。沙盒适配层位于`src/utils/sandbox/sandbox-adapter.ts`。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code应用层                             │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐         │
│  │  BashTool     │ │  权限系统     │ │  设置管理     │         │
│  └───────┬───────┘ └───────────────┘ └───────────────┘         │
└──────────┼──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│                SandboxManager (适配层)                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ convertToSandboxRuntimeConfig() - 配置转换                │  │
│  │ wrapWithSandbox() - 命令包装                               │  │
│  │ initialize() - 初始化                                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────┬──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│              @anthropic-ai/sandbox-runtime                       │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ macOS: Seatbelt│ Linux: bwrap │ 网络代理    │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

### 16.4.2 沙盒配置转换

Claude Code设置转换为沙盒运行时配置：

```typescript
// src/utils/sandbox/sandbox-adapter.ts

export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const permissions = settings.permissions || {}

  // 从WebFetch规则提取网络域名
  const allowedDomains: string[] = []
  const deniedDomains: string[] = []

  // 当allowManagedSandboxDomainsOnly启用时，只使用策略设置中的域名
  if (shouldAllowManagedSandboxDomainsOnly()) {
    const policySettings = getSettingsForSource('policySettings')
    for (const domain of policySettings?.sandbox?.network?.allowedDomains || []) {
      allowedDomains.push(domain)
    }
  } else {
    for (const domain of settings.sandbox?.network?.allowedDomains || []) {
      allowedDomains.push(domain)
    }
    for (const ruleString of permissions.allow || []) {
      const rule = permissionRuleValueFromString(ruleString)
      if (rule.toolName === WEB_FETCH_TOOL_NAME && rule.ruleContent?.startsWith('domain:')) {
        allowedDomains.push(rule.ruleContent.substring('domain:'.length))
      }
    }
  }

  // 从Edit和Read规则提取文件系统路径
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  const denyRead: string[] = []

  // 始终拒绝写入settings.json文件以防止沙盒逃逸
  const settingsPaths = SETTING_SOURCES.map(source =>
    getSettingsFilePathForSource(source),
  ).filter((p): p is string => p !== undefined)
  denyWrite.push(...settingsPaths)

  // 阻止对.claude/skills的写入（与commands和agents同级安全）
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))

  // 安全：阻止伪造git仓库攻击
  const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
  for (const dir of [originalCwd, cwd]) {
    for (const gitFile of bareGitRepoFiles) {
      const p = resolve(dir, gitFile)
      if (existsSync(p)) {
        denyWrite.push(p)
      } else {
        bareGitRepoScrubPaths.push(p)  // 命令后清理
      }
    }
  }

  // 包含--add-dir添加的目录
  const additionalDirs = new Set([
    ...(settings.permissions?.additionalDirectories || []),
    ...getAdditionalDirectoriesForClaudeMd(),
  ])
  allowWrite.push(...additionalDirs)

  return {
    network: {
      allowedDomains,
      deniedDomains,
      allowUnixSockets: settings.sandbox?.network?.allowUnixSockets,
      allowLocalBinding: settings.sandbox?.network?.allowLocalBinding,
    },
    filesystem: {
      denyRead,
      allowRead,
      allowWrite,
      denyWrite,
    },
    ignoreViolations: settings.sandbox?.ignoreViolations,
    ripgrep: ripgrepConfig,
  }
}
```

### 16.4.3 路径解析规则

沙盒使用特殊的路径解析规则：

```typescript
// Claude Code特有的路径模式：
// - `//path` → 文件系统根目录的绝对路径（变成 `/path`）
// - `/path` → 相对于设置文件目录（变成 `$SETTINGS_DIR/path`）
// - `~/path` → 用户主目录（透传）
// - `./path` 或 `path` → 相对路径（透传）

export function resolvePathPatternForSandbox(
  pattern: string,
  source: SettingSource,
): string {
  // 处理 // 前缀 - 从根目录的绝对路径
  if (pattern.startsWith('//')) {
    return pattern.slice(1)  // "//.aws/**" → "/.aws/**"
  }

  // 处理 / 前缀 - 相对于设置文件目录
  if (pattern.startsWith('/') && !pattern.startsWith('//')) {
    const root = getSettingsRootPathForSource(source)
    return resolve(root, pattern.slice(1))
  }

  // 其他模式透传
  return pattern
}
```

### 16.4.4 沙盒初始化与配置刷新

```typescript
// src/utils/sandbox/sandbox-adapter.ts

async function initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void> {
  if (initializationPromise) {
    return initializationPromise
  }

  if (!isSandboxingEnabled()) {
    return
  }

  initializationPromise = (async () => {
    try {
      // 解析worktree主仓库路径（会话期间缓存）
      if (worktreeMainRepoPath === undefined) {
        worktreeMainRepoPath = await detectWorktreeMainRepoPath(getCwdState())
      }

      const settings = getSettings_DEPRECATED()
      const runtimeConfig = convertToSandboxRuntimeConfig(settings)

      await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)

      // 订阅设置变更以动态更新沙盒配置
      settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
        const settings = getSettings_DEPRECATED()
        const newConfig = convertToSandboxRuntimeConfig(settings)
        BaseSandboxManager.updateConfig(newConfig)
      })
    } catch (error) {
      initializationPromise = undefined
      logForDebugging(`Failed to initialize sandbox: ${errorMessage(error)}`)
    }
  })()

  return initializationPromise
}

function refreshConfig(): void {
  if (!isSandboxingEnabled()) return
  const settings = getSettings_DEPRECATED()
  const newConfig = convertToSandboxRuntimeConfig(settings)
  BaseSandboxManager.updateConfig(newConfig)
}
```

### 16.4.5 命令包装与执行

```typescript
// src/utils/sandbox/sandbox-adapter.ts

async function wrapWithSandbox(
  command: string,
  binShell?: string,
  customConfig?: Partial<SandboxRuntimeConfig>,
  abortSignal?: AbortSignal,
): Promise<string> {
  if (isSandboxingEnabled()) {
    if (initializationPromise) {
      await initializationPromise
    } else {
      throw new Error('Sandbox failed to initialize.')
    }
  }

  return BaseSandboxManager.wrapWithSandbox(
    command,
    binShell,
    customConfig,
    abortSignal,
  )
}

// 获取排除的命令（不应被沙盒化的命令）
function getExcludedCommands(): string[] {
  const settings = getSettings_DEPRECATED()
  return settings?.sandbox?.excludedCommands ?? []
}
```

### 16.4.6 沙盒管理器接口

```typescript
export interface ISandboxManager {
  // 初始化与状态
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>
  isSandboxingEnabled(): boolean
  isSandboxEnabledInSettings(): boolean
  isSupportedPlatform(): boolean
  checkDependencies(): SandboxDependencyCheck

  // 配置
  getSandboxUnavailableReason(): string | undefined
  isAutoAllowBashIfSandboxedEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  isSandboxRequired(): boolean
  areSandboxSettingsLockedByPolicy(): boolean
  setSandboxSettings(options: {...}): Promise<void>

  // 文件系统配置
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig

  // 网络配置
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
  getAllowUnixSockets(): string[] | undefined
  getAllowLocalBinding(): boolean | undefined

  // 命令执行
  getExcludedCommands(): string[]
  wrapWithSandbox(command: string, binShell?: string, ...): Promise<string>
  cleanupAfterCommand(): void

  // 违规处理
  getSandboxViolationStore(): SandboxViolationStore
  annotateStderrWithSandboxFailures(command: string, stderr: string): string

  // 维护
  refreshConfig(): void
  reset(): Promise<void>
}
```

## 16.5 安全最佳实践

### 16.5.1 分层防御策略

Claude Code权限系统采用多层防御：

```
┌─────────────────────────────────────────────────────────────────┐
│                     第1层：静态规则检查                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ deny规则 → 直接拒绝                                      │   │
│  │ ask规则 → 强制询问                                       │   │
│  │ allow规则 → 直接允许                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     第2层：工具特定检查                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ tool.checkPermissions() - 工具自定义逻辑                  │   │
│  │ 安全路径检查（.git/, .claude/, shell配置）                │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     第3层：AI分类器                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ auto模式 - AI评估操作风险                                 │   │
│  │ 连续拒绝限制 → 降级到用户确认                             │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     第4层：用户交互                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 交互式确认对话框                                          │   │
│  │ 远程审批（CCR、Channel）                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     第5层：沙盒隔离                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ OS级别隔离执行                                            │   │
│  │ 文件系统访问控制                                          │   │
│  │ 网络访问控制                                              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 16.5.2 安全检查免疫

某些安全检查无法被绕过：

```typescript
// 1g. 安全检查绕过免疫
if (toolPermissionResult?.behavior === 'ask' &&
    toolPermissionResult.decisionReason?.type === 'safetyCheck') {
  return toolPermissionResult  // 即使bypassPermissions也需要确认
}

// 安全路径示例：
// - .git/ - Git仓库元数据
// - .claude/ - Claude Code配置
// - .vscode/ - VS Code配置
// - shell配置文件 (.bashrc, .zshrc等)
```

### 16.5.3 设置文件保护

沙盒自动阻止对设置文件的写入：

```typescript
// 始终拒绝写入settings.json以防止沙盒逃逸
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source),
).filter((p): p is string => p !== undefined)
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())

// 也阻止当前工作目录中的设置文件
if (cwd !== originalCwd) {
  denyWrite.push(resolve(cwd, '.claude', 'settings.json'))
  denyWrite.push(resolve(cwd, '.claude', 'settings.local.json'))
}
```

### 16.5.4 Git仓库保护

防止通过伪造Git仓库进行沙盒逃逸：

```typescript
// Git的is_git_directory()将包含HEAD + objects/ + refs/的目录视为裸仓库
// 攻击者可能通过创建这些文件（加上包含core.fsmonitor的config）逃逸沙盒

const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of [originalCwd, cwd]) {
  for (const gitFile of bareGitRepoFiles) {
    const p = resolve(dir, gitFile)
    if (existsSync(p)) {
      denyWrite.push(p)  // 存在的文件：只读挂载
    } else {
      bareGitRepoScrubPaths.push(p)  // 不存在：命令后清理
    }
  }
}

function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try {
      rmSync(p, { recursive: true })
    } catch {
      // ENOENT是预期情况
    }
  }
}
```

### 16.5.5 Auto模式拒绝限制

Auto模式（AI分类器）有拒绝次数限制，防止无限拒绝循环：

```typescript
// src/utils/permissions/denialTracking.ts

export const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 最大连续拒绝次数
  maxTotal: 10,       // 最大总拒绝次数
}

export function shouldFallbackToPrompting(
  state: DenialTrackingState,
): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}

// 达到限制时降级到用户确认
if (shouldFallbackToPrompting(newDenialState)) {
  return {
    ...result,
    decisionReason: {
      type: 'classifier',
      classifier: originalClassifier,
      reason: `${warning}\n\nLatest blocked action: ${classifierReason}`,
    },
  }
}
```

### 16.5.6 权限规则最佳实践

推荐的安全配置模式：

```json
// settings.json
{
  "permissions": {
    "allow": [
      "Read",                    // 允许所有读取操作
      "Bash(npm:*)",            // 允许npm命令
      "Bash(git:*)",            // 允许git命令
      "Edit(//src/**)",         // 允许编辑src目录
      "mcp__filesystem__*"      // 允许filesystem MCP服务器所有工具
    ],
    "deny": [
      "Bash(rm -rf:*)",         // 拒绝危险删除
      "Bash(sudo:*)",           // 拒绝sudo
      "Edit(//.env*)"           // 拒绝编辑环境文件
    ],
    "ask": [
      "Bash(npm publish:*)",    // 发布需要确认
      "Write"                   // 写入新文件需要确认
    ],
    "additionalDirectories": [
      "/path/to/trusted/dir"    // 信任的额外目录
    ]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "network": {
      "allowedDomains": ["api.anthropic.com", "github.com"]
    }
  }
}
```

### 16.5.7 审计与监控

权限系统提供完整的审计能力：

```typescript
// 权限决策存储在toolUseContext中
if (!toolUseContext.toolDecisions) {
  toolUseContext.toolDecisions = new Map()
}
toolUseContext.toolDecisions.set(toolUseID, {
  source: sourceString,     // 决策来源
  decision,                 // 决策类型
  timestamp: Date.now(),    // 时间戳
})

// 分析事件记录
logEvent('tengu_tool_use_granted_in_config', {
  messageID,
  toolName,
  sandboxEnabled: SandboxManager.isSandboxingEnabled(),
})

logEvent('tengu_auto_mode_decision', {
  decision: 'allowed' | 'blocked',
  toolName,
  classifierModel,
  consecutiveDenials,
  classifierDurationMs,
  classifierCostUSD,
})
```

## 总结

Claude Code的权限系统是一个多层次、高度可配置的安全框架，通过以下机制确保AI代理操作的安全性：

1. **类型安全的权限模型**：使用TypeScript类型系统确保权限配置的正确性
2. **灵活的规则系统**：支持allow/deny/ask三种行为，可针对工具和内容细粒度配置
3. **智能分类器**：在auto模式下使用AI评估操作风险，减少用户干预
4. **多渠道授权**：支持本地交互、远程Web审批、消息渠道（Telegram/iMessage）审批
5. **沙盒隔离**：OS级别的执行隔离，防止恶意操作影响系统
6. **完整的审计追踪**：所有权限决策都有详细记录，支持安全审计

这套权限系统平衡了安全性与可用性，既保护了用户系统和数据的安全，又不过度阻碍正常的开发工作流程。
