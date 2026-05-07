# 设计哲学 (Design Philosophy)

## 概述

pi-mono 项目的设计哲学深受其作者 Mario Zechner (@badlogic) 的背景影响，融合了游戏开发（LibGDX 框架创始人）、AI 系统架构和 CLI 工具设计的最佳实践。项目体现了以下核心价值观：

- **简洁性优于复杂性** - API 设计遵循最小惊讶原则
- **可扩展性优先** - 插件化架构支持无限扩展
- **类型安全** - 充分利用 TypeScript 类型系统
- **渐进式增强** - 从简单开始，按需增长
- **跨平台一致性** - 统一 API 抽遮平台差异

---

## 核心设计原则

### 1. 单一职责原则 (Single Responsibility)

每个包、类、函数都有明确的单一职责：

**包层级**：
```
pi-ai           → LLM API 抽象层
pi-agent-core   → Agent 运行时（无 UI 逻辑）
pi-coding-agent → 编程 Agent 应用
pi-tui          → 终端 UI 组件库
```

**类层级**：
```typescript
// /packages/ai/src/api-registry.ts
class APIRegistry {
  // 唯一职责：管理 Provider 注册与查找
  register(key: string, provider: Provider): void { }
  get(key: string): Provider | undefined { }
  list(): Provider[] { }
}

// /packages/tui/src/tui.ts
class TUI {
  // 唯一职责：终端 UI 渲染与事件处理
  render(): void { }
  handleInput(key: Key): void { }
}
```

### 2. 开放封闭原则 (Open/Closed)

系统对扩展开放，对修改封闭：

**Provider 系统**：
```typescript
// 用户可以添加新 Provider 而不修改核心代码
registerProvider({
  key: "my-custom-llm",
  create: (config) => new MyCustomLLM(config)
})
```

**工具系统**：
```typescript
// 用户可以定义新工具
registerTool({
  name: "my-tool",
  description: "My custom tool",
  parameters: { /* schema */ },
  execute: async (args) => { /* implementation */ }
})
```

**扩展系统**：
```typescript
// 完整的扩展可以添加工具、技能、UI 组件
export default defineExtension({
  id: "my-extension",
  tools: [/* tools */],
  skills: [/* skills */],
  hooks: { /* lifecycle hooks */ }
})
```

### 3. 依赖倒置原则 (Dependency Inversion)

高层模块不依赖低层模块，两者都依赖抽象：

**事件抽象**：
```typescript
// /packages/agent/src/types.ts
// Agent 依赖于抽象的 EventStream，而非具体实现
interface AgentEvent {
  type: string
  timestamp: number
  data?: unknown
}

class Agent {
  constructor(
    private events: EventStream<AgentEvent>  // 抽象依赖
  ) { }
}
```

**Provider 抽象**：
```typescript
// /packages/ai/src/types.ts
// Agent Core 不依赖具体 LLM 实现
interface LLMProvider {
  chat(messages: Message[], options: Options): AsyncGenerator<Chunk>
  embed(texts: string[]): Promise<number[][]>
}
```

### 4. 接口隔离原则 (Interface Segregation)

客户端不应依赖它不使用的接口：

**细粒度接口**：
```typescript
// 工具接口只定义必要方法
interface Tool {
  name: string
  description: string
  parameters?: ToolParameterSchema
  execute?(args: unknown): Promise<ToolResult>
  // 不强制实现可选功能
}

// 生命周期接口独立
interface LifecycleHooks {
  onLoad?(): Promise<void>
  onUnload?(): Promise<void>
  onMessage?(message: Message): Promise<void>
}
```

---

## 架构哲学

### 分层架构 (Layered Architecture)

项目采用清晰的四层架构：

```
┌─────────────────────────────────────────────┐
│  L4: 应用层 (Applications)                  │
│  - pi-coding-agent                          │
│  - pi-mom (Slack bot)                       │
│  - pi-web-ui                                │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  L3: 领域层 (Domain)                        │
│  - Extension System                         │
│  - Tool System                              │
│  - Session Management                       │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  L2: Agent 运行时 (Agent Runtime)           │
│  - pi-agent-core                            │
│  - Agent Loop                               │
│  - Event System                             │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  L1: 基础设施层 (Infrastructure)            │
│  - pi-ai (LLM 抽象)                         │
│  - pi-tui (UI 库)                           │
│  - pi-db (存储抽象)                         │
└─────────────────────────────────────────────┘
```

**关键约束**：
- L1 不依赖任何其他层级
- L2 只依赖 L1
- L3 只依赖 L2 和 L1
- L4 可以依赖任何层级

### 插件化架构 (Plugin Architecture)

系统的核心是插件化设计：

**扩展作为一等公民**：
```typescript
// /packages/coding-agent/src/core/extensions/types.ts
interface Extension {
  // 身份
  id: string
  name: string
  version: string

  // 能力
  tools?: Tool[]
  skills?: Skill[]
  templates?: Template[]

  // 生命周期
  hooks?: ExtensionHooks

  // UI 集成
  ui?: {
    components?: UIComponent[]
    routes?: Route[]
  }
}
```

