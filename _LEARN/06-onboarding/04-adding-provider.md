# 添加 Provider (Adding Provider)

## 概述

本文档指导开发者如何为 pi-ai 添加新的 LLM Provider。Provider 是与 LLM API 交互的适配器，统一的接口使 pi-coding-agent 能够支持多种 LLM 服务。

---

## Provider 基础

### Provider 接口

```typescript
// packages/ai/src/types.ts
interface LLMProvider {
  // 聊天接口（流式）
  async *chat(
    messages: Message[],
    options: ChatOptions
  ): AsyncGenerator<Chunk>

  // 嵌入接口（可选）
  embed?(texts: string[]): Promise<number[][]>

  // 元数据
  readonly name: string
  readonly models: ModelInfo[]
}

interface Message {
  role: "user" | "assistant" | "system" | "tool"
  content: string
  toolCalls?: ToolCall[]
  toolCallId?: string
}

interface Chunk {
  content: string
  toolCalls?: ToolCall[]
  done: boolean
  usage?: TokenUsage
}

interface ChatOptions {
  temperature?: number
  maxTokens?: number
  topP?: number
  stopSequences?: string[]
  tools?: Tool[]
}
```

### Provider 注册

```typescript
// packages/ai/src/api-registry.ts
class APIRegistry {
  register(
    key: string,
    factory: (config: ProviderConfig) => LLMProvider
  ): void {
    this.providers.set(key, factory)
  }

  create(
    key: string,
    config: ProviderConfig
  ): LLMProvider {
    const factory = this.providers.get(key)
    if (!factory) throw new Error(`Unknown provider: ${key}`)
    return factory(config)
  }
}

const registry = new APIRegistry()

// 注册 Provider
registry.register("my-provider", (config) => new MyProvider(config))
```

---

## 创建简单 Provider

### 步骤 1：定义 Provider 类

```typescript
// packages/ai/src/providers/my-provider.ts
import { LLMProvider, Message, Chunk, ChatOptions } from "../types"

export class MyProvider implements LLMProvider {
  readonly name = "my-provider"

  readonly models = [
    { id: "my-model-1", name: "My Model 1", context: 4096 },
    { id: "my-model-2", name: "My Model 2", context: 8192 }
  ]

  constructor(private config: ProviderConfig) {
    // 验证配置
    if (!config.apiKey) {
      throw new Error("MyProvider requires apiKey")
    }
  }

  async *chat(
    messages: Message[],
    options: ChatOptions
  ): AsyncGenerator<Chunk> {
    // 1. 转换消息格式
    const apiMessages = this.convertMessages(messages)

    // 2. 调用 API
    const response = await this.callAPI(apiMessages, options)

    // 3. 流式返回
    for await (const chunk of response) {
      yield this.convertChunk(chunk)
    }
  }
}
```

### 步骤 2：实现消息转换

```typescript
private convertMessages(messages: Message[]): MyAPIMessage[] {
  return messages.map(msg => ({
    role: msg.role,
    content: msg.content,
    tool_calls: msg.toolCalls?.map(tc => ({
      id: tc.id,
      type: tc.type,
      function: {
        name: tc.function.name,
        arguments: tc.function.arguments
      }
    }))
  }))
}
```

### 步骤 3：实现 API 调用

```typescript
private async callAPI(
  messages: MyAPIMessage[],
  options: ChatOptions
): AsyncGenerator<MyAPIChunk> {
  const response = await fetch("https://api.myprovider.com/v1/chat", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${this.config.apiKey}`
    },
    body: JSON.stringify({
      model: this.config.model || "my-model-1",
      messages,
      stream: true,
      temperature: options.temperature || 0.7,
      max_tokens: options.maxTokens || 4096
    })
  })

  if (!response.ok) {
    throw new Error(`API error: ${response.statusText}`)
  }

  // 解析流式响应
  const reader = response.body!.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const chunk = decoder.decode(value)
    const lines = chunk.split("\n").filter(line => line.trim())

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const data = JSON.parse(line.slice(6))
        yield data
      }
    }
  }
}
```

### 步骤 4：实现响应转换

```typescript
private convertChunk(apiChunk: MyAPIChunk): Chunk {
  return {
    content: apiChunk.content || "",
    toolCalls: apiChunk.tool_calls?.map(tc => ({
      id: tc.id,
      type: tc.type,
      function: {
        name: tc.function.name,
        arguments: tc.function.arguments
      }
    })),
    done: apiChunk.finish_reason === "stop",
    usage: apiChunk.usage ? {
      prompt: apiChunk.usage.prompt_tokens,
      completion: apiChunk.usage.completion_tokens,
      total: apiChunk.usage.total_tokens
    } : undefined
  }
}
```

### 步骤 5：注册 Provider

```typescript
// packages/ai/src/providers/register-builtins.ts
import { MyProvider } from "./my-provider"

