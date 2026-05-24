# 会话系统

Pi 的会话系统以 **SessionManager**（`packages/coding-agent/src/core/session-manager.ts`）为核心，将对话历史存储为 **append-only 树结构** 的 JSONL 文件，支持分支、fork、压缩摘要和 HTML 导出。

## 架构概览

```mermaid
graph TB
    subgraph "存储"
        JSONL["JSONL 文件<br/>~/.pi/agent/sessions/&lt;encoded-cwd&gt;/"]
        HEADER["SessionHeader"]
        ENTRIES["SessionEntry 树"]
    end

    subgraph "SessionManager"
        LEAF["leafId 指针"]
        INDEX["byId Map"]
        BUILD["buildSessionContext()"]
    end

    subgraph "消费方"
        AGENT["Agent LLM 上下文"]
        TUI["TUI 会话树选择器"]
        EXPORT["HTML 导出"]
    end

    JSONL --> HEADER
    JSONL --> ENTRIES
    ENTRIES --> INDEX
    LEAF --> BUILD
    INDEX --> BUILD
    BUILD --> AGENT
    ENTRIES --> TUI
    ENTRIES --> EXPORT
```

## 会话树结构

每个 entry 含 `id` 和 `parentId`，形成有向树（实际为链+分支）：

```mermaid
graph TD
    ROOT["parentId=null<br/>首条 entry"]
    M1["message: user"]
    M2["message: assistant"]
    MC["model_change"]
    M3["message: user"]
    M4["message: assistant"]
    BR["branch_summary"]
    M5["message: user (分支)"]

    ROOT --> M1 --> M2 --> MC --> M3 --> M4
    M2 --> BR --> M5

    style M4 fill:#aaa,stroke-dasharray: 5 5
    style M5 fill:#9f9
```

- **leafId** 指向当前活跃分支的末端 entry
- `appendXxx()` 创建 `parentId = leafId` 的新 entry，然后推进 leaf
- `branch(id)` 将 leaf 移回较早 entry，下次 append 形成新分支
- 历史 entry **永不修改或删除**（append-only）

### Entry 类型

| type | 参与 LLM 上下文 | 说明 |
|------|-----------------|------|
| `message` | 是 | 标准 AgentMessage（user/assistant/toolResult 等） |
| `custom_message` | 是 | 扩展注入的消息，按 customType 渲染 |
| `compaction` | 是（摘要形式） | 上下文压缩摘要 |
| `branch_summary` | 是（摘要形式） | 分支切换时的路径摘要 |
| `model_change` | 否（影响 model 解析） | 记录 provider/modelId |
| `thinking_level_change` | 否（影响 thinking 解析） | 记录思考级别 |
| `custom` | 否 | 扩展持久化状态（不参与上下文） |
| `label` | 否 | 用户书签/标记 |
| `session_info` | 否 | 会话显示名称等元数据 |

```mermaid
classDiagram
    class SessionEntryBase {
        +string type
        +string id
        +string|null parentId
        +string timestamp
    }

    class SessionMessageEntry {
        +AgentMessage message
    }

    class CompactionEntry {
        +string summary
        +string firstKeptEntryId
        +number tokensBefore
    }

    class BranchSummaryEntry {
        +string fromId
        +string summary
    }

    class CustomEntry {
        +string customType
        +unknown data
    }

    SessionEntryBase <|-- SessionMessageEntry
    SessionEntryBase <|-- CompactionEntry
    SessionEntryBase <|-- BranchSummaryEntry
    SessionEntryBase <|-- CustomEntry
```

## JSONL 存储

### 文件位置

```
~/.pi/agent/sessions/<encoded-cwd>/<timestamp>_<sessionId>.jsonl
```

### 目录名编码

`getDefaultSessionDir()` 将 cwd 编码为安全目录名：

```
原始: /home/user/my-project
编码: --home-user-my-project--
路径: ~/.pi/agent/sessions/--home-user-my-project--/
```

