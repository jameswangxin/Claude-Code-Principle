# 第15章：Skills技能系统

## 概述

Skills技能系统是Claude Code的核心扩展机制，允许用户和开发者通过声明式Markdown文件定义可重用的工作流程和领域知识。技能系统提供了标准化的方式来封装复杂操作、注入专业上下文，并实现自动化任务执行。

技能系统采用分层架构设计，支持多种技能来源：内置技能（bundled）、用户技能（user settings）、项目技能（project settings）、插件技能（plugin）以及MCP技能。这种设计确保了技能的灵活性和可移植性。

## 15.1 技能系统设计

### 15.1.1 核心设计理念

Claude Code的技能系统建立在以下核心原则之上：

1. **声明式定义**：技能通过Markdown文件（SKILL.md）定义，包含YAML前置元数据和指令内容
2. **延迟加载**：技能内容仅在调用时才完全加载，减少启动时的内存占用
3. **上下文隔离**：技能可选择在独立子代理（fork模式）中执行，拥有独立的token预算
4. **权限控制**：技能可声明所需的工具权限，系统自动进行权限检查
5. **热重载**：技能目录的变更会被自动检测，无需重启即可生效

### 15.1.2 技能类型定义

技能系统定义了多种命令类型，其中`PromptCommand`是技能的核心类型：

```typescript
// src/types/command.ts

export type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number  // 内容长度，用于token估算
  argNames?: string[]    // 参数名称列表
  allowedTools?: string[] // 允许使用的工具
  model?: string         // 模型覆盖
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: {
    pluginManifest: PluginManifest
    repository: string
  }
  disableNonInteractive?: boolean
  hooks?: HooksSettings   // 技能调用时注册的钩子
  skillRoot?: string      // 技能资源的基础目录
  context?: 'inline' | 'fork'  // 执行上下文
  agent?: string          // fork模式下使用的代理类型
  effort?: EffortValue    // 努力级别
  paths?: string[]        // 条件技能的路径模式
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>
}
```

### 15.1.3 技能来源层次

技能从多个来源加载，按优先级排列：

| 来源 | 目录路径 | 说明 |
|------|----------|------|
| managed | 管理路径/.claude/skills | 策略管理的技能 |
| userSettings | ~/.claude/skills | 用户全局技能 |
| projectSettings | .claude/skills | 项目级技能 |
| plugin | 插件内部 | 插件提供的技能 |
| bundled | 编译时内置 | CLI内置技能 |
| mcp | MCP服务器 | MCP协议提供的技能 |

### 15.1.4 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      SkillTool (工具层)                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ validateInput│ │checkPermissions│ │   call()   │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    命令注册表 (commands.ts)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ getCommands() → 合并所有来源的技能                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    技能加载器 (loadSkillsDir.ts)                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │loadSkillsFrom│ │loadSkillsFrom│ │createSkillCmd│            │
│  │SkillsDir()   │ │CommandsDir() │ │()            │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    内置技能注册 (bundledSkills.ts)                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │registerBundle│ │getBundledSkill│ │extractFiles │            │
│  │dSkill()      │ │s()           │ │()           │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

## 15.2 技能定义与加载

### 15.2.1 技能文件格式

技能通过目录结构组织，每个技能是一个包含`SKILL.md`文件的目录：

```
.claude/skills/
├── my-skill/
│   ├── SKILL.md        # 必需：技能定义文件
│   ├── schemas/        # 可选：JSON Schema文件
│   │   └── config.json
│   └── templates/      # 可选：模板文件
│       └── example.md
```

`SKILL.md`文件使用YAML前置元数据：

```markdown
---
name: skill-name
description: 简短描述
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: 详细描述何时使用此技能，包括触发短语
argument-hint: "[param1] [param2]"
arguments:
  - param1
  - param2
context: fork
model: sonnet
effort: high
user-invocable: true
paths:
  - "src/**/*.ts"
---

# 技能标题

技能的详细指令内容...

## 输入
- `$param1`: 参数1的描述
- `$param2`: 参数2的描述

## 目标
明确的目标描述

## 步骤

### 1. 第一步
具体操作指令

**成功标准**: 如何判断此步骤完成
```

### 15.2.2 前置元数据解析

`parseSkillFrontmatterFields`函数负责解析技能的前置元数据：

