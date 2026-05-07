# 会话系统深度分析

> 理解 pi-mono 的会话持久化、分支、压缩机制

---

## 1. 会话系统概览

pi-mono 的会话系统是其核心功能之一，提供：
- **JSONL 持久化**：增量式会话存储
- **树形结构**：通过 parentId 构建会话树
- **会话分支**：在任意点创建分支会话
- **上下文压缩**：智能压缩长对话
- **会话导航**：在会话树中任意切换
- **标签系统**：用户定义的书签

### 1.1 架构图

[MermaidChart:./_LEARN/docs/mmd/session-system-flow.mmd]

### 1.2 核心组件

| 组件 | 文件 | 行数 | 职责 |
|------|------|------|------|
| **SessionManager** | `session-manager.ts` | 1425 | 会话文件读写、索引、树遍历 |
| **AgentSession** | `agent-session.ts` | 3100+ | Agent 生命周期、压缩、分支 |
| **Compaction** | `compaction/` | 500+ | 上下文压缩逻辑 |

---

## 2. 会话文件格式

### 2.1 JSONL 结构

**文件位置**：`~/.pi/sessions/<timestamp>_<uuid>.jsonl`

**格式**：
```jsonl
{"type":"session","version":3,"id":"session-123","timestamp":"2026-04-27T10:00:00.000Z","cwd":"/project"}
{"type":"message","id":"msg-1","parentId":null,"timestamp":"2026-04-27T10:00:01.000Z","message":{"role":"user","content":[{"type":"text","text":"Hello"}]}}
{"type":"message","id":"msg-2","parentId":"msg-1","timestamp":"2026-04-27T10:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}]}}
{"type":"compaction","id":"comp-1","parentId":"msg-2","timestamp":"2026-04-27T10:05:00.000Z","summary":"...","firstKeptEntryId":"msg-100","tokensBefore":50000}
```

### 2.2 Entry 类型系统

**代码**：`session-manager.ts:79-193`

```typescript
// 基础条目
interface SessionEntryBase {
    type: string;
    id: string;
    parentId: string | null;
    timestamp: string;
}

// 消息条目
interface SessionMessageEntry extends SessionEntryBase {
    type: "message";
    message: AgentMessage;
}

// 压缩条目
interface CompactionEntry<T = unknown> extends SessionEntryBase {
    type: "compaction";
    summary: string;
    firstKeptEntryId: string;
    tokensBefore: number;
    details?: T;
    fromHook?: boolean;
}

// 分支摘要
interface BranchSummaryEntry<T = unknown> extends SessionEntryBase {
    type: "branch_summary";
    fromId: string;
    summary: string;
    details?: T;
    fromHook?: boolean;
}

// 自定义条目（扩展用，不参与 LLM 上下文）
interface CustomEntry<T = unknown> extends SessionEntryBase {
    type: "custom";
    customType: string;
    data?: T;
}

// 自定义消息（扩展用，参与 LLM 上下文）
interface CustomMessageEntry<T = unknown> extends SessionEntryBase {
    type: "custom_message";
    customType: string;
    content: string | (TextContent | ImageContent)[];
    details?: T;
    display: boolean;
}

// 标签
interface LabelEntry extends SessionEntryBase {
    type: "label";
    targetId: string;
    label: string | undefined;
}

// 会话信息（名称等）
interface SessionInfoEntry extends SessionEntryBase {
    type: "session_info";
    name?: string;
}

// 思考级别变更
interface ThinkingLevelChangeEntry extends SessionEntryBase {
    type: "thinking_level_change";
    thinkingLevel: string;
}

// 模型变更
interface ModelChangeEntry extends SessionEntryBase {
    type: "model_change";
    provider: string;
    modelId: string;
}
```

### 2.3 会话头部

```typescript
interface SessionHeader {
    type: "session";
    version?: number;           // v1 无此字段
    id: string;                 // 会话 ID
    timestamp: string;          // 创建时间
    cwd: string;                // 工作目录
    parentSession?: string;     // 父会话 ID（分支用）
}
```

---

## 3. SessionManager 类

### 3.1 核心状态

**代码**：`session-manager.ts:669-695`

