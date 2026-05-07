# 工具系统深度分析

> 理解 pi-mono 的可扩展工具执行框架

---

## 1. 工具系统概览

pi-mono 的工具系统是一个高度可扩展的框架，支持：
- **7 个内置工具**：read, bash, edit, write, grep, find, ls
- **TypeBox 参数验证**：完整的类型安全
- **自定义渲染**：工具调用和结果的 TUI 渲染
- **执行钩子**：beforeToolCall, afterToolCall 事件拦截
- **并行/串行执行**：灵活的工具执行模式
- **流式更新**：onUpdate 回调实现进度报告

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Loop                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │               Tool Execution Pipeline                      │  │
│  │  prepare → validate → before hook → execute → after hook  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                   Built-in Tools (7)                            │
│  read | bash | edit | write | grep | find | ls                 │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Extension Tools                                │
│  Extensions can register additional tools                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. ToolDefinition 接口

### 2.1 完整接口定义

**文件**：`/packages/coding-agent/src/core/extensions/types.ts:424-471`

```typescript
export interface ToolDefinition<TParams extends TSchema = any, TDetails = unknown, TState = any> {
    // === 基础信息 ===
    name: string;                              // 工具名称（LLM 调用使用）
    label: string;                             // 人类可读标签
    description: string;                       // LLM 描述

    // === 系统提示集成 ===
    promptSnippet?: string;                    // 单行代码片段，显示在"可用工具"部分
    promptGuidelines?: string[];               // 指南要点，附加到默认系统提示

    // === 参数定义 ===
    parameters: TParams;                       // TypeBox 参数 schema

    // === 兼容性层 ===
    prepareArguments?: (args: unknown) => Static<TParams>;  // 预处理参数

    // === 执行模式 ===
    executionMode?: ToolExecutionMode;         // "sequential" | "parallel"

    // === 渲染控制 ===
    renderShell?: "default" | "self";         // 渲染标准 shell 或自定义

    // === 核心执行函数 ===
    execute(
        toolCallId: string,
        params: Static<TParams>,
        signal: AbortSignal | undefined,
        onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
        ctx: ExtensionContext
    ): Promise<AgentToolResult<TDetails>>;

    // === 自定义渲染 ===
    renderCall?: (
        args: Static<TParams>,
        theme: Theme,
        context: ToolRenderContext<TState, Static<TParams>>
    ) => Component;

    renderResult?: (
        result: AgentToolResult<TDetails>,
        options: ToolRenderResultOptions,
        theme: Theme,
        context: ToolRenderContext<TState, Static<TParams>>
    ) => Component;
}
```

### 2.2 类型参数说明

| 参数 | 用途 | 示例 |
|------|------|------|
| `TParams` | TypeBox 参数 schema | `Type.Object({ path: Type.String() })` |
| `TDetails` | 工具详情数据类型 | `{ truncation?: TruncationResult }` |
| `TState` | 渲染状态类型 | `{ offset: number; lines: string[] }` |

---

## 3. 7 个内置工具

### 3.1 工具清单

| 工具 | 文件 | 主要功能 | 模式 |
|------|------|---------|------|
| **read** | `read.ts` (273 行) | 读取文件内容 | parallel |
| **bash** | `bash.ts` (448 行) | 执行 shell 命令 | parallel |
| **edit** | `edit.ts` | 编辑文件（基于 diff） | sequential |
| **write** | `write.ts` | 写入文件 | sequential |
| **grep** | `grep.ts` | 搜索文本 | parallel |
| **find** | `find.ts` | 查找文件 | parallel |
| **ls** | `ls.ts` | 列出目录 | parallel |

### 3.2 read 工具详解

**参数 Schema**：
```typescript
const readSchema = Type.Object({
    path: Type.String({ description: "文件路径（相对或绝对）" }),
    offset: Type.Optional(Type.Number({ description: "起始行号（1-indexed）" })),
    limit: Type.Optional(Type.Number({ description: "最大读取行数" })),
});
```

**核心功能**：
- 读取文本文件
- 支持 offset/limit 分页
- 自动语法高亮（基于文件扩展名）
- 图片文件支持（自动转换为 ImageContent）
- 自动调整图片大小（2000x2000 max）

