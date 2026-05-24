# pi-ai 源码深度分析

> `@earendil-works/pi-ai` — 统一多供应商 LLM API

## 包概览

pi-ai 是整个 Pi 栈的基础层，提供**供应商无关的 LLM 调用接口**。核心设计思路是"两轴分离"：

```mermaid
graph TB
    subgraph "Provider 轴 (谁提供服务)"
        P1["anthropic"]
        P2["openai"]
        P3["google"]
        P4["openrouter"]
        P5["deepseek"]
        P6["...30+ 供应商"]
    end

    subgraph "Api 轴 (用什么协议)"
        A1["anthropic-messages"]
        A2["openai-completions"]
        A3["openai-responses"]
        A4["google-generative-ai"]
        A5["...9 种 API"]
    end

    P1 -->|"使用"| A1
    P2 -->|"使用"| A3
    P4 -->|"使用"| A2
    P5 -->|"使用"| A2
```

一个 Provider 使用一种 Api 实现。多个 Provider 可以共享同一种 Api（例如 OpenRouter、DeepSeek、Groq 都使用 `openai-completions`）。

## 文件结构

```
packages/ai/src/
├── index.ts                    # 公共导出桶
├── types.ts                    # 核心类型定义 (~576 行)
├── stream.ts                   # 主 API: stream/complete
├── models.ts                   # 模型注册表
├── models.generated.ts         # 自动生成的模型目录 (~16000 行)
├── api-registry.ts             # API 供应商注册表
├── image-models.ts             # 图片模型
├── image-models.generated.ts   # 自动生成的图片模型
├── images.ts                   # 图片生成 API
├── images-api-registry.ts      # 图片 API 注册表
├── env-api-keys.ts             # 环境变量 API Key 解析
├── oauth.ts                    # OAuth 子路径导出
├── bedrock-provider.ts         # Bedrock Node-only 子路径
├── session-resources.ts        # 会话资源清理
├── cli.ts                      # pi-ai CLI (login/list)
├── providers/                  # 供应商实现
│   ├── register-builtins.ts    # 内置供应商注册
│   ├── anthropic.ts            # Anthropic Claude
│   ├── openai-completions.ts   # OpenAI Completions API
│   ├── openai-responses.ts     # OpenAI Responses API
│   ├── openai-responses-shared.ts
│   ├── openai-codex-responses.ts
│   ├── azure-openai-responses.ts
│   ├── google.ts               # Google AI Studio
│   ├── google-shared.ts
│   ├── google-vertex.ts        # Google Vertex AI
│   ├── mistral.ts              # Mistral AI
│   ├── amazon-bedrock.ts       # AWS Bedrock
│   ├── cloudflare.ts           # Cloudflare Workers AI
│   ├── faux.ts                 # 测试用模拟供应商
│   ├── simple-options.ts       # reasoning → 供应商参数映射
│   ├── transform-messages.ts   # 跨供应商消息转换
│   ├── github-copilot-headers.ts
│   ├── openai-prompt-cache.ts
│   └── images/
│       ├── register-builtins.ts
│       └── openrouter.ts
└── utils/
    ├── event-stream.ts         # EventStream + AssistantMessageEventStream
    ├── validation.ts           # TypeBox 工具参数验证
    ├── typebox-helpers.ts      # TypeBox 辅助
    ├── json-parse.ts           # 部分 JSON 解析
    ├── overflow.ts             # 上下文溢出处理
    ├── diagnostics.ts          # 错误诊断
    ├── hash.ts                 # 哈希工具
    ├── headers.ts              # HTTP 头处理
    ├── sanitize-unicode.ts     # Unicode 清理
    ├── node-http-proxy.ts      # HTTP 代理支持
    └── oauth/                  # OAuth 实现
        ├── index.ts
        ├── types.ts
        ├── anthropic.ts
        ├── github-copilot.ts
        ├── openai-codex.ts
        ├── device-code.ts
        ├── pkce.ts
        └── oauth-page.ts
```

## 核心类型体系 (types.ts)

### 消息模型