```typescript
export class SessionManager {
    private sessionId: string = "";
    private sessionFile: string | undefined;
    private sessionDir: string;
    private cwd: string;
    private persist: boolean;
    private flushed: boolean = false;

    // 核心数据结构
    private fileEntries: FileEntry[] = [];           // 所有文件条目
    private byId: Map<string, SessionEntry> = new Map();  // ID → Entry
    private labelsById: Map<string, string> = new Map();   // 目标 ID → 标签
    private labelTimestampsById: Map<string, string> = new Map();
    private leafId: string | null = null;            // 当前叶子节点
}
```

### 3.2 初始化

```typescript
// 创建新会话
newSession(options?: NewSessionOptions): string | undefined {
    this.sessionId = options?.id ?? createSessionId();
    const timestamp = new Date().toISOString();

    const header: SessionHeader = {
        type: "session",
        version: CURRENT_SESSION_VERSION,
        id: this.sessionId,
        timestamp,
        cwd: this.cwd,
        parentSession: options?.parentSession,
    };

    this.fileEntries = [header];
    this.byId.clear();
    this.labelsById.clear();
    this.leafId = null;
    this.flushed = false;

    if (this.persist) {
        const fileTimestamp = timestamp.replace(/[:.]/g, "-");
        this.sessionFile = join(this.getSessionDir(), `${fileTimestamp}_${this.sessionId}.jsonl`);
    }

    return this.sessionFile;
}

// 加载现有会话
setSessionFile(sessionFile: string): void {
    this.sessionFile = resolve(sessionFile);

    if (existsSync(this.sessionFile)) {
        this.fileEntries = loadEntriesFromFile(this.sessionFile);

        // 空文件或损坏处理
        if (this.fileEntries.length === 0) {
            const explicitPath = this.sessionFile;
            this.newSession();
            this.sessionFile = explicitPath;
            this._rewriteFile();
            return;
        }

        // 版本迁移
        if (migrateToCurrentVersion(this.fileEntries)) {
            this._rewriteFile();
        }

        this._buildIndex();
    }
}
```

### 3.3 索引构建

**代码**：`session-manager.ts:754-773`

```typescript
private _buildIndex(): void {
    this.byId.clear();
    this.labelsById.clear();
    this.labelTimestampsById.clear();
    this.leafId = null;

    for (const entry of this.fileEntries) {
        if (entry.type === "session") continue;

        // ID 索引
        this.byId.set(entry.id, entry);

        // 叶子节点（最后添加的条目）
        this.leafId = entry.id;

        // 标签索引
        if (entry.type === "label") {
            if (entry.label) {
                this.labelsById.set(entry.targetId, entry.label);
                this.labelTimestampsById.set(entry.targetId, entry.timestamp);
            } else {
                this.labelsById.delete(entry.targetId);
                this.labelTimestampsById.delete(entry.targetId);
            }
        }
    }
}
```

---

## 4. 条目追加与持久化

### 4.1 追加流程

```
Append Entry
     ↓
Add to fileEntries array
     ↓
Add to byId map
     ↓
Update leafId
     ↓
Persist to JSONL
```

### 4.2 持久化逻辑

**代码**：`session-manager.ts:801-828`

```typescript
private _persist(entry: SessionEntry): void {
    if (!this.persist || !this.sessionFile) return;

    // 延迟写入：等到第一个 assistant 消息再写
    const hasAssistant = this.fileEntries.some(
        (e) => e.type === "message" && e.message.role === "assistant"
    );

    if (!hasAssistant) {
        this.flushed = false;
        return;
    }

    // 首次刷新：写入所有条目
    if (!this.flushed) {
        for (const e of this.fileEntries) {
            appendFileSync(this.sessionFile, `${JSON.stringify(e)}\n`);
        }
        this.flushed = true;
    } else {
        // 增量写入
        appendFileSync(this.sessionFile, `${JSON.stringify(entry)}\n`);
    }
}
```

### 4.3 各种追加方法

**消息**：
```typescript
appendMessage(message: AgentMessage): string {
    const entry: SessionMessageEntry = {
        type: "message",
        id: generateId(this.byId),
        parentId: this.leafId,
        timestamp: new Date().toISOString(),
        message
    };
    this._appendEntry(entry);
    return entry.id;
}
```

