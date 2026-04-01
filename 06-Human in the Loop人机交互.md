# 第6章：Human in the Loop 人机交互

> 本章深入分析 Claude Code 的人机交互机制，它确保 AI 在执行敏感操作前获得用户确认，实现安全可控的 AI 助手。

## 6.1 概述

### 6.1.1 什么是 Human in the Loop

Human in the Loop（HITL）是一种设计模式，在 AI 系统执行关键操作前，暂停并请求人类确认。Claude Code 通过权限系统实现这一机制：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Human in the Loop 流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  AI 请求执行工具 → 权限检查 → 需要确认? → 是 → 显示确认对话框        │
│                                   │                                 │
│                                   ↓                                 │
│                                   否 → 直接执行                      │
│                                                                     │
│  用户响应: ┬─ Allow（允许）→ 执行工具                                │
│           ├─ Deny（拒绝）→ 返回错误给 AI                            │
│           └─ Abort（中止）→ 终止当前操作                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.1.2 核心目标

| 目标 | 描述 |
|------|------|
| **安全性** | 防止 AI 执行危险或未授权的操作 |
| **可控性** | 用户可以审查并修改 AI 的行为 |
| **透明性** | 用户清楚了解 AI 正在做什么 |
| **灵活性** | 支持多种权限模式和自动批准规则 |

### 6.1.3 与 Agent Loop 的关系

Human in the Loop 是 Agent Loop 中工具执行阶段的关键环节：

```
Agent Loop
    │
    ├── API 调用 → 生成 tool_use
    │
    └── 工具执行
            │
            ├── hasPermissionsToUseTool() ── 权限检查
            │       │
            │       ├── allow → 执行工具
            │       ├── deny → 返回错误
            │       └── ask → 触发 Human in the Loop
            │
            └── 执行工具 → 返回结果
```

---

## 6.2 权限检查流水线

### 6.2.1 hasPermissionsToUseTool 函数

权限检查的入口函数，定义在 `src/utils/permissions/permissions.ts`：

```typescript
export const hasPermissionsToUseTool: CanUseToolFn = async (
  tool: Tool,
  input: { [key: string]: unknown },
  context: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
): Promise<PermissionDecision> => {
  // 1. 执行内部权限检查
  const result = await hasPermissionsToUseToolInner(tool, input, context)

  // 2. 处理允许的情况
  if (result.behavior === 'allow') {
    return result
  }

  // 3. 处理需要询问的情况
  if (result.behavior === 'ask') {
    // 3a. dontAsk 模式：转换为拒绝
    if (appState.toolPermissionContext.mode === 'dontAsk') {
      return { behavior: 'deny', message: DONT_ASK_REJECT_MESSAGE(tool.name) }
    }

    // 3b. auto 模式：使用 AI classifier 判断
    if (appState.toolPermissionContext.mode === 'auto') {
      return await runAutoModeClassifier(tool, input, context)
    }

    // 3c. 异步代理：无法显示对话框时自动拒绝
    if (appState.toolPermissionContext.shouldAvoidPermissionPrompts) {
      return { behavior: 'deny', message: AUTO_REJECT_MESSAGE(tool.name) }
    }
  }

  return result
}
```

### 6.2.2 权限检查步骤

```
┌─────────────────────────────────────────────────────────────────────┐
│                    权限检查流水线                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: 规则检查                                                   │
│  ├── 1a. 拒绝规则 (deny rules) → 直接拒绝                           │
│  ├── 1b. 询问规则 (ask rules) → 需要确认                            │
│  ├── 1c. 工具自身权限检查 (tool.checkPermissions)                   │
│  ├── 1d. 工具拒绝 → 拒绝                                            │
│  ├── 1e. 需要用户交互 → 需要确认                                    │
│  ├── 1f. 内容询问规则 → 需要确认                                    │
│  └── 1g. 安全检查 (.git/, .claude/) → 需要确认                      │
│                                                                     │
│  Step 2: 模式检查                                                   │
│  ├── 2a. bypassPermissions 模式 → 直接允许                          │
│  └── 2b. 允许规则 (allow rules) → 直接允许                          │
│                                                                     │
│  Step 3: 默认行为                                                   │
│  └── 转换为 'ask' → 触发用户确认                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2.3 权限决策类型

```typescript
type PermissionDecision =
  | { behavior: 'allow'; updatedInput?: object; decisionReason: DecisionReason }
  | { behavior: 'deny'; message: string; decisionReason: DecisionReason }
  | { behavior: 'ask'; message: string; suggestions?: PermissionUpdate[] }