export function registerBuiltinProviders(registry: APIRegistry) {
  // ... 其他 Provider

  registry.register("my-provider", (config) => new MyProvider(config))
}
```

---

## 高级 Provider 功能

### 1. 支持嵌入

```typescript
class MyProvider implements LLMProvider {
  // ... chat 实现

  async embed(texts: string[]): Promise<number[][]> {
    const response = await fetch("https://api.myprovider.com/v1/embeddings", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${this.config.apiKey}`
      },
      body: JSON.stringify({
        model: "my-embedding-model",
        input: texts
      })
    })

    const data = await response.json()
    return data.data.map((item: any) => item.embedding)
  }
}
```

### 2. 支持工具调用

```typescript
async *chat(messages: Message[], options: ChatOptions): AsyncGenerator<Chunk> {
  const tools = options.tools?.map(tool => ({
    type: "function",
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.parameters
    }
  }))

  const response = await this.callAPI(messages, {
    ...options,
    tools
  })

  for await (const chunk of response) {
    // 处理工具调用
    if (chunk.tool_calls) {
      yield {
        content: "",
        toolCalls: chunk.tool_calls.map(tc => ({
          id: tc.id,
          type: "function",
          function: {
            name: tc.function.name,
            arguments: JSON.stringify(tc.function.arguments)
          }
        })),
        done: false
      }
    } else {
      yield {
        content: chunk.content,
        done: chunk.finish_reason === "stop"
      }
    }
  }
}
```

### 3. 支持思考模式

```typescript
async *chat(messages: Message[], options: ChatOptions): AsyncGenerator<Chunk> {
  const hasThinking = messages.some(m =>
    m.role === "system" && m.content.includes("think step by step")
  )

  if (hasThinking) {
    // 启用思考模式
    const response = await this.callAPI(messages, {
      ...options,
      reasoning_mode: "thinking"
    })

    for await (const chunk of response) {
      if (chunk.reasoning) {
        // 发送思考内容
        yield {
          content: chunk.reasoning,
          done: false
        }
      } else if (chunk.content) {
        // 发送最终回答
        yield {
          content: chunk.content,
          done: chunk.finish_reason === "stop"
        }
      }
    }
  } else {
    // 正常模式
    for await (const chunk of this.callAPI(messages, options)) {
      yield this.convertChunk(chunk)
    }
  }
}
```

### 4. 错误处理和重试

```typescript
class MyProvider implements LLMProvider {
  private maxRetries = 3
  private retryDelay = 1000

  private async callAPIWithRetry(
    messages: MyAPIMessage[],
    options: ChatOptions
  ): Promise<Response> {
    let lastError: Error

    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        const response = await this.callAPI(messages, options)

        if (response.ok) {
          return response
        }

        // 特定错误码处理
        if (response.status === 401) {
          throw new Error("Invalid API key")
        }

        if (response.status === 429) {
          // 速率限制，重试
          await this.delay(this.retryDelay * (attempt + 1))
          continue
        }

        lastError = new Error(`API error: ${response.statusText}`)
      } catch (error) {
        lastError = error as Error

        // 网络错误，重试
        if (error instanceof TypeError) {
          await this.delay(this.retryDelay * (attempt + 1))
          continue
        }

        throw error
      }
    }

    throw lastError!
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```

---

## 配置和模型定义

### 配置 Schema

```typescript
// packages/ai/src/providers/my-provider/config.ts
import { z } from "zod"

export const MyProviderConfigSchema = z.object({
  apiKey: z.string().min(1),
  endpoint: z.string().url().default("https://api.myprovider.com"),
  model: z.string().default("my-model-1"),
  timeout: z.number().default(30000),
  maxRetries: z.number().default(3)
})

export type MyProviderConfig = z.infer<typeof MyProviderConfigSchema>
```

### 模型定义

```typescript
// packages/ai/src/models.ts
export const MODELS: Record<string, ModelInfo[]> = {
  "my-provider": [
    {
      id: "my-model-1",
      name: "My Model 1",
      provider: "my-provider",
      context: 4096,
      supportsImages: false,
      supportsTools: true,
      supportsStreaming: true
    },
    {
      id: "my-model-2",
      name: "My Model 2",
      provider: "my-provider",
      context: 8192,
      supportsImages: true,
      supportsTools: true,
      supportsStreaming: true
    }
  ]
}
```

---

## 测试 Provider

### 单元测试

```typescript
// tests/providers/my-provider.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { MyProvider } from "@/providers/my-provider"

describe("MyProvider", () => {
  let provider: MyProvider

  beforeEach(() => {
    provider = new MyProvider({
      apiKey: "test-key",
      model: "my-model-1"
    })
  })

  it("should convert messages correctly", () => {
    const messages = [
      { role: "user", content: "Hello" }
    ]

    const result = provider["convertMessages"](messages)

    expect(result).toEqual([
      { role: "user", content: "Hello" }
    ])
  })

  it("should stream chat responses", async () => {
    const messages = [
      { role: "user", content: "Test" }
    ]

    const chunks = []
    for await (const chunk of provider.chat(messages, {})) {
      chunks.push(chunk)
    }

    expect(chunks.length).toBeGreaterThan(0)
    expect(chunks[chunks.length - 1].done).toBe(true)
  })
})
```

### 集成测试

```typescript
// tests/integration/my-provider-integration.test.ts
import { describe, it, expect } from "vitest"
import { APIRegistry } from "@/api-registry"

describe("MyProvider Integration", () => {
  it("should work with real API", { timeout: 30000 }, async () => {
    const registry = new APIRegistry()

    // 注册测试 Provider
    registry.register("my-provider-test", (config) => {
      return new MyProvider({
        ...config,
        endpoint: "https://api.myprovider.com/v1"
      })
    })

    // 创建 Provider
    const provider = registry.create("my-provider-test", {
      apiKey: process.env.MY_PROVIDER_API_KEY!
    })

    // 测试聊天
    const messages = [
      { role: "user", content: "Say 'Hello, World!'" }
    ]

    const chunks = []
    for await (const chunk of provider.chat(messages, {})) {
      chunks.push(chunk)
    }

    const response = chunks.map(c => c.content).join("")
    expect(response).toContain("Hello")
  })
})
```

---

## 性能优化

### 1. 连接池

```typescript
class MyProvider implements LLMProvider {
  private connectionPool: Map<string, Connection> = new Map()

  private async getConnection(): Promise<Connection> {
    const key = this.config.apiKey

    if (!this.connectionPool.has(key)) {
      const connection = await Connection.create(this.config.endpoint)
      this.connectionPool.set(key, connection)
    }

    return this.connectionPool.get(key)!
  }

  async *chat(messages: Message[], options: ChatOptions): AsyncGenerator<Chunk> {
    const connection = await this.getConnection()

    try {
      const response = await connection.send(messages, options)
      for await (const chunk of response) {
        yield this.convertChunk(chunk)
      }
    } catch (error) {
      // 连接失败，移除并重试
      this.connectionPool.clear()
      throw error
    }
  }
}
```

### 2. 批量嵌入

```typescript
async embed(texts: string[]): Promise<number[][]> {
  // 批量处理
  const batchSize = 100
  const results: number[][] = []

  for (let i = 0; i < texts.length; i += batchSize) {
    const batch = texts.slice(i, i + batchSize)
    const response = await this.callEmbedAPI(batch)
    results.push(...response)
  }

  return results
}
```

### 3. 缓存

```typescript
class CachedProvider implements LLMProvider {
  private cache = new Map<string, Chunk[]>()

  constructor(private provider: LLMProvider) { }

  async *chat(messages: Message[], options: ChatOptions): AsyncGenerator<Chunk> {
    const cacheKey = this.getCacheKey(messages, options)

    if (this.cache.has(cacheKey)) {
      yield* this.cache.get(cacheKey)!
      return
    }

    const chunks: Chunk[] = []
    for await (const chunk of this.provider.chat(messages, options)) {
      chunks.push(chunk)
      yield chunk
    }

    this.cache.set(cacheKey, chunks)
  }

  private getCacheKey(messages: Message[], options: ChatOptions): string {
    return JSON.stringify({ messages, options })
  }
}
```

---

## 文档和发布

### 编写文档

```markdown
# MyProvider

## 配置

\`\`\`json
{
  "provider": "my-provider",
  "apiKey": "your-api-key",
  "endpoint": "https://api.myprovider.com",
  "model": "my-model-1"
}
\`\`\`

## 支持的模型

- \`my-model-1\`: 基础模型，4K 上下文
- \`my-model-2\`: 高级模型，8K 上下文，支持图像

## 使用示例

\`\`\`typescript
import { APIRegistry } from "@mariozechner/pi-ai"

const registry = new APIRegistry()
const provider = registry.create("my-provider", {
  apiKey: "your-api-key"
})

const response = await provider.chat([
  { role: "user", content: "Hello!" }
], {})
\`\`\`
```

### 发布到 npm

```bash
# 1. 更新版本
npm version patch

# 2. 构建
npm run build

# 3. 发布
npm publish

# 4. 在 pi-mono 中使用
# npm install @your-org/my-provider
```

---

## 总结

✅ 你已经学会：
- 实现 LLMProvider 接口
- 处理消息转换和 API 调用
- 支持高级功能（工具调用、思考模式）
- 错误处理和重试
- 性能优化
- 测试和发布

**下一步**：
- [添加工具](./05-adding-tool.md)
- [创建 Skill](./06-creating-skill.md)
- [调试技巧](./07-debugging.md)

---

## 相关链接

- **Provider 系统**：`/LEARN/04-subsystems/04-provider-system.md`
- **pi-ai 包**：`/LEARN/03-packages/01-pi-ai.md`
- **API 参考手册**：`https://docs.pi-ai.dev`