**自定义条目**：
```typescript
appendCustomEntry<T = unknown>(
    customType: string,
    data?: T
): string {
    const entry: CustomEntry<T> = {
        type: "custom",
        customType,
        data,
        id: generateId(this.byId),
        parentId: this.leafId,
        timestamp: new Date().toISOString()
    };
    this._appendEntry(entry);
    return entry.id;
}
```

**自定义消息**：
```typescript
appendCustomMessageEntry<T = unknown>(
    customType: string,
    content: string | (TextContent | ImageContent)[],
    display: boolean,
    details?: T
): string {
    const entry: CustomMessageEntry<T> = {
        type: "custom_message",
        customType,
        content,
        display,
        details,
        id: generateId(this.byId),
        parentId: this.leafId,
        timestamp: new Date().toISOString()
    };
    this._appendEntry(entry);
    return entry.id;
}
```

---

## 5. 树形结构

### 5.1 树构建

**代码**：`session-manager.ts:1075-1112`

```typescript
getTree(): SessionTreeNode[] {
    const entries = this.getEntries();
    const nodeMap = new Map<string, SessionTreeNode>();
    const roots: SessionTreeNode[] = [];

    // 创建节点
    for (const entry of entries) {
        const label = this.labelsById.get(entry.id);
        const labelTimestamp = this.labelTimestampsById.get(entry.id);
        nodeMap.set(entry.id, {
            entry,
            children: [],
            label,
            labelTimestamp
        });
    }

    // 构建树
    for (const entry of entries) {
        const node = nodeMap.get(entry.id)!;

        if (entry.parentId === null || entry.parentId === entry.id) {
            roots.push(node);
        } else {
            const parent = nodeMap.get(entry.parentId);
            if (parent) {
                parent.children.push(node);
            } else {
                // 孤儿节点 → 视为根
                roots.push(node);
            }
        }
    }

    return roots;
}
```

### 5.2 树遍历

```typescript
getBranch(fromId?: string): SessionEntry[] {
    const path: SessionEntry[] = [];
    const startId = fromId ?? this.leafId;

    let current = startId ? this.byId.get(startId) : undefined;

    while (current) {
        path.unshift(current);
        current = current.parentId ? this.byId.get(current.parentId) : undefined;
    }

    return path;
}
```

### 5.3 树形示例

```
┌─────────────────────────────────────────────────────┐
│ msg-1: User: "Help me refactor"                     │
│ ├─ msg-2: Assistant: "Sure, what file?"            │
│ │   ├─ msg-3: User: "src/app.ts"                   │
│ │   │   └─ msg-4: Assistant: "Here's the diff..."  │
│ │   │       └─ msg-5: User: "Thanks"               │
│ │   │           └─ msg-6: Assistant: "You're welcome" [LEAF]
│ │   │                                                 │
│ │   └─ msg-7: User: "Actually, let's do it differently" [FORK]
│ │       └─ msg-8: Assistant: "OK, how?"             │
│ └─ msg-9: User: "Never mind" [COMPACT POINT]       │
└─────────────────────────────────────────────────────┘
```

---

## 6. 上下文构建

### 6.1 buildSessionContext

**代码**：`session-manager.ts:548-710`

```typescript
export function buildSessionContext(
    entries: SessionEntry[],
    leafId: string | null,
    byId: Map<string, SessionEntry>
): SessionContext {
    const messages: AgentMessage[] = [];
    const visitedIds = new Set<string>();
    const compactionEntries: CompactionEntry[] = [];

    // 从叶子回溯到根
    let currentId: string | null = leafId;
    while (currentId !== null && currentId !== undefined) {
        const entry = byId.get(currentId);
        if (!entry) break;

        // 收集压缩条目
        if (entry.type === "compaction") {
            compactionEntries.push(entry);
            currentId = entry.parentId;
            continue;
        }

        // 收集消息
        const message = getMessageFromEntry(entry);
        if (message && !visitedIds.has(entry.id)) {
            messages.unshift(message);
            visitedIds.add(entry.id);
        }

        currentId = entry.parentId;
    }

    // 反向添加压缩摘要
    const compactionSummaries: AgentMessage[] = [];
    for (let i = compactionEntries.length - 1; i >= 0; i--) {
        const entry = compactionEntries[i];
        const message = createCompactionSummaryMessage(
            entry.summary,
            entry.tokensBefore,
            entry.timestamp
        );
        compactionSummaries.push(message);
    }

    return {
        messages: [...compactionSummaries, ...messages],
        branchSummaries: getBranchSummaries(entries, byId)
    };
}
```