```

**决策类型说明**：

| 行为 | 描述 | 后续动作 |
|------|------|----------|
| `allow` | 允许执行 | 直接调用工具 |
| `deny` | 拒绝执行 | 返回错误消息给 AI |
| `ask` | 需要确认 | 显示权限对话框 |

### 6.2.4 决策原因类型

```typescript
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'safetyCheck'; reason: string }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'other'; reason: string }
```

---

## 6.3 交互式权限处理

### 6.3.1 handleInteractivePermission 函数

当权限检查返回 `ask` 时，调用此函数处理用户交互：

```typescript
// src/hooks/toolPermission/handlers/interactiveHandler.ts

function handleInteractivePermission(
  params: InteractivePermissionParams,
  resolve: (decision: PermissionDecision) => void,
): void {
  const { ctx, description, result } = params

  // 1. 推送权限确认到队列
  ctx.pushToQueue({
    assistantMessage: ctx.assistantMessage,
    tool: ctx.tool,
    description,
    input: displayInput,
    toolUseContext: ctx.toolUseContext,
    toolUseID: ctx.toolUseID,
    permissionResult: result,

    // 回调函数
    onUserInteraction() {
      userInteracted = true
      clearClassifierChecking(ctx.toolUseID)
    },
    onAbort() {
      resolveOnce(ctx.cancelAndAbort(undefined, true))
    },
    async onAllow(updatedInput, permissionUpdates, feedback) {
      resolveOnce(await ctx.handleUserAllow(...))
    },
    onReject(feedback) {
      resolveOnce(ctx.cancelAndAbort(feedback))
    },
    async recheckPermission() {
      // 重新检查权限（可能规则已更新）
    },
  })

  // 2. 异步运行 hooks
  void (async () => {
    const hookDecision = await ctx.runHooks(...)
    if (hookDecision && claim()) {
      resolveOnce(hookDecision)
    }
  })()

  // 3. 异步运行 classifier（如果适用）
  if (result.pendingClassifierCheck) {
    void executeAsyncClassifierCheck(...)
  }
}
```

### 6.3.2 竞争机制

权限确认使用"先响应获胜"的竞争机制：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    权限确认竞争机制                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
│  │  用户交互    │   │  Hook 响应   │   │  Classifier  │           │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘           │
│         │                  │                  │                    │
│         └──────────────────┼──────────────────┘                    │
│                            │                                       │
│                            ▼                                       │
│                   ┌────────────────┐                               │
│                   │  claim() 检查  │ ← 原子操作，只允许一个获胜者  │
│                   └────────┬───────┘                               │
│                            │                                       │
│                 ┌──────────┴──────────┐                            │
│                 ▼                     ▼                            │
│           获胜者执行            其他竞争者忽略                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3.3 resolveOnce 保护

```typescript
const { resolve: resolveOnce, isResolved, claim } = createResolveOnce(resolve)

// claim() 是原子操作：检查是否已解决，若未解决则标记为已解决
if (!claim()) return  // 已被其他竞争者解决，直接返回
```

---

## 6.4 权限请求 UI

### 6.4.1 PermissionRequest 组件

```typescript
// src/components/permissions/PermissionRequest.tsx

