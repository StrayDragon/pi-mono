# 调试技巧 (Debugging)

## 概述

本文档提供 pi-mono 项目的全面调试指南，涵盖调试工具、技巧、常见问题和解决方案。

---

## 调试模式

### 启用调试日志

```bash
# 方式 1：环境变量
PI_DEBUG=1 pi

# 方式 2：配置文件
# ~/.pi/config.json
{
  "debug": true,
  "logLevel": "debug"
}

# 方式 3：命令行参数
pi --debug
pi -d
```

### 日志级别

```typescript
// packages/coding-agent/src/utils/logger.ts
enum LogLevel {
  DEBUG = 0,    // 详细调试信息
  INFO = 1,     // 一般信息
  WARN = 2,     // 警告
  ERROR = 3,    // 错误
  SILENT = 4    // 静默
}

// 设置日志级别
export PI_LOG_LEVEL=debug
export PI_LOG_LEVEL=info
export PI_LOG_LEVEL=warn
export PI_LOG_LEVEL=error
```

### 查看日志

```bash
# 日志位置
~/.pi/logs/
├── pi.log              # 主日志
├── error.log           # 错误日志
└── debug.log           # 调试日志

# 实时查看日志
tail -f ~/.pi/logs/pi.log

# 查看最近 100 行
tail -n 100 ~/.pi/logs/pi.log

# 搜索特定错误
grep "ERROR" ~/.pi/logs/pi.log
grep "Tool.*failed" ~/.pi/logs/pi.log
```

---

## VS Code 调试

### 配置调试器

