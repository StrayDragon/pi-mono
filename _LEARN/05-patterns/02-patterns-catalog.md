# 设计模式索引

本文档 catalog Pi monorepo 中使用的设计模式，每个模式附 Mermaid 图与源码位置。

---

## 1. Registry 模式

**用途：** 运行时注册与查找 Provider、API 实现、模型。

```mermaid
graph TB
    subgraph ApiRegistry["ApiProvider Registry"]
        REG["apiProviderRegistry: Map&lt;Api, Provider&gt;"]
        R1["registerApiProvider()"]
        G1["getApiProvider()"]
        R1 --> REG
        G1 --> REG
    end

    subgraph ModelRegistry["Model Registry"]
        MR["ModelRegistry"]
        R2["registerProvider()"]
        G2["getModel() / listModels()"]
        MJ["models.json 合并"]
        R2 --> MR
        G2 --> MR
        MJ --> MR
    end

    subgraph ImagesRegistry["Images API Registry"]
        IR["imagesApiRegistry"]
    end

    STREAM["stream()"] --> G1
    CLI["--list-models"] --> G2
```

| 注册表 | 注册 API | 查找 API | 源码 |
|--------|---------|---------|------|
| ApiProvider | `registerApiProvider()` | `getApiProvider()` | `packages/ai/src/api-registry.ts` |
| Model | `ModelRegistry.registerProvider()` | `getModel()`、`findModels()` | `packages/coding-agent/src/core/model-registry.ts` |
| Images API | `registerImagesApiProvider()` | `getImagesApiProvider()` | `packages/ai/src/images-api-registry.ts` |
| 内置 Provider | `registerBuiltins()` | 启动时自动 | `packages/ai/src/providers/register-builtins.ts` |

---

## 2. Event Stream 模式

**用途：** LLM 响应的异步增量 delivery。

```mermaid
stateDiagram-v2
    [*] --> Active: new EventStream()
    Active --> Active: push(partial event)
    Active --> Complete: push(done)
    Active --> Error: push(error)
    Complete --> [*]: result() resolved
    Error --> [*]: result() resolved

    note right of Active
        消费者 for await 或
        等待 .result()
    end note
```

```mermaid
classDiagram
    class EventStream~T, R~ {
        -queue: T[]
        -waiting: resolver[]
        -done: boolean
        +push(event: T)
        +end(result?: R)
        +result(): Promise~R~
        +[asyncIterator]()
    }

    class AssistantMessageEventStream {
        完成条件: type=done|error
        结果: AssistantMessage
    }

    EventStream <|-- AssistantMessageEventStream
```

| 组件 | 源码 |
|------|------|
| `EventStream<T, R>` | `packages/ai/src/utils/event-stream.ts` |
| `AssistantMessageEventStream` | 同上 |
| `createAssistantMessageEventStream()` | 同上（扩展工厂） |
| Provider 实现 | `packages/ai/src/providers/*.ts` |

---

## 3. Declaration Merging（声明合并）

**用途：** 在不修改上游包的情况下扩展联合类型。

```mermaid
graph LR
    subgraph pi-agent-core
        CAM["interface CustomAgentMessages {}"]
        AM["type AgentMessage = ... | CustomAgentMessages[key]"]
    end

    subgraph pi-coding-agent
        MERGE["declare module '@earendil-works/pi-agent-core' {<br/>  interface CustomAgentMessages {<br/>    bashExecution: ...<br/>    custom: ...<br/>  }<br/>}"]
    end

    subgraph pi-tui
        KB["interface Keybindings {}"]
    end

    subgraph coding-agent
        KBM["interface Keybindings extends AppKeybindings {}"]
    end

    MERGE --> CAM
    KBM --> KB
```

| 扩展点 | 合并位置 | 源码 |
|--------|---------|------|
| `CustomAgentMessages` | coding-agent | `packages/coding-agent/src/core/messages.ts` |
| `Keybindings` | coding-agent → pi-tui | `packages/coding-agent/src/core/keybindings.ts` |
| 示例（web-ui） | 消费者项目 | `packages/web-ui/example/src/custom-messages.ts` |

---

## 4. Factory 模式

**用途：** 创建工具、会话、Agent 实例，隐藏组装细节。

