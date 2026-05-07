# 设计模式目录 (Design Patterns Catalog)

## 概述

pi-mono 项目中使用了大量经典和现代设计模式。本文档按类别列出项目中使用的主要设计模式，包括创建型、结构型、行为型模式，以及异步和响应式模式。

**查找模式**：
- 按模式名称查找（索引）
- 按使用场景查找（目录）
- 按源文件位置查找（交叉引用）

---

## 创建型模式 (Creational Patterns)

### 1. 工厂模式 (Factory Pattern)

**意图**：通过工厂函数创建对象，隐藏创建逻辑

**使用场景**：

1. **Provider 工厂** (`/packages/ai/src/api-registry.ts`)
```typescript
class APIRegistry {
  private factories = new Map<string, ProviderFactory>()

  register(key: string, factory: ProviderFactory): void {
    this.factories.set(key, factory)
  }

  create(key: string, config: Config): LLMProvider {
    const factory = this.factories.get(key)
    if (!factory) throw new Error(`Unknown provider: ${key}`)
    return factory(config)
  }
}

// 使用
registry.register("openai", (config) => new OpenAIProvider(config))
const provider = registry.create("openai", { apiKey: "..." })
```

2. **工具工厂** (`/packages/coding-agent/src/core/tools/registry.ts`)
```typescript
class ToolRegistry {
  create(config: ToolConfig): Tool {
    switch (config.type) {
      case "read": return new ReadTool(config)
      case "write": return new WriteTool(config)
      case "edit": return new EditTool(config)
      // ...
    }
  }
}
```

---

### 2. 建造者模式 (Builder Pattern)

**意图**：分步骤构建复杂对象

**使用场景**：

**Agent 配置构建器** (`/packages/agent/src/agent.ts`)
```typescript
class AgentBuilder {
  private config: Partial<AgentConfig> = {}

  withLLM(llm: LLMProvider): this {
    this.config.llm = llm
    return this
  }

  withTools(tools: Tool[]): this {
    this.config.tools = tools
    return this
  }

  withSystemPrompt(prompt: string): this {
    this.config.systemPrompt = prompt
    return this
  }

  build(): Agent {
    return new Agent({
      llm: this.config.llm ?? defaultLLM,
      tools: this.config.tools ?? [],
      systemPrompt: this.config.systemPrompt ?? defaultPrompt
    })
  }
}

// 使用
const agent = new AgentBuilder()
  .withLLM(openai)
  .withTools([readTool, writeTool])
  .withSystemPrompt("You are a helpful assistant.")
  .build()
```

---

### 3. 单例模式 (Singleton Pattern)

**意图**：确保全局只有一个实例

**使用场景**：

1. **全局配置** (`/packages/coding-agent/src/config/config.ts`)
```typescript
class ConfigManager {
  private static instance: ConfigManager

  private constructor() {
    // 私有构造函数
  }

  static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager()
    }
    return ConfigManager.instance
  }
}

// 使用
const config = ConfigManager.getInstance()
```

2. **Theme 全局实例** (`/packages/coding-agent/src/modes/interactive/theme/theme.ts`)
```typescript
let globalTheme: Theme | undefined

export function getGlobalTheme(): Theme {
  if (!globalTheme) {
    globalTheme = loadTheme(getDefaultTheme())
  }
  return globalTheme
}

export function setGlobalTheme(theme: Theme): void {
  globalTheme = theme
}
```

---

### 4. 依赖注入 (Dependency Injection)

**意图**：将依赖通过构造函数或参数传入，而非内部创建

**使用场景**：

**Agent 依赖注入** (`/packages/agent/src/agent.ts`)
```typescript
class Agent {
  constructor(
    private llm: LLMProvider,          // 注入
    private tools: ToolRegistry,       // 注入
    private events: EventStream,       // 注入
    private storage: Storage           // 注入
  ) { }
}

// 使用时注入具体实现
const agent = new Agent(
  new OpenAIProvider(apiKey),
  new ToolRegistry(),
  new MemoryEventStream(),
  new FileStorage("/tmp/sessions")
)
```

---

## 结构型模式 (Structural Patterns)

### 5. 适配器模式 (Adapter Pattern)

**意图**：将不兼容的接口转换为兼容的接口

**使用场景**：

