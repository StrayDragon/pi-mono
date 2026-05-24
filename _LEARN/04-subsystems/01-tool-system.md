# 工具系统

Pi 的工具系统位于 `packages/coding-agent/src/core/tools/`，为 LLM 提供可调用的能力：读文件、执行命令、编辑代码、搜索目录等。工具以 **ToolDefinition** 为统一抽象，经 **AgentTool** 包装后接入 `pi-agent-core` 运行时。

## 架构概览

```mermaid
graph TB
    subgraph "定义层"
        TD["ToolDefinition<br/>schema + execute + render"]
        CTD["createXxxToolDefinition()"]
        CT["createXxxTool()"]
        WTD["wrapToolDefinition()"]
    end

    subgraph "注册层"
        BUILTIN["内置工具<br/>read/bash/edit/write/grep/find/ls"]
        EXT["扩展工具<br/>registerTool()"]
        REG["_toolRegistry<br/>AgentSession"]
    end

    subgraph "运行时"
        AGENT["Agent (pi-agent-core)"]
        LLM["LLM 工具调用"]
        TUI["ToolExecutionComponent (TUI)"]
    end

    CTD --> TD
    CT --> WTD --> AGENT
    TD --> WTD
    BUILTIN --> REG
    EXT --> REG
    REG --> AGENT
    AGENT --> LLM
    AGENT --> TUI
```

## 七种内置工具

| 工具 | 用途 | 默认激活 |
|------|------|----------|
| `read` | 读取文件内容（支持行范围、图片） | 是 |
| `bash` | 执行 shell 命令 | 是 |
| `edit` | 对文件做 search/replace 编辑 | 是 |
| `write` | 写入或覆盖文件 | 是 |
| `grep` | 在文件中搜索正则模式 | 否 |
| `find` | 按 glob 查找文件 | 否 |
| `ls` | 列出目录内容 | 否 |

默认激活集由 `createCodingToolDefinitions()` / `createCodingTools()` 定义，仅包含 **read、bash、edit、write** 四个编码工具。只读工具集 `createReadOnlyToolDefinitions()` 包含 read、grep、find、ls。

```mermaid
mindmap
  root((内置工具))
    编码工具
      read
      bash
      edit
      write
    只读工具
      read
      grep
      find
      ls
    默认激活
      read
      bash
      edit
      write
```

## ToolDefinition 接口

扩展与内置工具共享 `ToolDefinition` 接口（定义于 `extensions/types.ts`）：

```typescript
interface ToolDefinition<TParams, TDetails, TState> {
  name: string;           // LLM 调用名
  label: string;          // UI 显示名
  description: string;    // LLM 描述
  parameters: TParams;    // TypeBox schema
  promptSnippet?: string; // 系统提示词中的单行摘要
  promptGuidelines?: string[];
  renderShell?: "default" | "self";
  prepareArguments?: (args: unknown) => Static<TParams>;
  executionMode?: "sequential" | "parallel";
  execute(toolCallId, params, signal, onUpdate, ctx): Promise<AgentToolResult>;
  renderCall?(args, theme, context): Component;
  renderResult?(result, options, theme, context): Component;
}
```

| 成员 | 作用 |
|------|------|
| `parameters` | TypeBox schema，定义 LLM 可见的参数结构 |
| `execute` | 实际执行逻辑，接收 `ExtensionContext` |
| `renderCall` | TUI 中工具调用行的自定义渲染 |
| `renderResult` | TUI 中工具结果行的自定义渲染 |
| `executionMode` | 覆盖默认并行/串行执行策略 |

```mermaid
classDiagram
    class ToolDefinition {
        +string name
        +string label
        +string description
        +TSchema parameters
        +execute()
        +renderCall()
        +renderResult()
    }

    class AgentTool {
        +string name
        +execute()
    }

    class ToolRenderContext {
        +TArgs args
        +string toolCallId
        +string cwd
        +boolean expanded
        +boolean isPartial
    }

    ToolDefinition --> AgentTool : wrapToolDefinition
    ToolDefinition --> ToolRenderContext : renderCall/renderResult
```

