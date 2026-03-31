# 第13章：MCP协议集成

> 本章深入解析 Claude Code 中 Model Context Protocol (MCP) 的实现原理，包括服务端实现、客户端集成、资源工具映射以及安全机制。

## 13.1 MCP协议概述

### 13.1.1 什么是MCP

Model Context Protocol (MCP) 是由 Anthropic 提出的开放协议，用于标准化 AI 模型与外部工具、资源和上下文之间的交互。它定义了一套统一的接口规范，使得 AI 应用能够：

1. **发现工具** - 动态获取可用工具列表及其输入输出模式
2. **调用工具** - 执行工具操作并获取结果
3. **访问资源** - 读取外部数据源
4. **使用提示模板** - 获取预定义的提示模板

### 13.1.2 MCP架构概览

Claude Code 中的 MCP 实现采用客户端-服务端架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Claude Code 主程序                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────────────────────┐    │
│  │  MCP Client     │    │        MCP Server               │    │
│  │  Management     │    │   (src/entrypoints/mcp.ts)      │    │
│  │                 │    │                                 │    │
│  │  - 连接管理      │    │   - 工具暴露 (ListTools)        │    │
│  │  - 工具发现      │    │   - 工具调用 (CallTool)         │    │
│  │  - 资源获取      │    │   - 标准输入输出传输            │    │
│  └────────┬────────┘    └─────────────────────────────────┘    │
│           │                                                      │
│           │ 多种传输协议                                          │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Transport Layer                        │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │  Stdio  │ │   SSE   │ │  HTTP   │ │WebSocket│       │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │   │
│  └───────┼───────────┼───────────┼───────────┼────────────┘   │
│          │           │           │           │                 │
└──────────┼───────────┼───────────┼───────────┼─────────────────┘
           │           │           │           │
           ▼           ▼           ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ 本地进程  │ │ 远程服务  │ │ HTTP API │ │ 实时连接  │
    │ MCP服务器 │ │ MCP服务器 │ │ MCP服务器│ │ MCP服务器 │
    └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### 13.1.3 核心类型定义

MCP 相关类型定义在 `src/services/mcp/types.ts` 中：

```typescript
// 配置作用域
export type ConfigScope =
  | 'local'      // 本地配置（仅当前项目可见）
  | 'user'       // 用户级配置（所有项目可用）
  | 'project'    // 项目级配置（.mcp.json）
  | 'dynamic'    // 动态配置（命令行传入）
  | 'enterprise' // 企业级配置（托管配置）
  | 'claudeai'   // claude.ai 代理配置
  | 'managed'    // 受管配置

// 传输协议类型
export type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'

// 服务器配置类型（联合类型）
export type McpServerConfig =
  | McpStdioServerConfig      // 本地进程通信
  | McpSSEServerConfig        // Server-Sent Events
  | McpHTTPServerConfig       // HTTP 流式传输
  | McpWebSocketServerConfig  // WebSocket 连接
  | McpSdkServerConfig        // SDK 内部传输
  | McpClaudeAIProxyServerConfig // claude.ai 代理
```

### 13.1.4 传输协议对比

| 传输类型 | 使用场景 | 特点 |
|---------|---------|------|
| `stdio` | 本地 MCP 服务器 | 通过标准输入输出通信，最常用 |
| `sse` | 远程 MCP 服务 | Server-Sent Events，支持 OAuth |
| `http` | HTTP API 服务 | Streamable HTTP，支持 OAuth |
| `ws` | 实时双向通信 | WebSocket 连接 |
| `sdk` | SDK 内部使用 | 内部传输，不暴露给用户 |
| `claudeai-proxy` | claude.ai 连接器 | 通过 claude.ai 代理访问 |

## 13.2 MCP服务端实现

### 13.2.1 服务端入口分析

MCP 服务端实现在 `src/entrypoints/mcp.ts`，它将 Claude Code 的内部工具暴露为 MCP 工具：

