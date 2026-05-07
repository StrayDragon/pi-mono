# 类型系统设计 (Type System Design)

## 概述

pi-mono 项目充分利用 TypeScript 的类型系统，实现了编译时类型安全、优秀的开发者体验和可维护的代码库。本文档深入分析项目的类型系统设计，包括高级类型技巧、模式和实践。

**核心理念**：
- **类型安全第一** - 利用类型系统防止运行时错误
- **渐进式类型** - 从简单开始，逐步增强类型约束
- **类型即文档** - 自描述的类型定义减少文档需求
- **推导优于显式** - 让 TypeScript 推导类型，减少重复

---

## 类型系统架构

### 分层类型设计

```
┌─────────────────────────────────────────────┐
│  应用层类型 (Application Types)              │
│  - AgentSession                              │
│  - Extension                                 │
│  - UI Components                             │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  领域层类型 (Domain Types)                   │
│  - Tool, Skill, Template                     │
│  - Message, Entry                            │
│  - AgentEvent                                │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  核心类型 (Core Types)                       │
│  - LLMProvider, Message                      │
│  - EventStream                               │
│  - Key, Color                                │
└─────────────────────────────────────────────┘
              ↓ 依赖
┌─────────────────────────────────────────────┐
│  基础类型 (Base Types)                       │
│  - primitives, branded types                 │
│  - utility types                             │
└─────────────────────────────────────────────┘
```

---

## 基础类型技巧

### 1. Branded Types (品牌类型)

用于区分语义上不同但底层类型相同的值：

```typescript
// /packages/agent/src/types.ts
type SessionId = string & { readonly __sessionId: unique symbol }
type ExtensionId = string & { readonly __extensionId: unique symbol }
type ToolName = string & { readonly __toolName: unique symbol }

// 创建辅助函数
function createSessionId(id: string): SessionId {
  return id as SessionId
}

function createExtensionId(id: string): ExtensionId {
  return id as ExtensionId
}

// 使用
function loadSession(id: SessionId): Promise<AgentSession> { }
function loadExtension(id: ExtensionId): Promise<Extension> { }

const sessionId = createSessionId("abc123")
const extId = createExtensionId("xyz789")

loadSession(sessionId)  // ✓
loadSession(extId)      // ✗ 编译错误：ExtensionId 不能赋值给 SessionId
```

**好处**：
- 编译时防止类型混淆
- 不影响运行时性能
- 保持 IDE 自动完成和类型检查

---

### 2. 严格字面量类型

使用字面量类型替代枚举：

```typescript
// /packages/ai/src/types.ts
// 不使用枚举
// enum ProviderType { OpenAI = "openai", Anthropic = "anthropic" }

// 使用字面量联合类型
type ProviderType = "openai" | "anthropic" | "ollama" | "google"

type MessageRole = "user" | "assistant" | "system" | "tool"

type ChunkType =
  | { type: "content"; content: string }
  | { type: "tool_call"; toolCall: ToolCall }
  | { type: "done" }

// 函数签名
async function chat(
  provider: ProviderType,
  messages: Array<{ role: MessageRole; content: string }>
): Promise<ChunkType>

// IDE 自动完成提供所有选项
chat("openai", [{ role: "user", content: "..." }])  // ✓
chat("invalid", [])  // ✗ 编译错误
```

---

### 3. 可辨识联合 (Discriminated Unions)

使用公共字段区分类型：

```typescript
// /packages/coding-agent/src/core/session/types.ts
type Entry =
  | { type: "message"; message: Message; timestamp: number }
  | { type: "compaction"; compacted: CompactedData; summary: string; timestamp: number }
  | { type: "branch"; branchId: string; parentEntryId: string; timestamp: number }
  | { type: "tool_execution"; tool: string; input: unknown; output: unknown; timestamp: number }

// 类型守卫
function isMessageEntry(entry: Entry): entry is Entry & { type: "message" } {
  return entry.type === "message"
}

// 使用
function processEntry(entry: Entry) {
  switch (entry.type) {
    case "message":
      // TypeScript 知道 entry.message 存在
      console.log(entry.message.content)
      break
    case "compaction":
      console.log(entry.summary)
      break
    // ...
  }
}
```

---

## 高级类型模式

### 4. 条件类型 (Conditional Types)

根据类型条件选择类型：