**Provider 适配器** (`/packages/ai/src/providers/`)
```typescript
// OpenAI API 的适配
class OpenAIAdapter implements LLMProvider {
  private client: OpenAI

  constructor(apiKey: string) {
    this.client = new OpenAI({ apiKey })
  }

  async *chat(messages: Message[], options: Options): AsyncGenerator<Chunk> {
    // 将统一格式转换为 OpenAI 格式
    const openaiMessages = messages.map(toOpenAIMessage)

    // 调用 OpenAI API
    const stream = await this.client.chat.completions.create({
      messages: openaiMessages,
      stream: true
    })

    // 将 OpenAI 响应转换为统一格式
    for await (const chunk of stream) {
      yield fromOpenAIChunk(chunk)
    }
  }
}
```

**终端 UI 适配器** (`/packages/tui/src/adapters/`)
```typescript
// Kitty 协议适配
class KittyKeyAdapter implements KeyInputAdapter {
  parse(sequence: string): Key | null {
    if (sequence.startsWith("\x1b[")) {
      // 解析 Kitty 转义序列
      return parseKittySequence(sequence)
    }
    return null
  }
}
```

---

### 6. 装饰器模式 (Decorator Pattern)

**意图**：动态添加功能，不修改原始对象

**使用场景**：

**LLM 响应缓存装饰器** (`/packages/ai/src/cache.ts`)
```typescript
class CachedLLM implements LLMProvider {
  constructor(
    private llm: LLMProvider,
    private cache: Cache
  ) { }

  async *chat(messages: Message[], options: Options): AsyncGenerator<Chunk> {
    const cacheKey = this.hashMessages(messages)

    // 检查缓存
    const cached = await this.cache.get(cacheKey)
    if (cached) {
      yield* cached
      return
    }

    // 调用原始 LLM
    const chunks: Chunk[] = []
    for await (const chunk of this.llm.chat(messages, options)) {
      chunks.push(chunk)
      yield chunk
    }

    // 缓存结果
    await this.cache.set(cacheKey, chunks)
  }
}

// 使用
const cachedLLM = new CachedLLM(
  new OpenAIProvider(apiKey),
  new MemoryCache()
)
```

**重试装饰器** (`/packages/ai/src/retry.ts`)
```typescript
class RetryLLM implements LLMProvider {
  constructor(
    private llm: LLMProvider,
    private maxRetries: number = 3
  ) { }

  async *chat(messages: Message[], options: Options): AsyncGenerator<Chunk> {
    let lastError: Error

    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        yield* this.llm.chat(messages, options)
        return
      } catch (error) {
        lastError = error
        await this.backoff(attempt)
      }
    }

    throw lastError
  }
}
```

---

### 7. 代理模式 (Proxy Pattern)

**意图**：控制对对象的访问

**使用场景**：

**权限检查代理** (`/packages/coding-agent/src/core/tools/proxy.ts`)
```typescript
class ToolProxy implements Tool {
  constructor(
    private tool: Tool,
    private permissions: PermissionChecker
  ) { }

  get name() { return this.tool.name }
  get description() { return this.tool.description }

  async execute(args: unknown): Promise<ToolResult> {
    // 执行前检查权限
    const hasPermission = await this.permissions.check(
      this.tool.name,
      args
    )

    if (!hasPermission) {
      throw new Error(`Permission denied for tool: ${this.tool.name}`)
    }

    // 调用原始工具
    return this.tool.execute(args)
  }
}
```

**会话加载代理** (`/packages/coding-agent/src/core/session-proxy.ts`)
```typescript
class SessionProxy implements AgentSession {
  private loaded = false
  private session?: AgentSession

  constructor(private path: string) { }

  private async ensureLoaded() {
    if (!this.loaded) {
      this.session = await loadSession(this.path)
      this.loaded = true
    }
  }

  async getEntries(): Promise<Entry[]> {
    await this.ensureLoaded()
    return this.session!.getEntries()
  }

  // 延迟加载会话数据
}
```

---

### 8. 桥接模式 (Bridge Pattern)

**意图**：分离抽象和实现，独立变化

**使用场景**：