```typescript
// src/entrypoints/mcp.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'

export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean,
): Promise<void> {
  // 设置工作目录
  setCwd(cwd)

  // 创建 MCP 服务器实例
  const server = new Server(
    {
      name: 'claude/tengu',
      version: MACRO.VERSION,
    },
    {
      capabilities: {
        tools: {},  // 声明支持工具能力
      },
    },
  )

  // 注册请求处理器...
}
```

### 13.2.2 工具列表处理器

服务端通过 `ListToolsRequestSchema` 处理器暴露工具列表：

```typescript
server.setRequestHandler(
  ListToolsRequestSchema,
  async (): Promise<ListToolsResult> => {
    const toolPermissionContext = getEmptyToolPermissionContext()
    const tools = getTools(toolPermissionContext)

    return {
      tools: await Promise.all(
        tools.map(async tool => {
          // 处理输出模式（MCP SDK 要求根级别 type: "object"）
          let outputSchema: ToolOutput | undefined
          if (tool.outputSchema) {
            const convertedSchema = zodToJsonSchema(tool.outputSchema)
            if (
              convertedSchema.type === 'object'
            ) {
              outputSchema = convertedSchema as ToolOutput
            }
          }

          return {
            ...tool,
            description: await tool.prompt({
              getToolPermissionContext: async () => toolPermissionContext,
              tools,
              agents: [],
            }),
            inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
            outputSchema,
          }
        }),
      ),
    }
  },
)
```

**关键点：**
1. 使用 `zodToJsonSchema` 将 Zod 模式转换为 JSON Schema
2. 输出模式必须是 `type: "object"` 才符合 MCP 规范
3. 工具描述通过 `tool.prompt()` 动态生成

### 13.2.3 工具调用处理器

```typescript
server.setRequestHandler(
  CallToolRequestSchema,
  async ({ params: { name, arguments: args } }): Promise<CallToolResult> => {
    const tools = getTools(toolPermissionContext)
    const tool = findToolByName(tools, name)

    if (!tool) {
      throw new Error(`Tool ${name} not found`)
    }

    // 构建工具使用上下文
    const toolUseContext: ToolUseContext = {
      abortController: createAbortController(),
      options: {
        commands: MCP_COMMANDS,
        tools,
        mainLoopModel: getMainLoopModel(),
        thinkingConfig: { type: 'disabled' },
        mcpClients: [],
        mcpResources: {},
        isNonInteractiveSession: true,
        debug,
        verbose,
        agentDefinitions: { activeAgents: [], allAgents: [] },
      },
      // ... 其他上下文属性
    }

    try {
      // 验证工具是否启用
      if (!tool.isEnabled()) {
        throw new Error(`Tool ${name} is not enabled`)
      }

      // 验证输入参数
      const validationResult = await tool.validateInput?.(args ?? {}, toolUseContext)
      if (validationResult && !validationResult.result) {
        throw new Error(`Tool ${name} input is invalid: ${validationResult.message}`)
      }

      // 执行工具调用
      const finalResult = await tool.call(
        args ?? {},
        toolUseContext,
        hasPermissionsToUseTool,
        createAssistantMessage({ content: [] }),
      )

      return {
        content: [{
          type: 'text',
          text: typeof finalResult === 'string'
            ? finalResult
            : jsonStringify(finalResult.data),
        }],
      }
    } catch (error) {
      return {
        isError: true,
        content: [{
          type: 'text',
          text: errorText,
        }],
      }
    }
  },
)
```

### 13.2.4 服务端架构图