```mermaid
flowchart TB
    subgraph Tool Factories
        CR["createReadTool()"]
        CB["createBashTool()"]
        CE["createEditTool()"]
        CTD["createXxxToolDefinition()"]
    end

    subgraph Session Factories
        CAS["createAgentSession()"]
        CASS["createAgentSessionServices()"]
        CASR["createAgentSessionRuntime()"]
    end

    subgraph Stream Factories
        CAES["createAssistantMessageEventStream()"]
    end

    SDK["SDK / CLI"] --> CAS
    CAS --> CASS
    CASS --> AgentSession
    CTD --> wrapToolDefinition --> AgentTool
```

| 工厂 | 产出 | 源码 |
|------|------|------|
| `createReadTool()` 等 | `AgentTool` | `packages/coding-agent/src/core/tools/*.ts` |
| `createXxxToolDefinition()` | `ToolDefinition`（扩展用） | 同上 |
| `createAgentSession()` | `{ session, ... }` | `packages/coding-agent/src/core/sdk.ts` |
| `createAgentSessionServices()` | 服务容器 | `packages/coding-agent/src/core/agent-session-services.ts` |
| `defineTool()` | 类型安全的扩展工具 | `packages/coding-agent/src/core/extensions/types.ts` |

---

## 5. Plugin / Extension 模式

**用途：** 用户 TS 模块在运行时注册工具、命令、Provider、UI。

```mermaid
sequenceDiagram
    participant Loader as Extension Loader
    participant Jiti as jiti
    participant Factory as ExtensionFactory
    participant API as ExtensionAPI
    participant Runner as ExtensionRunner

    Loader->>Jiti: import(extension.ts)
    Jiti->>Factory: default export function(pi)
    Factory->>API: pi.registerTool()
    Factory->>API: pi.on("session_start")
    Factory->>API: pi.registerCommand()
    API->>Runner: 注册 handler
    Note over Runner: 生命周期事件触发 handler
```

| 概念 | 类型/接口 | 源码 |
|------|----------|------|
| `ExtensionFactory` | `(pi: ExtensionAPI) => void \| Promise<void>` | `packages/coding-agent/src/core/extensions/types.ts` |
| `ExtensionAPI` | 注册/订阅/UI 上下文 | 同上 |
| Loader | jiti + virtualModules | `packages/coding-agent/src/core/extensions/loader.ts` |
| Runner | 事件分发 | `packages/coding-agent/src/core/extensions/runner.ts` |

示例：`packages/coding-agent/examples/extensions/hello.ts`

---

## 6. Result 类型模式

**用途：** 可预期失败以值返回，而非 throw。

```mermaid
flowchart LR
    OP["ExecutionEnv.readTextFile()"]
    R{"Result&lt;T, E&gt;"}
    OK["{ ok: true, value }"]
    ERR["{ ok: false, error }"]

    OP --> R
    R --> OK
    R --> ERR

    OK --> USE["getOrThrow / 模式匹配"]
    ERR --> HANDLE["结构化错误处理"]
```

```typescript
// packages/agent/src/harness/types.ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
function ok<T, E>(value: T): Result<T, E>;
function err<T, E>(error: E): Result<T, E>;
```

| 使用处 | 源码 |
|--------|------|
| `ExecutionEnv` 全部文件/进程操作 | `packages/agent/src/harness/types.ts` |
| Node.js 实现 | `packages/agent/src/harness/env/nodejs.ts` |
| `AuthStorage.withLock` | `packages/coding-agent/src/core/auth-storage.ts`（内部 LockResult） |

---

## 7. Observer 模式

**用途：** Agent 状态变化、会话事件、扩展钩子的订阅/通知。

```mermaid
graph TB
    subgraph Agent Observer
        AG["Agent"]
        SUB["subscribe(listener)"]
        EV["AgentEvent"]
        AG --> SUB
        SUB --> EV
    end

    subgraph Session Observer
        AS["AgentSession"]
        SE["AgentSessionEvent"]
        AS --> SE
    end

    subgraph Extension Observer
        PI["pi.on(event, handler)"]
        RUN["ExtensionRunner.emit()"]
        RUN --> PI
    end

    EV --> UI["InteractiveMode 渲染"]
    SE --> UI
    PI --> EXT["扩展逻辑"]
```

| 订阅 API | 事件类型 | 源码 |
|----------|---------|------|
| `Agent.subscribe()` | `AgentEvent` | `packages/agent/src/agent.ts` |
| `AgentSession` 事件 | `AgentSessionEvent` | `packages/coding-agent/src/core/agent-session.ts` |
| `pi.on()` | 20+ 扩展事件 | `packages/coding-agent/src/core/extensions/types.ts` |
| `EventBus` | 内部 pub/sub | `packages/coding-agent/src/core/event-bus.ts` |

---

