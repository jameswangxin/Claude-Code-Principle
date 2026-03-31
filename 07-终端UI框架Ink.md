# 第7章：终端UI框架（Ink）

> React-based CLI 应用框架，通过自定义 Ink 审擎实现高性能终端渲染

## 7.1 Ink框架原理

### 7.1.1 框架概述
Ink 是一个基于 React 的终端 UI 渲染框架，最初由 Vadim Demedes 开源，后被 Claude Code 团队进行了大规模定制和扩展。本章节分析的是 Claude Code 项目中的自定义 Ink 实现，包含约 56,000+ 行代码，分布在 70+ 个文件中。

### 核心架构图
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Ink Framework Architecture                         │
├─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │                   React Application                     │
        │              (JSX Components: Box, Text, etc.)             │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │              React Reconciler (reconciler.ts)             │
        │         React Components -> DOM Node Tree Mapping           │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │                 DOM Tree (dom.ts)                           │
        │     Virtual DOM with Yoga Layout Nodes for Flexbox Layout   │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │              Yoga Layout Engine (layout/yoga.ts)            │
        │            Flexbox Layout Calculation (Native WASM)          │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │               Renderer (renderer.ts)                        │
        │         DOM Tree -> Screen Buffer (Char/Style/Hyperlink)     │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │           Screen Buffer (screen.ts)                       │
        │    Packed Int32Array (CharPool, StylePool, HyperlinkPool)    │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │          Diff Engine (log-update.ts)                     │
        │        Frame Diffing + ANSI Escape Sequence Output         │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │              Terminal (terminal.ts)                        │
        │              stdout/stdin + ANSI Control Codes              │
        └───────────────────────────────────────────────────────┘
```

### 7.1.2 核心设计理念
1. **React 生态兼容**：使用标准 React JSX 语法和 hooks，保持开发体验一致性
2. **高性能渲染**：使用 TypedArray 避免对象分配，减少 GC 压力
3. **增量更新**：基于帧差异（Frame Diffing）的最小化终端输出
4. **Flexbox 娡型**：Yoga 布局引擎实现 CSS Flexbox 兼容的终端布局
5. **事件系统**：DOM 风格的捕获/冒泡事件传播机制

---

## 7.2 自定义 Ink 实现分析

### 7.2.1 项目结构概览
```
src/ink/
├── components/           # React 组件
│   ├── App.tsx           # 根组件，处理输入/输出/焦点
│   ├── Box.tsx           # Flexbox 容器组件
│   ├── Text.tsx           # 文本显示组件
│   ├── ScrollBox.tsx      # 可滚动容器组件
│   ├── AlternateScreen.tsx # 全屏模式组件
│   └── Button.tsx          # 可交互按钮组件
├── events/               # 事件系统
│   ├── dispatcher.ts      # 事件分发器
│   ├── keyboard-event.ts  # 键盘事件
│   ├── click-event.ts     # 点击事件
│   └── focus-event.ts     # 焦点事件
├── layout/               # 布局引擎
│   ├── engine.ts         # 布局引擎接口
│   ├── yoga.ts           # Yoga WASM 绑定
│   ├── node.ts           # 布局节点类型
│   └── geometry.ts       # 几何计算
├── hooks/                # React Hooks
│   ├── use-input.ts      # 输入处理
│   ├── use-app.ts        # 应用上下文
│   ├── use-selection.ts  # 文本选择
│   └── use-stdin.ts      # 标准输入访问
├── termio/               # 终端 I/O
│   ├── parser.ts         # 输入解析器
│   ├── ansi.ts           # ANSI 转义序列
│   ├── csi.ts            # CSI 控制序列
│   ├── osc.ts            # OSC 操作序列
│   └── dec.ts            # DEC 私有模式
├── ink.tsx               # Ink 核心类
├── reconciler.ts         # React 协调器
├── renderer.ts           # 帧渲染器
├── screen.ts             # 屏幕缓冲区
├── dom.ts                # 虚拟 DOM
├── focus.ts              # 焦点管理
└── log-update.ts         # 增量更新引擎
```

### 7.2.2 Ink 核心类 (ink.tsx)
`Ink` 类是整个框架的核心，负责协调 React 渲染、布局计算和终端输出。

```typescript
// D:/agent-framework/claude-code/src/ink/ink.tsx
export default class Ink {
  private readonly log: LogUpdate;           // 增量更新引擎
  private readonly terminal: Terminal;       // 终端接口
  private scheduleRender: (() => void) & {   // 节流渲染调度
    cancel?: () => void;
  };
  private readonly container: FiberRoot;      // React Fiber 根节点
  private rootNode: dom.DOMElement;          // 虚拟 DOM 根节点
  readonly focusManager: FocusManager;       // 焦点管理器
  private renderer: Renderer;               // 帧渲染器
  private readonly stylePool: StylePool;     // 样式池
  private charPool: CharPool;               // 字符池
  private hyperlinkPool: HyperlinkPool;      // 超链接池
  // ...
}
```

**核心职责**：
1. 初始化 React 容器和协调器
2. 管理 Yoga 布局节点和焦点管理器
3. 大制帧渲染和节流控制
4. 夌理终端模式切换（Alt Screen、Raw Mode）
5. 管理文本选择状态

### 7.2.3 React 协调器 (reconciler.ts)
使用 `react-reconciler` 将 React 组件树映射到自定义 DOM 结构。

```typescript
// D:/agent-framework/claude-code/src/ink/reconciler.ts
const reconciler = createReconciler<
  ElementNames,        // 元素类型：ink-box, ink-text 等
  Props,              // 组件属性
  DOMElement,         // 容器节点
  DOMElement,         // 宿主节点
  TextNode,           // 文本节点
  DOMElement,         // 挂载节点
  unknown,
  unknown,
  DOMElement,         // 公共实例
  HostContext,        // 宿主上下文
  null,
  NodeJS.Timeout,     // 调度超时
  -1,
  null