```
┌────────────────────────────────────────────────────────────┐
│                    MCP Server (mcp.ts)                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Server (MCP SDK)                        │ │
│  │                                                      │ │
│  │  capabilities: { tools: {} }                         │ │
│  └──────────────────────┬───────────────────────────────┘ │
│                         │                                  │
│           ┌─────────────┼─────────────┐                   │
│           │             │             │                    │
│           ▼             ▼             ▼                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │ ListTools   │ │ CallTool    │ │ Stdio       │         │
│  │ Handler     │ │ Handler     │ │ Transport   │         │
│  └──────┬──────┘ └──────┬──────┘ └─────────────┘         │
│         │               │                                  │
└─────────┼───────────────┼──────────────────────────────────┘
          │               │
          ▼               ▼
┌─────────────────────────────────────────────────────────┐
│                  Claude Code Tool System                │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │   Read   │ │   Write  │ │   Bash   │ │   Glob   │  │
│  │  Tool    │ │   Tool   │ │   Tool   │ │   Tool   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │  Grep    │ │  Notebook│ │  Skill   │ │   ...    │  │
│  │  Tool    │ │  Edit    │ │   Tool   │ │          │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 13.3 MCP客户端集成

### 13.3.1 客户端连接管理

客户端连接管理通过 `useManageMCPConnections` Hook 实现，位于 `src/services/mcp/useManageMCPConnections.ts`：

```typescript
export function useManageMCPConnections(
  dynamicMcpConfig: Record<string, ScopedMcpServerConfig> | undefined,
  isStrictMcpConfig = false,
) {
  const store = useAppStateStore()
  const setAppState = useSetAppState()

  // 重连定时器引用
  const reconnectTimersRef = useRef<Map<string, NodeJS.Timeout>>(new Map())

  // 批量更新状态
  const pendingUpdatesRef = useRef<PendingUpdate[]>([])
  const flushTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  // 连接尝试回调
  const onConnectionAttempt = useCallback(
    ({ client, tools, commands, resources }) => {
      updateServer({ ...client, tools, commands, resources })

      switch (client.type) {
        case 'connected': {
          // 注册 elicitation 处理器
          registerElicitationHandler(client.client, client.name, setAppState)

          // 设置关闭处理程序（自动重连）
          client.client.onclose = () => {
            // 检查是否被禁用
            if (isMcpServerDisabled(client.name)) {
              return
            }

            // 远程传输自动重连
            if (configType !== 'stdio' && configType !== 'sdk') {
              // 指数退避重连
              void reconnectWithBackoff()
            }
          }

          // 注册 list_changed 通知处理器
          if (client.capabilities?.tools?.listChanged) {
            client.client.setNotificationHandler(
              ToolListChangedNotificationSchema,
              async () => {
                const newTools = await fetchToolsForClient(client)
                updateServer({ ...client, tools: newTools })
              },
            )
          }
          break
        }
      }
    },
    [updateServer],
  )

  return { reconnectMcpServer, toggleMcpServer }
}
```

### 13.3.2 连接状态类型

MCP 服务器连接状态通过联合类型定义：

```typescript
// 已连接状态
export type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

// 需要认证状态
export type NeedsAuthMCPServer = {
  name: string
  type: 'needs-auth'
  config: ScopedMcpServerConfig
}

// 失败状态
export type FailedMCPServer = {
  name: string
  type: 'failed'
  config: ScopedMcpServerConfig
  error?: string
}

// 等待连接状态
export type PendingMCPServer = {
  name: string
  type: 'pending'
  config: ScopedMcpServerConfig
  reconnectAttempt?: number
  maxReconnectAttempts?: number
}

// 禁用状态
export type DisabledMCPServer = {
  name: string
  type: 'disabled'
  config: ScopedMcpServerConfig
}

// 联合类型
export type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

### 13.3.3 自动重连机制

