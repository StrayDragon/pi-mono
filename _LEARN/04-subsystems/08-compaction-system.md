# 上下文压缩系统 (Compaction System)

## 概述

上下文压缩系统是 pi-coding-agent 的核心子系统之一，负责在会话过程中智能压缩历史消息以避免超过 LLM 的上下文窗口限制。系统通过以下机制实现：

- **自动触发检测**：监控会话大小，当超过配置阈值时触发压缩
- **智能切点检测**：找到最佳的消息分割位置，保持对话连贯性
- **LLM 驱动摘要**：使用 LLM 生成历史对话的结构化摘要
- **文件操作追踪**：跨压缩边界追踪文件操作，确保工具上下文不丢失
- **分支摘要**：在会话树导航时压缩分支历史

**核心源文件**：
- `/packages/coding-agent/src/core/compaction/compaction.ts` (840 行) - 核心压缩逻辑
- `/packages/coding-agent/src/core/compaction/branch-summarization.ts` (356 行) - 分支摘要
- `/packages/coding-agent/src/core/compaction/utils.ts` - 工具函数
- `/packages/coding-agent/src/core/compaction/prompts.ts` - 摘要生成提示词
- `/packages/coding-agent/src/core/compaction/types.ts` - 类型定义

**生命周期概览**：

[MermaidChart:./_LEARN/docs/mmd/compaction-system-lifecycle.mmd]

---

## 触发检测 (Trigger Detection)

### 上下文令牌计算

系统使用两级令牌计算策略：

1. **精确计算** (`calculateContextTokens`) - 使用 tiktoken 实际计算
2. **估算计算** (`estimateContextTokens`) - 基于字符数的快速估算

```typescript
// src/core/compaction/compaction.ts:68-96
function calculateContextTokens(messages: Message[]): number {
  let tokens = 0
  for (const m of messages) {
    tokens += getTokenCount(m.role) // role overhead
    tokens += getTokenCount(m.content)
    if (m.reasoning) tokens += getTokenCount(m.reasoning)
    if (m.thought) tokens += getTokenCount(m.thought)
    // ... 其他字段
  }
  return tokens
}

function estimateContextTokens(messages: Message[]): number {
  // 快速估算：1 token ≈ 4 characters
  const chars = messages.reduce((sum, m) => sum + m.content.length, 0)
  return Math.ceil(chars / 4)
}
```

### 阈值检测

每次添加消息后检查是否需要压缩：

```typescript
// src/core/compaction/compaction.ts:98-117
export function shouldCompact(
  currentSize: number,
  threshold: number,
  lastCompactionIndex?: number,
  minDistance?: number
): boolean {
  if (currentSize < threshold) return false
  if (lastCompactionIndex === undefined) return true

  // 确保与上次压缩的最小间距
  const messagesSinceCompaction = session.entries.length - lastCompactionIndex
  return messagesSinceCompaction >= (minDistance ?? 10)
}
```

**配置参数**：
- `threshold` - 触发压缩的令牌数（默认为模型上下文窗口的 80%）
- `minDistance` - 两次压缩间的最小消息数（默认 10）

---

## 压缩准备 (Preparation)

### 查找上次压缩点

```typescript
// src/core/compaction/compaction.ts:119-144
function findPreviousCompaction(entries: Entry[]): Entry | undefined {
  // 从最新往回找最近的 compaction entry
  for (let i = entries.length - 1; i >= 0; i--) {
    if (entries[i].type === "compaction") {
      return entries[i]
    }
  }
  return undefined
}
```

### 边界确定

```typescript
// src/core/compaction/compaction.ts:146-192
function prepareCompactionBounds(
  session: AgentSession,
  options: CompactionOptions
): { start: number; end: number; budget: number } {
  const prevCompaction = findPreviousCompaction(session.entries)
  const startIndex = prevCompaction 
    ? session.entries.indexOf(prevCompaction) + 1 
    : 0

  const totalTokens = calculateContextTokens(
    session.entries.slice(startIndex).map(e => e.message)
  )

  // 目标：压缩到预算的 50%，为未来消息留空间
  const targetBudget = options.budget ?? DEFAULT_TOKEN_BUDGET
  const budget = Math.floor(targetBudget * 0.5)

  return { start: startIndex, end: session.entries.length - 1, budget }
}
```