**主题渲染桥接** (`/packages/tui/src/theme/bridge.ts`)
```typescript
// 抽象：渲染接口
interface ThemeRenderer {
  renderText(text: string, style: TextStyle): string
  renderBox(lines: string[], style: BoxStyle): string
}

// 实现 1：真彩色渲染器
class TruecolorRenderer implements ThemeRenderer {
  renderText(text: string, style: TextStyle): string {
    const fg = this.toRGB(style.foreground)
    return `\x1b[38;2;${fg.r};${fg.g};${fg.b}m${text}\x1b[0m`
  }
}

// 实现 2：256 色渲染器
class Color256Renderer implements ThemeRenderer {
  renderText(text: string, style: TextStyle): string {
    const index = this.to256(style.foreground)
    return `\x1b[38;5;${index}m${text}\x1b[0m`
  }
}

// 桥接：主题使用渲染接口
class Theme {
  constructor(private renderer: ThemeRenderer) { }

  highlight(text: string, color: string): string {
    return this.renderer.renderText(text, { foreground: color })
  }
}

// 使用
const theme = new Theme(
  detectTruecolorSupport()
    ? new TruecolorRenderer()
    : new Color256Renderer()
)
```

---

### 9. 组合模式 (Composite Pattern)

**意图**：统一处理单个对象和组合对象

**使用场景**：

**UI 组件树** (`/packages/tui/src/components/composite.ts`)
```typescript
interface UIComponent {
  render(): string
  handleInput(key: Key): boolean
}

// 叶子组件：按钮
class Button implements UIComponent {
  constructor(
    private label: string,
    private onPress: () => void
  ) { }

  render(): string {
    return `[ ${this.label} ]`
  }

  handleInput(key: Key): boolean {
    if (key.key === "Enter") {
      this.onPress()
      return true
    }
    return false
  }
}

// 组合组件：容器
class Container implements UIComponent {
  private children: UIComponent[] = []

  add(component: UIComponent): void {
    this.children.push(component)
  }

  render(): string {
    return this.children.map(c => c.render()).join("\n")
  }

  handleInput(key: Key): boolean {
    for (const child of this.children) {
      if (child.handleInput(key)) {
        return true
      }
    }
    return false
  }
}
```

**会话树组合** (`/packages/coding-agent/src/core/session/tree.ts`)
```typescript
interface SessionNode {
  id: string
  parent?: SessionNode
  children: SessionNode[]
}

class Entry implements SessionNode {
  constructor(
    public id: string,
    public parent: SessionNode | undefined,
    public children: SessionNode[] = []
  ) { }

  // 统一处理单个条目和树
  traverse(visitor: (node: SessionNode) => void): void {
    visitor(this)
    for (const child of this.children) {
      child.traverse(visitor)
    }
  }
}
```

---

## 行为型模式 (Behavioral Patterns)

### 10. 策略模式 (Strategy Pattern)

**意图**：定义算法族，可互换使用

**使用场景**：

**压缩策略** (`/packages/coding-agent/src/core/compaction/strategies.ts`)
```typescript
interface CompactionStrategy {
  shouldCompact(session: AgentSession): boolean
  findCutPoint(entries: Entry[]): number
  generateSummary(entries: Entry[]): Promise<string>
}

// 策略 1：基于令牌数的压缩
class TokenBasedCompaction implements CompactionStrategy {
  constructor(private threshold: number) { }

  shouldCompact(session: AgentSession): boolean {
    return calculateTokens(session) > this.threshold
  }

  findCutPoint(entries: Entry[]): number {
    // 实现基于令牌的切点查找
  }

  generateSummary(entries: Entry[]): Promise<string> {
    // 实现摘要生成
  }
}

// 策略 2：基于消息数的压缩
class MessageBasedCompaction implements CompactionStrategy {
  constructor(private maxMessages: number) { }

  shouldCompact(session: AgentSession): boolean {
    return session.entries.length > this.maxMessages
  }

  findCutPoint(entries: Entry[]): number {
    return entries.length - this.maxMessages
  }

  generateSummary(entries: Entry[]): Promise<string> {
    return Promise.resolve(`Compacted ${entries.length} messages`)
  }
}

// 使用
const strategy = config.useTokenCount
  ? new TokenBasedCompaction(8000)
  : new MessageBasedCompaction(100)
```

---

### 11. 观察者模式 (Observer Pattern)

**意图**：一对多依赖，状态变化时通知所有观察者

**使用场景**：

**事件流** (`/packages/agent/src/events.ts`)
```typescript
class EventStream<T> {
  private listeners = new Set<(event: T) => void>()

  on(listener: (event: T) => void): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  emit(event: T): void {
    for (const listener of this.listeners) {
      listener(event)
    }
  }
}

// Agent 使用事件流
class Agent {
  readonly events = new EventStream<AgentEvent>()

  async chat(message: string): Promise<Response> {
    this.events.emit({ type: "start", timestamp: Date.now() })

    try {
      const response = await this.llm.chat([message])
      this.events.emit({ type: "complete", timestamp: Date.now() })
      return response
    } catch (error) {
      this.events.emit({ type: "error", error, timestamp: Date.now() })
      throw error
    }
  }
}

// 订阅事件
agent.events.on(event => {
  if (event.type === "error") {
    logger.error("Agent error:", event.error)
  }
})
```

**主题热重载** (`/packages/coding-agent/src/modes/interactive/theme/theme.ts`)
```typescript
class ThemeManager {
  private watchers = new Set<(theme: Theme) => void>()

  subscribe(callback: (theme: Theme) => void): () => void {
    this.watchers.add(callback)
    return () => this.watchers.delete(callback)
  }

  async loadTheme(path: string): Promise<Theme> {
    const theme = await loadThemeFile(path)
    this.notify(theme)
    return theme
  }

  private notify(theme: Theme): void {
    for (const callback of this.watchers) {
      callback(theme)
    }
  }
}
```

---

### 12. 命令模式 (Command Pattern)

**意图**：将请求封装为对象

**使用场景**：

**撤销/重做** (`/packages/coding-agent/src/core/commands.ts`)
```typescript
interface Command {
  execute(): Promise<void>
  undo?(): Promise<void>
}

class EditFileCommand implements Command {
  constructor(
    private path: string,
    private edits: Edit[],
    private backup?: string
  ) { }

  async execute(): Promise<void> {
    // 创建备份
    this.backup = await fs.readFile(this.path, "utf-8")

    // 应用编辑
    await applyEdits(this.path, this.edits)
  }

  async undo(): Promise<void> {
    if (this.backup) {
      await fs.writeFile(this.path, this.backup)
    }
  }
}

// 命令历史
class CommandHistory {
  private history: Command[] = []
  private current = -1

  async execute(command: Command): Promise<void> {
    // 移除重做历史
    this.history = this.history.slice(0, this.current + 1)

    await command.execute()
    this.history.push(command)
    this.current++
  }

  async undo(): Promise<void> {
    if (this.current >= 0) {
      const command = this.history[this.current]
      await command.undo?.()
      this.current--
    }
  }

  async redo(): Promise<void> {
    if (this.current < this.history.length - 1) {
      this.current++
      const command = this.history[this.current]
      await command.execute()
    }
  }
}
```

---

### 13. 状态模式 (State Pattern)

**意图**：对象行为随内部状态变化

**使用场景**：

**Agent 状态** (`/packages/agent/src/agent-states.ts`)
```typescript
interface AgentState {
  enter(agent: Agent): void
  exit(agent: Agent): void
  handleInput(agent: Agent, input: string): Promise<void>
}

class IdleState implements AgentState {
  enter(agent: Agent): void {
    agent.showPrompt()
  }

  exit(agent: Agent): void {
    agent.hidePrompt()
  }

  async handleInput(agent: Agent, input: string): Promise<void> {
    agent.transition(new ThinkingState())
    await agent.process(input)
  }
}

class ThinkingState implements AgentState {
  enter(agent: Agent): void {
    agent.showSpinner()
  }

  exit(agent: Agent): void {
    agent.hideSpinner()
  }

  async handleInput(agent: Agent, input: string): Promise<void> {
    // 忽略输入，正在思考
  }
}

// Agent 状态机
class Agent {
  private state: AgentState = new IdleState()

  transition(state: AgentState): void {
    this.state.exit(this)
    this.state = state
    this.state.enter(this)
  }

  async handleInput(input: string): Promise<void> {
    await this.state.handleInput(this, input)
  }
}
```

---

### 14. 迭代器模式 (Iterator Pattern)

**意图**：遍历集合对象

**使用场景**：

**会话条目迭代** (`/packages/coding-agent/src/core/session/iterator.ts`)
```typescript
interface SessionIterator<T> {
  next(): IteratorResult<T>
  [Symbol.iterator](): SessionIterator<T>
}

class EntryIterator implements SessionIterator<Entry> {
  private index = 0

  constructor(
    private entries: Entry[],
    private filter?: (entry: Entry) => boolean
  ) { }

  next(): IteratorResult<Entry> {
    while (this.index < this.entries.length) {
      const entry = this.entries[this.index++]

      if (!this.filter || this.filter(entry)) {
        return { done: false, value: entry }
      }
    }

    return { done: true, value: undefined }
  }

  [Symbol.iterator]() {
    return this
  }
}

// 使用
for (const entry of new EntryIterator(session.entries, e => e.type === "message")) {
  console.log(entry.message.content)
}
```

---

### 15. 模板方法模式 (Template Method Pattern)

**意图**：定义算法骨架，子类实现细节

**使用场景**：

**Provider 基类** (`/packages/ai/src/providers/base.ts`)
```typescript
abstract class BaseProvider implements LLMProvider {
  abstract formatMessages(messages: Message[]): unknown
  abstract parseResponse(response: unknown): Message

  async *chat(messages: Message[], options: Options): AsyncGenerator<Chunk> {
    // 模板方法：定义流程
    const formatted = this.formatMessages(messages)
    const rawResponse = await this.doRequest(formatted, options)

    for await (const chunk of this.streamResponse(rawResponse)) {
      yield this.parseChunk(chunk)
    }
  }

  protected abstract doRequest(formatted: unknown, options: Options): Promise<unknown>
  protected abstract *streamResponse(response: unknown): Generator<unknown>
  protected abstract parseChunk(chunk: unknown): Chunk
}

// 子类实现细节
class OpenAIProvider extends BaseProvider {
  formatMessages(messages: Message[]): OpenAIMessage[] {
    return messages.map(toOpenAIFormat)
  }

  protected async doRequest(formatted: OpenAIMessage[]): Promise<OpenAIResponse> {
    return this.client.chat.completions.create({ messages: formatted })
  }

  // ...
}
```

---

## 异步模式 (Async Patterns)

### 16. 异步生成器模式 (Async Generator Pattern)

**意图**：流式处理异步数据

**使用场景**：

**流式聊天** (`/packages/ai/src/stream.ts`)
```typescript
async function* streamChat(
  llm: LLMProvider,
  messages: Message[]
): AsyncGenerator<MessageChunk> {
  const response = await llm.chat(messages, { stream: true })

  for await (const chunk of response) {
    if (chunk.choices?.[0]?.delta?.content) {
      yield {
        content: chunk.choices[0].delta.content,
        done: false
      }
    }
  }

  yield { content: "", done: true }
}

// 使用
for await (const chunk of streamChat(llm, messages)) {
  if (!chunk.done) {
    process.stdout.write(chunk.content)
  }
}
```

---

### 17. Promise 链模式 (Promise Chain Pattern)

**意图**：顺序执行异步操作

**使用场景**：

**扩展加载链** (`/packages/coding-agent/src/core/extensions/loader.ts`)
```typescript
async function loadExtension(ext: Extension): Promise<void> {
  // 顺序执行加载步骤
  await validateExtension(ext)
  await resolveDependencies(ext)
  await registerTools(ext.tools ?? [])
  await registerSkills(ext.skills ?? [])
  await ext.hooks?.onLoad?.()
}
```

---

## 响应式模式 (Reactive Patterns)

### 18. Reactor 模式 (Reactor Pattern)

**意图**：事件驱动处理

**使用场景**：

**TUI 事件循环** (`/packages/tui/src/tui.ts`)
```typescript
class TUI {
  private handlers = new Map<Key, Handler>()

  onKey(key: Key, handler: Handler): void {
    this.handlers.set(key, handler)
  }

  async run(): Promise<void> {
    // 事件循环
    for await (const event of this.stdin) {
      const key = parseKey(event)

      const handler = this.handlers.get(key)
      if (handler) {
        await handler(key)
      }

      this.render()
    }
  }
}
```

---

## 总结

| 模式 | 类别 | 主要使用场景 | 源文件位置 |
|------|------|--------------|-----------|
| 工厂模式 | 创建型 | Provider/Tool 创建 | `ai/src/api-registry.ts` |
| 建造者模式 | 创建型 | Agent 配置构建 | `agent/src/agent.ts` |
| 单例模式 | 创建型 | 全局配置/Theme | `coding-agent/src/config/` |
| 依赖注入 | 创建型 | Agent 依赖管理 | `agent/src/agent.ts` |
| 适配器模式 | 结构型 | Provider 适配 | `ai/src/providers/` |
| 装饰器模式 | 结构型 | 缓存/重试 | `ai/src/cache.ts` |
| 代理模式 | 结构型 | 权限检查/延迟加载 | `coding-agent/src/core/tools/` |
| 桥接模式 | 结构型 | 主题渲染 | `tui/src/theme/` |
| 组合模式 | 结构型 | UI 组件树/会话树 | `tui/src/components/` |
| 策略模式 | 行为型 | 压缩策略 | `coding-agent/src/core/compaction/` |
| 观察者模式 | 行为型 | 事件系统 | `agent/src/events.ts` |
| 命令模式 | 行为型 | 撤销/重做 | `coding-agent/src/core/commands.ts` |
| 状态模式 | 行为型 | Agent 状态 | `agent/src/agent-states.ts` |
| 异步生成器 | 异步模式 | 流式聊天 | `ai/src/stream.ts` |
| Reactor 模式 | 响应式 | TUI 事件循环 | `tui/src/tui.ts` |

---

## 相关链接

- **设计哲学**：`/LEARN/05-patterns/01-design-philosophy.md`
- **类型系统设计**：`/LEARN/05-patterns/03-type-system.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
