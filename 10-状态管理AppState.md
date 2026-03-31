# 第10章：状态管理（AppState）

## 概述

Claude Code 采用精心设计的状态管理架构，将全局状态分为两个层次：

1. **AppState**：React 响应式状态，驱动 UI 渲染
2. **Bootstrap State**：全局单例状态，存储会话级配置和运行时数据

本章深入分析状态管理的设计理念、实现细节和最佳实践。

---

## 10.1 状态架构设计

### 10.1.1 双层状态模型

Claude Code 的状态管理采用双层架构，分离了 UI 响应式状态和全局配置状态：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Claude Code 状态架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    React 组件树                                │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │              AppStateProvider                            │  │ │
│  │  │  ┌─────────────────────────────────────────────────┐    │  │ │
│  │  │  │            AppStoreContext                       │    │  │ │
│  │  │  │  ┌─────────────────────────────────────────┐    │    │  │ │
│  │  │  │  │          AppStateStore                  │    │    │  │ │
│  │  │  │  │  ┌─────────────────────────────────┐    │    │    │  │ │
│  │  │  │  │  │          AppState               │    │    │    │  │ │
│  │  │  │  │  │  - UI 状态                      │    │    │    │  │ │
│  │  │  │  │  │  - 任务状态                     │    │    │    │  │ │
│  │  │  │  │  │  - MCP 连接                     │    │    │    │  │ │
│  │  │  │  │  │  - 插件状态                     │    │    │    │  │ │
│  │  │  │  │  └─────────────────────────────────┘    │    │    │  │ │
│  │  │  │  └─────────────────────────────────────────┘    │    │  │ │
│  │  │  └─────────────────────────────────────────────────┘    │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              │ onChangeAppState                     │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                   Bootstrap State                              │ │
│  │  (src/bootstrap/state.ts)                                      │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  - sessionId / parentSessionId                          │  │ │
│  │  │  - cwd / projectRoot                                     │  │ │
│  │  │  - totalCostUSD / modelUsage                             │  │ │
│  │  │  - mainLoopModelOverride                                 │  │ │
│  │  │  - 遥测状态 (meter, counters)                            │  │ │
│  │  │  - 会话标志 (sessionTrustAccepted, etc.)                 │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.1.2 设计原则

**1. 关注点分离**

- **AppState**：需要触发 UI 重渲染的状态
- **Bootstrap State**：全局配置、成本追踪、会话元数据

**2. 不可变性**

AppState 使用 `DeepImmutable` 类型包装，确保状态只能通过 `setState` 更新：

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  // ...
}> & {
  // 排除 DeepImmutable 的特殊字段（包含函数类型）
  tasks: { [taskId: string]: TaskState }
  // ...
}
```

**3. 单向数据流**

```
Action → setState(updater) → newState → listeners → UI Re-render
                                     → onChangeAppState → 副作用
```

### 10.1.3 状态分类

```
AppState 字段分类
├── 核心配置
│   ├── settings          - 用户设置（来自配置文件）
│   ├── mainLoopModel     - 当前使用的模型
│   ├── verbose           - 详细日志模式
│   └── toolPermissionContext - 权限上下文
│
├── UI 状态
│   ├── expandedView      - 展开视图模式
│   ├── footerSelection   - 底部导航选中项
│   ├── viewSelectionMode - 视图选择模式
│   └── statusLineText    - 状态栏文本
│
├── 任务管理
│   ├── tasks             - 任务字典
│   ├── agentNameRegistry - Agent 名称注册表
│   ├── foregroundedTaskId - 前台任务 ID
│   └── viewingAgentTaskId - 正在查看的 Agent 任务
│
├── MCP & 插件
│   ├── mcp               - MCP 服务器连接
│   └── plugins           - 插件状态
│
├── 远程 & Bridge
│   ├── remoteSessionUrl  - 远程会话 URL
│   ├── remoteConnectionStatus - 连接状态
│   └── replBridge*       - Bridge 相关状态
│
└── 特性状态
    ├── speculation       - 推测执行状态
    ├── promptSuggestion  - 提示建议
    ├── todos             - 待办事项
    └── computerUseMcpState - Computer Use MCP 状态
```

---

## 10.2 AppStateStore 实现

### 10.2.1 Store 基础实现

Store 实现位于 `src/state/store.ts`，是一个极简的发布-订阅模式实现：

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      // Object.is 引用相等检查，避免不必要的更新
      if (Object.is(next, prev)) return
      state = next
      // 状态变更回调
      onChange?.({ newState: next, oldState: prev })
      // 通知所有订阅者
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      // 返回取消订阅函数
      return () => listeners.delete(listener)
    },
  }
}
```