**`.vscode/launch.json`**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug pi-coding-agent",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "start"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to pi",
      "port": 9229,
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Current Test",
      "program": "${workspaceFolder}/node_modules/.bin/vitest",
      "args": ["run", "${relativeFile}"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```

### 启动调试会话

```bash
# 方式 1：使用 inspect 标志
node --inspect-brk $(which pi)

# 方式 2：从源码调试
node --inspect-brk node_modules/.bin/ts-node packages/coding-agent/src/index.ts

# 方式 3：使用构建后的代码
npm run build
node --inspect-brk packages/coding-agent/dist/index.js
```

### 断点调试

```typescript
// packages/coding-agent/src/core/agent-session.ts

// 方式 1：使用 debugger 语句
async processMessage(message: string): Promise<void> {
  debugger;  // 程序会在此处暂停

  const response = await this.agent.chat(message)
  // ...
}

// 方式 2：条件断点（在 VS Code 中右键设置）
async processMessage(message: string): Promise<void> {
  const response = await this.agent.chat(message)

  // 在 VS Code 中设置条件：message.length > 100
  console.log("Processing long message")

  // ...
}

// 方式 3：日志断点
async processMessage(message: string): Promise<void> {
  // 在 VS Code 中设置日志断点：
  // console.log('Message:', message)
  // 不会停止执行，只记录日志

  const response = await this.agent.chat(message)
  // ...
}
```

---

## 调试工具

### 1. Chrome DevTools

```bash
# 启动并打开 Chrome DevTools
node --inspect $(which pi)

# 然后在 Chrome 中打开
chrome://inspect
```

### 2. ndb (改进的调试器)

```bash
# 安装 ndb
npm install -g ndb

# 使用 ndb 调试
ndb npm run start

# 或
ndb pi
```

### 3. debug 模块

```typescript
// packages/coding-agent/src/utils/debug.ts
import debug from "debug"

export const log = debug("pi:agent")
export const toolLog = debug("pi:tool")
export const llmLog = debug("pi:llm")

// 使用
log("Agent session started")
toolLog("Executing tool: %s", toolName)
llmLog("LLM response: %O", response)
```

```bash
# 启用特定命名空间的调试
DEBUG=pi:* pi
DEBUG=pi:agent pi
DEBUG=pi:tool,pi:llm pi
```

---

## 调试特定问题

### 调试工具执行

```typescript
// packages/coding-agent/src/core/tools/executor.ts

// 添加工具执行日志
async executeTool(toolName: string, args: unknown): Promise<ToolResult> {
  console.log(`[Tool Executor] Executing: ${toolName}`)
  console.log(`[Tool Executor] Arguments:`, JSON.stringify(args, null, 2))

  const startTime = Date.now()

  try {
    const tool = this.registry.get(toolName)

    if (!tool) {
      console.error(`[Tool Executor] Tool not found: ${toolName}`)
      return { success: false, error: `Tool not found: ${toolName}` }
    }

    console.log(`[Tool Executor] Tool description: ${tool.description}`)

    const result = await tool.execute!(args, this.context)

    const duration = Date.now() - startTime
    console.log(`[Tool Executor] Execution time: ${duration}ms`)
    console.log(`[Tool Executor] Result:`, JSON.stringify(result, null, 2))

    return result
  } catch (error) {
    const duration = Date.now() - startTime
    console.error(`[Tool Executor] Error after ${duration}ms:`, error)
    throw error
  }
}
```

### 调试 LLM 调用

```typescript
// packages/ai/src/stream.ts

export async function streamLLM(
  provider: LLMProvider,
  messages: Message[],
  options: ChatOptions
): Promise<void> {
  console.log("[LLM] === Request ===")
  console.log("[LLM] Provider:", provider.name)
  console.log("[LLM] Model:", options.model)
  console.log("[LLM] Messages:", messages.length)
  messages.forEach((msg, i) => {
    console.log(`[LLM] [${i}] ${msg.role}: ${msg.content.slice(0, 100)}...`)
  })
  console.log("[LLM] Options:", JSON.stringify(options, null, 2))

  const startTime = Date.now()

  try {
    let fullResponse = ""
    let chunkCount = 0

    for await (const chunk of provider.chat(messages, options)) {
      chunkCount++
      fullResponse += chunk.content

      if (chunkCount % 10 === 0) {
        console.log(`[LLM] Chunks received: ${chunkCount}, Length: ${fullResponse.length}`)
      }

      if (chunk.toolCalls) {
        console.log("[LLM] Tool calls:", JSON.stringify(chunk.toolCalls, null, 2))
      }

      if (chunk.done) {
        const duration = Date.now() - startTime
        console.log("[LLM] === Complete ===")
        console.log(`[LLM] Total chunks: ${chunkCount}`)
        console.log(`[LLM] Total length: ${fullResponse.length}`)
        console.log(`[LLM] Duration: ${duration}ms`)
        console.log(`[LLM] Tokens/sec: ${fullResponse.length / (duration / 1000)}`)

        if (chunk.usage) {
          console.log("[LLM] Usage:", chunk.usage)
        }
      }
    }
  } catch (error) {
    const duration = Date.now() - startTime
    console.error(`[LLM] Error after ${duration}ms:`, error)
    throw error
  }
}
```

### 调试扩展加载

```typescript
// packages/coding-agent/src/core/extensions/loader.ts

export async function loadExtensions(
  extensionsDir: string
): Promise<Extension[]> {
  console.log(`[Extension Loader] Scanning: ${extensionsDir}`)

  const extensions: Extension[] = []

  try {
    const files = await fs.readdir(extensionsDir)
    console.log(`[Extension Loader] Found ${files.length} items`)

    for (const file of files) {
      const fullPath = join(extensionsDir, file)
      const stat = await fs.stat(fullPath)

      if (stat.isDirectory()) {
        console.log(`[Extension Loader] Loading extension: ${file}`)

        try {
          const ext = await loadExtension(fullPath)
          extensions.push(ext)

          console.log(`[Extension Loader] ✓ Loaded: ${ext.id} v${ext.version}`)
          console.log(`[Extension Loader]   - ${ext.tools?.length || 0} tools`)
          console.log(`[Extension Loader]   - ${ext.skills?.length || 0} skills`)
        } catch (error) {
          console.error(`[Extension Loader] ✗ Failed: ${file}`)
          console.error(`[Extension Loader]   Error:`, error)
        }
      }
    }

    console.log(`[Extension Loader] Total loaded: ${extensions.length}`)

    return extensions
  } catch (error) {
    console.error(`[Extension Loader] Failed to scan directory:`, error)
    return []
  }
}
```

### 调试会话管理

```typescript
// packages/coding-agent/src/core/session-manager.ts

export class SessionManager {
  private sessions: Map<string, Session> = new Map()

  // 添加会话状态日志
  logState(): void {
    console.log("[Session Manager] === State ===")
    console.log(`[Session Manager] Total sessions: ${this.sessions.size}`)

    for (const [id, session] of this.sessions) {
      console.log(`[Session Manager] Session: ${id}`)
      console.log(`[Session Manager]   - Messages: ${session.messages.length}`)
      console.log(`[Session Manager]   - Branches: ${session.branches.length}`)
      console.log(`[Session Manager]   - Active: ${session.active}`)
      console.log(`[Session Manager]   - Created: ${session.createdAt}`)
    }
  }

  // 添加会话树可视化
  visualizeTree(): void {
    console.log("[Session Manager] === Session Tree ===")

    const root = this.sessions.get("root")

    if (!root) {
      console.log("[Session Manager] No root session")
      return
    }

    const printBranch = (session: Session, indent: number = 0) => {
      const prefix = "  ".repeat(indent)
      const marker = session.active ? "→" : " "
      console.log(`${prefix}${marker} ${session.id} (${session.messages.length} msgs)`)

      for (const branchId of session.branches) {
        const branch = this.sessions.get(branchId)
        if (branch) {
          printBranch(branch, indent + 1)
        }
      }
    }

    printBranch(root)
  }
}
```

---

## 性能分析

### CPU 性能分析

```bash
# 方式 1：使用 Node.js 内置分析器
node --prof pi
# 运行一些操作后按 Ctrl+C 退出
node --prof-process isolate-*.log > profile.txt

# 方式 2：使用 clinic.js
npm install -g clinic
clinic doctor -- pi

# 方式 3：使用 0x
npm install -g 0x
0x pi
```

### 内存分析

```typescript
// packages/coding-agent/src/utils/memory.ts

export class MemoryMonitor {
  private interval?: NodeJS.Timeout

  start(intervalMs: number = 5000): void {
    this.interval = setInterval(() => {
      const usage = process.memoryUsage()

      console.log("[Memory Monitor] === Usage ===")
      console.log(`[Memory Monitor] RSS: ${this.formatBytes(usage.rss)}`)
      console.log(`[Memory Monitor] Heap Total: ${this.formatBytes(usage.heapTotal)}`)
      console.log(`[Memory Monitor] Heap Used: ${this.formatBytes(usage.heapUsed)}`)
      console.log(`[Memory Monitor] External: ${this.formatBytes(usage.external)}`)

      const heapUsedPercent = (usage.heapUsed / usage.heapTotal) * 100
      console.log(`[Memory Monitor] Heap Usage: ${heapUsedPercent.toFixed(2)}%`)

      if (heapUsedPercent > 90) {
        console.warn("[Memory Monitor] ⚠️  High heap usage!")
      }
    }, intervalMs)
  }

  stop(): void {
    if (this.interval) {
      clearInterval(this.interval)
      this.interval = undefined
    }
  }

  private formatBytes(bytes: number): string {
    const units = ["B", "KB", "MB", "GB"]
    const size = Math.floor(Math.log(bytes) / Math.log(1024))
    return `${(bytes / Math.pow(1024, size)).toFixed(2)} ${units[size]}`
  }

  // 生成堆快照
  async generateSnapshot(filename: string): Promise<void> {
    const { writeHeapSnapshot } = await import("v8")
    const snapshotFile = writeHeapSnapshot(filename)
    console.log(`[Memory Monitor] Heap snapshot: ${snapshotFile}`)
  }
}
```

### 工具执行时间分析

```typescript
// packages/coding-agent/src/utils/performance.ts

export class PerformanceMonitor {
  private measurements = new Map<string, number[]>()

  startMeasure(label: string): () => void {
    const start = performance.now()

    return () => {
      const duration = performance.now() - start
      this.record(label, duration)
    }
  }

  record(label: string, duration: number): void {
    if (!this.measurements.has(label)) {
      this.measurements.set(label, [])
    }

    this.measurements.get(label)!.push(duration)
  }

  report(): void {
    console.log("[Performance] === Report ===")

    for (const [label, durations] of this.measurements.entries()) {
      const total = durations.reduce((a, b) => a + b, 0)
      const avg = total / durations.length
      const min = Math.min(...durations)
      const max = Math.max(...durations)
      const count = durations.length

      console.log(`[Performance] ${label}:`)
      console.log(`[Performance]   Count: ${count}`)
      console.log(`[Performance]   Avg: ${avg.toFixed(2)}ms`)
      console.log(`[Performance]   Min: ${min.toFixed(2)}ms`)
      console.log(`[Performance]   Max: ${max.toFixed(2)}ms`)
      console.log(`[Performance]   Total: ${total.toFixed(2)}ms`)
    }
  }

  // 装饰器用法
  static measure(target: unknown, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value
    const monitor = new PerformanceMonitor()

    descriptor.value = async function (...args: unknown[]) {
      const end = monitor.startMeasure(`${propertyKey}`)

      try {
        return await originalMethod.apply(this, args)
      } finally {
        end()
      }
    }

    return descriptor
  }
}

// 使用装饰器
class ToolExecutor {
  @PerformanceMonitor.measure
  async executeTool(name: string, args: unknown): Promise<ToolResult> {
    // ... 实现
  }
}
```

---

## 常见问题排查

### 问题 1：工具未找到

**症状**：
```
Error: Tool not found: my-tool
```

**排查步骤**：
```bash
# 1. 检查工具是否注册
pi --list-tools

# 2. 检查扩展是否加载
PI_DEBUG=1 pi | grep "Extension"

# 3. 检查工具定义
cat ~/.pi/extensions/my-ext/index.ts | grep "name:"

# 4. 验证工具语法
node -c ~/.pi/extensions/my-ext/index.ts
```

**解决方案**：
```typescript
// 确保工具正确导出
export default defineExtension({
  id: "my-extension",
  tools: [myTool]  // ← 确保工具在数组中
})
```

### 问题 2：LLM API 错误

**症状**：
```
Error: API request failed: 401 Unauthorized
```

**排查步骤**：
```bash
# 1. 验证 API Key
cat ~/.pi/config.json | grep apiKey

# 2. 测试 API 连接
curl -H "Authorization: Bearer $PI_API_KEY" \
  https://api.openai.com/v1/models

# 3. 检查 Provider 配置
pi --config

# 4. 启用详细日志
DEBUG=pi:llm PI_DEBUG=1 pi
```

**解决方案**：
```json
// ~/.pi/config.json
{
  "provider": "openai",
  "apiKey": "sk-...",  // 验证密钥有效性
  "baseURL": "https://api.openai.com/v1",  // 检查 URL
  "timeout": 30000  // 增加超时时间
}
```

### 问题 3：内存泄漏

**症状**：
```
JavaScript heap out of memory
```

**排查步骤**：
```bash
# 1. 监控内存使用
watch -n 1 'ps aux | grep "node.*pi"'

# 2. 生成堆快照
kill -USR2 <pid>

# 3. 使用 clinic.js
clinic heapprofiler -- on -- pi
```

**常见原因和解决**：
```typescript
// 1. 事件监听器未清理
class AgentSession {
  private disposables: Array<() => void> = []

  setupEventListeners() {
    const handler = () => { /* ... */ }
    this.events.on("message", handler)

    // 记录清理函数
    this.disposables.push(() => {
      this.events.off("message", handler)
    })
  }

  dispose() {
    // 清理所有监听器
    this.disposables.forEach(fn => fn())
    this.disposables = []
  }
}

// 2. 定时器未清除
class ToolExecutor {
  private timers: NodeJS.Timeout[] = []

  scheduleTask(fn: Function, delay: number) {
    const timer = setTimeout(fn, delay)
    this.timers.push(timer)
  }

  dispose() {
    this.timers.forEach(clearTimeout)
    this.timers = []
  }
}

// 3. 大对象缓存
class CacheManager {
  private cache = new Map<string, unknown>()
  private maxSize = 1000

  set(key: string, value: unknown): void {
    // LRU 淘汰
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }

    this.cache.set(key, value)
  }
}
```

### 问题 4：TUI 渲染异常

**症状**：
```
Terminal display corrupted, colors wrong, or flickering
```

**排查步骤**：
```bash
# 1. 检查终端能力
echo $TERM
tic -a