**Details 类型**：
```typescript
interface ReadToolDetails {
    truncation?: TruncationResult;    // 截断信息
}
```

**关键代码** (`read.ts:173-273`):
```typescript
async function executeRead(
    input: ReadToolInput,
    toolCallId: string,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<ReadToolDetails> | undefined,
    operations: ReadOperations,
    options?: ReadToolOptions
): Promise<AgentToolResult<ReadToolDetails>> {
    // 1. 解析路径
    const absolutePath = resolveReadPath(cwd, input.path);

    // 2. 检查可读性
    await operations.access(absolutePath);

    // 3. 读取文件
    const buffer = await operations.readFile(absolutePath);

    // 4. 检测图片
    const mimeType = await operations.detectImageMimeType?.(absolutePath);
    if (mimeType) {
        // 处理图片...
    }

    // 5. 文本截断
    const { content, details } = truncateText(buffer, input.offset, input.limit);

    return { content: [{ type: "text", text: content }], details };
}
```

### 3.3 bash 工具详解

**参数 Schema**：
```typescript
const bashSchema = Type.Object({
    command: Type.String({ description: "要执行的 Bash 命令" }),
    timeout: Type.Optional(Type.Number({ description: "超时时间（秒）" })),
});
```

**核心功能**：
- 执行任意 shell 命令
- 支持超时控制
- 输出截断（默认 100KB/1000 行）
- 后台进程支持（detached 模式）
- 进程树跟踪和清理

**Details 类型**：
```typescript
interface BashToolDetails {
    truncation?: TruncationResult;
    fullPath?: string;                  // 完整输出文件路径（detached 模式）
}
```

**BashOperations 可插拔接口**：
```typescript
interface BashOperations {
    spawn: (command: string, cwd: string) => ChildProcess;
    kill: (pid: number) => void;
    readFile: (path: string) => Promise<Buffer>;
    // 支持远程执行（如 SSH）
}
```

**关键代码** (`bash.ts:200-300`):
```typescript
async function executeBash(
    input: BashToolInput,
    toolCallId: string,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<BashToolDetails> | undefined,
    operations: BashOperations
): Promise<AgentToolResult<BashToolDetails>> {
    // 1. 解析 spawn context（shell 配置、环境变量）
    const context = resolveSpawnContext(input.command, cwd, spawnHook);

    // 2. Spawn 进程
    const child = operations.spawn(input.command, cwd);

    // 3. 流式收集输出
    for await (const chunk of child.stdout) {
        output += chunk.toString();
        // 流式更新
        onUpdate?.({ output: formatOutput(output) });
    }

    // 4. 截断处理
    const { content, details } = truncateTail(output, maxBytes, maxLines);

    return { content: [{ type: "text", text: content }], details };
}
```

### 3.4 edit 工具详解

**参数 Schema**：
```typescript
const editSchema = Type.Object({
    path: Type.String({ description: "文件路径" }),
    text: Type.String({ description: "要替换的文本" }),
    replace: Type.String({ description: "替换后的文本" }),
    occurrences: Type.Optional(Type.Union([
        Type.Integer({ minimum: 1 }),
        Type.Literal("all")
    ])),
});
```

**核心功能**：
- 基于文本匹配的编辑
- 支持单次或全部替换
- 原子性操作（FileMutationQueue）
- 清晰的 diff 输出

**FileMutationQueue** (`file-mutation-queue.ts`):
```typescript
export function withFileMutationQueue<T>(
    filePath: string,
    callback: (queue: FileMutationQueue) => Promise<T>
): Promise<T> {
    // 原子性文件修改队列
    // 确保多个 edit 操作不冲突
}
```

### 3.5 write 工具详解

**参数 Schema**：
```typescript
const writeSchema = Type.Object({
    path: Type.String({ description: "文件路径" }),
    content: Type.String({ description: "文件内容" }),
});
```

**核心功能**：
- 写入新文件或覆盖现有文件
- 自动创建目录
- 显示写入的行数和字节数

### 3.6 grep 工具详解