```typescript
// /packages/coding-agent/src/core/tools/types.ts
// 提取工具执行返回类型
type ToolResult<T extends Tool> =
  T extends { execute: infer E }
    ? E extends (...args: any[]) => infer R
      ? R extends Promise<infer U>
        ? U
        : R
      : never
    : never

// 使用
interface ReadTool extends Tool {
  execute(path: string): Promise<string>
}

type ReadResult = ToolResult<ReadTool>  // string
```

**条件类型用于配置**：
```typescript
// 根据配置类型决定返回值类型
type ConfigValue<T> =
  T extends { default: infer D }
    ? D
    : T extends { type: "string" }
      ? string
      : T extends { type: "number" }
        ? number
        : unknown

interface StringConfig {
  type: "string"
  default: string
}

interface NumberConfig {
  type: "number"
  default: number
}

type GetString = ConfigValue<StringConfig>  // string
type GetNumber = ConfigValue<NumberConfig>  // number
```

---

### 5. 映射类型 (Mapped Types)

转换对象类型的每个属性：

```typescript
// /packages/ai/src/types.ts
// 将所有属性变为可选
type PartialOptions<T> = {
  [K in keyof T]?: T[K]
}

// 将所有属性变为只读
type ReadonlyConfig<T> = {
  readonly [K in keyof T]: T[K]
}

// 提取 getter 返回类型
type Getters<T> = {
  [K in keyof T as T[K] extends () => any ? K : never]: T[K]
}

// 使用
interface Config {
  apiKey: string
  endpoint: string
  timeout: number
}

type PartialConfig = PartialOptions<Config>
// { apiKey?: string; endpoint?: string; timeout?: number }
```

**键名重映射**：
```typescript
// /packages/tui/src/keybindings.ts
// 创建按键 ID 类型
type KeyId<T extends string> = `key-${T}`

type Keys = KeyId<"enter" | "escape" | "space">
// "key-enter" | "key-escape" | "key-space"
```

---

### 6. 模板字面量类型 (Template Literal Types)

用于字符串操作的类型：

```typescript
// /packages/tui/src/keybindings.ts
// 按键组合类型
type Modifier = "ctrl" | "shift" | "alt" | "super"
type BaseKey = "a" | "b" | "c" | "enter" | "escape" | "space"

// 生成所有可能的组合
type KeyCombination = `${Modifier}-${BaseKey}`

// 结果："ctrl-a" | "ctrl-b" | ... | "super-space"

// 更复杂的：多重修饰键
type KeyCombo<M extends Modifier[], K extends BaseKey> =
  M extends [infer First extends Modifier, ...infer Rest extends Modifier[]]
    ? Rest extends []
      ? `${First}-${K}`
      : `${First}-${KeyCombo<Rest, K>}`
    : K

type CtrlShiftA = KeyCombo<["ctrl", "shift"], "a">  // "ctrl-shift-a"
```

---

### 7. 递归类型 (Recursive Types)

定义无限嵌套结构：

```typescript
// /packages/coding-agent/src/core/session/tree.ts
// 会话树结构
interface SessionNode {
  id: string
  entry: Entry
  children: SessionNode[]
  parent?: SessionNode
}

// JSON 类型
type JSON =
  | string
  | number
  | boolean
  | null
  | JSON[]
  | { [key: string]: JSON }

// 工具参数 Schema
type ToolParameterSchema =
  | { type: "string"; description?: string; enum?: string[] }
  | { type: "number"; description?: string; min?: number; max?: number }
  | { type: "boolean"; description?: string }
  | { type: "object"; properties: Record<string, ToolParameterSchema> }
  | { type: "array"; items: ToolParameterSchema }
```

---

### 8. 类型推断 (Type Inference)

让 TypeScript 推导复杂类型：

```typescript
// /packages/ai/src/stream.ts
// 推导 Promise 返回类型
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T

// 推导函数返回类型
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any

// 推解 AsyncGenerator 类型
type AsyncGeneratorType<T> =
  T extends AsyncGenerator<infer Y, any, any>
    ? Y
    : never

// 使用
async function* stream(): AsyncGenerator<string, void, unknown> {
  yield "hello"
}

type Chunk = AsyncGeneratorType<typeof stream>  // string
```

**推导工具调用结果**：
```typescript
// /packages/coding-agent/src/core/tools/types.ts
type ToolCallResult<T extends Tool> =
  T extends { execute: infer E }
    ? E extends (...args: any[]) => infer R
      ? R extends Promise<infer U>
        ? U
        : R
      : never
    : never
```