# 2. 测试颜色支持
for i in {0..255}; do print -Pn "%K{$i}  %k%F{$i}${(l:3::0:)i}%f " ${${(M)$((i%6)):#3}:+$'\n'}; done

# 3. 强制 256 色模式
PI_COLOR_MODE=256 pi

# 4. 禁用 TUI
PI_NO_TUI=1 pi
```

**解决方案**：
```json
// ~/.pi/config.json
{
  "theme": {
    "colorMode": "256",  // 或 "truecolor"
    "overrides": {
      "background": "black"
    }
  }
}
```

### 问题 5：上下文压缩失败

**症状**：
```
Error: Context compaction failed
```

**排查步骤**：
```bash
# 1. 检查压缩配置
cat ~/.pi/config.json | jq .compaction

# 2. 查看压缩日志
PI_DEBUG=1 pi 2>&1 | grep -i compaction

# 3. 测试 LLM 压缩能力
curl -H "Authorization: Bearer $PI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Summarize: Hello"}],"max_tokens":10}' \
  https://api.openai.com/v1/chat/completions
```

**解决方案**：
```json
// ~/.pi/config.json
{
  "compaction": {
    "enabled": true,
    "threshold": 0.8,           // 提高阈值
    "minDistance": 15,          // 增加最小距离
    "maxTokens": 8000,          // 降低最大 token 数
    "provider": "openai",       // 确保使用支持的 provider
    "model": "gpt-4o"           // 使用支持摘要的模型
  }
}
```

---

## 调试最佳实践

### 1. 结构化日志

```typescript
// 使用结构化日志格式
interface LogEntry {
  timestamp: string
  level: "info" | "warn" | "error" | "debug"
  component: string
  message: string
  data?: Record<string, unknown>
  error?: Error
}