**参数 Schema**：
```typescript
const grepSchema = Type.Object({
    pattern: Type.String({ description: "搜索模式（正则表达式）" }),
    path: Type.Optional(Type.String({ description: "搜索路径" })),
    include: Type.Optional(Type.String({ description: "文件名模式" })),
    exclude: Type.Optional(Type.String({ description: "排除模式" })),
    contextLines: Type.Optional(Type.Number({ description: "上下文行数" })),
});
```

**核心功能**：
- 正则表达式搜索
- 支持 include/exclude 文件过滤
- 上下文行显示
- 结果高亮

### 3.7 find 工具详解

**参数 Schema**：
```typescript
const findSchema = Type.Object({
    pattern: Type.String({ description: "文件名模式（支持 * 通配符）" }),
    path: Type.Optional(Type.String({ description: "搜索路径" })),
});
```

**核心功能**：
- 文件名模式匹配
- 递归搜索
- 结果按路径排序

### 3.8 ls 工具详解

**参数 Schema**：
```typescript
const lsSchema = Type.Object({
    path: Type.Optional(Type.String({ description: "目录路径" })),
});
```

**核心功能**：
- 列出目录内容
- 区分文件和目录
- 显示文件大小

---

## 4. 工具执行生命周期

### 4.1 完整流程图

[MermaidChart:./_LEARN/docs/mmd/tool-system-lifecycle.mmd]

### 4.2 代码路径

**入口**：`/packages/agent/src/agent-loop.ts:338-471`

```typescript
async function executeToolCalls(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink
): Promise<ExecutedToolCallBatch> {
    const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");

    // 检查是否有串行工具
    const hasSequentialToolCall = toolCalls.some(
        (tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential"
    );

    // 选择执行模式
    if (config.toolExecution === "sequential" || hasSequentialToolCall) {
        return executeToolCallsSequential(...);
    }
    return executeToolCallsParallel(...);
}
```

### 4.3 执行阶段

#### 阶段 1: 准备 (prepareToolCall)

**代码**：`/packages/agent/src/agent-loop.ts:517-567`

```typescript
async function prepareToolCall(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCall: AgentToolCall,
    config: AgentLoopConfig,
    signal: AbortSignal | undefined
): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
    // 1. 查找工具
    const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
    if (!tool) {
        return {
            kind: "immediate",
            result: createErrorToolResult(`Tool ${toolCall.name} not found`),
            isError: true
        };
    }

    // 2. 参数预处理
    const preparedToolCall = prepareToolCallArguments(tool, toolCall);

    // 3. TypeBox 验证
    const validatedArgs = validateToolArguments(tool, preparedToolCall);

    // 4. beforeToolCall hook
    if (config.beforeToolCall) {
        const beforeResult = await config.beforeToolCall({
            assistantMessage,
            toolCall,
            args: validatedArgs,
            context: currentContext
        }, signal);

        if (beforeResult?.block) {
            return {
                kind: "immediate",
                result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
                isError: true
            };
        }
    }

    return {
        kind: "prepared",
        toolCall,
        tool,
        args: validatedArgs
    };
}
```

#### 阶段 2: 执行 (executePreparedToolCall)

**代码**：`/packages/agent/src/agent-loop.ts:569-604`

```typescript
async function executePreparedToolCall(
    prepared: PreparedToolCall,
    signal: AbortSignal | undefined,
    emit: AgentEventSink
): Promise<ExecutedToolCallOutcome> {
    const updateEvents: Promise<void>[] = [];

    try {
        const result = await prepared.tool.execute(
            prepared.toolCall.id,
            prepared.args as never,
            signal,
            (partialResult) => {
                // onUpdate 回调 - 流式更新
                updateEvents.push(
                    Promise.resolve(
                        emit({
                            type: "tool_execution_update",
                            toolCallId: prepared.toolCall.id,
                            toolName: prepared.toolCall.name,
                            args: prepared.toolCall.arguments,
                            partialResult
                        })
                    )
                );
            }
        );
        await Promise.all(updateEvents);
        return { result, isError: false };
    } catch (error) {
        await Promise.all(updateEvents);
        return {
            result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
            isError: true
        };
    }
}
```