```typescript
// src/skills/loadSkillsDir.ts

export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
  descriptionFallbackLabel: 'Skill' | 'Custom command' = 'Skill',
): {
  displayName: string | undefined
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  argumentHint: string | undefined
  argumentNames: string[]
  whenToUse: string | undefined
  version: string | undefined
  model: ReturnType<typeof parseUserSpecifiedModel> | undefined
  disableModelInvocation: boolean
  userInvocable: boolean
  hooks: HooksSettings | undefined
  executionContext: 'fork' | undefined
  agent: string | undefined
  effort: EffortValue | undefined
  shell: FrontmatterShell | undefined
} {
  const validatedDescription = coerceDescriptionToString(
    frontmatter.description,
    resolvedName,
  )
  const description =
    validatedDescription ??
    extractDescriptionFromMarkdown(markdownContent, descriptionFallbackLabel)

  const userInvocable =
    frontmatter['user-invocable'] === undefined
      ? true
      : parseBooleanFrontmatter(frontmatter['user-invocable'])

  const model =
    frontmatter.model === 'inherit'
      ? undefined
      : frontmatter.model
        ? parseUserSpecifiedModel(frontmatter.model as string)
        : undefined

  const effortRaw = frontmatter['effort']
  const effort =
    effortRaw !== undefined ? parseEffortValue(effortRaw) : undefined

  return {
    displayName:
      frontmatter.name != null ? String(frontmatter.name) : undefined,
    description,
    hasUserSpecifiedDescription: validatedDescription !== null,
    allowedTools: parseSlashCommandToolsFromFrontmatter(
      frontmatter['allowed-tools'],
    ),
    argumentHint:
      frontmatter['argument-hint'] != null
        ? String(frontmatter['argument-hint'])
        : undefined,
    argumentNames: parseArgumentNames(
      frontmatter.arguments as string | string[] | undefined,
    ),
    whenToUse: frontmatter.when_to_use as string | undefined,
    version: frontmatter.version as string | undefined,
    model,
    disableModelInvocation: parseBooleanFrontmatter(
      frontmatter['disable-model-invocation'],
    ),
    userInvocable,
    hooks: parseHooksFromFrontmatter(frontmatter, resolvedName),
    executionContext: frontmatter.context === 'fork' ? 'fork' : undefined,
    agent: frontmatter.agent as string | undefined,
    effort,
    shell: parseShellFrontmatter(frontmatter.shell, resolvedName),
  }
}
```

### 15.2.3 技能命令创建

`createSkillCommand`函数将解析后的数据转换为Command对象：

```typescript
// src/skills/loadSkillsDir.ts

export function createSkillCommand({
  skillName,
  displayName,
  description,
  hasUserSpecifiedDescription,
  markdownContent,
  allowedTools,
  argumentHint,
  argumentNames,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  source,
  baseDir,
  loadedFrom,
  hooks,
  executionContext,
  agent,
  paths,
  effort,
  shell,
}: {...}): Command {
  return {
    type: 'prompt',
    name: skillName,
    description,
    hasUserSpecifiedDescription,
    allowedTools,
    argumentHint,
    argNames: argumentNames.length > 0 ? argumentNames : undefined,
    whenToUse,
    version,
    model,
    disableModelInvocation,
    userInvocable,
    context: executionContext,
    agent,
    effort,
    paths,
    contentLength: markdownContent.length,
    isHidden: !userInvocable,
    progressMessage: 'running',
    userFacingName(): string {
      return displayName || skillName
    },
    source,
    loadedFrom,
    hooks,
    skillRoot: baseDir,
    async getPromptForCommand(args, toolUseContext) {
      let finalContent = baseDir
        ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
        : markdownContent

      // 参数替换
      finalContent = substituteArguments(
        finalContent,
        args,
        true,
        argumentNames,
      )

      // 替换 ${CLAUDE_SKILL_DIR} 变量
      if (baseDir) {
        const skillDir =
          process.platform === 'win32' ? baseDir.replace(/\\/g, '/') : baseDir
        finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
      }

      // 替换 ${CLAUDE_SESSION_ID} 变量
      finalContent = finalContent.replace(
        /\$\{CLAUDE_SESSION_ID\}/g,
        getSessionId(),
      )

      // MCP技能不执行shell命令（安全考虑）
      if (loadedFrom !== 'mcp') {
        finalContent = await executeShellCommandsInPrompt(
          finalContent,
          {...},
          `/${skillName}`,
          shell,
        )
      }

      return [{ type: 'text', text: finalContent }]
    },
  } satisfies Command
}
```

### 15.2.4 技能目录加载

技能从多个目录加载，支持技能目录和遗留命令目录格式：

```typescript
// src/skills/loadSkillsDir.ts

export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const userSkillsDir = join(getClaudeConfigHomeDir(), 'skills')
    const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
    const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)

    // 并行加载所有技能来源
    const [
      managedSkills,
      userSkills,
      projectSkillsNested,
      additionalSkillsNested,
      legacyCommands,
    ] = await Promise.all([
      loadSkillsFromSkillsDir(managedSkillsDir, 'policySettings'),
      loadSkillsFromSkillsDir(userSkillsDir, 'userSettings'),
      Promise.all(
        projectSkillsDirs.map(dir =>
          loadSkillsFromSkillsDir(dir, 'projectSettings'),
        ),
      ),
      // 额外目录和遗留命令
      ...
    ])

    // 合并和去重
    const allSkillsWithPaths = [
      ...managedSkills,
      ...userSkills,
      ...projectSkillsNested.flat(),
      ...additionalSkillsNested.flat(),
      ...legacyCommands,
    ]

    // 基于realpath去重
    const fileIds = await Promise.all(
      allSkillsWithPaths.map(({ skill, filePath }) =>
        skill.type === 'prompt'
          ? getFileIdentity(filePath)
          : Promise.resolve(null),
      ),
    )

    // ... 去重逻辑

    return unconditionalSkills
  },
)
```