```typescript
// 重连常量
const MAX_RECONNECT_ATTEMPTS = 5
const INITIAL_BACKOFF_MS = 1000
const MAX_BACKOFF_MS = 30000

async function reconnectWithBackoff() {
  for (let attempt = 1; attempt <= MAX_RECONNECT_ATTEMPTS; attempt++) {
    // 检查服务器是否被禁用
    if (isMcpServerDisabled(client.name)) {
      return
    }

    // 更新状态为重连中
    updateServer({
      ...client,
      type: 'pending',
      reconnectAttempt: attempt,
      maxReconnectAttempts: MAX_RECONNECT_ATTEMPTS,
    })

    try {
      const result = await reconnectMcpServerImpl(client.name, client.config)

      if (result.client.type === 'connected') {
        onConnectionAttempt(result)
        return
      }
    } catch (error) {
      // 记录错误
    }

    // 指数退避等待
    const backoffMs = Math.min(
      INITIAL_BACKOFF_MS * Math.pow(2, attempt - 1),
      MAX_BACKOFF_MS,
    )
    await sleep(backoffMs)
  }
}
```

### 13.3.4 配置加载流程

配置加载在 `src/services/mcp/config.ts` 中实现：

```typescript
export async function getClaudeCodeMcpConfigs(
  dynamicServers: Record<string, ScopedMcpServerConfig> = {},
  extraDedupTargets: Promise<Record<string, ScopedMcpServerConfig>> = Promise.resolve({}),
): Promise<{
  servers: Record<string, ScopedMcpServerConfig>
  errors: PluginError[]
}> {
  // 1. 加载企业配置（如果存在，则独占控制）
  const { servers: enterpriseServers } = getMcpConfigsByScope('enterprise')
  if (doesEnterpriseMcpConfigExist()) {
    return { servers: filterByPolicy(enterpriseServers), errors: [] }
  }

  // 2. 加载各作用域配置
  const { servers: userServers } = getMcpConfigsByScope('user')
  const { servers: projectServers } = getMcpConfigsByScope('project')
  const { servers: localServers } = getMcpConfigsByScope('local')

  // 3. 加载插件 MCP 服务器
  const pluginResult = await loadAllPluginsCacheOnly()
  const pluginMcpServers: Record<string, ScopedMcpServerConfig> = {}
  for (const plugin of pluginResult.enabled) {
    const servers = await getPluginMcpServers(plugin, mcpErrors)
    if (servers) Object.assign(pluginMcpServers, servers)
  }

  // 4. 去重处理（手动配置优先于插件）
  const { servers: dedupedPluginServers } = dedupPluginMcpServers(
    enabledPluginServers,
    enabledManualServers,
  )

  // 5. 合并配置（优先级：plugin < user < project < local）
  const configs = Object.assign(
    {},
    dedupedPluginServers,
    userServers,
    approvedProjectServers,
    localServers,
  )

  // 6. 应用策略过滤
  return { servers: filterByPolicy(configs), errors: mcpErrors }
}
```

### 13.3.5 客户端架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                  useManageMCPConnections Hook                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    配置加载阶段                          │   │
│  │                                                         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│  │  │ Enterprise│ │  User   │ │ Project  │ │  Local   │  │   │
│  │  │  Config  │ │ Config  │ │ Config   │ │ Config   │  │   │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │   │
│  │       │            │            │            │         │   │
│  │       └────────────┴─────┬──────┴────────────┘         │   │
│  │                          ▼                              │   │
│  │                ┌─────────────────┐                     │   │
│  │                │  Plugin Servers │                     │   │
│  │                │  (dynamic scope)│                     │   │
│  │                └────────┬────────┘                     │   │
│  │                         │                               │   │
│  │                         ▼                               │   │
│  │                ┌─────────────────┐                     │   │
│  │                │  Deduplication  │                     │   │
│  │                └────────┬────────┘                     │   │
│  └─────────────────────────┼───────────────────────────────┘   │
│                            │                                    │
│  ┌─────────────────────────┼───────────────────────────────┐   │
│  │                    连接管理阶段                          │   │
│  │                         │                               │   │
│  │         ┌───────────────┼───────────────┐              │   │
│  │         │               │               │              │   │
│  │         ▼               ▼               ▼              │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐         │   │
│  │  │  Stdio    │  │   SSE     │  │   HTTP    │         │   │
│  │  │ Transport │  │ Transport │  │ Transport │         │   │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘         │   │
│  │        │              │              │                │   │
│  │        └──────────────┼──────────────┘                │   │
│  │                       ▼                               │   │
│  │              ┌─────────────────┐                      │   │
│  │              │  State Update   │                      │   │
│  │              │  (Batched)      │                      │   │
│  │              └─────────────────┘                      │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                   自动重连机制                          │   │
│  │                                                        │   │
│  │  onclose ──▶ check disabled ──▶ backoff ──▶ reconnect │   │
│  │                            │                           │   │
│  │                            ▼                           │   │
│  │                    MAX_RECONNECT_ATTEMPTS              │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 13.4 资源与工具映射