### 6.2 getMessageFromEntry

**代码**：`session-manager.ts:79-93`

```typescript
function getMessageFromEntry(entry: SessionEntry): AgentMessage | undefined {
    if (entry.type === "message") {
        return entry.message;
    }

    if (entry.type === "custom_message") {
        return createCustomMessage(
            entry.customType,
            entry.content,
            entry.display,
            entry.details,
            entry.timestamp
        );
    }

    if (entry.type === "branch_summary") {
        return createBranchSummaryMessage(
            entry.summary,
            entry.fromId,
            entry.timestamp
        );
    }

    if (entry.type === "compaction") {
        return createCompactionSummaryMessage(
            entry.summary,
            entry.tokensBefore,
            entry.timestamp
        );
    }

    return undefined;
}
```

---

## 7. 上下文压缩

### 7.1 压缩触发条件

**代码**：`compaction/compaction.ts:595-613`

```typescript
export function shouldCompact(
    messages: AgentMessage[],
    model: Model<any> | undefined,
    settings: CompactionSettings
): boolean {
    if (!settings.enabled) return false;

    const estimate = estimateContextTokens(messages);
    const contextWindow = model?.input?.maxTokens ?? 200000;

    // 检查是否超出保留空间
    const remainingTokens = contextWindow - settings.reserveTokens;
    const wouldOverflow = estimate.tokens > remainingTokens;

    // 检查是否接近上限
    const isNearLimit = estimate.tokens > settings.keepRecentTokens;

    return wouldOverflow || isNearLimit;
}
```

### 7.2 压缩准备

**代码**：`compaction/compaction.ts:239-302`

```typescript
export async function prepareCompaction(
    entries: SessionEntry[],
    leafId: string | null,
    byId: Map<string, SessionEntry>,
    model: Model<any>,
    options: PrepareCompactionOptions
): Promise<CompactionPreparation> {
    // 1. 收集要压缩的条目
    const toCompact: SessionEntry[] = [];
    const toKeep: SessionEntry[] = [];
    let currentId: string | null = leafId;

    while (currentId !== null) {
        const entry = byId.get(currentId);
        if (!entry) break;

        const estimate = estimateTokens(entry);
        if (accumulatedTokens + estimate <= options.keepRecentTokens) {
            toKeep.unshift(entry);
            accumulatedTokens += estimate;
        } else {
            toCompact.unshift(entry);
        }

        currentId = entry.parentId;
    }

    // 2. 提取文件操作
    const fileOps = extractFileOperations(messages, entries, prevCompactionIndex);

    // 3. 序列化对话
    const conversation = serializeConversation(toCompact, byId);

    // 4. 生成系统提示
    const systemPrompt = SUMMARIZATION_SYSTEM_PROMPT(formatFileOperations(fileOps));

    return {
        toCompact,
        toKeep,
        conversation,
        systemPrompt,
        fileOps
    };
}
```

### 7.3 压缩执行

**代码**：`compaction/compaction.ts:321-401`

```typescript
export async function compact(
    preparation: CompactionPreparation,
    model: Model<any>,
    options: CompactOptions
): Promise<CompactionResult> {
    const { toCompact, toKeep, conversation, systemPrompt, fileOps } = preparation;

    // 1. 调用 LLM 生成摘要
    const summary = await completeSimple(
        model,
        [
            { role: "user", content: systemPrompt },
            { role: "user", content: conversation }
        ],
        {
            apiKey: options.apiKey,
            maxTokens: 4000
        }
    );

    const summaryText = summary.content[0].text;

    // 2. 计算压缩前 token 数
    const tokensBefore = toCompact.reduce((sum, entry) => {
        return sum + estimateTokens(entry);
    }, 0);

    // 3. 确定保留的起始条目
    const firstKeptEntry = toKeep[0];
    const firstKeptEntryId = firstKeptEntry?.id ?? toCompact[toCompact.length - 1].id;

    // 4. 构建详情
    const details: CompactionDetails = {
        readFiles: Array.from(fileOps.read),
        modifiedFiles: Array.from(fileOps.edited)
    };

    return {
        summary: summaryText,
        firstKeptEntryId,
        tokensBefore,
        details
    };
}
```