**设计亮点**：

1. **引用相等优化**：使用 `Object.is` 检查，如果 updater 返回相同引用则跳过更新
2. **函数式更新**：`setState` 接收 updater 函数，支持基于前一个状态计算
3. **自动清理**：subscribe 返回取消订阅函数，防止内存泄漏

### 10.2.2 AppState 类型定义

AppState 类型定义位于 `src/state/AppStateStore.ts`，包含 100+ 个状态字段：

```typescript
export type AppState = DeepImmutable<{
  // 核心配置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null

  // 权限上下文
  toolPermissionContext: ToolPermissionContext

  // 远程会话
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number

  // Bridge 状态
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  // ... 更多 Bridge 相关字段

  // Kairos/Assistant 模式
  kairosEnabled: boolean
  agent: string | undefined
}> & {
  // 非 DeepImmutable 字段
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // MCP 状态
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // 插件状态
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { ... }
    needsRefresh: boolean
  }

  // 更多状态...
}
```

### 10.2.3 默认状态初始化

```typescript
export function getDefaultAppState(): AppState {
  // 确定初始权限模式
  const teammateUtils = require('../utils/teammate.js')
  const initialMode: PermissionMode =
    teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
      ? 'plan'
      : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    mainLoopModelForSession: null,
    statusLineText: undefined,
    expandedView: 'none',
    isBriefOnly: false,
    // ... 初始化所有字段

    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },

    mcp: {
      clients: [],
      tools: [],
      commands: [],
      resources: {},
      pluginReconnectKey: 0,
    },

    plugins: {
      enabled: [],
      disabled: [],
      commands: [],
      errors: [],
      installationStatus: { marketplaces: [], plugins: [] },
      needsRefresh: false,
    },
    // ...
  }
}
```

---

## 10.3 响应式状态更新

### 10.3.1 AppStateProvider 组件

AppStateProvider 是状态的顶层提供者，负责创建 Store 并注入 React 上下文：

```typescript
// src/state/AppState.tsx

export const AppStoreContext = React.createContext<AppStateStore | null>(null)
const HasAppStateContext = React.createContext<boolean>(false)

type Props = {
  children: React.ReactNode
  initialState?: AppState
  onChangeAppState?: (args: { newState: AppState; oldState: AppState }) => void
}

export function AppStateProvider({ children, initialState, onChangeAppState }: Props) {
  // 防止嵌套 Provider
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error("AppStateProvider can not be nested within another AppStateProvider")
  }

  // Store 只创建一次，永不改变
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )

  // 挂载时检查 bypass permissions 模式
  useEffect(() => {
    const { toolPermissionContext } = store.getState()
    if (
      toolPermissionContext.isBypassPermissionsModeAvailable &&
      isBypassPermissionsModeDisabled()
    ) {
      logForDebugging("Disabling bypass permissions mode on mount")
      store.setState(prev => ({
        ...prev,
        toolPermissionContext: createDisabledBypassPermissionsContext(
          prev.toolPermissionContext
        ),
      }))
    }
  }, [])

  // 监听外部设置变更
  const onSettingsChange = useEffectEvent((source: SettingSource) =>
    applySettingsChange(source, store.setState)
  )
  useSettingsChange(onSettingsChange)

  return (
    <HasAppStateContext.Provider value={true}>
      <AppStoreContext.Provider value={store}>
        <MailboxProvider>
          <VoiceProvider>{children}</VoiceProvider>
        </MailboxProvider>
      </AppStoreContext.Provider>
    </HasAppStateContext.Provider>
  )
}
```

### 10.3.2 useAppState Hook

useAppState 是最常用的状态订阅 Hook，支持选择性订阅状态切片：

```typescript
/**
 * 订阅 AppState 的一个切片。仅当选中的值变化时（通过 Object.is 比较）才重渲染。
 *
 * 对于多个独立字段，多次调用 Hook：
 * const verbose = useAppState(s => s.verbose)
 * const model = useAppState(s => s.mainLoopModel)
 *
 * 不要从 selector 返回新对象 —— Object.is 会认为它们已改变。
 * 应该选择现有的子对象引用：
 * const { text, promptId } = useAppState(s => s.promptSuggestion) // 正确
 */
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()

  const get = () => {
    const state = store.getState()
    const selected = selector(state)
    return selected
  }

  // 使用 React 的同步外部 Store API
  return useSyncExternalStore(store.subscribe, get, get)
}
```

