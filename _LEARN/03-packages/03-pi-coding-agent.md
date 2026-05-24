# pi-coding-agent 源码深度分析

> `@earendil-works/pi-coding-agent` — 编码智能体 CLI

## 包概览

pi-coding-agent 是面向用户的顶层包（~152 个源文件），整合了 Agent 核心、LLM API、终端 UI，提供完整的编码智能体体验。

```mermaid
graph TB
    subgraph "入口"
        CLI["cli.ts<br/>启动入口"]
        MAIN["main.ts<br/>编排器"]
    end

    subgraph "运行时"
        ASR["AgentSessionRuntime<br/>会话运行时"]
        ASS["AgentSessionServices<br/>服务捆绑"]
        AS["AgentSession<br/>核心业务逻辑"]
    end

    subgraph "模式"
        INT["InteractiveMode<br/>TUI"]
        PRINT["PrintMode<br/>文本输出"]
        RPC["RpcMode<br/>JSONL 协议"]
    end

    subgraph "核心子系统"
        TOOLS["工具系统"]
        EXT["扩展系统"]
        SESS["会话管理"]
        MODEL["模型注册"]
        AUTH["认证存储"]
    end

    CLI --> MAIN --> ASR --> ASS --> AS
    MAIN --> INT & PRINT & RPC
    AS --> TOOLS & EXT & SESS & MODEL & AUTH
```

## 启动流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as cli.ts
    participant Main as main.ts
    participant Svc as Services
    participant AS as AgentSession
    participant Mode as 运行模式

    User->>CLI: pi [args]
    CLI->>CLI: 设置环境 (HTTP dispatcher, 进程标题)
    CLI->>Main: main()
    Main->>Main: parseArgs()
    Main->>Main: runMigrations()
    Main->>Main: loadSettings()
    Main->>Main: resolveModel()
    
    alt 非交互模式
        Main->>Main: 处理 --help, --version, --list-models 等
    end
    
    Main->>Svc: createAgentSessionServices()
    Note over Svc: AuthStorage, ModelRegistry,<br/>SettingsManager, ResourceLoader
    
    Main->>AS: createAgentSession()
    Note over AS: 加载扩展, 发现技能,<br/>构建系统提示
    
    alt Interactive
        Main->>Mode: new InteractiveMode()
        Mode->>Mode: 创建 TUI + 编辑器
    else Print
        Main->>Mode: runPrintMode()
    else RPC
        Main->>Mode: runRpcMode()
    end
```

## 核心模块

### AgentSessionServices: 服务捆绑

```mermaid
classDiagram
    class AgentSessionServices {
        +authStorage: AuthStorage
        +modelRegistry: ModelRegistry
        +settingsManager: SettingsManager
        +resourceLoader: DefaultResourceLoader
        +cwd: string
    }

    class AuthStorage {
        +getCredentials(provider)
        +setCredentials(provider, creds)
        +refresh(provider)
    }

    class ModelRegistry {
        +getModel(provider, id)
        +getModels()
        +registerProvider(name, config)
        +resolveApiKey(model)
    }

    class SettingsManager {
        +get(key)
        +getTheme()
        +getCompactionSettings()
    }

    class DefaultResourceLoader {
        +loadExtensions()
        +loadSkills()
        +loadPromptTemplates()
        +loadThemes()
        +loadContextFiles()
    }

    AgentSessionServices --> AuthStorage
    AgentSessionServices --> ModelRegistry
    AgentSessionServices --> SettingsManager
    AgentSessionServices --> DefaultResourceLoader
```

### AgentSession: 核心业务逻辑

AgentSession 是 pi-coding-agent 最核心的类，桥接 Agent 运行时与应用层：

```mermaid
graph TB
    subgraph "AgentSession 职责"
        P["处理用户输入"]
        A["管理 Agent 生命周期"]
        T["注册和管理工具"]
        E["运行扩展事件"]
        S["持久化到会话"]
        C["上下文压缩"]
        M["模型切换"]
        SP["系统提示构建"]
    end

    subgraph "关联组件"
        AGENT["Agent (pi-agent-core)"]
        TOOLS["工具定义"]
        EXTR["ExtensionRunner"]
        SESSMGR["SessionManager"]
        SYSPR["系统提示构建器"]
    end

    P --> AGENT
    A --> AGENT
    T --> TOOLS
    E --> EXTR
    S --> SESSMGR
    SP --> SYSPR
```

### 系统提示构建

系统提示是一个精心构建的多段文本：

```mermaid
flowchart TD
    A["基础人格指令"] --> B["工具使用指南"]
    B --> C["编辑工具最佳实践"]
    C --> D["可用工具列表"]
    D --> E["技能列表"]
    E --> F["上下文文件<br/>(AGENTS.md / CLAUDE.md)"]
    F --> G["自定义指令"]
    G --> H["扩展指令"]
    H --> I["最终系统提示"]
```

## 工具系统详解

### 工具注册与双表示

```mermaid
graph TB
    subgraph "ToolDefinition (扩展级)"
        TD["name, label, description"]
        TP["parameters: TSchema"]
        TE["execute(id, params, signal, onUpdate, ctx)"]
        TR["renderCall() / renderResult()"]
    end

    subgraph "AgentTool (运行时级)"
        AT["name, label, description"]
        ATP["parameters: TSchema"]
        ATE["execute(id, params, signal, onUpdate)"]
    end

    subgraph "转换"
        W["ToolDefinitionWrapper"]
    end

    TD -->|"wraps"| W -->|"produces"| AT