```mermaid
classDiagram
    class Message {
        <<union>>
    }
    class UserMessage {
        +role: "user"
        +content: string | Content[]
        +timestamp: number
    }
    class AssistantMessage {
        +role: "assistant"
        +content: Content[]
        +api: Api
        +provider: Provider
        +model: string
        +usage: Usage
        +stopReason: StopReason
        +timestamp: number
    }
    class ToolResultMessage {
        +role: "toolResult"
        +toolCallId: string
        +toolName: string
        +content: Content[]
        +isError: boolean
        +timestamp: number
    }
    
    Message <|-- UserMessage
    Message <|-- AssistantMessage
    Message <|-- ToolResultMessage

    class TextContent {
        +type: "text"
        +text: string
    }
    class ThinkingContent {
        +type: "thinking"
        +thinking: string
        +redacted?: boolean
    }
    class ImageContent {
        +type: "image"
        +data: string
        +mimeType: string
    }
    class ToolCall {
        +type: "toolCall"
        +id: string
        +name: string
        +arguments: Record
    }
```

### Model 接口

每个模型包含完整的元数据，用于成本计算、能力检测和兼容性处理：

```typescript
interface Model<TApi extends Api> {
   id: string;                    // 模型 ID
   name: string;                  // 显示名称
   api: TApi;                     // API 协议类型
   provider: Provider;            // 供应商名称
   baseUrl: string;               // API 端点
   reasoning: boolean;            // 支持推理?
   thinkingLevelMap?: ThinkingLevelMap;  // 思考级别映射
   input: ("text" | "image")[];   // 输入类型
   cost: { input, output, cacheRead, cacheWrite };  // $/百万 token
   contextWindow: number;         // 上下文窗口大小
   maxTokens: number;             // 最大输出 token
   headers?: Record<string, string>;  // 自定义请求头
   compat?: ...;                  // 兼容性覆盖
}
```

### 流式协议

```mermaid
graph LR
    SF["StreamFunction"] -->|"返回"| AMES["AssistantMessageEventStream"]
    AMES -->|"异步迭代"| Events["AssistantMessageEvent"]
    AMES -->|".result()"| Final["AssistantMessage"]
```

**核心合约**：
- `StreamFunction` **不得抛异常**
- 错误通过流内 `error` 事件传递
- 错误消息的 `stopReason` 为 `"error"` 或 `"aborted"`

## API 注册表 (api-registry.ts)

注册表是一个简单的 `Map<Api, RegisteredApiProvider>`：

```mermaid
graph TB
    subgraph "注册表"
        REG["apiProviderRegistry<br/>Map&lt;Api, Provider&gt;"]
    end

    subgraph "注册"
        R1["registerApiProvider()"]
    end

    subgraph "查询"
        G1["getApiProvider(api)"]
        G2["getApiProviders()"]
    end

    subgraph "清理"
        U1["unregisterApiProviders(sourceId)"]
        U2["clearApiProviders()"]
    end

    R1 --> REG
    REG --> G1
    REG --> G2
    U1 --> REG
```

注册时会包裹 `stream` 和 `streamSimple` 函数，添加 API 类型校验。

## 主 API: stream.ts

```typescript
// 完整控制的流式调用
function stream<TApi>(model, context, options?) → AssistantMessageEventStream

// 统一 reasoning 参数的简化版
function streamSimple<TApi>(model, context, options?) → AssistantMessageEventStream

// 阻塞式版本
async function complete<TApi>(model, context, options?) → AssistantMessage
async function completeSimple<TApi>(model, context, options?) → AssistantMessage
```

`streamSimple` 是 Agent 层的默认调用方式，它将统一的 `reasoning: ThinkingLevel` 参数映射为各供应商的具体参数。

## 供应商实现模式

每个供应商实现遵循统一模式：

```mermaid
flowchart TD
    A["streamXxx(model, context, options)"] --> B["构建请求体<br/>Context → 供应商格式"]
    B --> C["transformMessages()<br/>跨供应商兼容"]
    C --> D["发送 HTTP 请求<br/>SSE/WebSocket"]
    D --> E["解析供应商响应流"]
    E --> F["转换为 AssistantMessageEvent"]
    F --> G["计算 Usage/Cost"]
    G --> H["返回 AssistantMessageEventStream"]
```

### 各供应商的关键差异