**工作原理**：

1. `useSyncExternalStore` 是 React 18 的内置 Hook
2. 它在订阅函数返回的快照变化时触发重渲染
3. 使用 `Object.is` 比较快照，实现细粒度订阅

### 10.3.3 useSetAppState Hook

当只需要更新状态而不需要订阅时使用：

```typescript
/**
 * 获取 setAppState 更新器而不订阅任何状态。
 * 返回一个永不改变的稳定引用 —— 只使用此 Hook 的组件
 * 永远不会因状态变化而重渲染。
 */
export function useSetAppState(): (
  updater: (prev: AppState) => AppState
) => void {
  return useAppStore().setState
}
```

### 10.3.4 状态选择器（Selectors）

选择器位于 `src/state/selectors.ts`，用于派生计算状态：

```typescript
/**
 * 获取当前查看的 teammate 任务
 * 选择器保持纯粹和简单 —— 只是数据提取，无副作用
 */

export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined {
  const { viewingAgentTaskId, tasks } = appState

  if (!viewingAgentTaskId) return undefined

  const task = tasks[viewingAgentTaskId]
  if (!task) return undefined

  if (!isInProcessTeammateTask(task)) return undefined

  return task
}

/**
 * 确定用户输入应该路由到哪里
 * 使用判别联合实现类型安全的输入路由
 */
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) {
    return { type: 'viewed', task: viewedTask }
  }

  const { viewingAgentTaskId, tasks } = appState
  if (viewingAgentTaskId) {
    const task = tasks[viewingAgentTaskId]
    if (task?.type === 'local_agent') {
      return { type: 'named_agent', task }
    }
  }

  return { type: 'leader' }
}
```

### 10.3.5 状态更新流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        状态更新流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐                                                  │
│  │   用户操作    │                                                  │
│  │  或工具调用   │                                                  │
│  └──────┬───────┘                                                  │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              setAppState(updater)                         │     │
│  │  const setAppState = useSetAppState()                    │     │
│  │                                                          │     │
│  │  setAppState(prev => ({                                  │     │
│  │    ...prev,                                              │     │
│  │    verbose: !prev.verbose                                │     │
│  │  }))                                                     │     │
│  └────────────────────────┬─────────────────────────────────┘     │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              Store.setState(updater)                      │     │
│  │                                                          │     │
│  │  1. const prev = state                                   │     │
│  │  2. const next = updater(prev)                           │     │
│  │  3. if (Object.is(next, prev)) return  // 优化跳过       │     │
│  │  4. state = next                                         │     │
│  └────────────────────────┬─────────────────────────────────┘     │
│                           │                                        │
│           ┌───────────────┴───────────────┐                       │
│           │                               │                        │
│           ▼                               ▼                        │
│  ┌────────────────────┐        ┌────────────────────────┐         │
│  │   onChangeAppState │        │    通知 listeners       │         │
│  │                    │        │                        │         │
│  │ • 权限模式同步     │        │ for (const listener    │         │
│  │ • 模型设置持久化   │        │   of listeners)        │         │
│  │ • expandedView 同步│        │   listener()           │         │
│  │ • verbose 持久化   │        │                        │         │
│  │ • 缓存清理         │        └───────────┬────────────┘         │
│  └────────────────────┘                    │                       │
│                                            │                       │
│                                            ▼                       │
│                              ┌────────────────────────┐            │
│                              │  React 重渲染          │            │
│                              │                        │            │
│                              │ useSyncExternalStore   │            │
│                              │ 检测到变化，触发       │            │
│                              │ 使用 selector 的组件   │            │
│                              │ 重渲染                 │            │
│                              └────────────────────────┘            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 10.4 状态持久化

### 10.4.1 onChangeAppState 回调

状态变更时的副作用处理位于 `src/state/onChangeAppState.ts`：