```

### 文件操作队列

`edit` 和 `write` 工具通过 `FileMutationQueue` 序列化，避免并发写入冲突：

```mermaid
sequenceDiagram
    participant LLM as LLM
    participant Loop as AgentLoop
    participant Queue as FileMutationQueue
    participant FS as 文件系统

    LLM->>Loop: toolCall: edit file1
    LLM->>Loop: toolCall: edit file2
    LLM->>Loop: toolCall: write file3
    
    par 并行执行
        Loop->>Queue: edit file1 (获取锁)
        Queue->>FS: 写入 file1
        FS->>Queue: 完成
        Queue->>Loop: 释放锁
    and
        Loop->>Queue: edit file2 (获取锁)
        Queue->>FS: 写入 file2
        FS->>Queue: 完成
        Queue->>Loop: 释放锁
    and
        Loop->>Queue: write file3 (获取锁)
        Queue->>FS: 写入 file3
        FS->>Queue: 完成
        Queue->>Loop: 释放锁
    end
```

## 三种运行模式

```mermaid
graph LR
    subgraph "Interactive 模式"
        TUI["TUI 应用"]
        CHAT["聊天容器"]
        EDITOR["多行编辑器"]
        FOOTER["状态栏"]
        OVERLAY["叠加层 (选择器)"]
    end

    subgraph "Print 模式"
        STDIN["stdin"]
        PROMPT["--prompt 参数"]
        STDOUT["stdout 输出"]
    end

    subgraph "RPC 模式"
        JSONL_IN["JSONL stdin"]
        JSONL_OUT["JSONL stdout"]
        PROTO["命令协议"]
    end
```

### Interactive 模式 UI 布局

```
┌──────────────────────────────────┐
│ headerContainer                  │ ← 快捷键提示
├──────────────────────────────────┤
│                                  │
│ chatContainer                    │ ← 消息 + 工具输出
│   ├ UserMessageComponent         │
│   ├ AssistantMessageComponent    │
│   │   └ ToolExecutionComponent   │
│   ├ CompactionSummaryComponent   │
│   └ ...                          │
│                                  │
├──────────────────────────────────┤
│ pendingMessagesContainer         │ ← 队列中的消息
├──────────────────────────────────┤
│ statusContainer                  │ ← 加载动画
├──────────────────────────────────┤
│ widgetContainerAbove             │ ← 扩展 widget
├──────────────────────────────────┤
│ editorContainer                  │ ← 输入编辑器
├──────────────────────────────────┤
│ widgetContainerBelow             │ ← 扩展 widget
├──────────────────────────────────┤
│ footer                           │ ← 模型/Token/思考
└──────────────────────────────────┘
```

## 扩展系统

### 扩展加载流程

```mermaid
flowchart TD
    DISCOVER["发现扩展文件"]
    DISCOVER -->|".pi/extensions/"| LOCAL["项目扩展"]
    DISCOVER -->|"~/.pi/agent/extensions/"| GLOBAL["全局扩展"]
    DISCOVER -->|"settings.extensions[]"| CONFIG["配置扩展"]
    DISCOVER -->|"--extensions"| CLI_EXT["CLI 扩展"]
    
    LOCAL & GLOBAL & CONFIG & CLI_EXT --> LOAD["jiti 加载 .ts"]
    LOAD --> EXEC["执行 ExtensionFactory"]
    EXEC -->|"pi.on()"| HANDLERS["注册事件处理器"]
    EXEC -->|"pi.registerTool()"| REG_TOOLS["注册工具"]
    EXEC -->|"pi.registerCommand()"| REG_CMDS["注册命令"]
    EXEC -->|"pi.registerProvider()"| REG_PROV["注册供应商"]
    EXEC -->|"pi.registerShortcut()"| REG_SC["注册快捷键"]
```

### ExtensionRunner: 事件分发中心

```mermaid
graph TB
    subgraph "ExtensionRunner"
        DISPATCH["dispatch(event)"]
        HANDLERS["handlers: Map<event, fn[]>"]
        TOOLS["tools: Map<name, RegisteredTool>"]
        CMDS["commands: Map<name, RegisteredCommand>"]
        RENDERERS["messageRenderers: Map<type, Renderer>"]
        FLAGS["flags: Map<name, ExtensionFlag>"]
        SHORTCUTS["shortcuts: Map<key, ExtensionShortcut>"]
    end

    subgraph "连接点"
        CTX["ExtensionContext<br/>(每次事件创建)"]
        UI["ExtensionUIContext<br/>(模式提供)"]
        ACTIONS["ExtensionActions<br/>(AgentSession 提供)"]
    end

    DISPATCH --> HANDLERS
    CTX --> UI
    CTX --> ACTIONS
```

## 配置系统

```mermaid
graph TB
    subgraph "配置层次 (低 → 高优先级)"
        DEFAULT["内置默认值"]
        GLOBAL["~/.pi/agent/settings.json"]
        PROJECT[".pi/settings.json"]
        CLI_ARG["CLI 参数"]
        ENV["环境变量"]
    end

    DEFAULT --> GLOBAL --> PROJECT --> CLI_ARG --> ENV

    subgraph "配置内容"
        THEME["主题"]
        MODELS["模型列表"]
        COMPACT["压缩设置"]
        TOOLS_CFG["工具配置"]
        EXT_CFG["扩展配置"]
        KB["快捷键"]
    end
```

## SDK 使用

pi-coding-agent 也提供编程 API：

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";

const session = await createAgentSession({
   model: getModel("anthropic", "claude-sonnet-4-20250514"),
   apiKey: process.env.ANTHROPIC_API_KEY,
   cwd: process.cwd(),
});

session.subscribe((event) => {
   // 处理事件
});

await session.prompt("修复这个 bug");
```