### 7.4 压缩摘要消息

**代码**：`messages.ts`

```typescript
export function createCompactionSummaryMessage(
    summary: string,
    tokensBefore: number,
    timestamp?: string
): CustomMessage {
    return {
        role: "user",
        customType: "compaction",
        content: `The conversation was compacted to save tokens. Previous summary:\n\n${summary}`,
        display: false,
        details: { tokensBefore },
        timestamp
    };
}
```

---

## 8. 分支摘要

### 8.1 目的

当用户从会话的一个分支切换到另一个分支时，生成"分支摘要"来记录之前分支的内容。

### 8.2 生成逻辑

**代码**：`compaction/branch-summarization.ts`

```typescript
export async function generateBranchSummary(
    entries: SessionEntry[],
    fromId: string,
    byId: Map<string, SessionEntry>,
    model: Model<any>,
    options: GenerateBranchSummaryOptions
): Promise<string> {
    // 1. 收集分支条目
    const branchEntries = collectEntriesForBranchSummary(entries, fromId, byId);

    // 2. 序列化
    const conversation = serializeConversation(branchEntries, byId);

    // 3. 生成摘要
    const summary = await completeSimple(
        model,
        [
            {
                role: "user",
                content: `Summarize this conversation branch:\n\n${conversation}`
            }
        ],
        options
    );

    return summary.content[0].text;
}
```

### 8.3 分支摘要消息

```typescript
export function createBranchSummaryMessage(
    summary: string,
    fromId: string,
    timestamp?: string
): CustomMessage {
    return {
        role: "user",
        customType: "branch_summary",
        content: `Switched from another conversation branch. Previous branch summary:\n\n${summary}`,
        display: false,
        details: { fromId },
        timestamp
    };
}
```

---

## 9. 会话导航

### 9.1 navigateTree

**代码**：`agent-session.ts:2670-2740`

```typescript
async navigateTree(
    targetId: string,
    options?: {
        summarize?: boolean;
        customInstructions?: string;
        replaceInstructions?: boolean;
        label?: string;
    }
): Promise<{ cancelled: boolean }> {
    // 1. 验证目标条目
    const targetEntry = this.sessionManager.getEntry(targetId);
    if (!targetEntry) {
        throw new Error(`Entry ${targetId} not found`);
    }

    // 2. 触发 before_tree 事件
    const beforeResult = await this.extensionRunner.emit({
        type: "session_before_tree",
        targetId,
        targetEntry
    });

    if (beforeResult.cancelled) {
        return { cancelled: true };
    }

    // 3. 生成分支摘要（如果需要）
    let summary: string | undefined;
    if (options?.summarize !== false) {
        summary = await generateBranchSummary(...);
    }

    // 4. 添加分支摘要条目
    if (summary) {
        this.sessionManager.appendBranchSummary(targetId, summary, {
            fromHook: false
        });
    }

    // 5. 更新叶子指针
    this.sessionManager.setLeafId(targetId);

    // 6. 重新构建上下文
    const context = this.sessionManager.buildSessionContext();
    this.updateContext(context);

    return { cancelled: false };
}
```

### 9.2 会话切换

```typescript
async switchSession(
    sessionPath: string,
    options?: {
        withSession?: (ctx: ReplacedSessionContext) => Promise<void>;
    }
): Promise<{ cancelled: boolean }> {
    // 1. 触发 before_switch 事件
    const beforeResult = await this.extensionRunner.emit({
        type: "session_before_switch",
        reason: "resume",
        targetSessionFile: sessionPath
    });

    if (beforeResult.cancelled) {
        return { cancelled: true };
    }

    // 2. 保存当前会话
    await this.saveSession();

    // 3. 加载新会话
    this.sessionManager.setSessionFile(sessionPath);

    // 4. 重新初始化
    const context = this.sessionManager.buildSessionContext();
    this.updateContext(context);

    // 5. 执行 withSession 回调
    if (options?.withSession) {
        await options.withSession(this.createReplacedContext());
    }

    return { cancelled: false };
}
```