```typescript
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // ========== 权限模式同步 ==========
  // 单一入口点，确保所有权限模式变更都通知 CCR 和 SDK
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    const prevExternal = toExternalPermissionMode(prevMode)
    const newExternal = toExternalPermissionMode(newMode)
    if (prevExternal !== newExternal) {
      notifySessionMetadataChanged({
        permission_mode: newExternal,
        is_ultraplan_mode: /* ... */,
      })
    }
    notifyPermissionModeChanged(newMode)
  }

  // ========== 模型设置持久化 ==========
  // 模型被移除
  if (newState.mainLoopModel !== oldState.mainLoopModel &&
      newState.mainLoopModel === null) {
    updateSettingsForSource('userSettings', { model: undefined })
    setMainLoopModelOverride(null)
  }

  // 模型被添加
  if (newState.mainLoopModel !== oldState.mainLoopModel &&
      newState.mainLoopModel !== null) {
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    setMainLoopModelOverride(newState.mainLoopModel)
  }

  // ========== expandedView 持久化 ==========
  if (newState.expandedView !== oldState.expandedView) {
    const showExpandedTodos = newState.expandedView === 'tasks'
    const showSpinnerTree = newState.expandedView === 'teammates'
    // 向后兼容：保存为两个独立的布尔值
    saveGlobalConfig(current => ({
      ...current,
      showExpandedTodos,
      showSpinnerTree,
    }))
  }

  // ========== verbose 持久化 ==========
  if (newState.verbose !== oldState.verbose) {
    saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
  }

  // ========== 设置变更时的缓存清理 ==========
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()

    // 重新应用环境变量
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()
    }
  }
}
```

### 10.4.2 Bootstrap State 持久化

Bootstrap State 位于 `src/bootstrap/state.ts`，管理全局单例状态：

```typescript
// 全局状态对象
const STATE: State = getInitialState()

// 状态字段定义（部分）
type State = {
  // 路径
  originalCwd: string
  projectRoot: string
  cwd: string

  // 会话
  sessionId: SessionId
  parentSessionId: SessionId | undefined
  sessionProjectDir: string | null

  // 成本追踪
  totalCostUSD: number
  modelUsage: { [modelName: string]: ModelUsage }

  // 模型
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting

  // 交互模式
  isInteractive: boolean
  clientType: string

  // 遥测
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // ... 更多计数器

  // 会话标志
  sessionTrustAccepted: boolean
  sessionPersistenceDisabled: boolean
  hasExitedPlanMode: boolean
  // ...
}
```

**会话切换机制**：

```typescript
// 原子切换会话
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void {
  // 清理旧会话的缓存
  STATE.planSlugCache.delete(STATE.sessionId)
  // 更新会话 ID 和项目目录
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  // 发出会话切换信号
  sessionSwitched.emit(sessionId)
}

// 会话切换信号
const sessionSwitched = createSignal<[id: SessionId]>()
export const onSessionSwitch = sessionSwitched.subscribe
```

### 10.4.3 状态恢复机制

成本状态的恢复用于会话恢复场景：

```typescript
export function setCostStateForRestore({
  totalCostUSD,
  totalAPIDuration,
  modelUsage,
  lastDuration,
  // ...
}): void {
  STATE.totalCostUSD = totalCostUSD
  STATE.totalAPIDuration = totalAPIDuration
  STATE.modelUsage = modelUsage ?? {}

  // 调整 startTime 以正确计算墙上时间
  if (lastDuration) {
    STATE.startTime = Date.now() - lastDuration
  }
}
```

---

## 10.5 状态调试工具

### 10.5.1 Signal 原语

Signal 是一个轻量级的事件信号原语，用于纯事件通知（无存储状态）：

```typescript
// src/utils/signal.ts

export type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}

export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() {
      listeners.clear()
    },
  }
}
```

**使用场景**：

```typescript
// 会话切换信号
const sessionSwitched = createSignal<[id: SessionId]>()
export const onSessionSwitch = sessionSwitched.subscribe

// 发出信号
sessionSwitched.emit(sessionId)
```

### 10.5.2 测试状态重置

```typescript
// 仅在测试环境中可用
export function resetStateForTests(): void {
  if (process.env.NODE_ENV !== 'test') {
    throw new Error('resetStateForTests can only be called in tests')
  }
  Object.entries(getInitialState()).forEach(([key, value]) => {
    STATE[key as keyof State] = value as never
  })
  // 重置其他模块级变量
  outputTokensAtTurnStart = 0
  currentTurnTokenBudget = null
  budgetContinuationCount = 0
  sessionSwitched.clear()
}
```

### 10.5.3 调试日志

在开发模式下可以启用状态调试：

```typescript
// 在 AppStateProvider 中
if (DEBUG_MODE) {
  store.subscribe(() => {
    console.log('[AppState]', store.getState())
  })
}
```

### 10.5.4 状态监控最佳实践

**1. 使用 selector 避免不必要的重渲染**

```typescript
// 好的做法：选择具体字段
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)

// 不好的做法：返回新对象
const { verbose, model } = useAppState(s => ({
  verbose: s.verbose,
  model: s.mainLoopModel
}))
```