#### 阶段 3: 完成处理 (finalizeExecutedToolCall)

**代码**：`/packages/agent/src/agent-loop.ts:606-649`

```typescript
async function finalizeExecutedToolCall(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    prepared: PreparedToolCall,
    executed: ExecutedToolCallOutcome,
    config: AgentLoopConfig,
    signal: AbortSignal | undefined
): Promise<FinalizedToolCallOutcome> {
    let result = executed.result;
    let isError = executed.isError;

    // afterToolCall hook
    if (config.afterToolCall) {
        try {
            const afterResult = await config.afterToolCall({
                assistantMessage,
                toolCall: prepared.toolCall,
                args: prepared.args,
                result,
                isError,
                context: currentContext
            }, signal);

            if (afterResult) {
                result = {
                    content: afterResult.content ?? result.content,
                    details: afterResult.details ?? result.details,
                    terminate: afterResult.terminate ?? result.terminate
                };
                isError = afterResult.isError ?? isError;
            }
        } catch (error) {
            result = createErrorToolResult(error instanceof Error ? error.message : String(error));
            isError = true;
        }
    }

    return {
        toolCall: prepared.toolCall,
        result,
        isError
    };
}
```

---

## 5. 参数验证系统

### 5.1 TypeBox Schema

pi-mono 使用 **TypeBox** 进行运行时类型验证：

```typescript
import { Type, Static } from "typebox";

// 定义 schema
const readSchema = Type.Object({
    path: Type.String({ description: "文件路径" }),
    offset: Type.Optional(Type.Number({ description: "起始行号" })),
    limit: Type.Optional(Type.Number({ description: "最大行数" })),
});

// 提取静态类型
export type ReadToolInput = Static<typeof readSchema>;
// 等价于：
// type ReadToolInput = {
//     path: string;
//     offset?: number;
//     limit?: number;
// }
```

### 5.2 验证流程

**代码**：`/packages/ai/src/validation.ts`

```typescript
import { Value } from "@sinclair/typebox/value";

export function validateToolArguments(
    tool: AgentTool<any>,
    toolCall: AgentToolCall
): unknown {
    try {
        // TypeBox 验证
        const errors = Value.Errors(tool.parameters, toolCall.arguments);

        if (errors.length > 0) {
            // 收集所有错误
            const errorMessages = [...errors].map((error) => {
                return `${error.path}: ${error.message}`;
            });

            throw new Error(
                `Invalid arguments for tool '${tool.name}':\n${errorMessages.join("\n")}`
            );
        }

        return toolCall.arguments;
    } catch (error) {
        // 验证失败
        throw new Error(`Tool validation failed: ${error.message}`);
    }
}
```

### 5.3 prepareArguments 钩子

用于兼容性参数转换：

```typescript
export interface ToolDefinition<TParams, TDetails, TState> {
    prepareArguments?: (args: unknown) => Static<TParams>;
}

// 示例：处理别名参数
const readTool: ToolDefinition<typeof readSchema> = {
    name: "read",
    parameters: readSchema,

    // 兼容 file_path 别名
    prepareArguments: (args) => {
        if (args && typeof args === "object" && "file_path" in args) {
            return { path: (args as any).file_path, offset: args.offset, limit: args.limit };
        }
        return args as Static<typeof readSchema>;
    },

    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
        // params 已经是验证后的类型
        const { path, offset, limit } = params;
        // ...
    }
};
```

---

## 6. 执行钩子系统

### 6.1 beforeToolCall 钩子

**目的**：在工具执行前拦截或修改参数

**事件定义** (`extensions/types.ts:763-810`):
```typescript
interface ToolCallEvent {
    type: "tool_call";
    assistantMessage: AssistantMessage;
    toolCall: AgentToolCall;
    args: unknown;
    context: AgentContext;
    block: () => void;              // 调用此函数阻止执行
    setResult: (result: AgentToolResult) => void;  // 设置自定义结果
}
```

