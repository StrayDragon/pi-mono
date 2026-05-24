# pi-agent-core 源码深度分析

> `@earendil-works/pi-agent-core` — Agent 运行时与会话管理

## 包概览

pi-agent-core 是 Pi 栈的**运行时心脏**，提供两层抽象：

```mermaid
graph TB
    subgraph "上层: AgentHarness"
        AH["AgentHarness<br/>持久化、钩子、技能"]
        ENV["ExecutionEnv<br/>文件系统 + Shell"]
        SESS["Session 树"]
        COMP["压缩系统"]
        SKILLS["技能/提示模板"]
    end

    subgraph "下层: Agent + Loop"
        AGENT["Agent 类<br/>有状态包裹"]
        LOOP["agentLoop<br/>函数式核心"]
        TYPES["类型系统<br/>AgentMessage, AgentTool, AgentEvent"]
    end

    subgraph "依赖"
        AI["pi-ai<br/>stream, Model, Message"]
    end

    AH --> AGENT
    AGENT --> LOOP
    LOOP --> AI
    AH --> ENV
    AH --> SESS
    AH --> COMP
```

## 文件结构

```
packages/agent/src/
├── index.ts                    # 公共导出
├── node.ts                     # Node 子路径 (NodeExecutionEnv)
├── agent.ts                    # Agent 类 (~350 行)
├── agent-loop.ts               # 核心循环 (~743 行)
├── types.ts                    # 类型定义 (~419 行)
├── proxy.ts                    # 流代理传输
└── harness/
    ├── agent-harness.ts        # AgentHarness (~800 行)
    ├── types.ts                # Harness 类型
    ├── messages.ts             # 自定义消息类型 + convertToLlm
    ├── system-prompt.ts        # 系统提示构建
    ├── prompt-templates.ts     # 提示模板
    ├── skills.ts               # 技能加载
    ├── compaction/
    │   ├── compaction.ts       # 压缩逻辑
    │   ├── branch-summarization.ts
    │   └── utils.ts
    ├── session/
    │   ├── session.ts          # 会话上下文构建
    │   ├── uuid.ts             # UUID 生成
    │   ├── repo-utils.ts       # 仓库工具
    │   ├── jsonl-repo.ts       # JSONL 持久化
    │   ├── jsonl-storage.ts    # JSONL 存储
    │   ├── memory-repo.ts      # 内存仓库
    │   └── memory-storage.ts   # 内存存储
    ├── env/
    │   └── nodejs.ts           # Node.js 执行环境
    └── utils/
        ├── shell-output.ts
        └── truncate.ts
```

## 核心类型 (types.ts)

### AgentMessage: 可扩展的消息类型

```mermaid
classDiagram
    class AgentMessage {
        <<union type>>
    }
    class Message {
        <<from pi-ai>>
        UserMessage
        AssistantMessage
        ToolResultMessage
    }
    class CustomAgentMessages {
        <<declaration merging>>
        bashExecution: BashExecutionMessage
        custom: CustomMessage
        branchSummary: BranchSummaryMessage
        compactionSummary: CompactionSummaryMessage
    }
    
    AgentMessage <|-- Message: 标准 LLM 消息
    AgentMessage <|-- CustomAgentMessages: 应用自定义消息
```

通过 TypeScript 声明合并，上层应用可以添加新的消息类型：

```typescript
declare module "@earendil-works/pi-agent-core" {
   interface CustomAgentMessages {
      myCustom: MyCustomMessage;
   }
}
```

### AgentTool: 运行时工具定义

```mermaid
classDiagram
    class Tool {
        <<from pi-ai>>
        +name: string
        +description: string
        +parameters: TSchema
    }
    class AgentTool {
        +label: string
        +execute(id, params, signal, onUpdate)
        +prepareArguments?(args)
        +executionMode?: "sequential" | "parallel"
    }
    
    Tool <|-- AgentTool
```

### AgentLoopConfig: 循环配置

```mermaid
graph TB
    subgraph "AgentLoopConfig"
        M["model: 使用的模型"]
        CTL["convertToLlm: 消息转换"]
        TC["transformContext: 上下文变换"]
        GAK["getApiKey: 动态 Key"]
        SST["shouldStopAfterTurn: 停止条件"]
        PNT["prepareNextTurn: 下轮准备"]
        GSM["getSteeringMessages: steering 消息"]
        GFM["getFollowUpMessages: follow-up 消息"]
        TE["toolExecution: 串行/并行"]
        BTC["beforeToolCall: 工具前钩子"]
        ATC["afterToolCall: 工具后钩子"]
    end
```

## Agent 循环 (agent-loop.ts)

这是 Pi 最核心的 743 行代码。循环实现了 LLM 调用 → 工具执行 → 再次调用的自主循环。