function log(entry: LogEntry): void {
  const output = JSON.stringify(entry)
  console.log(output)

  // 可选：写入文件
  fs.appendFileSync("~/.pi/logs/structured.log", output + "\n")
}

// 使用
log({
  timestamp: new Date().toISOString(),
  level: "info",
  component: "ToolExecutor",
  message: "Tool executed successfully",
  data: {
    tool: "read",
    duration: 45,
    fileSize: 1024
  }
})
```

### 2. 错误边界

```typescript
// packages/coding-agent/src/utils/error-boundary.ts

export class ErrorBoundary {
  static async run<T>(
    fn: () => Promise<T>,
    context: {
      operation: string
      onError?: (error: Error) => void
      fallback?: T
    }
  ): Promise<T> {
    try {
      return await fn()
    } catch (error) {
      const err = error instanceof Error ? error : new Error(String(error))

      console.error(`[ErrorBoundary] ${context.operation}:`, err)

      // 记录错误堆栈
      console.error(`[ErrorBoundary] Stack:`, err.stack)

      // 调用自定义错误处理
      if (context.onError) {
        context.onError(err)
      }

      // 返回 fallback 或抛出
      if (context.fallback !== undefined) {
        return context.fallback
      }

      throw err
    }
  }
}

// 使用
const result = await ErrorBoundary.run(
  () => executeTool(toolName, args),
  {
    operation: `Execute tool: ${toolName}`,
    onError: (error) => {
      // 发送错误报告
      reportError(error)
    },
    fallback: { success: false, error: "Tool execution failed" }
  }
)
```

### 3. 调试友好的错误消息

```typescript
// ❌ 差的错误消息
throw new Error("Failed")