规则：去掉 leading `/` 或 `\`，将 `/`、`\`、`:` 替换为 `-`，前后加 `--`。

```mermaid
flowchart LR
    CWD["resolvePath(cwd)"] --> STRIP["去掉 leading 分隔符"]
    STRIP --> REPLACE["/ \\ : → -"]
    REPLACE --> WRAP["--{path}--"]
    WRAP --> JOIN["join(agentDir, sessions, safePath)"]
```

### 文件格式

每行一个 JSON 对象，首行必须是 `SessionHeader`：

```json
{"type":"session","version":3,"id":"...","timestamp":"...","cwd":"/path","parentSession":"..."}
{"type":"message","id":"abc12345","parentId":null,"timestamp":"...","message":{...}}
{"type":"model_change","id":"def67890","parentId":"abc12345",...}
```

- **版本迁移**：v1→v2 添加 id/parentId 树；v2→v3 重命名 hookMessage→custom
- **延迟刷盘**：首条 assistant 消息到达前缓冲，避免空会话文件
- **append-only**：正常运行只追加行；迁移/修复时整文件 rewrite

## 会话操作

```mermaid
stateDiagram-v2
    [*] --> Create: SessionManager.create()
    Create --> Active: newSession()

    Active --> Continue: continueRecent()
    Continue --> Active: 打开最近文件

    Active --> Open: SessionManager.open(path)
    Open --> Active

    Active --> Fork: forkFrom(source, targetCwd)
    Fork --> Active: 新文件 + parentSession 链接

    Active --> Branch: branch(entryId)
    Branch --> Active: leaf 移回

    Active --> Navigate: branchWithSummary()
    Navigate --> Active: 新分支 + 摘要 entry

    Active --> Extract: createBranchedSession(leafId)
    Extract --> Active: 提取单路径为新文件
```

| 操作 | API | 说明 |
|------|-----|------|
| 创建 | `SessionManager.create(cwd)` | 新会话，自动生成文件 |
| 打开 | `SessionManager.open(path)` | 加载已有 JSONL |
| 继续 | `SessionManager.continueRecent(cwd)` | 最近会话或新建 |
| Fork | `SessionManager.forkFrom(source, targetCwd)` | 跨项目复制会话 |
| 分支 | `branch(entryId)` | leaf 移回，下次 append 分叉 |
| 带摘要分支 | `branchWithSummary(id, summary)` | 分支 + branch_summary entry |
| 树导航 | `resetLeaf()` / `branch(null)` | 回到根或指定点 |
| 列表 | `SessionManager.list(cwd)` / `listAll()` | 枚举会话 |

CLI 对应：`--continue`、`-r`/`--resume`、`--session`、`--fork`。

### Fork 流程

```mermaid
sequenceDiagram
    participant User
    participant SM as SessionManager
    participant FS as 文件系统

    User->>SM: forkFrom(sourcePath, targetCwd)
    SM->>FS: 读取 source JSONL
    SM->>FS: 写入新 header (parentSession=source)
    SM->>FS: 复制所有非 header entries
    SM-->>User: 新 SessionManager (targetCwd)
```

## buildSessionContext

从 leaf 沿 `parentId` 向上走到根，构建 LLM 可见上下文：

```mermaid
flowchart TD
    START["leafId"] --> WALK["沿 parentId 向上收集 path"]
    WALK --> SCAN["扫描 path 提取设置"]
    SCAN --> TL["thinkingLevel"]
    SCAN --> MD["model (provider/modelId)"]
    SCAN --> CP["compaction entry"]

    CP -->|有 compaction| COMPACT["1. 插入压缩摘要<br/>2. firstKeptEntryId 起的 kept messages<br/>3. compaction 之后的 messages"]
    CP -->|无 compaction| ALL["emit 全部 message/custom_message/branch_summary"]

    COMPACT --> CTX["SessionContext"]
    ALL --> CTX

    CTX --> OUT["{ messages, thinkingLevel, model }"]