### 15.2.5 内置技能注册

内置技能通过`registerBundledSkill`函数注册：

```typescript
// src/skills/bundledSkills.ts

export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // 额外参考文件
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}

const bundledSkills: Command[] = []

export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition

  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  // 如果有额外文件，需要提取到磁盘
  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }

  const command: Command = {
    type: 'prompt',
    name: definition.name,
    description: definition.description,
    // ... 其他属性
    source: 'bundled',
    loadedFrom: 'bundled',
    getPromptForCommand,
  }
  bundledSkills.push(command)
}
```

内置技能初始化在启动时完成：

```typescript
// src/skills/bundled/index.ts

export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()

  // 条件性注册的技能
  if (feature('AGENT_TRIGGERS')) {
    const { registerLoopSkill } = require('./loop.js')
    registerLoopSkill()
  }
  // ... 其他条件技能
}
```

### 15.2.6 动态技能发现

系统支持在会话过程中动态发现技能：

```typescript
// src/skills/loadSkillsDir.ts

// 动态发现的技能目录
const dynamicSkillDirs = new Set<string>()
const dynamicSkills = new Map<string, Command>()

// 条件技能（带paths前端的技能）
const conditionalSkills = new Map<string, Command>()
const activatedConditionalSkillNames = new Set<string>()

/**
 * 根据文件路径发现技能目录
 */
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]> {
  const newDirs: string[] = []

  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)

    // 向上遍历到cwd（不包括cwd本身）
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')

      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)
        try {
          await fs.stat(skillDir)
          // 检查是否被gitignore
          if (await isPathGitignored(currentDir, resolvedCwd)) {
            continue
          }
          newDirs.push(skillDir)
        } catch {
          // 目录不存在
        }
      }

      currentDir = dirname(currentDir)
      if (parent === currentDir) break
    }
  }

  // 按路径深度排序（最深的优先）
  return newDirs.sort(
    (a, b) => b.split(pathSep).length - a.split(pathSep).length,
  )
}

/**
 * 激活匹配路径模式的条件技能
 */
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  const activated: string[] = []

  for (const [name, skill] of conditionalSkills) {
    if (!skill.paths || skill.paths.length === 0) continue

    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)

      if (skillIgnore.ignores(relativePath)) {
        // 激活此技能
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        break
      }
    }
  }

  return activated
}
```

## 15.3 技能执行流程

### 15.3.1 SkillTool工具

SkillTool是执行技能的核心工具，它将技能名称转换为可执行的提示内容：

```typescript
// src/tools/SkillTool/SkillTool.ts

export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME,
  searchHint: 'invoke a slash-command skill',
  maxResultSizeChars: 100_000,

  inputSchema: z.object({
    skill: z.string().describe('The skill name. E.g., "commit", "review-pr"'),
    args: z.string().optional().describe('Optional arguments for the skill'),
  }),

  outputSchema: z.union([
    // 内联技能输出
    z.object({
      success: z.boolean(),
      commandName: z.string(),
      allowedTools: z.array(z.string()).optional(),
      model: z.string().optional(),
      status: z.literal('inline').optional(),
    }),
    // Fork技能输出
    z.object({
      success: z.boolean(),
      commandName: z.string(),
      status: z.literal('fork'),
      agentId: z.string(),
      result: z.string(),
    }),
  ]),

  description: async ({ skill }) => `Execute skill: ${skill}`,

  // ... 其他方法
})
```

### 15.3.2 输入验证

`validateInput`方法验证技能调用的有效性：

```typescript
// src/tools/SkillTool/SkillTool.ts

async validateInput({ skill }, context): Promise<ValidationResult> {
  const trimmed = skill.trim()
  if (!trimmed) {
    return {
      result: false,
      message: `Invalid skill format: ${skill}`,
      errorCode: 1,
    }
  }

  // 移除前导斜杠（兼容性处理）
  const hasLeadingSlash = trimmed.startsWith('/')
  const normalizedCommandName = hasLeadingSlash
    ? trimmed.substring(1)
    : trimmed

  // 获取所有可用命令（包括MCP技能）
  const commands = await getAllCommands(context)

  // 检查命令是否存在
  const foundCommand = findCommand(normalizedCommandName, commands)
  if (!foundCommand) {
    return {
      result: false,
      message: `Unknown skill: ${normalizedCommandName}`,
      errorCode: 2,
    }
  }

  // 检查是否禁用模型调用
  if (foundCommand.disableModelInvocation) {
    return {
      result: false,
      message: `Skill ${normalizedCommandName} cannot be used with Skill tool`,
      errorCode: 4,
    }
  }

  // 检查是否为提示类型命令
  if (foundCommand.type !== 'prompt') {
    return {
      result: false,
      message: `Skill ${normalizedCommandName} is not a prompt-based skill`,
      errorCode: 5,
    }
  }

  return { result: true }
}
```