## 工具创建模式

每个内置工具遵循 **双工厂函数** 模式：

```mermaid
flowchart LR
    A["createReadToolDefinition(cwd, options)"] --> B["ToolDefinition"]
    C["createReadTool(cwd, options)"] --> D["wrapToolDefinition()"]
    B --> D
    D --> E["AgentTool"]
```

- **`createXxxToolDefinition()`**：返回完整 `ToolDefinition`（含 schema、execute、renderCall、renderResult）
- **`createXxxTool()`**：调用 definition 工厂 + `wrapToolDefinition()`，返回 `AgentTool`

聚合入口在 `tools/index.ts`：

- `createToolDefinition(toolName, cwd)` / `createTool(toolName, cwd)` — 按名称创建单个工具
- `createAllToolDefinitions(cwd)` — 全部 7 个工具的 Record
- `createCodingToolDefinitions(cwd)` — 默认 4 个编码工具

每个工具还支持 **可插拔 Operations**（如 `ReadOperations`、`BashOperations`），便于扩展或测试时替换底层实现（例如 SSH 远程文件系统）。

## FileMutationQueue

`edit` 和 `write` 通过 `withFileMutationQueue()` 序列化对同一文件的变更：

```mermaid
sequenceDiagram
    participant T1 as edit(path=A)
    participant T2 as write(path=A)
    participant Q as FileMutationQueue
    participant FS as 文件系统

    T1->>Q: 注册队列 key=realpath(A)
    T2->>Q: 注册队列 key=realpath(A)
    Q->>T1: 等待 currentQueue
    T1->>FS: 执行编辑
    T1->>Q: releaseNext()
    Q->>T2: currentQueue 完成
    T2->>FS: 执行写入
```

机制要点：

- 按 **realpath** 作为队列 key，不同文件并行、同文件串行
- 全局 `registrationQueue` 保证注册顺序，避免竞态
- 队列链式 Promise：前一个操作完成后才执行下一个

## CLI 工具配置

| 标志 | 缩写 | 效果 |
|------|------|------|
| `--tools read,grep,ls` | `-t` | 显式 allowlist，仅启用列出的工具 |
| `--no-tools` | `-nt` | 禁用所有工具（内置 + 扩展） |
| `--no-builtin-tools` | `-nbt` | 禁用内置工具，保留扩展/自定义工具 |

SDK 层（`sdk.ts`）解析逻辑：

```mermaid
flowchart TD
    START["启动"] --> T{"--tools 指定?"}
    T -->|是| ALLOW["allowedToolNames = 指定列表"]
    T -->|否| NT{"--no-tools?"}
    NT -->|all| EMPTY["allowedToolNames = []"]
    NT -->|builtin| EXTONLY["禁用内置，保留扩展"]
    NT -->|否| DEFAULT["默认 active: read,bash,edit,write"]
    ALLOW --> REG["_refreshToolRegistry()"]
    EMPTY --> REG
    EXTONLY --> REG
    DEFAULT --> REG
```

`AgentSession._refreshToolRegistry()` 合并内置工具、扩展工具和 CLI allowlist，最终通过 `setActiveToolsByName()` 同步到 Agent。

## 扩展工具注册

扩展通过 `ExtensionAPI.registerTool()` 注册自定义工具：

```mermaid
flowchart TB
    subgraph "扩展加载"
        FACTORY["ExtensionFactory(pi)"]
        RT["registerTool(definition)"]
        EXT["Extension.tools Map"]
    end

    subgraph "运行时绑定"
        RUNNER["ExtensionRunner.bindCore()"]
        REFRESH["refreshTools()"]
        WRAP["wrapRegisteredTools()"]
        REGISTRY["_toolRegistry"]
    end

    FACTORY --> RT --> EXT
    EXT --> RUNNER
    RUNNER --> REFRESH --> WRAP --> REGISTRY
```

注册流程：