>({
  getRootHostContext: () => ({ isInsideText: false }),
  createInstance(type, props, root, hostContext): DOMElement {
  createTextInstance(text, root, hostContext): TextNode
  appendChildNode, insertBeforeNode, removeChildNode
  commitUpdate, commitTextUpdate
  // ...
})
```

**关键生命周期方法**：
- `createInstance`: 创建 DOM 元素节点，绑定 Yoga 劂局节点
- `appendChildNode/insertBeforeNode/removeChildNode`: DOM 树操作
- `commitUpdate`: 属性更新时同步样式到 Yoga 节点
- `resetAfterCommit`: 提交后触发布局计算和渲染

### 7.2.4 衸幕缓冲区 (screen.ts)
使用紧凑的 TypedArray 存储屏幕内容，避免 GC 压力。

```typescript
// D:/agent-framework/claude-code/src/ink/screen.ts
export type Screen = Size & {
  // 每个单元格 22 个 Int32：[charId, packed(styleId|hyperlinkId|width)]
  cells: Int32Array
  cells64: BigInt64Array    // 用于批量清空的 BigInt 视图

  // 共享池 - ID 跨屏幕有效
  charPool: CharPool        // 字符串驻留
  hyperlinkPool: HyperlinkPool // 超链接驻留

  emptyStyleId: number       // 空样式 ID
  damage: Rectangle | undefined // 损坏区域追踪
  noSelect: Uint8Array       // 禁止选择标记
  softWrap: Int32Array       // 软换行标记
}

// 单元格宽度枚举（处理 CJK/emoji 宽字符）
export const enum CellWidth {
  Narrow = 0       // 单宽字符
  Wide = 2         // 宽字符
  SpacerTail = 3   // 宽字符后续 spacer
  SpacerHead = 4   // 软换行 spacer
}
```

**内存优化设计**：
1. **字符串驻留**：`CharPool` 对重复字符串使用唯一 ID，比较时直接对比整数
2. **样式驻留**：`StylePool` 缓存 ANSI 转义序列，避免重复计算
3. **打包布局**：每个单元格 8 字节（2 个 Int32），包含字符、样式、超链接、宽度

---

## 7.3 组件渲染系统

### 7.3.1 渲染流程图
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Render Pipeline                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │     React commit -> reconciler.resetAfterCommit()      │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │         rootNode.onComputeLayout() (Yoga)              │
        │     Calculate Flexbox Layout with terminal width         │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │           rootNode.onRender() -> scheduleRender()         │
        │              Throttled render (16ms interval)               │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │                 renderer() -> Frame                        │
        │      renderNodeToOutput: DOM -> Screen Buffer              │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │           log.render() -> Diff Patches                    │
        │    Compare frontFrame vs backFrame, generate ANSI output   │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │            writeDiffToTerminal() -> stdout                │
        │              Apply ANSI escape sequences                   │
        └───────────────────────────────────────────────────────┘
```

### 7.3.2 渲染器 (renderer.ts)
```typescript
// D:/agent-framework/claude-code/src/ink/renderer.ts
export default function createRenderer(node: DOMElement, stylePool: StylePool): Renderer
{
  let output: Output | undefined
  return options => {
    const { frontFrame, backFrame, terminalWidth, terminalRows } = options

    // 1. 检查 Yoga 布局是否有效
    const computedHeight = node.yogaNode?.getComputedHeight()
    if (!node.yogaNode || hasInvalidHeight || hasInvalidWidth) {
      return emptyFrame(terminalWidth, terminalRows, stylePool, charPool, hyperlinkPool)
    }

    // 2. 创建或重用 Output 缓冲区
    const screen = backScreen ?? createScreen(width, height, stylePool, charPool, hyperlinkPool)
    output = output ?? new Output({ width, height, stylePool, screen })
    output.reset(width, height, screen)

    // 3. 渲染 DOM 树到 Output
    renderNodeToOutput(node, output, { prevScreen })

    // 4. 返回渲染帧
    return {
      screen: output.get(),
      viewport: { width: terminalWidth, height: terminalRows },
      cursor: { x: 0, y: screen.height, visible: !isTTY }
    }
  }
}
```

### 7.3.3 DOM 节点到输出 (render-node-to-output.ts)
将虚拟 DOM 树递归渲染到屏幕缓冲区。

**核心渲染逻辑**：
1. **脏检查优化**：只渲染 `dirty` 标记为 true 的子树
2. **缓存复用**：未改变的子树从 `prevScreen` 直接 blit
3. **滚动处理**：`overflow: scroll` 的节点实现虚拟滚动
4. **绝对定位**：`position: absolute` 节点最后渲染（覆盖层）

```typescript
// 核心渲染函数结构
function renderNodeToOutput(
  node: DOMElement,
  output: Output,
  options: { prevScreen?: Screen }
): void {
  // 跳过隐藏节点
  if (node.isHidden || yoga.getDisplay() === LayoutDisplay.None) return

  // 缓存命中检查
  if (!node.dirty && prevScreen) {
    const cached = nodeCache.get(node)
    if (cached) {
      blitRegion(output.screen, prevScreen, cached.x, cached.y, ...)
      return
    }
  }

  // 根据节点类型分发渲染
  switch (node.nodeName) {
    case 'ink-text':
      renderTextNode(node, output, offsetX, offsetY)
      break
    case 'ink-box':
      renderBox(node, output, offsetX, offsetY)
      break
    // ...
  }

  // 清除脏标记
  node.dirty = false
}
```

### 7.3.4 核心组件

#### Box 组件
```typescript
// D:/agent-framework/claude-code/src/ink/components/Box.tsx
type Props = Except<Styles, 'textWrap'> & {
  ref?: Ref<DOMElement>
  tabIndex?: number       // Tab 导航索引
  autoFocus?: boolean     // 自动获取焦点
  onClick?: (event: ClickEvent) => void
  onFocus?: (event: FocusEvent) => void
  onBlur?: (event: FocusEvent) => void
  onKeyDown?: (event: KeyboardEvent) => void
  onMouseEnter?: () => void
  onMouseLeave?: () => void
}

function Box({ children, flexWrap, flexDirection, flexGrow, ... }): React.ReactNode {
  return (
    <ink-box
      style={{
        flexWrap: flexWrap ?? 'nowrap',
        flexDirection: flexDirection ?? 'row',
        flexGrow: flexGrow ?? 0,
        flexShrink: flexShrink ?? 1,
        overflowX: style.overflowX ?? style.overflow ?? 'visible',
        overflowY: style.overflowY ?? style.overflow ?? 'visible',
        ...style
      }}
    >
      {children}
    </ink-box>
  )
}
```

#### Text 组件
```typescript
// D:/agent-framework/claude-code/src/ink/components/Text.tsx
type Props = {
  color?: Color              // 前景色
  backgroundColor?: Color    // 背景色
  bold?: boolean             // 粗体
  dim?: boolean              // 暗淡
  italic?: boolean           // 斜体
  underline?: boolean        // 下划线
  strikethrough?: boolean    // 删除线
  inverse?: boolean          // 反色
  wrap?: Styles['textWrap']  // 换行模式
  children?: ReactNode
}

function Text({ color, backgroundColor, bold, dim, italic, ... }): React.ReactNode {
  const textStyles = {
    ...(color && { color }),
    ...(backgroundColor && { backgroundColor }),
    ...(dim && { dim }),
    ...(bold && { bold }),
    // ...
  }

  return (
    <ink-text style={memoizedStylesForWrap[wrap]} textStyles={textStyles}>
      {children}
    </ink-text>
  )
}
```

#### ScrollBox 组件
支持虚拟滚动的容器组件，是 Claude Code 消息列表的核心。

```typescript
// D:/agent-framework/claude-code/src/ink/components/ScrollBox.tsx
export type ScrollBoxHandle = {
  scrollTo: (y: number) => void
  scrollBy: (dy: number) => void
  scrollToElement: (el: DOMElement, offset?: number) => void
  scrollToBottom: () => void
  getScrollTop: () => number
  getScrollHeight: () => number
  getViewportHeight: () => number
  isSticky: () => boolean
  subscribe: (listener: () => void) => () => void
}

function ScrollBox({ children, stickyScroll, ...style }): React.ReactNode {
  // 命令式滚动 API
  useImperativeHandle(ref, () => ({
    scrollTo(y) { el.scrollTop = y; scrollMutated(el) },
    scrollBy(dy) { el.pendingScrollDelta = (el.pendingScrollDelta ?? 0) + dy },
    scrollToBottom() { el.stickyScroll = true; forceRender(n => n + 1) },
    // ...
  }))

  return (
    <ink-box style={{ overflowX: 'scroll', overflowY: 'scroll', ... }}>
      <Box flexDirection="column" flexGrow={1} flexShrink={0}>
        {children}
      </Box>
    </ink-box>
  )
}
```

---

## 7.4 事件处理机制

### 7.4.1 事件系统架构
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Event System Architecture                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │              stdin (Raw Mode)                           │
        │           Terminal Input Stream                         │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │          parse-keypress.ts                             │
        │   State Machine for ANSI Sequence Parsing               │
        │   - Keyboard sequences (CSI u, CSI ~)                    │
        │   - Mouse sequences (SGR, DEC)                           │
        │   - Terminal responses (DA1, DECRPM, XTVERSION)           │
        └───────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │             App.processInput()                          │
        │        Batch keys in single discreteUpdates()            │
        └───────────────────────────────────────────────────────┘
                                    │
                      ┌─────────────┴─────────────┐
                      ▼                           ▼
        ┌───────────────────────┐   ┌───────────────────────┐
        │  EventEmitter (Legacy) │   │  Dispatcher (DOM-style) │
        │   useInput() hook      │   │  Capture/Bubble phases  │
        └───────────────────────┘   └───────────────────────┘
                                    │
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │               Event Handlers                            │
        │   onClick, onKeyDown, onFocus, onMouseEnter, etc.        │
        └───────────────────────────────────────────────────────┘
```

### 7.4.2 输入解析器 (parse-keypress.ts)
使用状态机解析终端输入序列。

```typescript
// 解析状态
type ParseState = {
  mode: 'GROUND' | 'ESCAPE' | 'CSI' | 'SS3' | 'OSC' | 'DCS' | 'IN_PASTE'
  params: number[]
  intermediate: string
  incomplete: boolean  // 是否有待处理的转义序列
}

// 解析结果
type ParsedInput =
  | { kind: 'key'; sequence: string; input: string; key: Key }
  | { kind: 'mouse'; col: number; row: number; button: number; modifiers: number }
  | { kind: 'response'; response: TerminalResponse }

// 状态机解析
function parseMultipleKeypresses(state: ParseState, input: string | null): [ParsedInput[], ParseState]
```

**支持的输入类型**：
1. **键盘事件**：单字符、功能键、修饰键组合
2. **鼠标事件**：SGR 编码的点击、拖拽、滚轮
3. **终端响应**：DA1、DECRPM、XTVERSION 等

### 7.4.3 事件分发器 (dispatcher.ts)
实现 DOM 风格的捕获/冒泡事件传播。

```typescript
// D:/agent-framework/claude-code/src/ink/events/dispatcher.ts
export class Dispatcher {
  currentEvent: TerminalEvent | null = null
  currentUpdatePriority: number = DefaultEventPriority
  discreteUpdates: DiscreteUpdates | null = null

  // 事件优先级映射
  function getEventPriority(eventType: string): number {
    switch (eventType) {
      case 'keydown':
      case 'click':
      case 'focus':
        return DiscreteEventPriority  // 同步高优先级
      case 'scroll':
      case 'mousemove':
        return ContinuousEventPriority  // 可批处理
      default:
        return DefaultEventPriority
    }
  }

  // 收集事件监听器（捕获 + 冒泡）
  function collectListeners(target: EventTarget, event: TerminalEvent): DispatchListener[] {
    // 从目标向上遍历到根
    // 捕获阶段：unshift（根在前）
    // 冒泡阶段：push（目标在前）
  }

  // 分发事件
  dispatch(target: EventTarget, event: TerminalEvent): boolean {
    const listeners = collectListeners(target, event)
    processDispatchQueue(listeners, event)
    return !event.defaultPrevented
  }
}
```

### 7.4.4 键盘事件 (keyboard-event.ts)
```typescript
export class KeyboardEvent extends TerminalEvent {
  readonly type: 'keydown' | 'keyup' = 'keydown'
  readonly key: string        // 键名：'a', 'Enter', 'ArrowUp'
  readonly code: string       // 物理键码
  readonly metaKey: boolean
  readonly ctrlKey: boolean
  readonly shiftKey: boolean
  readonly altKey: boolean

  // 辅助属性
  readonly leftArrow: boolean
  readonly rightArrow: boolean
  readonly upArrow: boolean
  readonly downArrow: boolean
  readonly return: boolean
  readonly escape: boolean
  readonly tab: boolean
  readonly delete: boolean
}
```

---

## 7.5 焦点管理与键盘交互

### 7.5.1 焦点管理器 (focus.ts)
实现浏览器风格的焦点管理系统。

```typescript
// D:/agent-framework/claude-code/src/ink/focus.ts
export class FocusManager {
  activeElement: DOMElement | null = null  // 当前焦点元素
  private focusStack: DOMElement[] = []     // 焦点历史栈
  private enabled: boolean = true

  // 获取焦点
  focus(node: DOMElement): void {
    if (node === this.activeElement) return
    const previous = this.activeElement

    // 推入焦点栈
    if (previous) {
      this.focusStack.push(previous)
      this.dispatchFocusEvent(previous, new FocusEvent('blur', node))
    }

    this.activeElement = node
    this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
  }

  // Tab 导航
  focusNext(root: DOMElement): void {
    const tabbable = collectTabbable(root)
    const nextIndex = (currentIndex + 1) % tabbable.length
    this.focus(tabbable[nextIndex])
  }

  focusPrevious(root: DOMElement): void {
    const tabbable = collectTabbable(root)
    const prevIndex = (currentIndex - 1 + tabbable.length) % tabbable.length
    this.focus(tabbable[prevIndex])
  }

  // 节点移除时恢复焦点
  handleNodeRemoved(node: DOMElement, root: DOMElement): void {
    // 从栈中清除已移除的节点
    // 如果移除的是焦点节点，从栈中恢复
  }
}

// 收集可 Tab 导航的节点
function collectTabbable(root: DOMElement): DOMElement[] {
  // 遍历树，收集 tabIndex >= 0 的节点
}
```

### 7.5.2 焦点流程图
```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Focus Management Flow                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│   autoFocus   │         │   Tab/Shift   │         │    Click      │
│  (on mount)   │         │     + Tab     │         │  (on click)   │
└───────────────┘         └───────────────┘         └───────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      FocusManager.focus(node)                            │
│   1. Dispatch 'blur' on previous activeElement                          │
│   2. Push previous to focusStack                                        │
│   3. Set activeElement = node                                           │
│   4. Dispatch 'focus' on new activeElement                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       FocusEvent Propagation                             │
│   Capture Phase: root → ... → parent (onFocusCapture)                   │
│   Target Phase: node (onFocus)                                          │
│   Bubble Phase: node → parent → ... → root (onFocus)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.5.3 useInput Hook
提供声明式的键盘输入处理。

```typescript
// D:/agent-framework/claude-code/src/ink/hooks/use-input.ts
type Handler = (input: string, key: Key, event: InputEvent) => void

const useInput = (inputHandler: Handler, options: Options = {}) => {
  const { setRawMode, internal_eventEmitter } = useStdin()

  // 使用 useLayoutEffect 确保原始模式在渲染前启用
  useLayoutEffect(() => {
    if (options.isActive === false) return
    setRawMode(true)
    return () => setRawMode(false)
  }, [options.isActive, setRawMode])

  // 注册事件监听器
  useEffect(() => {
    const handleData = (event: InputEvent) => {
      if (options.isActive === false) return
      const { input, key } = event
      if (!(input === 'c' && key.ctrl) || !internal_exitOnCtrlC) {
        inputHandler(input, key, event)
      }
    }

    internal_eventEmitter?.on('input', handleData)
    return () => internal_eventEmitter?.removeListener('input', handleData)
  }, [internal_eventEmitter, handleData])
}
```

**使用示例**：
```typescript
useInput((input, key) => {
  if (input === 'q') {
    // 处理 q 键
  }
  if (key.leftArrow) {
    // 处理左箭头
  }
  if (key.ctrl && input === 'c') {
    // 处理 Ctrl+C
  }
})
```

### 7.5.4 Button 组件
可交互的按钮组件，集成焦点和点击处理。

```typescript
// D:/agent-framework/claude-code/src/ink/components/Button.tsx
function Button({
  children,
  onAction,
  tabIndex = 0,
  autoFocus = false,
  ...style
}): React.ReactNode {
  const [focused, setFocused] = useState(false)
  const [hovered, setHovered] = useState(false)
  const [active, setActive] = useState(false)

  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault()
      setActive(true)
    }
  }

  return (
    <Box
      ref={ref}
      tabIndex={tabIndex}
      autoFocus={autoFocus}
      onKeyDown={handleKeyDown}
      onClick={() => onAction?.()}
      onFocus={() => setFocused(true)}
      onBlur={() => setFocused(false)}
      onMouseEnter={() => setHovered(true)}
      onMouseLeave={() => setHovered(false)}
      {...style}
    >
      {typeof children === 'function'
        ? children({ focused, hovered, active })
        : children
      }
    </Box>
  )
}
```

---

## 7.6 性能优化策略

### 7.6.1 帧节流
```typescript
// ink.tsx
const FRAME_INTERVAL_MS = 16  // ~60fps

const deferredRender = (): void => queueMicrotask(this.onRender)
this.scheduleRender = throttle(deferredRender, FRAME_INTERVAL_MS, {
  leading: true,   // 立即执行首次渲染
  trailing: true   // 确保最后一次更新被渲染
})
```

### 7.6.2 脏检查渲染
```typescript
// render-node-to-output.ts
// 只渲染标记为 dirty 的节点
if (!node.dirty && prevScreen) {
  const cached = nodeCache.get(node)
  if (cached) {
    blitRegion(output.screen, prevScreen, ...)
    return  // 跳过子树渲染
  }
}
```

### 7.6.3 增量差异更新
```typescript
// log-update.ts
render(prev: Frame, next: Frame): Diff {
  // 比较前后帧
  diffEach(prev.screen, next.screen, (x, y, removed, added) => {
    // 只输出变化的单元格
  })
}
```

### 7.6.4 字符串驻留
```typescript
// screen.ts
class CharPool {
  private strings: string[] = [' ', '']  // 空格和空字符串预驻留
  private stringMap = new Map<string, number>()
  private ascii: Int32Array  // ASCII 快速查找表

  intern(char: string): number {
    // ASCII 直接数组查找
    if (char.length === 1 && char.charCodeAt(0) < 128) {
      const cached = this.ascii[char.charCodeAt(0)]
      if (cached !== -1) return cached
    }
    // 否则使用 Map
  }
}
```

---

## 7.7 终端特性支持

### 7.7.1 全屏模式 (AlternateScreen)
```typescript
// D:/agent-framework/claude-code/src/ink/components/AlternateScreen.tsx
function AlternateScreen({ children, mouseTracking = true }): React.ReactNode {
  const writeRaw = useContext(TerminalWriteContext)

  useInsertionEffect(() => {
    const ink = instances.get(process.stdout)
    writeRaw(ENTER_ALT_SCREEN + '\x1b[2J\x1b[H' + (mouseTracking ? ENABLE_MOUSE_TRACKING : ''))
    ink?.setAltScreenActive(true, mouseTracking)

    return () => {
      ink?.setAltScreenActive(false)
      writeRaw((mouseTracking ? DISABLE_MOUSE_TRACKING : '') + EXIT_ALT_SCREEN)
    }
  }, [mouseTracking])

  return <Box flexDirection="column" height={size?.rows} width="100%">{children}</Box>
}
```

### 7.7.2 文本选择支持
```typescript
// selection.ts
export type SelectionState = {
  anchor: { row: number; col: number; rowAnchor?: boolean } | null
  focus: { row: number; col: number } | null
  isDragging: boolean
  mode: 'char' | 'word' | 'line'
  scrolledOffAbove: string[]
  scrolledOffBelow: string[]
}

// 鼠标拖拽选择
export function extendSelection(state: SelectionState, col: number, row: number): void
// 双击选择单词
export function selectWordAt(screen: Screen, col: number, row: number): void
// 三击选择行
export function selectLineAt(screen: Screen, row: number): void
// 获取选中文本
export function getSelectedText(state: SelectionState, screen: Screen): string
```

### 7.7.3 扩展键盘协议
支持 Kitty 键盘协议和 modifyOtherKeys，区分修饰键组合。

```typescript
// 启用扩展键盘报告
writeRaw(ENABLE_KITTY_KEYBOARD + ENABLE_MODIFY_OTHER_KEYS)

// 解析扩展序列
// CSI u 格式：\x1b[<code>;<modifiers>u
// 例如 Ctrl+Shift+A: \x1b[65;6u
```

---

## 7.8 小结

Claude Code 的自定义 Ink 实现是一个高度优化的终端 UI 框架，主要特点包括：

1. **React 兼容**：完整的 React 协调器实现，支持 hooks、context、refs
2. **高性能渲染**：TypedArray 屏幕缓冲、脏检查、增量差异更新
3. **Flexbox 布局**：Yoga 布局引擎，CSS 兼容的 Flexbox 模型
4. **DOM 风格事件**：捕获/冒泡事件传播、焦点管理、Tab 导航
5. **终端特性**：全屏模式、鼠标跟踪、文本选择、扩展键盘协议

这套框架支撑了 Claude Code 的核心交互界面，包括消息列表滚动、输入框焦点、快捷键绑定等关键功能。