**热加载支持**：
```typescript
// 运行时加载/卸载扩展
await extensions.load("/path/to/extension")
await extensions.unload("extension-id")
await extensions.reload("extension-id")
```

**依赖管理**：
```typescript
// 扩展可以声明依赖
{
  dependencies: {
    extensions: ["other-extension"],
    packages: {
      "some-package": "^1.0.0"
    }
  }
}
```

---

## API 设计哲学

### 流式优先 (Streaming First)

LLM 交互以流式为默认方式：

```typescript
// /packages/ai/src/stream.ts
// 所有响应都是 AsyncGenerator
async function* streamChat(
  messages: Message[]
): AsyncGenerator<Chunk> {
  for await (const chunk of llm.chat(messages)) {
    yield chunk
    // 每个块立即传递给调用者
  }
}

// 使用方
for await (const chunk of streamChat(messages)) {
  console.log(chunk.content)  // 实时输出
}
```

**好处**：
- 降低延迟感知
- 支持取消操作
- 更好的内存效率

### 错误处理哲学

**显式优于隐式**：
```typescript
// 错误不静默失败
async function executeTool(tool: Tool, args: unknown) {
  try {
    return await tool.execute(args)
  } catch (error) {
    // 明确报告错误，不返回 undefined
    return {
      success: false,
      error: error.message,
      content: null
    }
  }
}
```

**结构化错误**：
```typescript
// /packages/ai/src/errors.ts
class APIError extends Error {
  constructor(
    public code: string,
    public provider: string,
    message: string,
    public details?: unknown
  ) {
    super(message)
  }
}
```

### 配置即代码

配置使用 TypeScript/JSON，而非专有格式：

```typescript
// /packages/coding-agent/src/config/config.ts
interface Config {
  // 类型安全的配置
  provider: {
    key: string
    model: string
  }
  theme: string
  keybindings: KeybindingConfig
  extensions: ExtensionConfig[]
}

// 支持热重载
config.on("change", (newConfig) => {
  applyConfig(newConfig)
})
```

---

## 类型系统哲学

### 渐进式类型安全

从宽松到严格的渐进路径：

```typescript
// 1. 基础类型 - 最小约束
interface Message {
  role: string
  content: string
}

// 2. 字面量类型 - 更精确
interface Message {
  role: "user" | "assistant" | "system"
  content: string
}

// 3. 泛型 - 最大灵活性
interface Message<T = string> {
  role: "user" | "assistant" | "system"
  content: T
}

// 4. 条件类型 - 编译时验证
type ToolResult<T extends Tool> =
  T extends { execute: infer E }
    ? Awaited<ReturnType<E>>
    : unknown
```

### 类型推导

充分利用 TypeScript 类型推导：

```typescript
// /packages/tui/src/keybindings.ts
// 递归类型推导所有可能的组合
type KeyId = {
  [M in Modifier]: {
    [K in BaseKey]: `${M}-${K}`
  }
}[Modifier][BaseKey]

// 编译时验证
function bind<T extends KeyId>(
  key: T,
  action: Action
): Binding<T> {
  return { key, action }
}

bind("ctrl-a", action)  // ✓ 正确
bind("ctrl-invalid", action)  // ✗ 编译错误
```

### Branded Types

使用 branded types 防止类型混淆：

```typescript
// /packages/agent/src/types.ts
type SessionId = string & { readonly __sessionId: unique symbol }
type ExtensionId = string & { readonly __extensionId: unique symbol }

function createSession(id: SessionId) { }
function loadExtension(id: ExtensionId) { }

// 编译时捕获错误
const sessionId: SessionId = "abc" as SessionId
const extId: ExtensionId = "xyz" as ExtensionId

createSession(sessionId)  // ✓
createSession(extId)      // ✗ 类型错误
```

---

## 性能哲学

### 懒加载 (Lazy Loading)

非关键路径延迟加载：

```typescript
// /packages/ai/src/providers/register-builtins.ts
// Provider 按需加载
const PROVIDER_REGISTRY = {
  openai: () => import("./providers/openai"),
  anthropic: () => import("./providers/anthropic"),
  ollama: () => import("./providers/ollama"),
  // ...
}

async function loadProvider(key: string) {
  const loader = PROVIDER_REGISTRY[key]
  if (!loader) throw new Error(`Unknown provider: ${key}`)

  const module = await loader()  // 只在需要时加载
  return module.default
}
```

### 缓存策略