### 13.4.1 工具名称规范化

MCP 工具名称遵循特定格式 `mcp__<server>__<tool>`，规范化逻辑在 `src/services/mcp/normalization.ts`：

```typescript
// Claude.ai 服务器名称前缀
const CLAUDEAI_SERVER_PREFIX = 'claude.ai '

/**
 * 将服务器名称规范化为符合 API 模式 ^[a-zA-Z0-9_-]{1,64}$
 * 将无效字符（包括点和空格）替换为下划线
 */
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')

  // 对于 claude.ai 服务器，合并连续下划线并去除首尾下划线
  if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }

  return normalized
}
```

### 13.4.2 工具名称构建

```typescript
// src/services/mcp/mcpStringUtils.ts
export function buildMcpToolName(serverName: string, toolName: string): string {
  const normalizedServerName = normalizeNameForMCP(serverName)
  return `mcp__${normalizedServerName}__${toolName}`
}

// 解析 MCP 工具名称
export function mcpInfoFromString(toolName: string): { serverName: string; toolName: string } | null {
  const match = toolName.match(/^mcp__(.+)__(.+)$/)
  if (!match) return null
  return { serverName: match[1], toolName: match[2] }
}

// 获取 MCP 前缀
export function getMcpPrefix(serverName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__`
}
```

### 13.4.3 工具过滤器

```typescript
// src/services/mcp/utils.ts

/**
 * 按服务器名称过滤工具
 */
export function filterToolsByServer(tools: Tool[], serverName: string): Tool[] {
  const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
  return tools.filter(tool => tool.name?.startsWith(prefix))
}

/**
 * 检查工具是否属于 MCP 服务器
 */
export function isMcpTool(tool: Tool): boolean {
  return tool.name?.startsWith('mcp__') || tool.isMcp === true
}

/**
 * 排除特定服务器的工具
 */
export function excludeToolsByServer(tools: Tool[], serverName: string): Tool[] {
  const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
  return tools.filter(tool => !tool.name?.startsWith(prefix))
}
```

### 13.4.4 命令过滤

```typescript
/**
 * 检查命令是否属于 MCP 服务器
 * MCP prompts: mcp__<server>__<prompt>
 * MCP skills: <server>:<skill>
 */
export function commandBelongsToServer(command: Command, serverName: string): boolean {
  const normalized = normalizeNameForMCP(serverName)
  const name = command.name
  if (!name) return false
  return (
    name.startsWith(`mcp__${normalized}__`) ||
    name.startsWith(`${normalized}:`)
  )
}

/**
 * 过滤 MCP prompts（不包括 skills）
 */
export function filterMcpPromptsByServer(commands: Command[], serverName: string): Command[] {
  return commands.filter(
    c =>
      commandBelongsToServer(c, serverName) &&
      !(c.type === 'prompt' && c.loadedFrom === 'mcp'),
  )
}
```

### 13.4.5 资源映射

```typescript
// 资源类型定义
export type ServerResource = Resource & { server: string }

/**
 * 按服务器过滤资源
 */
export function filterResourcesByServer(
  resources: ServerResource[],
  serverName: string,
): ServerResource[] {
  return resources.filter(resource => resource.server === serverName)
}