### 15.3.3 权限检查

`checkPermissions`方法执行权限验证：

```typescript
// src/tools/SkillTool/SkillTool.ts

async checkPermissions({ skill, args }, context): Promise<PermissionDecision> {
  const commandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed
  const commands = await getAllCommands(context)
  const commandObj = findCommand(commandName, commands)

  // 规则匹配函数
  const ruleMatches = (ruleContent: string): boolean => {
    const normalizedRule = ruleContent.startsWith('/')
      ? ruleContent.substring(1)
      : ruleContent

    // 精确匹配
    if (normalizedRule === commandName) return true

    // 前缀匹配（如 "review:*" 匹配 "review-pr 123"）
    if (normalizedRule.endsWith(':*')) {
      const prefix = normalizedRule.slice(0, -2)
      return commandName.startsWith(prefix)
    }
    return false
  }

  // 检查拒绝规则
  const denyRules = getRuleByContentsForTool(...)
  for (const [ruleContent, rule] of denyRules.entries()) {
    if (ruleMatches(ruleContent)) {
      return { behavior: 'deny', ... }
    }
  }

  // 检查允许规则
  const allowRules = getRuleByContentsForTool(...)
  for (const [ruleContent, rule] of allowRules.entries()) {
    if (ruleMatches(ruleContent)) {
      return { behavior: 'allow', ... }
    }
  }

  // 自动允许只使用安全属性的技能
  if (commandObj?.type === 'prompt' && skillHasOnlySafeProperties(commandObj)) {
    return { behavior: 'allow', ... }
  }

  // 默认：询问用户
  return { behavior: 'ask', message: `Execute skill: ${commandName}`, ... }
}
```

安全属性白名单：

```typescript
// src/tools/SkillTool/SkillTool.ts

const SAFE_SKILL_PROPERTIES = new Set([
  // PromptCommand属性
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // CommandBase属性
  'name', 'description', 'hasUserSpecifiedDescription', 'isEnabled',
  'isHidden', 'aliases', 'isMcp', 'argumentHint', 'whenToUse',
  'paths', 'version', 'disableModelInvocation', 'userInvocable',
  'loadedFrom', 'immediate', 'userFacingName',
])
```

### 15.3.4 技能执行

`call`方法执行技能的核心逻辑：

```typescript
// src/tools/SkillTool/SkillTool.ts

async call({ skill, args }, context, canUseTool, parentMessage, onProgress) {
  const commandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed
  const commands = await getAllCommands(context)
  const command = findCommand(commandName, commands)

  // 记录技能使用
  recordSkillUsage(commandName)

  // 检查是否应作为forked子代理运行
  if (command?.type === 'prompt' && command.context === 'fork') {
    return executeForkedSkill(
      command, commandName, args, context, canUseTool, parentMessage, onProgress
    )
  }

  // 处理内联技能
  const processedCommand = await processPromptSlashCommand(
    commandName,
    args || '',
    commands,
    context,
  )

  // 提取元数据
  const allowedTools = processedCommand.allowedTools || []
  const model = processedCommand.model
  const effort = command?.type === 'prompt' ? command.effort : undefined

  // 获取工具使用ID
  const toolUseID = getToolUseIDFromParentMessage(parentMessage, SKILL_TOOL_NAME)

  // 标记消息
  const newMessages = tagMessagesWithToolUseID(
    processedCommand.messages.filter(...),
    toolUseID,
  )

  // 返回结果
  return {
    data: {
      success: true,
      commandName,
      allowedTools: allowedTools.length > 0 ? allowedTools : undefined,
      model,
    },
    newMessages,
    contextModifier(ctx) {
      // 更新允许的工具
      // 覆盖模型
      // 覆盖努力级别
      return modifiedContext
    },
  }
}
```

### 15.3.5 Fork模式执行

当技能设置为`context: fork`时，它在独立的子代理中执行：