**使用示例**：
```typescript
// 扩展中拦截敏感命令
api.on("tool_call", async (event) => {
    if (event.toolCall.name === "bash") {
        const command = (event.args as BashToolInput).command;

        // 阻止危险命令
        if (command.includes("rm -rf /")) {
            event.block();
            event.setResult({
                content: [{ type: "text", text: "Dangerous command blocked!" }],
                details: {}
            });
        }
    }
});
```

### 6.2 afterToolCall 钩子

**目的**：修改工具结果或添加副作用

**事件定义** (`extensions/types.ts:824-875`):
```typescript
interface ToolResultEvent {
    type: "tool_result";
    assistantMessage: AssistantMessage;
    toolCall: AgentToolCall;
    args: unknown;
    result: AgentToolResult;
    isError: boolean;
    context: AgentContext;
    setResult: (result: AgentToolResult) => void;
    setIsError: (isError: boolean) => void;
}
```

**使用示例**：
```typescript
// 扩展中修改 read 结果
api.on("tool_result", async (event) => {
    if (event.toolCall.name === "read") {
        // 添加注释
        const originalText = event.result.content[0].text;
        event.setResult({
            content: [{
                type: "text",
                text: `// File read by pi-coding-agent\n${originalText}`
            }],
            details: event.result.details
        });
    }
});
```

---

## 7. 工具包装机制

### 7.1 ToolDefinition → AgentTool

**代码**：`/packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:5-19`

```typescript
export function wrapToolDefinition<TDetails = unknown>(
    definition: ToolDefinition<any, TDetails>,
    ctxFactory?: () => ExtensionContext
): AgentTool<any, TDetails> {
    return {
        name: definition.name,
        label: definition.label,
        description: definition.description,
        parameters: definition.parameters,
        prepareArguments: definition.prepareArguments,
        executionMode: definition.executionMode,
        execute: (toolCallId, params, signal, onUpdate) =>
            definition.execute(
                toolCallId,
                params,
                signal,
                onUpdate,
                ctxFactory?.() as ExtensionContext
            )
    };
}
```

### 7.2 AgentTool → ToolDefinition

**代码**：`tool-definition-wrapper.ts:35-46`

```typescript
export function createToolDefinitionFromAgentTool(
    tool: AgentTool<any>
): ToolDefinition<any, unknown> {
    return {
        name: tool.name,
        label: tool.label,
        description: tool.description,
        parameters: tool.parameters as any,
        prepareArguments: tool.prepareArguments,
        executionMode: tool.executionMode,
        execute: async (toolCallId, params, signal, onUpdate) =>
            tool.execute(toolCallId, params, signal, onUpdate)
    };
}
```

### 7.3 包装时机

```
扩展定义 ToolDefinition
         ↓
    wrapToolDefinition()
         ↓
AgentSession 内部使用 AgentTool
         ↓
Agent Loop 调用 tool.execute()
         ↓
执行扩展的 definition.execute()
```

---

## 8. 自定义工具渲染

### 8.1 renderCall

**用途**：自定义工具调用显示

**签名**：
```typescript
renderCall?: (
    args: Static<TParams>,
    theme: Theme,
    context: ToolRenderContext<TState, Static<TParams>>
) => Component;
```

**示例**：
```typescript
const myTool: ToolDefinition = {
    name: "my_tool",
    label: "My Tool",
    description: "A custom tool",
    parameters: mySchema,

    renderCall: (args, theme, context) => {
        return new Container()
            .add(new Text(theme.fg("toolTitle", "my_tool")))
            .add(new Text(`Executing with param: ${args.param}`));
    },

    execute: async (toolCallId, params, signal, onUpdate, ctx) => {
        // ...
    }
};
```

### 8.2 renderResult

**用途**：自定义工具结果显示

**签名**：
```typescript
renderResult?: (
    result: AgentToolResult<TDetails>,
    options: ToolRenderResultOptions,
    theme: Theme,
    context: ToolRenderContext<TState, Static<TParams>>
) => Component;
```

**示例**：
```typescript
const myTool: ToolDefinition = {
    // ...
    renderResult: (result, options, theme, context) => {
        const text = result.content[0].text;
        return new Container()
            .add(new Text(theme.fg("success", text)))
            .add(new Text(options.expanded ? "(expanded)" : "(collapsed)"));
    }
};
```

### 8.3 ToolRenderContext

**包含**：
- `args`: 工具调用参数（与 renderCall 共享）
- `toolCallId`: 本次执行唯一 ID
- `invalidate`: 使当前工具执行组件无效

---

## 9. 工具注册与创建

### 9.1 工具工厂函数

**代码**：`/packages/coding-agent/src/core/tools/index.ts:96-196`

```typescript
// 创建单个工具定义
export function createToolDefinition(
    toolName: ToolName,
    cwd: string,
    options?: ToolsOptions
): ToolDef {
    switch (toolName) {
        case "read":
            return createReadToolDefinition(cwd, options?.read);
        case "bash":
            return createBashToolDefinition(cwd, options?.bash);
        // ...
    }
}