export function PermissionRequest({
  toolUseConfirm,
  toolUseContext,
  onDone,
  onReject,
  verbose,
  workerBadge,
  setStickyFooter,
}: PermissionRequestProps): React.ReactNode {
  // 1. 注册中断快捷键 (Ctrl+C)
  useKeybinding('app:interrupt', () => {
    onDone()
    onReject()
    toolUseConfirm.onReject()
  }, { context: 'Confirmation' })

  // 2. 超时后发送通知
  useNotifyAfterTimeout(notificationMessage, 'permission_prompt')

  // 3. 根据工具类型选择对应的权限请求组件
  const PermissionComponent = permissionComponentForTool(toolUseConfirm.tool)

  return <PermissionComponent ... />
}
```

### 6.4.2 工具特定的权限组件

```typescript
function permissionComponentForTool(tool: Tool): React.ComponentType {
  switch (tool) {
    case FileEditTool:
      return FileEditPermissionRequest
    case FileWriteTool:
      return FileWritePermissionRequest
    case BashTool:
      return BashPermissionRequest
    case PowerShellTool:
      return PowerShellPermissionRequest
    case WebFetchTool:
      return WebFetchPermissionRequest
    case ExitPlanModeV2Tool:
      return ExitPlanModePermissionRequest
    case EnterPlanModeTool:
      return EnterPlanModePermissionRequest
    case SkillTool:
      return SkillPermissionRequest
    case AskUserQuestionTool:
      return AskUserQuestionPermissionRequest
    case GlobTool:
    case GrepTool:
    case FileReadTool:
      return FilesystemPermissionRequest
    default:
      return FallbackPermissionRequest
  }
}
```

### 6.4.3 ToolUseConfirm 类型

```typescript
type ToolUseConfirm<Input = AnyObject> = {
  assistantMessage: AssistantMessage
  tool: Tool<Input>
  description: string
  input: z.infer<Input>
  toolUseContext: ToolUseContext
  toolUseID: string
  permissionResult: PermissionDecision
  permissionPromptStartTimeMs: number

  // Classifier 相关
  classifierCheckInProgress?: boolean
  classifierAutoApproved?: boolean
  classifierMatchedRule?: string

  // 回调
  onUserInteraction(): void
  onAbort(): void
  onDismissCheckmark?(): void
  onAllow(updatedInput, permissionUpdates, feedback?, contentBlocks?): void
  onReject(feedback?, contentBlocks?): void
  recheckPermission(): Promise<void>
}
```

---

## 6.5 权限模式

### 6.5.1 模式类型

```typescript
type PermissionMode =
  | 'default'           // 默认模式：需要用户确认
  | 'bypassPermissions' // 跳过权限检查
  | 'acceptEdits'       // 自动接受文件编辑
  | 'plan'              // 计划模式
  | 'auto'              // 自动模式（使用 AI classifier）
  | 'dontAsk'           // 不询问，自动拒绝
```

### 6.5.2 模式行为对比

| 模式 | allow 规则 | deny 规则 | ask 规则 | 默认行为 |
|------|:----------:|:---------:|:--------:|:--------:|
| default | ✅ 直接允许 | ✅ 直接拒绝 | ✅ 需要确认 | 需要确认 |
| bypassPermissions | ✅ 直接允许 | ✅ 直接拒绝 | ⚠️ 跳过 | 直接允许 |
| acceptEdits | ✅ 直接允许 | ✅ 直接拒绝 | ✅ 需要确认 | 编辑类允许 |
| auto | ✅ 直接允许 | ✅ 直接拒绝 | ✅ Classifier | Classifier |
| dontAsk | ✅ 直接允许 | ✅ 直接拒绝 | ❌ 转为拒绝 | 转为拒绝 |

### 6.5.3 Auto 模式详解

Auto 模式使用 AI Classifier 自动判断操作是否安全：

```typescript
// Auto 模式流程
if (appState.toolPermissionContext.mode === 'auto') {
  // 1. 检查 acceptEdits 快速路径
  const acceptEditsResult = await tool.checkPermissions(input, {
    ...context,
    getAppState: () => ({ ...state, mode: 'acceptEdits' })
  })
  if (acceptEditsResult.behavior === 'allow') {
    return { behavior: 'allow', decisionReason: { type: 'mode', mode: 'auto' } }
  }

  // 2. 检查安全工具白名单
  if (isAutoModeAllowlistedTool(tool.name)) {
    return { behavior: 'allow', ... }
  }

  // 3. 运行 Classifier
  const classifierResult = await classifyYoloAction(
    context.messages,
    action,
    context.options.tools,
    appState.toolPermissionContext,
    context.abortController.signal,
  )

  if (classifierResult.shouldBlock) {
    // 4. 检查拒绝限制
    if (shouldFallbackToPrompting(denialState)) {
      return { behavior: 'ask', ... }  // 回退到用户确认
    }
    return { behavior: 'deny', message: buildYoloRejectionMessage(...) }
  }

  return { behavior: 'allow', ... }
}
```

### 6.5.4 拒绝限制机制

为防止 Classifier 误判导致无限拒绝，实现了限制机制：

```typescript
const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 连续拒绝上限
  maxTotal: 10,       // 总拒绝上限
}

function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

---

## 6.6 权限规则系统

### 6.6.1 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource    // 规则来源
  ruleBehavior: PermissionBehavior // 行为类型
  ruleValue: PermissionRuleValue  // 规则值
}

type PermissionRuleValue = {
  toolName: string          // 工具名称
  ruleContent?: string      // 规则内容（可选）
}

type PermissionRuleSource =
  | 'userSettings'    // 用户设置
  | 'projectSettings' // 项目设置
  | 'localSettings'   // 本地设置
  | 'policySettings'  // 策略设置
  | 'cliArg'          // 命令行参数
  | 'session'         // 会话临时

type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

### 6.6.2 规则语法示例