```typescript
// src/tools/SkillTool/SkillTool.ts

async function executeForkedSkill(
  command: Command & { type: 'prompt' },
  commandName: string,
  args: string | undefined,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<Progress>,
): Promise<ToolResult<Output>> {
  const startTime = Date.now()
  const agentId = createAgentId()

  // 准备forked上下文
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  // 合并努力级别
  const agentDefinition =
    command.effort !== undefined
      ? { ...baseAgent, effort: command.effort }
      : baseAgent

  const agentMessages: Message[] = []

  try {
    // 运行子代理
    for await (const message of runAgent({
      agentDefinition,
      promptMessages,
      toolUseContext: {
        ...context,
        getAppState: modifiedGetAppState,
      },
      canUseTool,
      isAsync: false,
      querySource: 'agent:custom',
      model: command.model as ModelAlias | undefined,
      availableTools: context.options.tools,
      override: { agentId },
    })) {
      agentMessages.push(message)

      // 报告进度
      if ((message.type === 'assistant' || message.type === 'user') && onProgress) {
        // ... 进度处理
      }
    }

    // 提取结果文本
    const resultText = extractResultText(agentMessages, 'Skill execution completed')

    return {
      data: {
        success: true,
        commandName,
        status: 'forked',
        agentId,
        result: resultText,
      },
    }
  } finally {
    // 清理技能内容
    clearInvokedSkillsForAgent(agentId)
  }
}
```

### 15.3.6 执行流程图

```
┌──────────────────────────────────────────────────────────────────┐
│                     模型调用 SkillTool                            │
│                   {skill: "name", args: "..."}                    │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      validateInput()                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 1. 验证技能格式                                              │  │
│  │ 2. 规范化技能名称（移除前导斜杠）                              │  │
│  │ 3. 查找技能是否存在                                          │  │
│  │ 4. 检查disableModelInvocation                               │  │
│  │ 5. 验证是prompt类型                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                    checkPermissions()                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 1. 检查deny规则                                             │  │
│  │ 2. 检查allow规则                                            │  │
│  │ 3. 检查安全属性白名单                                        │  │
│  │ 4. 默认询问用户                                              │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                         call()                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                     检查context模式                          │  │
│  └───────────────────────────┬────────────────────────────────┘  │
│                              │                                    │
│              ┌───────────────┴───────────────┐                   │
│              │                               │                   │
│              ▼                               ▼                   │
│  ┌─────────────────────┐         ┌─────────────────────┐        │
│  │   inline模式        │         │    fork模式          │        │
│  │                     │         │                     │        │
│  │ processPromptSlash  │         │ executeForkedSkill  │        │
│  │ Command()           │         │                     │        │
│  │                     │         │ runAgent()          │        │
│  │ getPromptForCommand │         │   └─子代理循环       │        │
│  │                     │         │                     │        │
│  │ newMessages注入     │         │ 返回result          │        │
│  │ conversation        │         │                     │        │
│  └─────────────────────┘         └─────────────────────┘        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 15.4 内置技能分析

### 15.4.1 update-config技能

`update-config`技能用于配置Claude Code的settings.json文件：

```typescript
// src/skills/bundled/updateConfig.ts

export function registerUpdateConfigSkill(): void {
  registerBundledSkill({
    name: 'update-config',
    description:
      'Use this skill to configure the Claude Code harness via settings.json...',
    allowedTools: ['Read'],
    userInvocable: true,
    async getPromptForCommand(args) {
      // 生成动态JSON Schema
      const jsonSchema = generateSettingsSchema()

      let prompt = UPDATE_CONFIG_PROMPT
      prompt += `\n\n## Full Settings JSON Schema\n\n\`\`\`json\n${jsonSchema}\n\`\`\``

      if (args) {
        prompt += `\n\n## User Request\n\n${args}`
      }

      return [{ type: 'text', text: prompt }]
    },
  })
}
```

该技能的特点：
- 动态生成JSON Schema以保持与类型定义同步
- 提供完整的Hooks配置文档
- 包含配置验证流程

### 15.4.2 simplify技能

`simplify`技能用于代码审查和清理：

```typescript
// src/skills/bundled/simplify.ts

const SIMPLIFY_PROMPT = `# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix any issues found.

## Phase 1: Identify Changes

Run \`git diff\` (or \`git diff HEAD\` if there are staged changes)...

## Phase 2: Launch Three Review Agents in Parallel

Use the ${AGENT_TOOL_NAME} tool to launch all three agents concurrently...

### Agent 1: Code Reuse Review
### Agent 2: Code Quality Review
### Agent 3: Efficiency Review

## Phase 3: Fix Issues
`