### 循环结构

```mermaid
flowchart TD
    START["runLoop()"] --> OUTER_LOOP["外循环: while(true)"]
    
    OUTER_LOOP --> INNER_LOOP["内循环: while(hasMoreToolCalls || pendingMessages)"]
    
    INNER_LOOP --> EMIT_TS["emit(turn_start)"]
    EMIT_TS --> INJECT{"有 pending 消息?"}
    INJECT -->|"是"| ADD["注入到上下文"]
    INJECT -->|"否"| SKIP
    ADD --> STREAM
    SKIP --> STREAM
    
    STREAM["streamAssistantResponse()"]
    STREAM --> CHECK_ERR{"error/aborted?"}
    CHECK_ERR -->|"是"| EMIT_END["emit(agent_end)<br/>return"]
    
    CHECK_ERR -->|"否"| CHECK_TC{"有 tool calls?"}
    CHECK_TC -->|"是"| EXEC["executeToolCalls()"]
    CHECK_TC -->|"否"| TC_FALSE["hasMoreToolCalls = false"]
    EXEC --> EMIT_TE
    TC_FALSE --> EMIT_TE
    
    EMIT_TE["emit(turn_end)"]
    EMIT_TE --> PNT["prepareNextTurn()"]
    PNT --> SHOULD_STOP{"shouldStopAfterTurn?"}
    SHOULD_STOP -->|"是"| EMIT_END2["emit(agent_end)<br/>return"]
    SHOULD_STOP -->|"否"| GET_STEER["getSteeringMessages()"]
    GET_STEER --> INNER_LOOP
    
    INNER_LOOP -->|"退出"| GET_FU["getFollowUpMessages()"]
    GET_FU -->|"有消息"| OUTER_LOOP
    GET_FU -->|"无"| FINAL_END["emit(agent_end)"]
```

### streamAssistantResponse(): LLM 调用

```mermaid
sequenceDiagram
    participant Loop as runLoop
    participant SAR as streamAssistantResponse
    participant TF as transformContext
    participant CTL as convertToLlm
    participant SF as StreamFn
    participant Provider as LLM Provider

    Loop->>SAR: 调用
    SAR->>TF: messages (AgentMessage[])
    TF->>SAR: 裁剪后的 messages
    SAR->>CTL: AgentMessage[] → Message[]
    CTL->>SAR: LLM 兼容的 Message[]
    SAR->>SAR: 构建 LLM Context
    SAR->>SAR: 解析 API Key
    SAR->>SF: stream(model, llmContext, options)
    SF->>Provider: HTTP 请求
    Provider-->>SF: SSE 事件流
    
    loop for await event of response
        SF-->>SAR: AssistantMessageEvent
        SAR->>Loop: emit(message_start/update/end)
    end
    
    SAR->>Loop: return AssistantMessage
```

### executeToolCalls(): 工具执行

工具执行支持两种模式：

```mermaid
graph TB
    TC["toolCalls[]"] --> MODE{"执行模式"}
    
    MODE -->|"sequential"| SEQ["按顺序执行"]
    SEQ --> P1["prepare → execute → finalize"]
    P1 --> P2["prepare → execute → finalize"]
    P2 --> P3["..."]
    
    MODE -->|"parallel"| PAR["并行执行"]
    PAR --> PP1["prepare 1"]
    PAR --> PP2["prepare 2"]
    PAR --> PP3["prepare 3"]
    PP1 --> EX["Promise.all(execute)"]
    PP2 --> EX
    PP3 --> EX
    EX --> FIN["按原始顺序 finalize"]
```

### 工具执行阶段

```mermaid
flowchart LR
    subgraph "Prepare 阶段"
        FIND["查找工具定义"]
        PREP_ARGS["prepareArguments()"]
        VALIDATE["validateToolArguments()"]
        BEFORE["beforeToolCall()"]
    end

    subgraph "Execute 阶段"
        EXEC["tool.execute()"]
        UPDATE["onUpdate() 回调"]
    end

    subgraph "Finalize 阶段"
        AFTER["afterToolCall()"]
        EMIT["emit(tool_execution_end)"]
        MSG["创建 ToolResultMessage"]
    end

    FIND --> PREP_ARGS --> VALIDATE --> BEFORE --> EXEC
    EXEC --> UPDATE
    EXEC --> AFTER --> EMIT --> MSG
```

## Agent 类 (agent.ts)

Agent 是 agentLoop 的**有状态包裹器**：