---

## 类型安全模式

### 9. 类型守卫 (Type Guards)

运行时类型检查：

```typescript
// /packages/coding-agent/src/core/session/guards.ts
// 基本类型守卫
function isString(value: unknown): value is string {
  return typeof value === "string"
}

function isArray(value: unknown): value is unknown[] {
  return Array.isArray(value)
}

// 可辨识联合守卫
function isMessage(entry: Entry): entry is MessageEntry {
  return entry.type === "message"
}

// 属性检查守卫
function hasToolCalls(message: Message): message is Message & { toolCalls: ToolCall[] } {
  return "toolCalls" in message && Array.isArray(message.toolCalls)
}

// 使用
function processMessage(message: Message) {
  if (hasToolCalls(message)) {
    // TypeScript 知道 message.toolCalls 存在
    for (const tc of message.toolCalls) {
      executeToolCall(tc)
    }
  }
}
```

---

### 10. 断言函数 (Assertion Functions)

抛出错误的类型守卫：

```typescript
// /packages/ai/src/assert.ts
function assert(value: unknown, message: string): asserts value {
  if (!value) {
    throw new Error(message)
  }
}

function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`)
  }
}

// 使用
function process(value: unknown) {
  assertIsString(value)
  // 此后 value 被推断为 string
  console.log(value.toUpperCase())
}
```

---

### 11. 模板函数类型

确保模板字符串和参数匹配：

```typescript
// /packages/tui/src/template.ts
type TemplateStringsArray = TemplateStringsArray & { readonly length: number }

function template(strings: TemplateStringsArray, ...values: unknown[]): string {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] ?? "")
  }, "")
}

// 强类型模板
function highlight(text: string, color: Color): string {
  return template`\x1b[${color.code}m${text}\x1b[0m`
}
```

---

## 实用工具类型

### 12. 内置工具类型组合

```typescript
// /packages/coding-agent/src/core/types.ts
// 必需属性
type RequiredConfig = Required<PartialConfig>

// 只读属性
type ReadonlySession = Readonly<AgentSession>

// 提取属性
type MessageContent = Pick<Message, "content" | "role">

// 排除属性
type EntryWithoutTimestamp = Omit<Entry, "timestamp">

// 提取函数属性
type SessionMethods = Pick<AgentSession, {
  [K in keyof AgentSession]: AgentSession[K] extends Function ? K : never
}[keyof AgentSession]>
```

---

### 13. 自定义工具类型

```typescript
// /packages/coding-agent/src/core/utils/types.ts
// 深度只读
type DeepReadonly<T> =
  T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T

// 深度可选
type DeepPartial<T> =
  T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T

// 提取联合类型的所有可能值
type UnionValues<T> = T extends any ? T[] : never

type Keys = UnionValues<"a" | "b" | "c">  // "a" | "b" | "c"
```

---

## 类型级编程

### 14. 类型级算术

```typescript
// /packages/tui/src/types.ts
// 元组长度
type Length<T extends readonly unknown[]> = T["length"]

type ArrLength = Length<[1, 2, 3]>  // 3

// 数组访问
type ElementType<T extends unknown[]> = T[number]

type Elements = ElementType<[string, number, boolean]>  // string | number | boolean

// 字符串操作
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}`
    ? [Head, ...Split<Tail, D>]
    : [S]

type Parts = Split<"a-b-c", "-">  // ["a", "b", "c"]
```

---

### 15. Schema 推导

从 JSON Schema 推导类型：

```typescript
// /packages/coding-agent/src/core/tools/schema.ts
// 简化版
type FromSchema<T> =
  T extends { type: "string" }
    ? string
    : T extends { type: "number" }
      ? number
      : T extends { type: "boolean" }
        ? boolean
        : T extends { type: "array"; items: infer I }
          ? FromSchema<I>[]
          : T extends { type: "object"; properties: infer P }
            ? { [K in keyof P]: FromSchema<P[K]> }
            : unknown

// 使用
interface UserSchema {
  type: "object"
  properties: {
    name: { type: "string" }
    age: { type: "number" }
    active: { type: "boolean" }
  }
}

type User = FromSchema<UserSchema>
// { name: string; age: number; active: boolean }
```

---

## 泛型约束与默认值

### 16. 泛型约束