```

### 压缩路径处理

存在 `compaction` entry 时：

1. 先插入 `CompactionSummaryMessage`（含 tokensBefore）
2. 从 `firstKeptEntryId` 到 compaction 之间的 message 保留
3. compaction 之后的 message 正常追加
4. `branch_summary` 转为 `BranchSummaryMessage` 注入上下文

`custom` 和 `label` entry 被忽略（不参与 LLM 上下文）。

### leafId 特殊值

| leafId | 行为 |
|--------|------|
| `undefined` | 使用最后一条 entry |
| 具体 id | 从该 entry 向上构建 |
| `null` | 返回空 messages（导航到首条之前） |

## 会话生命周期

```mermaid
sequenceDiagram
    participant CLI
    participant SM as SessionManager
    participant AS as AgentSession
    participant LLM

    CLI->>SM: create / open / continueRecent
    SM->>SM: 加载/创建 JSONL
    SM->>AS: sessionManager 注入
    AS->>SM: buildSessionContext()
    SM-->>AS: messages + model + thinkingLevel

    loop 对话
        AS->>SM: appendMessage(user)
        AS->>LLM: stream
        AS->>SM: appendMessage(assistant)
        AS->>SM: appendModelChange / appendThinkingLevelChange
    end

    opt 分支
        AS->>SM: branch(entryId)
        AS->>SM: appendMessage (新分支)
    end

    opt 压缩
        AS->>SM: appendCompaction(summary, firstKeptEntryId)
    end
```

## 上下文构建详解

```mermaid
graph TB
    subgraph "path 上的 entry 类型"
        direction LR
        E1["thinking_level_change → 更新 thinkingLevel"]
        E2["model_change → 更新 model"]
        E3["message → push to messages"]
        E4["custom_message → 转 CustomMessage"]
        E5["branch_summary → 转 BranchSummaryMessage"]
        E6["compaction → 触发压缩逻辑"]
    end

    subgraph "SessionContext 输出"
        M["messages: AgentMessage[]"]
        T["thinkingLevel: string"]
        MO["model: {provider, modelId} | null"]
    end

    E1 --> T
    E2 --> MO
    E3 --> M
    E4 --> M
    E5 --> M
    E6 --> M
```

Agent 在每次 LLM 调用前调用 `sessionManager.buildSessionContext()`，再经扩展 `context` 事件修改 messages。

## HTML 导出

位于 `packages/coding-agent/src/core/export-html/`：

```mermaid
flowchart LR
    SM["SessionManager"] --> ENTRIES["getEntries() / getBranch()"]
    ENTRIES --> RENDER["渲染消息 + 工具调用"]
    RENDER --> TEMPLATE["template.html + template.css"]
    TEMPLATE --> OUT["单文件 HTML"]

    subgraph "工具渲染"
        TR["tool-renderer.ts"]
        ANSI["ansi-to-html.ts"]
        HL["highlight.js"]
    end

    RENDER --> TR
    TR --> ANSI
    TR --> HL
```

特性：

- 读取会话 entry，渲染完整对话时间线
- 内置工具 + 扩展工具的 HTML 渲染（`ToolHtmlRenderer` 接口）
- 主题色从 TUI theme 导出（`getThemeExportColors()`）
- CLI：`--export <path>` 或 SDK `exportSession()`

导出文件为自包含 HTML（内联 CSS/JS），可离线浏览。

## 关键 API 速查

| 方法 | 用途 |
|------|------|
| `appendMessage(msg)` | 追加消息 entry |
| `appendCompaction(...)` | 追加压缩 entry |
| `appendBranchSummary(...)` | 追加分支摘要 |
| `appendCustom(type, data)` | 扩展状态持久化 |
| `appendCustomMessage(...)` | 扩展上下文消息 |
| `getBranch(fromId?)` | 获取 root→leaf 路径 |
| `getTree()` | 完整树结构（含 children） |
| `buildSessionContext()` | 构建 LLM 上下文 |
| `createBranchedSession(leafId)` | 提取单路径为新会话 |

## 关键源文件

| 文件 | 职责 |
|------|------|
| `session-manager.ts` | SessionManager 核心实现 |
| `messages.ts` | 摘要/自定义消息工厂 |
| `export-html/index.ts` | HTML 导出入口 |
| `export-html/tool-renderer.ts` | 工具 HTML 渲染 |
| `agent-session.ts` | 会话操作与扩展集成 |