// ✅ 好的错误消息
throw new Error(
  `Tool execution failed: ${toolName}\n` +
  `Args: ${JSON.stringify(args)}\n` +
  `Reason: ${error.message}\n` +
  `Hint: Check if the tool is properly configured`
)

// 使用 Error 子类
class ToolExecutionError extends Error {
  constructor(
    public toolName: string,
    public args: unknown,
    cause: Error
  ) {
    super(
      `Tool execution failed: ${toolName}\n` +
      `Args: ${JSON.stringify(args)}\n` +
      `Reason: ${cause.message}`
    )

    this.name = "ToolExecutionError"
    this.cause = cause
  }
}

throw new ToolExecutionError(toolName, args, error)
```

### 4. 调试模式切换

```typescript
// packages/coding-agent/src/config/debug.ts

export class DebugConfig {
  private static instance: DebugConfig

  private flags = {
    tools: false,
    llm: false,
    extensions: false,
    sessions: false,
    performance: false
  }

  static getInstance(): DebugConfig {
    if (!this.instance) {
      this.instance = new DebugConfig()
    }

    // 从环境变量读取
    const debugFlags = process.env.PI_DEBUG?.split(",") || []
    debugFlags.forEach(flag => {
      this.instance.flags[flag.trim()] = true
    })

    return this.instance
  }

  isEnabled(flag: keyof typeof DebugConfig.prototype.flags): boolean {
    return this.flags[flag]
  }

  enable(flag: keyof typeof DebugConfig.prototype.flags): void {
    this.flags[flag] = true
  }

  disable(flag: keyof typeof DebugConfig.prototype.flags): void {
    this.flags[flag] = false
  }
}

// 使用条件日志
const debug = DebugConfig.getInstance()

if (debug.isEnabled("tools")) {
  console.log("[Tool Debug] Executing:", toolName)
}
```

---

## 总结

✅ 你已经学会：
- 启用和使用调试模式
- 配置和使用 VS Code 调试器
- 调试工具执行、LLM 调用、扩展加载、会话管理
- 性能分析和内存分析
- 排查常见问题
- 编写调试友好的代码

**调试清单**：
- ✓ 启用调试日志
- ✓ 使用断点调试
- ✓ 记录性能指标
- ✓ 监控内存使用
- ✓ 验证错误处理
- ✓ 检查日志文件

**下一步**：
- [终端用户指南](../07-user-guide/01-end-user-guide.md)
- [团队集成](../07-user-guide/02-team-integration.md)
- [高级配置](../07-user-guide/03-advanced-configuration.md)

---

## 相关链接

- **快速上手**：`/LEARN/06-onboarding/01-quick-start.md`
- **代码库导航**：`/LEARN/06-onboarding/02-codebase-navigation.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