```typescript
// /packages/ai/src/types.ts
// 约束为函数
type FunctionType<T extends (...args: any[]) => any> = T

// 约束为类
type ClassType<T extends new (...args: any[]) => any> = T

// 约束为对象
type ObjectType<T extends Record<string, any>> = T

// 使用
function createInstance<T extends new (...args: any[]) => any>(
  Class: T,
  ...args: ConstructorParameters<T>
): InstanceType<T> {
  return new Class(...args)
}

const instance = createInstance(SomeClass, "arg1", "arg2")
```

---

### 17. 条件默认值

```typescript
// /packages/agent/src/types.ts
// 有默认值时优先使用默认值类型
type WithDefault<T, D> = T extends undefined ? D : T

// 配置类型
interface Config<T = string> {
  value: WithDefault<T, string>
}

const config1: Config = { value: "default" }  // value: string
const config2: Config<number> = { value: 123 }  // value: number
```

---

## 类型体操实战

### 18. 深度提取

```typescript
// /packages/coding-agent/src/core/utils/types.ts
// 提取数组元素类型
type ArrayElement<T> = T extends (infer U)[] ? U : never

// 提取 Promise 返回类型
type PromiseValue<T> = T extends Promise<infer U> ? U : never

// 提取函数参数类型
type FunctionArgs<T> = T extends (...args: infer A) => any ? A : never

// 提取函数返回类型
type FunctionReturn<T> = T extends (...args: any[]) => infer R ? R : never

// 组合使用
type AsyncReturnType<T extends (...args: any[]) => Promise<any>> =
  T extends (...args: any[]) => Promise<infer R> ? R : any
```

---

### 19. 不可变性类型

```typescript
// /packages/agent/src/types.ts
// 所有属性变为只读
type Immutable<T> =
  T extends (infer U)[]
    ? ReadonlyArray<U>
    : T extends object
      ? { readonly [K in keyof T]: Immutable<T[K]> }
      : T

// 可变引用类型
type Mutable<T> =
  T extends ReadonlyArray<infer U>
    ? U[]
    : T extends object
      ? { -readonly [K in keyof T]: Mutable<T[K]> }
      : T

// 使用
const immutableSession: Immutable<AgentSession> = {
  ...session
}

// immutableSession.entries = []  // ✗ 错误：只读
```

---

## 类型测试

### 20. 类型级测试

使用 `Expect` 类型进行编译时测试：

```typescript
// /packages/test/types.ts
type Expect<T extends true> = T
type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends
  (<T>() => T extends Y ? 1 : 2)
    ? true
    : false

// 测试
type Test1 = Expect<Equal<string, string>>  // ✓
type Test2 = Expect<Equal<number, string>>  // ✗ 编译错误
```

---

## 最佳实践

### 类型设计原则

1. **优先使用类型推导**：
```typescript
// 不好的做法
const config: { apiKey: string; endpoint: string } = {
  apiKey: "...",
  endpoint: "..."
}

// 好的做法
const config = {
  apiKey: "...",
  endpoint: "..."
}
// config 被自动推导为 { apiKey: string; endpoint: string }
```

2. **使用字面量类型而非枚举**：
```typescript
// 不好的做法
enum Mode { Production, Development }

// 好的做法
type Mode = "production" | "development"
```

3. **避免 any，使用 unknown**：
```typescript
// 不好的做法
function parse(input: any): any { }

// 好的做法
function parse(input: unknown): unknown {
  if (isString(input)) {
    return JSON.parse(input)
  }
  throw new Error("Invalid input")
}
```

4. **使用品牌类型防止混淆**：
```typescript
type UserId = string & { readonly __userId: unique symbol }
type SessionId = string & { readonly __sessionId: unique symbol }
```

---

## 总结

pi-mono 项目的类型系统设计体现了：

1. **深度类型安全** - Branded types、字面量类型、可辨识联合
2. **高级类型技巧** - 条件类型、映射类型、模板字面量类型
3. **类型级编程** - 类型推导、Schema 推导、类型级算术
4. **实用工具** - 类型守卫、断言函数、工具类型
5. **最佳实践** - 类型推导优于显式、unknown 优于 any

这些技巧共同构建了一个既强大又易维护的类型系统。

---

## 相关链接

- **设计哲学**：`/LEARN/05-patterns/01-design-philosophy.md`
- **设计模式目录**：`/LEARN/05-patterns/02-patterns-catalog.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