```mermaid
classDiagram
    class Agent {
        -_messages: AgentMessage[]
        -_tools: AgentTool[]
        -abortController: AbortController
        -steeringQueue: AgentMessage[]
        -followUpQueue: AgentMessage[]
        -subscribers: AgentEventCallback[]
        
        +state: AgentState
        +prompt(messages): Promise
        +continue(): Promise
        +abort(): void
        +subscribe(callback): Unsubscribe
        +steer(messages): void
        +followUp(messages): void
    }
```

### 关键行为

1. **prompt()**: 添加用户消息，启动 agentLoop
2. **continue()**: 不添加新消息，从当前上下文继续（用于重试）
3. **steer()**: 在 Agent 运行时注入消息（steering 队列）
4. **followUp()**: Agent 即将停止时注入消息（follow-up 队列）
5. **abort()**: 通过 AbortController 中断当前运行

## Harness 层

### AgentHarness

AgentHarness 在 Agent 之上添加了：

```mermaid
graph TB
    subgraph "AgentHarness"
        AH["AgentHarness"]
    end

    subgraph "新增能力"
        S["会话持久化<br/>Session 树"]
        C["上下文压缩"]
        H["钩子系统"]
        E["执行环境<br/>FileSystem + Shell"]
        SK["技能 & 提示模板"]
        BS["分支摘要"]
    end

    AH --> S
    AH --> C
    AH --> H
    AH --> E
    AH --> SK
    AH --> BS
```

### 自定义消息与 convertToLlm

`harness/messages.ts` 定义了 4 种自定义消息并声明合并到 `CustomAgentMessages`：

| 消息类型 | 角色 | LLM 转换方式 |
|---------|------|-------------|
| `BashExecutionMessage` | `bashExecution` | → UserMessage (命令 + 输出文本) |
| `CustomMessage` | `custom` | 按 `display` 配置转换 |
| `BranchSummaryMessage` | `branchSummary` | → UserMessage (XML 包裹的摘要) |
| `CompactionSummaryMessage` | `compactionSummary` | → UserMessage (XML 包裹的摘要) |

### 会话树

```mermaid
graph TB
    subgraph "Session 树 (JSONL)"
        ROOT["根节点"]
        M1["message: user"]
        A1["message: assistant"]
        M2["message: user"]
        A2["message: assistant"]
        FORK["分支点"]
        B1["branch_summary"]
        M3A["message: user (分支 A)"]
        M3B["message: user (分支 B)"]
        COMP["compaction"]
        M4["message: user (压缩后)"]
    end

    ROOT --> M1 --> A1 --> M2 --> A2
    A2 --> FORK
    FORK --> B1
    B1 --> M3A
    FORK --> M3B
    A1 --> COMP --> M4
```

### 会话上下文构建

```mermaid
flowchart TD
    TREE["会话树"] --> PATH["提取 root → leaf 路径"]
    PATH --> WALK["遍历路径条目"]
    
    WALK --> CHECK{"遇到 compaction?"}
    CHECK -->|"是"| USE_COMP["使用压缩摘要<br/>+ 之后的条目"]
    CHECK -->|"否"| USE_ALL["收集所有消息"]
    
    WALK --> META["提取 thinkingLevel<br/>提取 model"]
    
    USE_COMP --> BUILD["SessionContext"]
    USE_ALL --> BUILD
    META --> BUILD
    
    BUILD --> SC["{messages, thinkingLevel, model}"]
```

### ExecutionEnv: 文件系统与 Shell 抽象

```mermaid
classDiagram
    class ExecutionEnv {
        <<interface>>
        +fileSystem: FileSystem
        +shell: Shell
    }
    class FileSystem {
        <<interface>>
        +readFile(path): Result~string~
        +writeFile(path, content): Result~void~
        +exists(path): Result~boolean~
        +mkdir(path): Result~void~
        +readdir(path): Result~string[]~
        +stat(path): Result~Stats~
    }
    class Shell {
        <<interface>>
        +exec(cmd, args, opts): Result~ExecResult~
    }
    class NodeExecutionEnv {
        +fileSystem: NodeFileSystem
        +shell: NodeShell
    }
    
    ExecutionEnv <|.. NodeExecutionEnv
```

所有 I/O 返回 `Result<T, Error>`，不抛异常——这是函数式错误处理的体现。

## 流代理 (proxy.ts)

`streamProxy` 将 LLM 调用代理到后端服务器，用于浏览器应用：

```mermaid
sequenceDiagram
    participant Browser as 浏览器
    participant Proxy as streamProxy
    participant Backend as 后端 /api/stream
    participant LLM as LLM Provider

    Browser->>Proxy: streamProxy(model, context)
    Proxy->>Backend: POST /api/stream
    Backend->>LLM: 调用 LLM
    LLM-->>Backend: SSE 流
    Backend-->>Proxy: SSE 流
    Proxy-->>Browser: AssistantMessageEventStream
```