---

## 切点检测 (Cut Point Detection)

### 反向遍历算法

系统从最新消息往回遍历，累积令牌直到达到预算：

```typescript
// src/core/compaction/compaction.ts:265-323
function findCutPoint(
  entries: Entry[],
  startIndex: number,
  budget: number
): { cutIndex: number; cutPoint: CutPoint } {
  let accumulatedTokens = 0
  let cutIndex = startIndex

  // 从最新往回走
  for (let i = entries.length - 1; i >= startIndex; i--) {
    const entry = entries[i]
    const entryTokens = estimateEntryTokens(entry)

    if (accumulatedTokens + entryTokens > budget) {
      // 找到切点
      break
    }

    accumulatedTokens += entryTokens
    cutIndex = i
  }

  const cutEntry = entries[cutIndex]
  const cutPoint = determineCutPointType(cutEntry)

  return { cutIndex, cutPoint }
}
```

### 切点类型分析

根据消息类型决定如何切分：

```typescript
// src/core/compaction/compaction.ts:194-263
type CutPoint =
  | { type: "user-message" }           // 用户消息 - 干净切分
  | { type: "assistant-message"; turnStartIndex: number; prefix: string; suffix: string }  // 需要分割对话轮次
  | { type: "other" }                  // 其他类型 - 包含非消息条目

function determineCutPointType(entry: Entry): CutPoint {
  if (entry.type === "message" && entry.message.role === "user") {
    return { type: "user-message" }
  }

  if (entry.type === "message" && entry.message.role === "assistant") {
    // 找到这一轮对话的开始
    const turnStartIndex = findTurnStart(entries, entries.indexOf(entry))
    const { prefix, suffix } = splitAssistantMessage(entry.message)

    return {
      type: "assistant-message",
      turnStartIndex,
      prefix,
      suffix
    }
  }

  return { type: "other" }
}
```

**切点策略**：
- **用户消息**：最佳切点，对话在此自然结束
- **助手消息**：需要分割 - 将部分前缀保留在摘要中，后缀保留在活跃上下文
- **其他条目**：通常压缩到最近的用户消息

---

## 摘要生成 (Summarization)

### 提示词策略

系统使用两种提示词模式：

1. **初始摘要** (`initial-summary.txt`) - 第一次压缩
2. **更新摘要** (`update-summary.txt`) - 已有摘要时的增量更新

```typescript
// src/core/compaction/compaction.ts:415-488
async function generateSummary(
  entriesToSummarize: Entry[],
  previousSummary?: string,
  fileOps?: FileOperations
): Promise<string> {
  const promptTemplate = previousSummary
    ? PROMPTS.updateSummary
    : PROMPTS.initialSummary

  const conversation = serializeConversation(entriesToSummarize)
  const fileOpsText = formatFileOperations(fileOps)

  const prompt = promptTemplate
    .replace("{{CONVERSATION}}", conversation)
    .replace("{{PREVIOUS_SUMMARY}}", previousSummary ?? "")
    .replace("{{FILE_OPERATIONS}}", fileOpsText)

  const summary = await callLLM(prompt, { temperature: 0.3 })
  return cleanupSummaryResponse(summary)
}
```

### 提示词结构

**初始摘要模板**：
```
You are analyzing a conversation between a user and an AI coding assistant.
Summarize the following conversation, focusing on:
1. The user's goals and requests
2. Key decisions made
3. Code changes and file operations
4. Current state and next steps

Conversation:
{{CONVERSATION}}

File operations:
{{FILE_OPERATIONS}}

Provide a concise summary (3-5 paragraphs) that captures the essential context.
```