/**
 * 排除特定服务器的资源
 */
export function excludeResourcesByServer(
  resources: Record<string, ServerResource[]>,
  serverName: string,
): Record<string, ServerResource[]> {
  const result = { ...resources }
  delete result[serverName]
  return result
}
```

### 13.4.6 工具映射架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     工具映射系统                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   MCP Server (外部)                       │  │
│  │                                                          │  │
│  │  Tools: ["read_file", "search", "execute"]              │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  名称规范化层                             │  │
│  │                                                          │  │
│  │  "my-server" ──▶ "my_server"                            │  │
│  │  "claude.ai Slack" ──▶ "claude_ai_Slack"                │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   前缀构建层                              │  │
│  │                                                          │  │
│  │  Server: "my_server"                                    │  │
│  │  Tool: "read_file"                                      │  │
│  │                   │                                      │  │
│  │                   ▼                                      │  │
│  │  Result: "mcp__my_server__read_file"                    │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   工具注册表                              │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────┐     │  │
│  │  │ mcp__my_server__read_file  ──▶ MCPTool         │     │  │
│  │  │ mcp__my_server__search     ──▶ MCPTool         │     │  │
│  │  │ mcp__my_server__execute    ──▶ MCPTool         │     │  │
│  │  └────────────────────────────────────────────────┘     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 13.5 MCP安全考虑

### 13.5.1 OAuth认证流程

MCP 客户端的 OAuth 认证在 `src/services/mcp/auth.ts` 中实现：

```typescript
export class ClaudeAuthProvider implements OAuthClientProvider {
  private serverName: string
  private serverConfig: McpSSEServerConfig | McpHTTPServerConfig
  private redirectUri: string

  // 获取客户端元数据
  get clientMetadata(): OAuthClientMetadata {
    return {
      client_name: `Claude Code (${this.serverName})`,
      redirect_uris: [this.redirectUri],
      grant_types: ['authorization_code', 'refresh_token'],
      response_types: ['code'],
      token_endpoint_auth_method: 'none', // 公共客户端
    }
  }

  // 获取令牌
  async tokens(): Promise<OAuthTokens | undefined> {
    const storage = getSecureStorage()
    const data = await storage.readAsync()
    const tokenData = data?.mcpOAuth?.[serverKey]

    if (!tokenData) return undefined

    // 检查令牌是否过期
    const expiresIn = (tokenData.expiresAt - Date.now()) / 1000

    // 主动刷新即将过期的令牌
    if (expiresIn <= 300 && tokenData.refreshToken) {
      const refreshed = await this.refreshAuthorization(tokenData.refreshToken)
      if (refreshed) return refreshed
    }

    return {
      access_token: tokenData.accessToken,
      refresh_token: tokenData.refreshToken,
      expires_in: expiresIn,
      scope: tokenData.scope,
      token_type: 'Bearer',
    }
  }

  // 重定向到授权 URL
  async redirectToAuthorization(authorizationUrl: URL): Promise<void> {
    // 验证 URL 协议
    if (!urlString.startsWith('http://') && !urlString.startsWith('https://')) {
      throw new Error('Invalid authorization URL: must use http:// or https://')
    }

    // 通知 UI 并打开浏览器
    if (this.onAuthorizationUrlCallback) {
      this.onAuthorizationUrlCallback(urlString)
    }

    if (!this.skipBrowserOpen) {
      await openBrowser(urlString)
    }
  }
}
```

### 13.5.2 企业策略控制

```typescript
// src/services/mcp/config.ts

/**
 * 检查 MCP 服务器是否被企业策略拒绝
 */