export function registerSimplifySkill(): void {
  registerBundledSkill({
    name: 'simplify',
    description:
      'Review changed code for reuse, quality, and efficiency, then fix any issues found.',
    userInvocable: true,
    async getPromptForCommand(args) {
      let prompt = SIMPLIFY_PROMPT
      if (args) {
        prompt += `\n\n## Additional Focus\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

该技能展示了如何：
- 使用Agent工具并行执行多个审查任务
- 结构化多阶段工作流
- 接受用户额外关注点参数

### 15.4.3 loop技能

`loop`技能用于设置循环任务：

```typescript
// src/skills/bundled/loop.ts

const DEFAULT_INTERVAL = '10m'

const USAGE_MESSAGE = `Usage: /loop [interval] <prompt>

Run a prompt or slash command on a recurring interval.

Intervals: Ns, Nm, Nh, Nd (e.g. 5m, 30m, 2h, 1d). Minimum granularity is 1 minute.
If no interval is specified, defaults to ${DEFAULT_INTERVAL}.
`

function buildPrompt(args: string): string {
  return `# /loop — schedule a recurring prompt

Parse the input below into \`[interval] <prompt…>\` and schedule it with ${CRON_CREATE_TOOL_NAME}.

## Parsing (in priority order)

1. **Leading token**: if the first whitespace-delimited token matches \`^\\d+[smhd]$\`...
2. **Trailing "every" clause**: otherwise, if the input ends with \`every <N><unit>\`...
3. **Default**: otherwise, interval is \`${DEFAULT_INTERVAL}\`...

## Action

1. Call ${CRON_CREATE_TOOL_NAME} with:
   - \`cron\`: the expression from the table above
   - \`prompt\`: the parsed prompt from above, verbatim
   - \`recurring\`: \`true\`
2. Briefly confirm...
3. **Then immediately execute the parsed prompt now**...

## Input

${args}`
}

export function registerLoopSkill(): void {
  registerBundledSkill({
    name: 'loop',
    description:
      'Run a prompt or slash command on a recurring interval (e.g. /loop 5m /foo, defaults to 10m)',
    whenToUse:
      'When the user wants to set up a recurring task, poll for status...',
    argumentHint: '[interval] <prompt>',
    userInvocable: true,
    isEnabled: isKairosCronEnabled,  // 条件启用
    async getPromptForCommand(args) {
      const trimmed = args.trim()
      if (!trimmed) {
        return [{ type: 'text', text: USAGE_MESSAGE }]
      }
      return [{ type: 'text', text: buildPrompt(trimmed) }]
    },
  })
}
```

该技能展示了：
- 条件启用（`isEnabled`回调）
- 参数解析逻辑
- 与其他工具（CronTool）的集成

### 15.4.4 skillify技能

`skillify`技能用于从当前会话创建新技能：

```typescript
// src/skills/bundled/skillify.ts

const SKILLIFY_PROMPT = `# Skillify {{userDescriptionBlock}}

You are capturing this session's repeatable process as a reusable skill.

## Your Session Context

Here is the session memory summary:
<session_memory>
{{sessionMemory}}
</session_memory>

Here are the user's messages during this session...
<user_messages>
{{userMessages}}
</user_messages>

## Your Task

### Step 1: Analyze the Session
### Step 2: Interview the User
### Step 3: Write the SKILL.md
### Step 4: Confirm and Save
`

export function registerSkillifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // 仅内部用户可用
  }

  registerBundledSkill({
    name: 'skillify',
    description:
      "Capture this session's repeatable process into a skill...",
    allowedTools: [
      'Read', 'Write', 'Edit', 'Glob', 'Grep', 'AskUserQuestion', 'Bash(mkdir:*)',
    ],
    userInvocable: true,
    disableModelInvocation: true,  // 禁止模型自动调用
    argumentHint: '[description of the process you want to capture]',
    async getPromptForCommand(args, context) {
      const sessionMemory =
        (await getSessionMemoryContent()) ?? 'No session memory available.'
      const userMessages = extractUserMessages(
        getMessagesAfterCompactBoundary(context.messages),
      )

      const userDescriptionBlock = args
        ? `The user described this process as: "${args}"`
        : ''

      const prompt = SKILLIFY_PROMPT
        .replace('{{sessionMemory}}', sessionMemory)
        .replace('{{userMessages}}', userMessages.join('\n\n---\n\n'))
        .replace('{{userDescriptionBlock}}', userDescriptionBlock)

      return [{ type: 'text', text: prompt }]
    },
  })
}
```

该技能展示了：
- 访问会话上下文（session memory和消息历史）
- 使用`disableModelInvocation`防止自动调用
- 模板变量替换

### 15.4.5 verify技能

`verify`技能使用外部文件资源：

```typescript
// src/skills/bundled/verify.ts

import { SKILL_FILES, SKILL_MD } from './verifyContent.js'

const { frontmatter, content: SKILL_BODY } = parseFrontmatter(SKILL_MD)

export function registerVerifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return
  }

  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,
    userInvocable: true,
    files: SKILL_FILES,  // 额外参考文件
    async getPromptForCommand(args) {
      const parts: string[] = [SKILL_BODY.trimStart()]
      if (args) {
        parts.push(`## User Request\n\n${args}`)
      }
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
  })
}
```

该技能展示了如何使用`files`属性将额外资源提取到磁盘。

## 15.5 技能扩展开发

### 15.5.1 创建自定义技能

要创建自定义技能，在`.claude/skills/`或`~/.claude/skills/`目录下创建技能目录：

```
.claude/skills/
└── my-workflow/
    └── SKILL.md