1. 扩展工厂函数调用 `pi.registerTool(toolDefinition)`
2. 工具存入 `Extension.tools` Map（含 `sourceInfo` 溯源）
3. `refreshTools()` 触发 `AgentSession._refreshToolRegistry()`
4. 扩展工具经 `wrapRegisteredTools()` 包装，与内置工具合并
5. 新注册的扩展工具默认自动加入 active 列表（除非被 allowlist 限制）

扩展工具可覆盖同名内置工具（后加载者优先）。

## 工具生命周期

```mermaid
stateDiagram-v2
    [*] --> Registered: createToolDefinition / registerTool
    Registered --> Active: setActiveToolsByName
    Registered --> Inactive: 不在 active 列表
    Active --> LLMVisible: 注入 Agent.tools
    Inactive --> LLMVisible: 重新激活

    LLMVisible --> ToolCall: LLM 发起调用
    ToolCall --> Validating: schema 校验
    Validating --> Blocked: tool_call 事件 block
    Validating --> Executing: 通过校验
    Executing --> Streaming: onUpdate 回调
    Streaming --> Complete: 返回 AgentToolResult
    Complete --> Rendering: renderResult
    Blocked --> [*]
    Rendering --> [*]
```

## 工具执行流

```mermaid
sequenceDiagram
    participant LLM as LLM
    participant Agent as Agent
    participant ER as ExtensionRunner
    participant Tool as Tool.execute
    participant TUI as ToolExecutionComponent

    LLM->>Agent: tool_call (name, args)
    Agent->>ER: emitToolCall (可 block/修改 args)
    ER-->>Agent: block? / mutated args
    alt blocked
        Agent->>LLM: tool_result (error)
    else allowed
        Agent->>TUI: tool_execution_start
        Agent->>Tool: execute(id, params, signal, onUpdate, ctx)
        loop 流式输出
            Tool->>TUI: onUpdate(partial)
            Agent->>ER: tool_execution_update
        end
        Tool-->>Agent: AgentToolResult
        Agent->>ER: emitToolResult (可修改 content)
        Agent->>TUI: tool_execution_end
        Agent->>LLM: tool_result
    end
```

执行阶段扩展介入点：

| 事件 | 时机 | 能力 |
|------|------|------|
| `tool_call` | 执行前 | `block` 阻止；就地修改 `event.input` |
| `tool_execution_start/update/end` | 执行中/后 | 观察，不可修改 |
| `tool_result` | 返回后 | 修改 `content`、`details`、`isError` |

## 工具注册流

```mermaid
flowchart TD
    subgraph "内置"
        A1["createAllToolDefinitions(cwd)"] --> A2["_baseToolDefinitions"]
    end

    subgraph "扩展"
        B1["loadExtensions(paths)"] --> B2["factory(api)"]
        B2 --> B3["api.registerTool()"]
        B3 --> B4["Extension.tools"]
    end

    subgraph "合并"
        C1["_refreshToolRegistry()"]
        C2["filter by allowedToolNames"]
        C3["wrapRegisteredTools(builtin)"]
        C4["wrapRegisteredTools(extension)"]
        C5["_toolRegistry Map"]
        C6["setActiveToolsByName()"]
    end

    A2 --> C1
    B4 --> C1
    C1 --> C2 --> C3 --> C5
    C1 --> C4 --> C5
    C5 --> C6
    C6 --> D["agent.setTools(active)"]
```

## 关键源文件

| 文件 | 职责 |
|------|------|
| `tools/index.ts` | 工具聚合、工厂函数、类型导出 |
| `tools/tool-definition-wrapper.ts` | ToolDefinition ↔ AgentTool 转换 |
| `tools/file-mutation-queue.ts` | 文件变更串行化 |
| `tools/read.ts` … `ls.ts` | 各内置工具实现 |
| `extensions/types.ts` | ToolDefinition 接口定义 |
| `agent-session.ts` | 工具注册表、active 工具管理 |
| `cli/args.ts` | CLI 工具标志解析 |