function isMcpServerDenied(serverName: string, config?: McpServerConfig): boolean {
  const settings = getMcpDenylistSettings()
  if (!settings.deniedMcpServers) return false

  // 名称拒绝检查
  for (const entry of settings.deniedMcpServers) {
    if (isMcpServerNameEntry(entry) && entry.serverName === serverName) {
      return true
    }
  }

  // 命令拒绝检查（stdio 服务器）
  if (config) {
    const serverCommand = getServerCommandArray(config)
    if (serverCommand) {
      for (const entry of settings.deniedMcpServers) {
        if (
          isMcpServerCommandEntry(entry) &&
          commandArraysMatch(entry.serverCommand, serverCommand)
        ) {
          return true
        }
      }
    }

    // URL 拒绝检查（远程服务器）
    const serverUrl = getServerUrl(config)
    if (serverUrl) {
      for (const entry of settings.deniedMcpServers) {
        if (
          isMcpServerUrlEntry(entry) &&
          urlMatchesPattern(serverUrl, entry.serverUrl)
        ) {
          return true
        }
      }
    }
  }

  return false
}

/**
 * URL 模式匹配（支持通配符）
 */
function urlMatchesPattern(url: string, pattern: string): boolean {
  // 转义正则特殊字符，保留 * 作为通配符
  const escaped = pattern.replace(/[.+?^${}()|[\]\\]/g, '\\$&')
  const regexStr = escaped.replace(/\*/g, '.*')
  return new RegExp(`^${regexStr}$`).test(url)
}
```

### 13.5.3 安全存储

```typescript
// 生成服务器唯一密钥
export function getServerKey(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
): string {
  const configJson = jsonStringify({
    type: serverConfig.type,
    url: serverConfig.url,
    headers: serverConfig.headers || {},
  })

  const hash = createHash('sha256')
    .update(configJson)
    .digest('hex')
    .substring(0, 16)

  return `${serverName}|${hash}`
}

// 撤销服务器令牌
export async function revokeServerTokens(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
): Promise<void> {
  const storage = getSecureStorage()
  const tokenData = storage.read()?.mcpOAuth?.[serverKey]

  if (tokenData?.accessToken || tokenData?.refreshToken) {
    // 尝试服务器端撤销
    const metadata = await fetchAuthServerMetadata(...)
    if (metadata?.revocation_endpoint) {
      // 先撤销 refresh token
      await revokeToken({
        endpoint: metadata.revocation_endpoint,
        token: tokenData.refreshToken,
        tokenTypeHint: 'refresh_token',
        clientId: tokenData.clientId,
      })

      // 再撤销 access token
      await revokeToken({
        endpoint: metadata.revocation_endpoint,
        token: tokenData.accessToken,
        tokenTypeHint: 'access_token',
        clientId: tokenData.clientId,
      })
    }
  }

  // 清除本地存储
  clearServerTokensFromLocalStorage(serverName, serverConfig)
}
```

### 13.5.4 敏感信息保护

```typescript
// 敏感 OAuth 参数列表
const SENSITIVE_OAUTH_PARAMS = [
  'state',
  'nonce',
  'code_challenge',
  'code_verifier',
  'code',
]

/**
 * 从 URL 中脱敏敏感参数，用于日志记录
 */
function redactSensitiveUrlParams(url: string): string {
  try {
    const parsedUrl = new URL(url)
    for (const param of SENSITIVE_OAUTH_PARAMS) {
      if (parsedUrl.searchParams.has(param)) {
        parsedUrl.searchParams.set(param, '[REDACTED]')
      }
    }
    return parsedUrl.toString()
  } catch {
    return url
  }
}

/**
 * 获取安全的 MCP 基础 URL（用于分析日志）
 * 移除查询字符串（可能包含访问令牌）
 */