```
# 允许规则
Bash(npm:*)           # 允许所有 npm 命令
Bash(git:*)           # 允许所有 git 命令
Read                  # 允许读取任意文件
Edit(*.ts)            # 允许编辑 TypeScript 文件

# 拒绝规则
Bash(rm -rf:*)        # 拒绝危险删除命令
Write(.env:*)         # 拒绝写入环境文件

# 询问规则
Bash(npm publish:*)   # 发布前需要确认
WebFetch              # 网络请求需要确认
```

### 6.6.3 规则匹配优先级

```
1. deny 规则（最高优先级）
   └── 完全匹配 > 前缀匹配 > 通配符匹配

2. ask 规则
   └── 完全匹配 > 前缀匹配 > 通配符匹配

3. allow 规则
   └── 完全匹配 > 前缀匹配 > 通配符匹配

4. 默认行为（最低优先级）
   └── 根据权限模式决定
```

---

## 6.7 远程权限确认

### 6.7.1 Bridge 权限转发

当连接到 claude.ai 时，权限请求可以转发到 Web UI：

```typescript
// Bridge 权限请求
if (bridgeCallbacks && bridgeRequestId) {
  bridgeCallbacks.sendRequest(
    bridgeRequestId,
    ctx.tool.name,
    displayInput,
    ctx.toolUseID,
    description,
    result.suggestions,
    result.blockedPath,
  )

  // 订阅响应
  const unsubscribe = bridgeCallbacks.onResponse(
    bridgeRequestId,
    response => {
      if (!claim()) return  // 本地已响应

      if (response.behavior === 'allow') {
        resolveOnce(ctx.buildAllow(response.updatedInput ?? displayInput))
      } else {
        resolveOnce(ctx.cancelAndAbort(response.message))
      }
    },
  )
}
```

### 6.7.2 Channel 权限中继

支持通过 Telegram、iMessage 等渠道确认权限：

```typescript
// Channel 权限请求
if (channelCallbacks && !ctx.tool.requiresUserInteraction?.()) {
  const params: ChannelPermissionRequestParams = {
    request_id: channelRequestId,
    tool_name: ctx.tool.name,
    description,
    input_preview: truncateForPreview(displayInput),
  }

  // 发送到所有活跃渠道
  for (const client of channelClients) {
    void client.client.notification({
      method: CHANNEL_PERMISSION_REQUEST_METHOD,
      params,
    })
  }

  // 订阅渠道响应
  const unsubscribe = channelCallbacks.onResponse(
    channelRequestId,
    response => {
      if (!claim()) return
      // 处理响应...
    },
  )
}
```

---

## 6.8 安全检查

### 6.8.1 敏感路径保护

某些路径即使在 bypass 模式下也需要确认：

```typescript
// 1g. 安全检查是 bypass 免疫的
if (
  toolPermissionResult?.behavior === 'ask' &&
  toolPermissionResult.decisionReason?.type === 'safetyCheck'
) {
  return toolPermissionResult  // 必须提示用户
}
```

**受保护的路径**：
- `.git/` - Git 仓库配置
- `.claude/` - Claude Code 配置
- `.vscode/` - VS Code 配置
- Shell 配置文件 (`.bashrc`, `.zshrc` 等)
- SSH 密钥 (`.ssh/`)

### 6.8.2 用户交互强制

某些工具必须要求用户交互：

```typescript
// 1e. 工具需要用户交互（即使在 bypass 模式下）
if (
  tool.requiresUserInteraction?.() &&
  toolPermissionResult?.behavior === 'ask'
) {
  return toolPermissionResult
}
```

**需要交互的工具**：
- `AskUserQuestionTool` - 向用户提问
- `ExitPlanModeTool` - 退出计划模式
- `ReviewArtifactTool` - 审查产物

---

## 6.9 总结

Human in the Loop 机制是 Claude Code 安全性的核心保障：

| 机制 | 描述 |
|------|------|
| **权限检查流水线** | 多步骤检查，支持规则、模式、安全检查 |
| **竞争机制** | 用户、Hook、Classifier 竞争响应 |
| **权限模式** | 支持 default/auto/bypass/dontAsk 等多种模式 |
| **规则系统** | 灵活的 allow/deny/ask 规则配置 |
| **远程确认** | 支持通过 Web UI 或移动端确认 |
| **安全保护** | 敏感路径和交互工具强制确认 |

这套机制确保了 AI 助手在保持高效的同时，始终处于用户的可控范围内。

---

*本章基于 `src/utils/permissions/permissions.ts`、`src/hooks/toolPermission/` 和 `src/components/permissions/` 源代码分析编写*