## 8. Strategy 模式

**用途：** 运行时切换 LLM 传输策略、工具执行模式。

```mermaid
graph TB
    subgraph StreamFn Strategy
        AL["AgentLoop"]
        SF["StreamFn 注入"]
        S1["streamSimple"]
        S2["streamProxy"]
        S3["faux mock"]
        AL --> SF
        SF --> S1 & S2 & S3
    end

    subgraph ToolExecutionMode Strategy
        TC["Tool Call"]
        MODE{"ToolExecutionMode"}
        SYNC["sync 阻塞"]
        ASYNC["async 并行"]
        TC --> MODE
        MODE --> SYNC & ASYNC
    end
```

| 策略 | 配置点 | 源码 |
|------|--------|------|
| `StreamFn` | `Agent` 构造 / `createAgentSession({ streamFn })` | `packages/agent/src/types.ts` |
| `ToolExecutionMode` | Agent 配置 / 扩展 | `packages/agent/src/types.ts` |
| `QueueMode`（steering/followUp） | Settings `steeringMode` / `followUpMode` | `packages/coding-agent/src/core/settings-manager.ts` |

---

## 9. Builder 模式

**用途：** 分步构建 system prompt，组合工具说明、技能、上下文文件。

```mermaid
flowchart TB
    OPT["BuildSystemPromptOptions"]
    OPT --> CWD["cwd"]
    OPT --> TOOLS["selectedTools"]
    OPT --> SKILLS["skills[]"]
    OPT --> CTX["contextFiles[]"]
    OPT --> APPEND["appendSystemPrompt"]
    OPT --> CUSTOM["customPrompt?"]

    CWD & TOOLS & SKILLS & CTX & APPEND --> BUILD["buildSystemPrompt()"]
    CUSTOM -->|"若存在"| OVERRIDE["完全替换默认模板"]
    BUILD --> PROMPT["最终 system prompt 字符串"]
    OVERRIDE --> PROMPT
```

| 组件 | 源码 |
|------|------|
| `BuildSystemPromptOptions` | `packages/coding-agent/src/core/system-prompt.ts` |
| `buildSystemPrompt()` | 同上 |
| `formatSkillsForPrompt()` | `packages/coding-agent/src/core/skills.ts` |
| 扩展 hook | `pi.on("before_agent_start")` 可修改 prompt |

---

## 10. Queue 模式

**用途：** 串行化文件 mutation；排队 steering/follow-up 消息。

```mermaid
graph TB
    subgraph File Mutation Queue
        T1["edit(path=A)"]
        T2["write(path=A)"]
        Q["withFileMutationQueue(A)"]
        T1 --> Q
        T2 --> Q
        Q --> SERIAL["同路径串行"]
    end

    subgraph Message Queues
        STEER["Steering Queue<br/>流式期间 Enter 排队"]
        FOLLOW["Follow-up Queue<br/>Alt+Enter 排队"]
        MODE{"one-at-a-time | all"}
        STEER --> MODE
        FOLLOW --> MODE
        MODE --> LOOP["下一轮 AgentLoop 注入"]
    end
```

| 队列 | API | 源码 |
|------|-----|------|
| 文件 mutation | `withFileMutationQueue(path, fn)` | `packages/coding-agent/src/core/tools/file-mutation-queue.ts` |
| Steering | `session.steer()` / Enter while streaming | `packages/coding-agent/src/core/agent-session.ts` |
| Follow-up | `session.followUp()` / Alt+Enter | 同上 |
| 队列模式 | `steeringMode`、`followUpMode` settings | `packages/coding-agent/src/core/settings-manager.ts` |

---

## 模式关系总览

```mermaid
graph TB
    REG["Registry"] --> STREAM["Event Stream"]
    FACT["Factory"] --> EXT["Plugin/Extension"]
    EXT --> OBS["Observer"]
    OBS --> STRAT["Strategy"]
    BUILD["Builder"] --> PROMPT["System Prompt"]
    QUEUE["Queue"] --> TOOLS["Tool Execution"]
    MERGE["Declaration Merge"] --> TYPES["Type System"]
    RESULT["Result"] --> ENV["ExecutionEnv"]

    STREAM --> UI["TUI 渲染"]
    EXT --> REG
    FACT --> REG
```

---

## 延伸阅读

- [TypeScript 类型体系](./03-type-system.md)
- [扩展系统](../04-subsystems/02-extension-system.md)
- [工具系统](../04-subsystems/01-tool-system.md)
- [事件系统](../02-architecture/04-event-system.md)