export function getLoggingSafeMcpBaseUrl(config: McpServerConfig): string | undefined {
  if (!('url' in config) || typeof config.url !== 'string') {
    return undefined
  }

  try {
    const url = new URL(config.url)
    url.search = '' // 移除查询字符串
    return url.toString().replace(/\/$/, '') // 移除尾部斜杠
  } catch {
    return undefined
  }
}
```

### 13.5.5 项目级服务器审批

```typescript
export function getProjectMcpServerStatus(
  serverName: string,
): 'approved' | 'rejected' | 'pending' {
  const settings = getSettings_DEPRECATED()
  const normalizedName = normalizeNameForMCP(serverName)

  // 检查拒绝列表
  if (settings?.disabledMcpjsonServers?.some(
    name => normalizeNameForMCP(name) === normalizedName
  )) {
    return 'rejected'
  }

  // 检查批准列表
  if (
    settings?.enabledMcpjsonServers?.some(
      name => normalizeNameForMCP(name) === normalizedName
    ) ||
    settings?.enableAllProjectMcpServers
  ) {
    return 'approved'
  }

  // 危险模式跳过权限检查
  if (
    hasSkipDangerousModePermissionPrompt() &&
    isSettingSourceEnabled('projectSettings')
  ) {
    return 'approved'
  }

  // 非交互模式自动批准
  if (
    getIsNonInteractiveSession() &&
    isSettingSourceEnabled('projectSettings')
  ) {
    return 'approved'
  }

  return 'pending'
}
```

### 13.5.6 安全架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     MCP 安全架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    配置安全层                             │  │
│  │                                                          │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐          │  │
│  │  │  Allowlist │ │  Denylist  │ │ Enterprise │          │  │
│  │  │   Policy   │ │   Policy   │ │   Config   │          │  │
│  │  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘          │  │
│  │        │              │              │                  │  │
│  │        └──────────────┼──────────────┘                  │  │
│  │                       ▼                                 │  │
│  │              ┌─────────────────┐                        │  │
│  │              │ Policy Filter   │                        │  │
│  │              └─────────────────┘                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    认证安全层                             │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │              OAuth 2.0 + PKCE                      │ │  │
│  │  │                                                    │ │  │
│  │  │  1. Authorization Code Flow                       │ │  │
│  │  │  2. State 参数防 CSRF                             │ │  │
│  │  │  3. PKCE 防授权码拦截                             │ │  │
│  │  │  4. Token 自动刷新                                │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │                                                          │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │           Secure Storage (Keychain)                │ │  │
│  │  │                                                    │ │  │
│  │  │  - access_token                                   │ │  │
│  │  │  - refresh_token                                  │ │  │
│  │  │  - client_id / client_secret                      │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    审批安全层                             │  │
│  │                                                          │  │
│  │  Project .mcp.json ──▶ Approval Dialog ──▶ Allow/Deny  │  │
│  │                              │                           │  │
│  │                              ▼                           │  │
│  │                    ┌─────────────────┐                  │  │
│  │                    │ Permission Store │                  │  │
│  │                    │ (enabled/disabled)│                 │  │
│  │                    └─────────────────┘                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    日志安全层                             │  │
│  │                                                          │  │
│  │  - URL 查询字符串脱敏                                    │  │
│  │  - OAuth 参数替换为 [REDACTED]                           │  │
│  │  - 仅记录基础 URL                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 13.6 小结

本章详细介绍了 Claude Code 中 MCP 协议的集成实现：

1. **协议概述**：MCP 是标准化 AI 模型与外部工具交互的开放协议，支持工具发现、调用和资源访问

2. **服务端实现**：通过 `src/entrypoints/mcp.ts` 将 Claude Code 内部工具暴露为 MCP 工具，支持 `ListTools` 和 `CallTool` 请求

3. **客户端集成**：`useManageMCPConnections` Hook 管理多传输协议连接，实现自动重连和状态同步

4. **资源映射**：工具名称规范化为 `mcp__<server>__<tool>` 格式，支持过滤和排除操作

5. **安全机制**：包括 OAuth 2.0 + PKCE 认证、企业策略控制、安全存储和项目级审批

MCP 协议的集成使得 Claude Code 能够无缝扩展功能，连接各种外部工具和服务，同时保持安全性和可管理性。

---

*下一章：插件系统*