**多级缓存**：
```typescript
// 1. 内存缓存
class LLMCache {
  private cache = new Map<string, Response>()

  async get(key: string): Promise<Response | undefined> {
    return this.cache.get(key)
  }

  async set(key: string, value: Response): Promise<void> {
    this.cache.set(key, value)
    // LRU 淘汰策略
    if (this.cache.size > MAX_CACHE_SIZE) {
      const first = this.cache.keys().next().value
      this.cache.delete(first)
    }
  }
}

// 2. 磁盘缓存（用于嵌入向量）
class EmbeddingCache {
  async get(texts: string[]): Promise<number[][] | undefined> {
    const hash = hashTexts(texts)
    const cached = await diskCache.get(hash)
    return cached ? JSON.parse(cached) : undefined
  }
}
```

### 增量操作

最小化重新计算：

```typescript
// /packages/tui/src/tui.ts
// 差分渲染，只更新变化的部分
class TUI {
  private prevLines: string[] = []

  render(lines: string[]) {
    const diff = computeDiff(this.prevLines, lines)

    for (const change of diff) {
      if (change.type === "insert") {
        this.insertLine(change.index, change.line)
      } else if (change.type === "delete") {
        this.deleteLine(change.index)
      } else if (change.type === "update") {
        this.updateLine(change.index, change.line)
      }
    }

    this.prevLines = lines
  }
}
```

---

## 错误恢复哲学

### 优雅降级

功能失败不影响核心：

```typescript
// /packages/coding-agent/src/modes/interactive/theme/theme.ts
function detectColorMode(): ColorMode {
  // 检测失败时回退到安全模式
  try {
    if (process.env.COLORTERM?.includes("truecolor")) {
      return "truecolor"
    }
  } catch {
    // 忽略检测错误
  }

  // 默认回退
  return "256color"
}
```

### 幂等操作

操作可以安全重试：

```typescript
// /packages/coding-agent/src/core/extensions/loader.ts
async function loadExtension(
  extension: Extension
): Promise<void> {
  // 已加载则直接返回
  if (this.loaded.has(extension.id)) {
    return
  }

  // 加载失败可重试
  await extension.hooks?.onLoad?.()
  this.loaded.add(extension.id)
}
```

---

## 可测试性哲学

### 依赖注入

所有外部依赖可注入：

```typescript
class Agent {
  constructor(
    private llm: LLMProvider,      // 可注入
    private tools: ToolRegistry,   // 可注入
    private events: EventStream    // 可注入
  ) { }
}

// 测试时注入 mock
const agent = new Agent(
  new MockLLM(),
  new MockToolRegistry(),
  new MockEventStream()
)
```

### 纯函数优先

逻辑与副作用分离：

```typescript
// 纯函数 - 易于测试
function calculateTokens(text: string): number {
  return Math.ceil(text.length / 4)
}

// 副作用分离
async function sendMessage(text: string) {
  const tokens = calculateTokens(text)  // 纯计算
  await api.chat({ text, tokens })      // 副作用
}
```

---

## 可观测性哲学

### 事件驱动日志

所有关键操作发出事件：

```typescript
// /packages/agent/src/agent-loop.ts
class AgentLoop {
  async run() {
    this.emit("start", { timestamp: Date.now() })

    try {
      const response = await this.llm.chat(messages)
      this.emit("llm-response", {
        tokens: response.tokens,
        latency: response.latency
      })
    } catch (error) {
      this.emit("error", {
        error: error.message,
        stack: error.stack
      })
    }

    this.emit("complete", { timestamp: Date.now() })
  }
}
```

### 结构化指标

```typescript
// /packages/ai/src/metrics.ts
class Metrics {
  histogram(
    name: string,
    value: number,
    tags: Record<string, string>
  ) {
    // 发送到指标系统
    metricsClient.record(name, value, tags)
  }
}

// 使用
metrics.histogram("llm.latency", latency, {
  provider: "openai",
  model: "gpt-4"
})
```

---

## 文档哲学

### 代码即文档

自描述的 API 设计：

```typescript
// 好的 API 设计不需要大量注释
interface Tool {
  name: string                    // 自解释
  description: string             // 自解释
  parameters?: ToolParameterSchema  // 类型即文档

  execute?(args: unknown): Promise<ToolResult>
}
```

### 类型级文档

```typescript
/**
 * 工具执行的参数
 * @template T - 工具类型
 */
type ToolParameters<T extends Tool> =
  T extends { parameters: infer P }
    ? P extends ToolParameterSchema
      ? FromSchema<P>  // 从 Schema 推导类型
      : unknown
    : unknown
```

---

## 总结

pi-mono 项目的设计哲学体现了：

1. **实用主义** - 解决实际问题，不过度设计
2. **渐进式复杂度** - 从简单开始，按需增长
3. **类型安全** - 充分利用 TypeScript，但不被其束缚
4. **可扩展性** - 插件化架构支持无限扩展
5. **可维护性** - 清晰的分层、单一职责、开放封闭原则

这些哲学指导下的代码库既强大又灵活，易于理解和扩展。

---

## 相关链接

- **设计模式目录**：`/LEARN/05-patterns/02-patterns-catalog.md`
- **类型系统设计**：`/LEARN/05-patterns/03-type-system.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