// 创建所有工具定义
export function createAllToolDefinitions(
    cwd: string,
    options?: ToolsOptions
): Record<ToolName, ToolDef> {
    return {
        read: createReadToolDefinition(cwd, options?.read),
        bash: createBashToolDefinition(cwd, options?.bash),
        edit: createEditToolDefinition(cwd, options?.edit),
        write: createWriteToolDefinition(cwd, options?.write),
        grep: createGrepToolDefinition(cwd, options?.grep),
        find: createFindToolDefinition(cwd, options?.find),
        ls: createLsToolDefinition(cwd, options?.ls),
    };
}

// 创建编码工具子集
export function createCodingToolDefinitions(
    cwd: string,
    options?: ToolsOptions
): ToolDef[] {
    return [
        createReadToolDefinition(cwd, options?.read),
        createBashToolDefinition(cwd, options?.bash),
        createEditToolDefinition(cwd, options?.edit),
        createWriteToolDefinition(cwd, options?.write),
    ];
}

// 创建只读工具子集
export function createReadOnlyToolDefinitions(
    cwd: string,
    options?: ToolsOptions
): ToolDef[] {
    return [
        createReadToolDefinition(cwd, options?.read),
        createGrepToolDefinition(cwd, options?.grep),
        createFindToolDefinition(cwd, options?.find),
        createLsToolDefinition(cwd, options?.ls),
    ];
}
```

### 9.2 扩展工具注册

```typescript
// 扩展中注册工具
export default function (api: ExtensionAPI) {
    api.registerTool({
        name: "my_custom_tool",
        label: "My Custom Tool",
        description: "Does something custom",
        parameters: Type.Object({
            input: Type.String()
        }),

        execute: async (toolCallId, params, signal, onUpdate, ctx) => {
            // 执行逻辑
            return {
                content: [{ type: "text", text: "Result" }],
                details: {}
            };
        }
    });
}
```

---

## 10. 执行模式

### 10.1 模式对比

| 模式 | 行为 | 用例 |
|------|------|------|
| **parallel** | 多个工具并发执行 | read, grep, find, ls, bash |
| **sequential** | 串行执行，前一个完成后才执行下一个 | edit, write |

### 10.2 串行执行

**代码**：`/packages/agent/src/agent-loop.ts:360-410`

```typescript
async function executeToolCallsSequential(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCalls: AgentToolCall[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink
): Promise<ExecutedToolCallBatch> {
    const messages: ToolResultMessage[] = [];

    // 逐个执行
    for (const toolCall of toolCalls) {
        await emit({
            type: "tool_execution_start",
            toolCallId: toolCall.id,
            toolName: toolCall.name,
            args: toolCall.arguments
        });

        // 准备
        const preparation = await prepareToolCall(...);

        // 执行
        const executed = await executePreparedToolCall(preparation, signal, emit);

        // 完成处理
        const finalized = await finalizeExecutedToolCall(...);

        const toolResultMessage = createToolResultMessage(finalized);
        messages.push(toolResultMessage);
    }

    return { messages, terminate: shouldTerminateToolBatch(finalizedCalls) };
}
```

### 10.3 并行执行

**代码**：`/packages/agent/src/agent-loop.ts:412-471`

```typescript
async function executeToolCallsParallel(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCalls: AgentToolCall[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink
): Promise<ExecutedToolCallBatch> {
    const finalizedCalls: FinalizedToolCallEntry[] = [];

    // 并行准备和执行
    for (const toolCall of toolCalls) {
        // 准备阶段
        const preparation = await prepareToolCall(...);

        // 包装为异步函数
        finalizedCalls.push(async () => {
            const executed = await executePreparedToolCall(preparation, signal, emit);
            const finalized = await finalizeExecutedToolCall(...);
            return finalized;
        });
    }

    // 并行执行
    const orderedFinalizedCalls = await Promise.all(
        finalizedCalls.map((entry) =>
            typeof entry === "function" ? entry() : Promise.resolve(entry)
        )
    );

    // 按顺序创建结果消息
    const messages: ToolResultMessage[] = [];
    for (const finalized of orderedFinalizedCalls) {
        const toolResultMessage = createToolResultMessage(finalized);
        messages.push(toolResultMessage);
    }

    return { messages, terminate: shouldTerminateToolBatch(orderedFinalizedCalls) };
}
```

---

## 11. 流式更新机制

### 11.1 onUpdate 回调

**类型**：
```typescript
type AgentToolUpdateCallback<TDetails> = (partialResult: TDetails) => void;
```

### 11.2 使用示例

**bash 工具**：
```typescript
async function executeBash(
    input: BashToolInput,
    toolCallId: string,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<BashToolDetails> | undefined,
    operations: BashOperations
): Promise<AgentToolResult<BashToolDetails>> {
    let output = "";

    const child = operations.spawn(input.command, cwd);

    // 流式读取输出
    for await (const chunk of child.stdout) {
        output += chunk.toString();

        // 触发更新
        onUpdate?.({
            output: formatOutput(output)
        });
    }

    return { content: [{ type: "text", text: output }], details: {} };
}
```

### 11.3 UI 渲染

**ToolExecutionComponent** (`interactive/modes/components/tool-execution.ts`):
```typescript
class ToolExecutionComponent extends Component {
    private onUpdate = (event: AgentEvent) => {
        if (event.type === "tool_execution_update" && event.toolCallId === this.toolCallId) {
            // 更新局部显示
            this.partialResult = event.partialResult;
            this.invalidate();
        }
    };
}
```

---

## 12. 最佳实践

### 12.1 工具设计原则

1. **幂等性**：相同输入产生相同输出
2. **原子性**：要么完全成功，要么完全失败
3. **可观测性**：通过 onUpdate 提供进度
4. **可取消性**：尊重 AbortSignal

### 12.2 错误处理

```typescript
execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    try {
        // 执行逻辑
        const result = await doSomething(params);

        return {
            content: [{ type: "text", text: result }],
            details: {}
        };
    } catch (error) {
        // 返回错误而非抛出
        return {
            content: [{
                type: "text",
                text: `Error: ${error instanceof Error ? error.message : String(error)}`
            }],
            details: {},
            terminate: false  // 是否终止对话
        };
    }
}
```

### 12.3 性能优化

1. **大文件处理**：使用流式读取，避免内存爆炸
2. **并行工具**：尽可能使用并行模式
3. **缓存**：考虑结果缓存（如文件读取）
4. **截断**：合理限制输出大小

### 12.4 安全考虑

1. **输入验证**：严格验证所有参数
2. **路径解析**：防止目录遍历攻击
3. **命令注入**：小心处理 bash 命令
4. **权限检查**：验证操作权限

---

## 13. 总结

pi-mono 的工具系统设计：

1. **类型安全**：TypeBox 完整运行时验证
2. **高度可扩展**：自定义工具、渲染、钩子
3. **流式优先**：onUpdate 实时进度报告
4. **执行模式**：灵活的并行/串行控制
5. **可插拔**：BashOperations、ReadOperations 等接口
6. **UI 集成**：完整的 TUI 渲染支持

这种设计使得 pi-mono 能够轻松扩展到各种用例，同时保持核心逻辑的简洁性。

---

**相关文档**：
- [Agent Loop 详解](../03-packages/02-pi-agent-core.md)
- [扩展系统](./02-extension-system.md)
- [事件系统](../02-architecture/04-event-system.md)