**更新摘要模板**：
```
Previous summary:
{{PREVIOUS_SUMMARY}}

New conversation to integrate:
{{CONVERSATION}}

Additional file operations:
{{FILE_OPERATIONS}}

Update the summary to incorporate the new conversation while maintaining
the essential context from the previous summary.
```

### 分割轮次处理

当切点落在助手消息上时，需要分别处理：

```typescript
// src/core/compaction/compaction.ts:363-413
async function generateTurnPrefixSummary(
  prefixMessages: Message[],
  suffixContent: string
): Promise<string> {
  const prompt = PROMPTS.turnPrefixSummary
    .replace("{{PREFIX_MESSAGES}}", serializeMessages(prefixMessages))
    .replace("{{SUFFIX_CONTENT}}", suffixContent)

  return await callLLM(prompt, { temperature: 0.2 })
}
```

---

## 文件操作追踪 (File Operations Tracking)

### 跨压缩边界追踪

关键挑战：确保工具调用（如 Read、Write）的上下文在压缩后仍然有效。

```typescript
// src/core/compaction/utils.ts:18-67
export function extractFileOpsFromMessage(message: Message): FileOperations {
  const ops: FileOperations = {
    reads: new Set(),
    writes: new Set(),
    edits: new Set()
  }

  if (message.toolCalls) {
    for (const tc of message.toolCalls) {
      switch (tc.function.name) {
        case "read":
        case "read_file":
          ops.reads.add(extractPath(tc))
          break

        case "write":
        case "create":
          ops.writes.add(extractPath(tc))
          break

        case "edit":
          ops.edits.add(extractPath(tc))
          break
      }
    }
  }

  return ops
}
```

### 合并策略

```typescript
// src/core/compaction/compaction.ts:602-631
function mergeFileOperations(
  previous: FileOperations,
  current: FileOperations
): FileOperations {
  return {
    reads: new Set([...previous.reads, ...current.reads]),
    writes: new Set([...previous.writes, ...current.writes]),
    edits: new Set([...previous.edits, ...current.edits])
  }
}
```

### XML 格式化

文件操作以 XML 标签格式插入摘要中：

```typescript
// src/core/compaction/compaction.ts:633-658
function formatFileOperations(ops?: FileOperations): string {
  if (!ops) return ""

  const parts: string[] = []

  if (ops.reads.size > 0) {
    parts.push(`<file_ops type="read" files="${[...ops.reads].join(", ")}" />`)
  }

  if (ops.writes.size > 0) {
    parts.push(`<file_ops type="write" files="${[...ops.writes].join(", ")}" />`)
  }

  if (ops.edits.size > 0) {
    parts.push(`<file_ops type="edit" files="${[...ops.edits].join(", ")}" />`)
  }

  return parts.join("\n")
}
```

---

## 压缩结果创建 (Compaction Result)

### Compaction Entry 结构

```typescript
// src/core/compaction/types.ts:1-32
export interface CompactionEntry extends Entry {
  type: "compaction"
  timestamp: number
  compacted: {
    startIndex: number
    endIndex: number
    entryCount: number
  }
  summary: string
  fileOperations?: {
    reads: string[]
    writes: string[]
    edits: string[]
  }
  turnContext?: {
    prefixSummary?: string
    suffixContent?: string
  }
}
```

### 创建压缩条目