**2. 使用 useSetAppState 避免订阅**

```typescript
// 如果只需要更新状态，不需要订阅
const setAppState = useSetAppState()

// 此组件永远不会因状态变化而重渲染
const handleClick = () => {
  setAppState(prev => ({ ...prev, verbose: !prev.verbose }))
}
```

**3. 安全访问 Provider 外部状态**

```typescript
// 可能在 Provider 外部调用的组件使用此 Hook
export function useAppStateMaybeOutsideProvider<T>(
  selector: (state: AppState) => T,
): T | undefined {
  const store = useContext(AppStoreContext)
  return useSyncExternalStore(
    store ? store.subscribe : NOOP_SUBSCRIBE,
    () => store ? selector(store.getState()) : undefined
  )
}
```

---

## 10.6 实战案例

### 10.6.1 Teammate 视图管理

```typescript
// src/state/teammateViewHelpers.ts

/**
 * 进入 teammate 转录视图
 * 设置 viewingAgentTaskId，对于 local_agent 设置 retain: true
 */
export function enterTeammateView(
  taskId: string,
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  logEvent('tengu_transcript_view_enter', {})
  setAppState(prev => {
    const task = prev.tasks[taskId]
    const prevId = prev.viewingAgentTaskId
    const prevTask = prevId !== undefined ? prev.tasks[prevId] : undefined

    // 检查是否正在切换 Agent
    const switching =
      prevId !== undefined &&
      prevId !== taskId &&
      isLocalAgent(prevTask) &&
      prevTask.retain

    // 需要更新 retain 标志吗？
    const needsRetain =
      isLocalAgent(task) && (!task.retain || task.evictAfter !== undefined)

    // 需要更新视图状态吗？
    const needsView =
      prev.viewingAgentTaskId !== taskId ||
      prev.viewSelectionMode !== 'viewing-agent'

    // 无变化则跳过
    if (!needsRetain && !needsView && !switching) return prev

    let tasks = prev.tasks
    if (switching || needsRetain) {
      tasks = { ...prev.tasks }
      if (switching) tasks[prevId] = release(prevTask)
      if (needsRetain) {
        tasks[taskId] = { ...task, retain: true, evictAfter: undefined }
      }
    }

    return {
      ...prev,
      viewingAgentTaskId: taskId,
      viewSelectionMode: 'viewing-agent',
      tasks,
    }
  })
}

/**
 * 退出 teammate 视图
 */
export function exitTeammateView(
  setAppState: (updater: (prev: AppState) => AppState) => void,
): void {
  logEvent('tengu_transcript_view_exit', {})
  setAppState(prev => {
    const id = prev.viewingAgentTaskId
    const cleared = {
      ...prev,
      viewingAgentTaskId: undefined,
      viewSelectionMode: 'none' as const,
    }
    if (id === undefined) {
      return prev.viewSelectionMode === 'none' ? prev : cleared
    }
    const task = prev.tasks[id]
    if (!isLocalAgent(task) || !task.retain) return cleared
    return {
      ...cleared,
      tasks: { ...prev.tasks, [id]: release(task) },
    }
  })
}
```

### 10.6.2 外部元数据同步

```typescript
// 从外部元数据恢复 AppState
export function externalMetadataToAppState(
  metadata: SessionExternalMetadata,
): (prev: AppState) => AppState {
  return prev => ({
    ...prev,
    ...(typeof metadata.permission_mode === 'string'
      ? {
          toolPermissionContext: {
            ...prev.toolPermissionContext,
            mode: permissionModeFromString(metadata.permission_mode),
          },
        }
      : {}),
    ...(typeof metadata.is_ultraplan_mode === 'boolean'
      ? { isUltraplanMode: metadata.is_ultraplan_mode }
      : {}),
  })
}
```

---

## 10.7 总结

Claude Code 的状态管理架构体现了以下设计智慧：

1. **双层分离**：React 响应式状态与全局配置状态的清晰分离
2. **极简 Store**：基于发布-订阅的轻量级实现，避免过度工程化
3. **细粒度订阅**：通过 selector 和 `useSyncExternalStore` 实现高效的组件更新
4. **不可变更新**：`DeepImmutable` 类型确保状态只能通过官方 API 修改
5. **副作用集中**：`onChangeAppState` 提供状态变更的单一副作用入口点
6. **会话管理**：原子化的会话切换机制，支持会话恢复和继承

这种架构在保持简单性的同时，提供了足够的灵活性来处理复杂的状态管理需求，是 React 应用状态管理的一个优秀参考实现。