```

`SKILL.md`的基本结构：

```markdown
---
name: my-workflow
description: 简短描述这个技能做什么
allowed-tools:
  - Bash(npm:*)
  - Read
  - Write
when_to_use: |
  使用此技能当用户想要执行X操作。
  触发短语示例：'执行X', '运行X流程'
argument-hint: "[project-name]"
arguments:
  - project-name
context: inline
model: sonnet
---

# 我的工作流

这个技能用于执行特定的工作流程。

## 输入
- `$project-name`: 项目名称

## 目标
完成X操作并生成Y输出。

## 步骤

### 1. 准备阶段
执行准备工作...

**成功标准**: 准备工作完成，所有依赖就绪

### 2. 执行阶段
执行主要操作...

**成功标准**: 操作成功完成

### 3. 验证阶段
验证结果...

**成功标准**: 验证通过
```

### 15.5.2 前置元数据详解

| 属性 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | 否 | 显示名称，默认使用目录名 |
| `description` | string | 是 | 简短描述 |
| `allowed-tools` | string[] | 否 | 允许的工具模式列表 |
| `when_to_use` | string | 推荐 | 详细的使用场景描述 |
| `argument-hint` | string | 否 | 参数提示文本 |
| `arguments` | string[] | 否 | 参数名称列表 |
| `context` | string | 否 | `inline`或`fork`，默认`inline` |
| `model` | string | 否 | 模型覆盖（`sonnet`, `opus`, `haiku`） |
| `effort` | string/number | 否 | 努力级别（`low`, `medium`, `high`或整数） |
| `user-invocable` | boolean | 否 | 是否可由用户调用，默认`true` |
| `disable-model-invocation` | boolean | 否 | 禁止模型自动调用 |
| `paths` | string[] | 否 | 条件激活的路径模式 |
| `hooks` | object | 否 | 技能调用时注册的钩子 |
| `shell` | object | 否 | Shell命令执行配置 |

### 15.5.3 参数替换

技能内容支持参数替换：

```
## 输入
- `$arg1`: 第一个参数
- `$arg2`: 第二个参数

使用参数：$arg1 和 $arg2
```

调用时：
```
/my-skill value1 value2
```

参数会按位置或名称替换：
- `$arg1` → `value1`
- `$arg2` → `value2`

### 15.5.4 内置变量

技能内容中可以使用以下内置变量：

| 变量 | 说明 |
|------|------|
| `${CLAUDE_SKILL_DIR}` | 技能目录的绝对路径 |
| `${CLAUDE_SESSION_ID}` | 当前会话ID |

示例：
```markdown
运行脚本：
\`\`\`bash
${CLAUDE_SKILL_DIR}/scripts/setup.sh
\`\`\`
```

### 15.5.5 Shell命令注入

技能支持在内容中注入Shell命令执行结果：

```markdown
当前分支：!`git branch --show-current`

文件列表：
\`\`\`!ls -la\`\`\`
```

通过`shell`前置元数据配置：

```yaml
shell:
  commands:
    - name: build-info
      command: echo "Build at $(date)"
  timeout: 30
```

### 15.5.6 条件技能

使用`paths`属性创建条件技能，仅在操作匹配文件时激活：

```yaml
---
name: typescript-helper
description: TypeScript开发辅助
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
---

# TypeScript辅助工具

这个技能仅在操作TypeScript文件时激活。
```

### 15.5.7 Fork模式

对于自包含的任务，使用`context: fork`在独立子代理中执行：

```yaml
---
name: complex-analysis
description: 复杂代码分析
context: fork
agent: general-purpose
effort: high
---

# 复杂分析任务

这个技能在独立的子代理中运行，拥有自己的token预算。
```

Fork模式的优势：
- 独立的上下文，不影响主会话
- 独立的token预算
- 适合长时间运行的任务
- 结果以摘要形式返回

### 15.5.8 开发内置技能

要开发内置技能：

1. 在`src/skills/bundled/`创建新文件：

```typescript
// src/skills/bundled/mySkill.ts

import { registerBundledSkill } from '../bundledSkills.js'

const MY_SKILL_PROMPT = `# My Skill
...
`

export function registerMySkill(): void {
  registerBundledSkill({
    name: 'my-skill',
    description: '技能描述',
    whenToUse: '使用场景描述',
    argumentHint: '[args]',
    userInvocable: true,
    async getPromptForCommand(args) {
      let prompt = MY_SKILL_PROMPT
      if (args) {
        prompt += `\n\n## User Request\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

2. 在`src/skills/bundled/index.ts`注册：

```typescript
export function initBundledSkills(): void {
  // ... 其他技能
  registerMySkill()
}
```

### 15.5.9 技能文件变更检测

系统使用`skillChangeDetector`模块监控技能文件变更：