---

## 10. 会话分支

### 10.1 fork

```typescript
async fork(
    entryId: string,
    options?: {
        position?: "before" | "at";
        withSession?: (ctx: ReplacedSessionContext) => Promise<void>;
    }
): Promise<{ cancelled: boolean }> {
    // 1. 查找目标条目
    const targetEntry = this.sessionManager.getEntry(entryId);
    if (!targetEntry) {
        throw new Error(`Entry ${entryId} not found`);
    }

    // 2. 创建新会话
    const newSessionFile = this.sessionManager.newSession({
        parentSession: this.sessionManager.getSessionId()
    });

    // 3. 复制条目到分叉点
    const branch = this.sessionManager.getBranch(entryId);
    for (const entry of branch) {
        this.sessionManager.appendEntry(entry);
    }

    // 4. 切换到新会话
    return this.switchSession(newSessionFile!, options);
}
```

### 10.2 分叉点选择

| position | 行为 |
|----------|------|
| `"before"` | 在目标条目之前分叉，保留分支后的内容 |
| `"at"` | 在目标条目处分叉，从该条目继续 |

---

## 11. 标签系统

### 11.1 标签设置

**代码**：`session-manager.ts:1006-1027`

```typescript
appendLabelChange(
    targetId: string,
    label: string | undefined
): string {
    if (!this.byId.has(targetId)) {
        throw new Error(`Entry ${targetId} not found`);
    }

    const entry: LabelEntry = {
        type: "label",
        id: generateId(this.byId),
        parentId: this.leafId,
        timestamp: new Date().toISOString(),
        targetId,
        label
    };

    this._appendEntry(entry);

    // 更新标签索引
    if (label) {
        this.labelsById.set(targetId, label);
        this.labelTimestampsById.set(targetId, entry.timestamp);
    } else {
        this.labelsById.delete(targetId);
        this.labelTimestampsById.delete(targetId);
    }

    return entry.id;
}
```

### 11.2 标签查询

```typescript
getLabel(id: string): string | undefined {
    return this.labelsById.get(id);
}

getLabelTimestamp(id: string): string | undefined {
    return this.labelTimestampsById.get(id);
}
```

---

## 12. 版本迁移

### 12.1 V1 → V2 迁移

**代码**：`session-manager.ts:227-289`

```typescript
function migrateV1ToV2(entries: FileEntry[]): void {
    // V1: 使用 request/response 消息类型
    // V2: 统一为 user/assistant/toolResult

    for (const entry of entries) {
        if (entry.type === "message") {
            const msg = entry.message as any;
            if (msg.role === "request") {
                msg.role = "user";
            } else if (msg.role === "response") {
                msg.role = "assistant";
            }
        }
    }

    // 更新头部版本
    const header = entries.find((e) => e.type === "session") as SessionHeader;
    if (header) {
        header.version = 2;
    }
}
```

### 12.2 V2 → V3 迁移

**代码**：`session-manager.ts:295-340`

```typescript
function migrateV2ToV3(entries: FileEntry[]): void {
    // V3: 添加 parentId 字段

    let previousId: string | null = null;

    for (const entry of entries) {
        if (entry.type === "session") {
            (entry as SessionHeader).version = 3;
            continue;
        }

        if (!("parentId" in entry)) {
            (entry as SessionEntry & { parentId: string | null }).parentId = previousId;
        }

        previousId = entry.id;
    }
}
```

---

## 13. Token 估算

### 13.1 估算算法

**代码**：`compaction/utils.ts:18-108`

```typescript
export function estimateTokens(message: AgentMessage): number {
    // 1. 检查 usage 信息（assistant 消息）
    if (message.role === "assistant" && "usage" in message) {
        const assistantMsg = message as AssistantMessage;
        if (assistantMsg.usage) {
            return calculateContextTokens(assistantMsg.usage);
        }
    }

    // 2. 粗略估算
    let text = "";

    for (const content of message.content) {
        if (content.type === "text") {
            text += content.text;
        } else if (content.type === "toolCall") {
            text += JSON.stringify(content);
        } else if (content.type === "image") {
            // 图片按固定值估算
            text += "X".repeat(1500);
        }
    }

    // 粗略：1 token ≈ 4 字符
    return Math.ceil(text.length / 4);
}
```