| 供应商 | 消息格式 | 流式协议 | 思考支持 | 特殊处理 |
|--------|---------|---------|---------|---------|
| Anthropic | messages API | SSE | extended thinking | cache_control, 自适应思考 |
| OpenAI Completions | chat/completions | SSE | reasoning_effort | developer role, store |
| OpenAI Responses | responses API | SSE | reasoning | session_id 缓存 |
| Google | generateContent | SSE | thinkingConfig | thoughtSignature |
| Bedrock | ConverseStream | 自有协议 | 不支持 | Node-only, AWS SDK |
| Mistral | chat/completions | SSE | 不支持 | 自有 SDK |

### 懒加载策略

供应商通过动态 `import()` 懒加载，确保浏览器和 Node 环境只加载需要的供应商：

```mermaid
graph LR
    REG["register-builtins.ts"] -->|"懒加载包裹"| A["anthropic.ts"]
    REG -->|"懒加载包裹"| B["openai-completions.ts"]
    REG -->|"懒加载包裹"| C["google.ts"]
    REG -->|"条件 Node-only"| D["amazon-bedrock.ts"]
```

## 模型目录 (models.ts + models.generated.ts)

```mermaid
graph TB
    GEN["scripts/generate-models.ts"] -->|"生成"| MG["models.generated.ts<br/>~16000 行"]
    MG -->|"注册到"| REG["内存注册表<br/>Map&lt;Provider, Map&lt;ModelId, Model&gt;&gt;"]
    
    subgraph "查询 API"
        G1["getModel(provider, id)"]
        G2["getModels()"]
        G3["getProviders()"]
        G4["calculateCost(model, usage)"]
    end
    
    REG --> G1
    REG --> G2
    REG --> G3
```

### 模型目录生成流程

```mermaid
flowchart LR
    SRC["generate-models.ts<br/>供应商元数据"] --> PROC["处理和验证"]
    PROC --> TS["models.generated.ts<br/>TypeScript 代码"]
    TS --> BUILD["编译到 dist/"]
```

模型目录定义了每个供应商的每个模型的完整元数据，包括 API 端点、token 价格、上下文窗口大小等。

## 跨供应商兼容 (transform-messages.ts)

当用户在同一会话中切换供应商时，`transformMessages` 自动处理消息格式差异：

```mermaid
graph TB
    INPUT["原始 Message[]"] --> CHECK{"需要转换?"}
    
    CHECK -->|"图片"| IMG["downgradeImages()<br/>移除非视觉模型的图片"]
    CHECK -->|"思考块"| THINK["convertThinking()<br/>thinking → &lt;thinking&gt; text"]
    CHECK -->|"工具 ID"| TID["normalizeToolIds()<br/>修复格式不兼容"]
    CHECK -->|"签名"| SIGN["清理供应商特有字段"]
    
    IMG --> OUTPUT["转换后 Message[]"]
    THINK --> OUTPUT
    TID --> OUTPUT
    SIGN --> OUTPUT
```

## EventStream 实现 (utils/event-stream.ts)

```mermaid
classDiagram
    class EventStream~T,R~ {
        -buffer: T[]
        -resolvers: Function[]
        -completed: boolean
        -result: R
        +push(event: T)
        +end(result: R)
        +result(): Promise~R~
        +[Symbol.asyncIterator]()
    }
    
    class AssistantMessageEventStream {
        +result(): Promise~AssistantMessage~
    }
    
    EventStream <|-- AssistantMessageEventStream
```

`EventStream` 是一个**推式异步可迭代流**：
- 生产者通过 `push()` 推入事件
- 消费者通过 `for await` 消费
- `result()` 等待流完成并返回最终值
- 终止条件由构造函数的谓词定义

## 工具参数验证 (utils/validation.ts)

使用 TypeBox 进行运行时类型验证：

```mermaid
graph LR
    TC["工具调用参数"] --> V["validateToolArguments()"]
    V --> TB["TypeBox Schema 校验"]
    TB -->|"通过"| OK["返回验证后的参数"]
    TB -->|"失败"| ERR["抛出验证错误"]
```

## 测试支持: faux provider

```mermaid
graph LR
    TEST["测试代码"] --> FR["registerFauxProvider()"]
    FR --> FP["FauxProvider<br/>内存模拟"]
    FP --> Q["预设响应队列"]
    TEST --> FA["fauxAssistantMessage()<br/>fauxText()<br/>fauxToolCall()"]
    FA --> Q
```

faux provider 是完整的内存供应商实现，支持：
- 预设响应队列
- 模拟流式输出
- 模拟缓存命中
- 测试各种错误场景