```typescript
// src/core/compaction/compaction.ts:489-600
async function compact(
  session: AgentSession,
  options: CompactionOptions
): Promise<CompactionEntry> {
  // 1. 准备边界
  const { start, end, budget } = prepareCompactionBounds(session, options)

  // 2. 找到切点
  const { cutIndex, cutPoint } = findCutPoint(
    session.entries,
    start,
    budget
  )

  // 3. 提取要摘要的内容
  const entriesToSummarize = session.entries.slice(start, cutIndex)

  // 4. 追踪文件操作
  const fileOps = aggregateFileOperations(entriesToSummarize)

  // 5. 生成摘要
  const previousSummary = findPreviousCompaction(session.entries)?.summary
  const summary = await generateSummary(
    entriesToSummarize,
    previousSummary,
    fileOps
  )

  // 6. 处理分割的轮次
  let turnContext
  if (cutPoint.type === "assistant-message") {
    const prefixSummary = await generateTurnPrefixSummary(
      extractPrefixMessages(entriesToSummarize, cutPoint),
      cutPoint.suffix
    )
    turnContext = {
      prefixSummary,
      suffixContent: cutPoint.suffix
    }
  }

  // 7. 创建压缩条目
  const compactionEntry: CompactionEntry = {
    type: "compaction",
    timestamp: Date.now(),
    compacted: {
      startIndex: start,
      endIndex: cutIndex - 1,
      entryCount: cutIndex - start
    },
    summary,
    fileOperations: {
      reads: [...fileOps.reads],
      writes: [...fileOps.writes],
      edits: [...fileOps.edits]
    },
    turnContext
  }

  return compactionEntry
}
```

### 应用到会话

```typescript
// src/core/agent-session.ts (简化)
async function applyCompaction(entry: CompactionEntry) {
  // 1. 移除被压缩的条目
  const { startIndex, endIndex } = entry.compacted
  this.entries.splice(startIndex, endIndex - startIndex + 1)

  // 2. 插入压缩条目
  this.entries.splice(startIndex, 0, entry)

  // 3. 重新加载会话
  await this.reload()

  // 4. 通知 UI
  this.emit("compacted", { entry })
}
```

---

## 分支摘要 (Branch Summarization)

当用户在会话树中导航时，压缩非活跃分支：

### 收集分支条目

```typescript
// src/core/compaction/branch-summarization.ts:18-72
export function collectEntriesForBranchSummary(
  session: AgentSession,
  currentEntry: Entry
): Entry[] {
  // 1. 找到公共祖先
  const commonAncestor = findCommonAncestor(
    session.rootEntry,
    session.currentEntry
  )

  // 2. 收集分支上的所有条目
  const branchEntries: Entry[] = []
  let entry = currentEntry

  while (entry && entry !== commonAncestor) {
    branchEntries.unshift(entry)
    entry = entry.parent
  }

  return branchEntries
}
```

### 准备分支内容

```typescript
// src/core/compaction/branch-summarization.ts:74-138
function prepareBranchEntries(entries: Entry[]): {
  conversation: string
  fileOps: FileOperations
} {
  const messages: Message[] = []
  const fileOps: FileOperations = {
    reads: new Set(),
    writes: new Set(),
    edits: new Set()
  }

  for (const entry of entries) {
    if (entry.type === "message") {
      messages.push(entry.message)
    }

    // 聚合文件操作
    const entryOps = extractFileOpsFromEntry(entry)
    mergeFileOperations(fileOps, entryOps)
  }

  return {
    conversation: serializeMessages(messages),
    fileOps
  }
}
```

### 生成分支摘要

```typescript
// src/core/compaction/branch-summarization.ts:140-220
async function generateBranchSummary(
  branchEntries: Entry[],
  parentSummary?: string
): Promise<string> {
  const { conversation, fileOps } = prepareBranchEntries(branchEntries)

  const prompt = PROMPTS.branchSummary
    .replace("{{PARENT_SUMMARY}}", parentSummary ?? "No previous context.")
    .replace("{{BRANCH_CONVERSATION}}", conversation)
    .replace("{{FILE_OPERATIONS}}", formatFileOperations(fileOps))

  return await callLLM(prompt, { temperature: 0.3 })
}
```

### 插入分支摘要

```typescript
// src/core/compaction/branch-summarization.ts:222-356
export async function summarizeBranch(
  session: AgentSession,
  branchEntry: Entry
): Promise<void> {
  // 1. 收集分支条目
  const branchEntries = collectEntriesForBranchSummary(session, branchEntry)

  // 2. 生成摘要
  const parentSummary = findParentSummary(session, branchEntry)
  const summary = await generateBranchSummary(branchEntries, parentSummary)

  // 3. 创建分支摘要条目
  const branchSummaryEntry: BranchSummaryEntry = {
    type: "branch-summary",
    timestamp: Date.now(),
    summary,
    branch: {
      from: branchEntries[0].id,
      to: branchEntries[branchEntries.length - 1].id,
      count: branchEntries.length
    }
  }

  // 4. 插入到新位置
  session.insertEntry(branchSummaryEntry)
}
```