### 13.2 上下文使用估算

**代码**：`compaction/compaction.ts:186-221`

```typescript
export function estimateContextTokens(
    messages: AgentMessage[]
): ContextUsageEstimate {
    const usageInfo = getLastAssistantUsageInfo(messages);

    if (!usageInfo) {
        let estimated = 0;
        for (const message of messages) {
            estimated += estimateTokens(message);
        }
        return {
            tokens: estimated,
            usageTokens: 0,
            trailingTokens: estimated,
            lastUsageIndex: null
        };
    }

    const { usage, index } = usageInfo;
    const usageTokens = calculateContextTokens(usage);

    // 估算 usage 之后的消息
    let trailingTokens = 0;
    for (let i = index + 1; i < messages.length; i++) {
        trailingTokens += estimateTokens(messages[i]);
    }

    return {
        tokens: usageTokens + trailingTokens,
        usageTokens,
        trailingTokens,
        lastUsageIndex: index
    };
}
```

---

## 14. 会话列表

### 14.1 列出会话

**代码**：`session-manager.ts:1269-1364`

```typescript
async function listSessionsFromDir(
    sessionDir: string,
    progress?: SessionListProgress
): Promise<SessionInfo[]> {
    const files = await readdir(sessionDir);
    const sessionFiles = files.filter(isValidSessionFile);

    const sessions: SessionInfo[] = [];

    for (let i = 0; i < sessionFiles.length; i++) {
        const filePath = join(sessionDir, sessionFiles[i]);
        const info = await buildSessionInfo(filePath);
        if (info) {
            sessions.push(info);
        }

        progress?.(i + 1, sessionFiles.length);
    }

    // 按修改时间排序
    sessions.sort((a, b) => b.modifiedAt.getTime() - a.modifiedAt.getTime());

    return sessions;
}
```

### 14.2 会话信息

```typescript
interface SessionInfo {
    path: string;
    id: string;
    createdAt: Date;
    modifiedAt: Date;
    name: string | undefined;
    cwd: string;
    parentSession: string | undefined;
    entryCount: number;
    lastActivityAt: Date;
}
```

---

## 15. 最佳实践

### 15.1 扩展使用会话

```typescript
// 保存扩展状态
export default function (api: ExtensionAPI) {
    api.on("session_start", async (event) => {
        // 扫描自定义条目
        const entries = api.sessionManager.getEntries();

        for (const entry of entries) {
            if (entry.type === "custom" && entry.customType === "my-extension") {
                // 恢复状态
                const state = entry.data;
                // ...
            }
        }
    });

    // 持久化状态
    function saveState(state: unknown) {
        api.sessionManager.appendCustomEntry("my-extension", state);
    }
}
```

### 15.2 会话命名

```typescript
// 设置会话名称
api.sessionManager.appendSessionInfo("Feature X implementation");
```

### 15.3 压缩策略

```typescript
// 自定义压缩设置
const settings = {
    enabled: true,
    reserveTokens: 16384,    // 保留空间
    keepRecentTokens: 20000  // 保留最近内容
};

// 手动触发压缩
await api.compact({
    customInstructions: "Focus on code changes and decisions"
});
```

---

## 16. 总结

pi-mono 的会话系统设计：

1. **增量持久化**：JSONL 格式，高性能写入
2. **树形结构**：parentId 构建完整的会话树
3. **灵活导航**：任意切换会话点
4. **智能压缩**：基于 token 使用自动压缩
5. **版本迁移**：平滑升级会话格式
6. **扩展友好**：CustomEntry/CustomMessageEntry 支持扩展状态

这种设计使得 pi-mono 能够处理长对话，同时保持优秀的性能和灵活性。

---

**相关文档**：
- [数据流](../02-architecture/03-data-flow.md)
- [事件系统](../02-architecture/04-event-system.md)
- [pi-coding-agent 包分析](../03-packages/03-pi-coding-agent.md)