```typescript
// src/utils/skills/skillChangeDetector.ts

export async function initialize(): Promise<void> {
  const paths = await getWatchablePaths()
  if (paths.length === 0) return

  watcher = chokidar.watch(paths, {
    persistent: true,
    ignoreInitial: true,
    depth: 2,
    awaitWriteFinish: {
      stabilityThreshold: FILE_STABILITY_THRESHOLD_MS,
      pollInterval: FILE_STABILITY_POLL_INTERVAL_MS,
    },
    usePolling: USE_POLLING,  // Bun使用polling避免FSWatcher死锁
    interval: POLLING_INTERVAL_MS,
  })

  watcher.on('add', handleChange)
  watcher.on('change', handleChange)
  watcher.on('unlink', handleChange)
}

function handleChange(path: string): void {
  scheduleReload(path)
}

function scheduleReload(changedPath: string): void {
  // 防抖处理
  pendingChangedPaths.add(changedPath)
  if (reloadTimer) clearTimeout(reloadTimer)
  reloadTimer = setTimeout(async () => {
    // 执行ConfigChange钩子
    const results = await executeConfigChangeHooks('skills', paths[0]!)
    if (hasBlockingResult(results)) return

    // 清除缓存
    clearSkillCaches()
    clearCommandsCache()
    resetSentSkillNames()

    // 通知变更
    skillsChanged.emit()
  }, RELOAD_DEBOUNCE_MS)
}
```

### 15.5.10 技能提示预算

技能列表在系统提示中占用预算控制：

```typescript
// src/tools/SkillTool/prompt.ts

// 技能列表占用上下文窗口的1%
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const CHARS_PER_TOKEN = 4
export const DEFAULT_CHAR_BUDGET = 8_000

// 每个条目的硬上限
export const MAX_LISTING_DESC_CHARS = 250

export function formatCommandsWithinBudget(
  commands: Command[],
  contextWindowTokens?: number,
): string {
  const budget = getCharBudget(contextWindowTokens)

  // 分区：bundled技能永不截断，其他按需截断
  const bundledIndices = new Set<number>()
  const restCommands: Command[] = []

  for (let i = 0; i < commands.length; i++) {
    if (commands[i]!.type === 'prompt' && commands[i]!.source === 'bundled') {
      bundledIndices.add(i)
    } else {
      restCommands.push(commands[i]!)
    }
  }

  // 计算非bundled技能的最大描述长度
  const maxDescLen = Math.floor(availableForDescs / restCommands.length)

  // 格式化输出
  return commands.map((cmd, i) => {
    if (bundledIndices.has(i)) return fullEntries[i]!.full
    return `- ${cmd.name}: ${truncate(description, maxDescLen)}`
  }).join('\n')
}
```

## 15.6 技能系统工具

### 15.6.1 SkillTool提示

SkillTool的提示定义了模型如何使用技能：

```typescript
// src/tools/SkillTool/prompt.ts

export const getPrompt = memoize(async (_cwd: string): Promise<string> => {
  return `Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match. Skills provide specialized capabilities and domain knowledge.

When users reference a "slash command" or "/<something>" (e.g., "/commit", "/review-pr"), they are referring to a skill. Use this tool to invoke it.

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - \`skill: "pdf"\` - invoke the pdf skill
  - \`skill: "commit", args: "-m 'Fix bug'"\` - invoke with arguments
  - \`skill: "review-pr", args: "123"\` - invoke with arguments
  - \`skill: "ms-office-suite:pdf"\` - invoke using fully qualified name

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- Do not use this tool for built-in CLI commands (like /help, /clear, etc.)
- If you see a <${COMMAND_NAME_TAG}> tag in the current conversation turn, the skill has ALREADY been loaded - follow the instructions directly instead of calling this tool again
`
})
```

### 15.6.2 MCP技能构建器

为避免循环依赖，MCP技能通过注册器模式构建：

```typescript
// src/skills/mcpSkillBuilders.ts

export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error(
      'MCP skill builders not registered — loadSkillsDir.ts has not been evaluated yet',
    )
  }
  return builders
}
```

在`loadSkillsDir.ts`中注册：

```typescript
// 模块初始化时注册
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

## 15.7 总结

Claude Code的技能系统是一个强大而灵活的扩展机制，它通过以下特性支持各种使用场景：

1. **声明式定义**：使用Markdown和YAML前置元数据定义技能，易于编写和维护
2. **多来源支持**：从用户、项目、插件、内置和MCP服务器加载技能
3. **延迟加载**：技能内容仅在调用时加载，优化启动性能
4. **上下文隔离**：Fork模式提供独立的执行环境和token预算
5. **权限控制**：细粒度的工具权限和用户确认机制
6. **热重载**：文件变更自动检测和缓存刷新
7. **条件激活**：基于路径模式的技能自动激活
8. **参数化**：支持参数替换和内置变量

通过理解技能系统的设计和实现，开发者可以创建高效、可重用的工作流程，扩展Claude Code的能力边界。