---

## 令牌估算 (Token Estimation)

### 令牌计数策略

```typescript
// src/core/compaction/compaction.ts:32-66
const TOKEN_OVERRIDES: Record<string, number> = {
  "gpt-4": 8192,
  "gpt-4-32k": 32768,
  "gpt-4-turbo": 128000,
  "claude-3-opus": 200000,
  "claude-3-sonnet": 200000,
  // ...
}

function getModelTokenBudget(model: string): number {
  // 使用模型定义的上下文窗口
  const override = TOKEN_OVERRIDES[model]
  if (override) return override

  // 默认回退值
  return DEFAULT_TOKEN_BUDGET // 8000
}
```

### 条目令牌估算

```typescript
// src/core/compaction/compaction.ts:241-263
function estimateEntryTokens(entry: Entry): number {
  if (entry.type === "message") {
    return estimateMessageTokens(entry.message)
  }

  if (entry.type === "compaction") {
    // 摘要本身 + 元数据
    return estimateSummaryTokens(entry.summary) + 200
  }

  // 其他条目类型
  return 100 // 基础开销
}

function estimateMessageTokens(message: Message): number {
  let tokens = Math.ceil(message.content.length / 4)

  // 添加各字段开销
  if (message.toolCalls) {
    tokens += message.toolCalls.length * 50
  }

  return tokens
}
```

---

## 最佳实践 (Best Practices)

### 配置建议

1. **阈值设置**：
   ```json
   {
     "compaction": {
       "threshold": 0.8,           // 上下文窗口的 80%
       "minDistance": 10,          // 最小间距
       "targetBudget": 0.5         // 压缩到 50%
     }
   }
   ```

2. **模型选择**：
   - 使用快速的模型生成摘要（如 GPT-3.5-turbo）
   - 低温度设置（0.2-0.3）确保一致性

3. **摘要质量**：
   - 包含用户目标和意图
   - 记录关键决策
   - 列出文件操作
   - 说明当前状态

### 避免陷阱

1. **不要压缩过于频繁**：
   - 设置合理的 `minDistance`
   - 避免在关键任务中途压缩

2. **保留重要上下文**：
   - 确保文件操作被正确追踪
   - 保留"分割轮次"的后缀内容

3. **摘要可读性**：
   - 使用结构化格式
   - 包含时间戳和关键事件
   - 便于未来恢复上下文

### 调试技巧

```typescript
// 启用详细日志
session.on("compaction", (event) => {
  console.log(`Compacted ${event.entry.compacted.entryCount} entries`)
  console.log(`Summary length: ${event.entry.summary.length} chars`)
  console.log(`File ops:`, event.entry.fileOperations)
})

// 检查压缩效果
function analyzeCompaction(session: AgentSession) {
  const compactions = session.entries.filter(e => e.type === "compaction")
  const totalCompacted = compactions.reduce(
    (sum, c) => sum + c.compacted.entryCount,
    0
  )
  console.log(`Total entries compacted: ${totalCompacted}`)
}
```

---

## 相关链接

- **会话系统**：`/LEARN/04-subsystems/03-session-system.md`
- **工具系统**：`/LEARN/04-subsystems/01-tool-system.md`
- **Agent Loop**：`/LEARN/03-packages/02-pi-agent-core.md`

---

## 源文件索引

| 文件 | 行数 | 描述 |
|------|------|------|
| `compaction.ts` | 840 | 核心压缩逻辑 |
| `branch-summarization.ts` | 356 | 分支摘要 |
| `utils.ts` | ~150 | 工具函数 |
| `prompts.ts` | ~100 | 提示词模板 |
| `types.ts` | ~50 | 类型定义 |
