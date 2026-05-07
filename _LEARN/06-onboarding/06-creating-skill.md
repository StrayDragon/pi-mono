# 创建 Skill (Creating Skill)

## 概述

本文档指导开发者如何为 pi-coding-agent 创建自定义 Skills。Skills 是触发式任务自动化系统，通过模式匹配自动执行预定义的工作流。

---

## Skill 基础

### 什么是 Skill

Skill 是一个可重用的任务单元，具有：
- 🎯 **触发器** - 激活条件（正则表达式或关键词）
- 📋 **处理器** - 执行逻辑
- 🔄 **可选流** - 后续动作

### Skill 接口

```typescript
// packages/coding-agent/src/core/skills/types.ts
interface Skill {
  // 基本信息
  name: string                          // Skill 名称（唯一）
  description: string                   // Skill 描述
  category: SkillCategory               // 分类

  // 触发条件
  trigger: SkillTrigger                 // 触发器定义

  // 执行逻辑
  handler: SkillHandler                 // 处理函数

  // 配置
  config?: SkillConfig                  // 可选配置
  permissions?: SkillPermissions        // 权限要求
}

type SkillTrigger =
  | RegExp                              // 正则表达式匹配
  | string[]                            // 关键词列表
  | SkillTriggerFn                      // 自定义触发函数

type SkillHandler = (
  input: string,
  context: SkillContext
) => Promise<SkillResult>

interface SkillResult {
  success: boolean
  content?: string
  data?: unknown
  followUp?: {
    action: string
    parameters?: Record<string, unknown>
  }
}
```

### Skill 注册

```typescript
// packages/coding-agent/src/core/skills/registry.ts
class SkillRegistry {
  private skills = new Map<string, Skill>()

  register(skill: Skill): void {
    if (this.skills.has(skill.name)) {
      throw new Error(`Skill already registered: ${skill.name}`)
    }
    this.skills.set(skill.name, skill)
  }

  match(input: string): Skill[] {
    const matched: Skill[] = []

    for (const skill of this.skills.values()) {
      if (this.testTrigger(skill.trigger, input)) {
        matched.push(skill)
      }
    }

    return matched
  }

  private testTrigger(trigger: SkillTrigger, input: string): boolean {
    if (trigger instanceof RegExp) {
      return trigger.test(input)
    }

    if (Array.isArray(trigger)) {
      const lowerInput = input.toLowerCase()
      return trigger.some(keyword =>
        lowerInput.includes(keyword.toLowerCase())
      )
    }

    if (typeof trigger === "function") {
      return trigger(input)
    }

    return false
  }
}

const registry = new SkillRegistry()
registry.register(mySkill)
```

---

## 创建简单 Skill

### 步骤 1：定义 Skill

```typescript
// my-skill.ts
import { Skill } from "@mariozechner/pi-coding-agent"

export const helloSkill: Skill = {
  name: "hello",
  description: "Respond to greetings",
  category: "general",

  // 正则触发器
  trigger: /^(hello|hi|hey|greetings)/i,

  // 处理函数
  handler: async (input, context) => {
    try {
      const userName = context.config.get("user.name") || "there"

      const response = `Hello, ${userName}! How can I help you today?`

      return {
        success: true,
        content: response
      }
    } catch (error) {
      return {
        success: false,
        content: "Sorry, I couldn't process that greeting.",
        data: { error: (error as Error).message }
      }
    }
  }
}
```

### 步骤 2：通过扩展注册

```typescript
// extensions/hello-extension/index.ts
import { defineExtension, Skill } from "@mariozechner/pi-coding-agent"
import { helloSkill } from "./my-skill"

export default defineExtension({
  id: "hello-extension",
  name: "Hello Extension",
  version: "1.0.0",

  skills: [helloSkill]
})
```

### 步骤 3：测试 Skill

```bash
# 在 pi 中使用
pi

You: Hello!

Agent: Hello, there! How can I help you today?

[Skill Matched: hello]
```

---

## 高级 Skill 功能

### 1. 参数提取

```typescript
export const deploySkill: Skill = {
  name: "deploy",
  description: "Deploy application to environment",
  category: "devops",

  // 使用捕获组提取参数
  trigger: /deploy\s+(?:(to|for)\s+)?(\w+)/i,

  handler: async (input, context) => {
    try {
      // 从触发器提取环境
      const match = input.match(deploySkill.trigger as RegExp)
      const environment = match?.[2] || "production"

      // 验证环境
      const validEnvs = ["development", "staging", "production"]
      if (!validEnvs.includes(environment.toLowerCase())) {
        return {
          success: false,
          content: `Invalid environment: ${environment}. Valid options: ${validEnvs.join(", ")}`
        }
      }

      // 获取部署配置
      const deployConfig = await context.config.get(`deploy.${environment}`)

      // 执行部署
      const result = await context.tools.exec("bash", {
        command: `kubectl apply -f manifests/${environment} -n ${environment}`
      })

      if (!result.success) {
        throw new Error(result.error)
      }

      return {
        success: true,
        content: `Successfully deployed to ${environment}!`,
        data: {
          environment,
          timestamp: new Date().toISOString(),
          output: result.content
        },
        followUp: {
          action: "verify_deployment",
          parameters: { environment }
        }
      }
    } catch (error) {
      return {
        success: false,
        content: `Deployment failed: ${(error as Error).message}`,
        data: { error: (error as Error).stack }
      }
    }
  }
}
```

### 2. 多步骤工作流

```typescript
export const codeReviewSkill: Skill = {
  name: "code_review",
  description: "Perform comprehensive code review",
  category: "development",

  trigger: /review\s+(?:this\s+)?(?:code|changes|pr)/i,

  handler: async (input, context) => {
    try {
      // 步骤 1: 获取 Git 状态
      const gitResult = await context.tools.exec("bash", {
        command: "git status --short"
      })

      if (!gitResult.success) {
        throw new Error("Failed to get git status")
      }

      const modifiedFiles = gitResult.content
        .split("\n")
        .filter(line => line.trim())
        .map(line => line.substring(3))
        .filter(file => file.match(/\.(ts|js|tsx|jsx)$/))

      if (modifiedFiles.length === 0) {
        return {
          success: true,
          content: "No TypeScript/JavaScript files to review."
        }
      }

      // 步骤 2: 读取修改的文件
      const fileContents: Record<string, string> = {}

      for (const file of modifiedFiles) {
        const readResult = await context.tools.exec("read", {
          filePath: file
        })

        if (readResult.success) {
          fileContents[file] = readResult.content as string
        }
      }

      // 步骤 3: 获取 Git diff
      const diffResult = await context.tools.exec("bash", {
        command: "git diff"
      })

      // 步骤 4: 使用 LLM 进行审查
      const reviewPrompt = `
        Review these code changes:

        Files modified:
        ${Object.keys(fileContents).join("\n")}

        Diff:
        ${diffResult.content}

        Please analyze:
        1. Potential bugs or issues
        2. Security vulnerabilities
        3. Performance concerns
        4. Code style and best practices
        5. Suggestions for improvement

        Provide specific, actionable feedback.
      `

      const llmResult = await context.llm.chat([
        {
          role: "system",
          content: "You are a senior code reviewer. Provide thorough, constructive feedback."
        },
        {
          role: "user",
          content: reviewPrompt
        }
      ])

      return {
        success: true,
        content: llmResult.content || "Review completed",
        data: {
          filesReviewed: Object.keys(fileContents),
          reviewContent: llmResult.content
        }
      }
    } catch (error) {
      return {
        success: false,
        content: `Code review failed: ${(error as Error).message}`
      }
    }
  }
}
```

### 3. 条件触发

```typescript
// 自定义触发函数
const createConditionalTrigger = (
  conditions: {
    fileExists?: string[]
    gitBranch?: string
    timeRange?: { start: number; end: number }
  }
) => {
  return (input: string): boolean => {
    // 检查文件存在条件
    if (conditions.fileExists) {
      for (const file of conditions.fileExists) {
        if (!fs.existsSync(file)) {
          return false
        }
      }
    }

    // 检查 Git 分支条件
    if (conditions.gitBranch) {
      // 需要通过 context 获取分支信息
      // 这里简化处理
    }

    // 检查时间范围
    if (conditions.timeRange) {
      const hour = new Date().getHours()
      if (hour < conditions.timeRange.start || hour > conditions.timeRange.end) {
        return false
      }
    }

    return true
  }
}

export const officeHoursSkill: Skill = {
  name: "office_hours",
  description: "Only available during office hours",
  category: "utility",

  trigger: createConditionalTrigger({
    timeRange: { start: 9, end: 17 }
  }),

  handler: async (input, context) => {
    return {
      success: true,
      content: "This action is only available during office hours (9 AM - 5 PM)."
    }
  }
}
```

### 4. 组合多个 Skills

```typescript
export const ciPipelineSkill: Skill = {
  name: "ci_pipeline",
  description: "Run complete CI pipeline",
  category: "devops",

  trigger: /run\s+(?:ci\s+)?pipeline/i,

  handler: async (input, context) => {
    const steps = [
      { name: "lint", command: "npm run lint" },
      { name: "test", command: "npm test" },
      { name: "build", command: "npm run build" },
      { name: "e2e", command: "npm run test:e2e" }
    ]

    const results: Record<string, { success: boolean; output?: string }> = {}

    for (const step of steps) {
      const result = await context.tools.exec("bash", {
        command: step.command
      })

      results[step.name] = {
        success: result.success,
        output: result.success ? result.content : undefined
      }

      if (!result.success) {
        return {
          success: false,
          content: `CI pipeline failed at step: ${step.name}`,
          data: {
            failedStep: step.name,
            error: result.error,
            results
          }
        }
      }
    }

    return {
      success: true,
      content: "CI pipeline completed successfully! ✅",
      data: { results }
    }
  }
}
```

---

## Skill 分类体系

### 标准分类

```typescript
enum SkillCategory {
  // 开发相关
  DEVELOPMENT = "development",         // 开发工具
  CODE_REVIEW = "code_review",         // 代码审查
  DEBUGGING = "debugging",             // 调试
  REFACTORING = "refactoring",         // 重构

  // DevOps 相关
  DEVOPS = "devops",                   // DevOps
  DEPLOYMENT = "deployment",           // 部署
  MONITORING = "monitoring",           // 监控

  // Git 相关
  GIT = "git",                         // Git 操作
  COMMIT = "commit",                   // 提交
  BRANCH = "branch",                   // 分支管理

  // 文件相关
  FILE_OPS = "file_ops",               // 文件操作
  SEARCH = "search",                   // 搜索

  // 通用
  GENERAL = "general",                 // 通用
  UTILITY = "utility",                 // 工具
  PRODUCTIVITY = "productivity"        // 生产力
}
```

### 分类标签系统

```typescript
interface SkillMetadata {
  category: SkillCategory
  tags: string[]                       // 自定义标签
  complexity: "simple" | "medium" | "complex"
  estimatedTime?: number               // 预计执行时间（秒）
  dangerous?: boolean                  // 是否有风险
  requiredTools?: string[]             // 需要的工具
}

export const categorizedSkill: Skill = {
  name: "database_migration",
  description: "Run database migrations",
  category: SkillCategory.DEVOPS,

  // 元数据
  config: {
    metadata: {
      category: SkillCategory.DEVOPS,
      tags: ["database", "migration", "sql"],
      complexity: "complex",
      estimatedTime: 60,
      dangerous: true,
      requiredTools: ["bash", "database"]
    }
  },

  trigger: /run\s+migrations?/i,

  handler: async (input, context) => {
    // 实现...
    return { success: true, content: "Migrations completed" }
  }
}
```

---

## Skill 上下文

### SkillContext 接口

```typescript
interface SkillContext {
  // 用户输入
  input: string

  // 工具访问
  tools: {
    exec(name: string, args: unknown): Promise<ToolResult>
    register(tool: Tool): void
  }

  // LLM 访问
  llm: {
    chat(messages: Message[]): Promise<Response>
    stream(messages: Message[]): AsyncGenerator<Chunk>
  }

  // 配置访问
  config: {
    get<T = unknown>(key: string): T | undefined
    set(key: string, value: unknown): Promise<void>
    has(key: string): boolean
  }

  // 状态管理
  state: {
    get<T = unknown>(key: string): T | undefined
    set(key: string, value: unknown): void
    clear(): void
  }

  // 事件
  events: {
    emit(event: string, data?: unknown): void
    on(event: string, handler: Function): void
  }

  // 会话信息
  session: {
    id: string
    branch: string
    root: string
  }

  // 元数据
  metadata: {
    timestamp: Date
    userId?: string
    workspace: string
  }
}
```

### 使用上下文

```typescript
export const contextAwareSkill: Skill = {
  name: "context_aware",
  description: "Demonstrates context usage",
  category: SkillCategory.GENERAL,

  trigger: /show\s+context/i,

  handler: async (input, context) => {
    // 访问配置
    const theme = context.config.get("theme")

    // 访问状态
    const previousRuns = context.state.get<number[]>("runHistory") || []

    // 访问工具
    const gitStatus = await context.tools.exec("bash", {
      command: "git status --short"
    })

    // 访问会话信息
    const currentBranch = context.session.branch

    // 记录状态
    previousRuns.push(Date.now())
    context.state.set("runHistory", previousRuns)

    // 发送事件
    context.events.emit("skill_executed", {
      skill: "context_aware",
      timestamp: context.metadata.timestamp
    })

    return {
      success: true,
      content: `
        Context Information:
        - Theme: ${theme}
        - Branch: ${currentBranch}
        - Previous runs: ${previousRuns.length}
        - Git status: ${gitStatus.content ? "Changed" : "Clean"}
      `
    }
  }
}
```

---

## Skill 权限

### 权限定义

```typescript
interface SkillPermissions {
  // 需要的工具
  tools?: string[]

  // 需要的配置键
  configKeys?: string[]

  // 危险操作
  destructive?: boolean                // 需要额外确认
  modifiesFiles?: boolean              // 修改文件
  networkAccess?: boolean              // 网络访问

  // 速率限制
  rateLimit?: {
    maxCalls: number                   // 最大调用次数
    period: number                     // 时间窗口（毫秒）
  }
}
```

### 权限检查

```typescript
export const protectedSkill: Skill = {
  name: "protected",
  description: "Requires special permissions",
  category: SkillCategory.DANGEROUS,

  permissions: {
    tools: ["bash", "write"],
    destructive: true,
    modifiesFiles: true,
    rateLimit: {
      maxCalls: 3,
      period: 3600000  // 1 hour
    }
  },

  trigger: /delete\s+all\s+logs/i,

  handler: async (input, context) => {
    // 检查权限
    if (!context.user.hasPermission("logs.delete")) {
      return {
        success: false,
        content: "You don't have permission to delete logs."
      }
    }

    // 检查速率限制
    const calls = context.state.get<number[]>("deleteCalls") || []
    const oneHourAgo = Date.now() - 3600000
    const recentCalls = calls.filter(c => c > oneHourAgo)

    if (recentCalls.length >= 3) {
      return {
        success: false,
        content: "Rate limit exceeded. Please try again later."
      }
    }

    // 执行操作
    const result = await context.tools.exec("bash", {
      command: "rm -rf logs/*"
    })

    // 记录调用
    recentCalls.push(Date.now())
    context.state.set("deleteCalls", recentCalls)

    return {
      success: result.success,
      content: result.success ? "All logs deleted" : result.error
    }
  }
}
```

---

## Skill 测试

### 单元测试

```typescript
// tests/skills/my-skill.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { helloSkill } from "@/skills/hello-skill"
import { createMockContext } from "../fixtures/context"

describe("hello skill", () => {
  let context: MockSkillContext

  beforeEach(() => {
    context = createMockContext({
      config: new Map([["user.name", "Alice"]])
    })
  })

  it("should match hello input", () => {
    const trigger = helloSkill.trigger as RegExp

    expect(trigger.test("hello")).toBe(true)
    expect(trigger.test("Hello there!")).toBe(true)
    expect(trigger.test("hi")).toBe(true)
    expect(trigger.test("goodbye")).toBe(false)
  })

  it("should greet user", async () => {
    const result = await helloSkill.handler("hello", context)

    expect(result.success).toBe(true)
    expect(result.content).toContain("Alice")
  })

  it("should handle missing config", async () => {
    const emptyContext = createMockContext({})

    const result = await helloSkill.handler("hello", emptyContext)

    expect(result.success).toBe(true)
    expect(result.content).toContain("there")
  })
})
```

### 集成测试

```typescript
// tests/integration/skills.test.ts
import { describe, it, expect } from "vitest"
import { SkillRegistry } from "@/skills/registry"
import { codeReviewSkill } from "@/skills/code-review"

describe("skill integration", () => {
  it("should match and execute skill", async () => {
    const registry = new SkillRegistry()
    registry.register(codeReviewSkill)

    const input = "Please review this code"
    const matched = registry.match(input)

    expect(matched).toHaveLength(1)
    expect(matched[0].name).toBe("code_review")

    const context = createMockContext({
      tools: { exec: mockToolExec }
    })

    const result = await matched[0].handler(input, context)

    expect(result.success).toBeDefined()
  })
})
```

---

## Skill 最佳实践

### 1. 触发器设计

```typescript
// ✅ 好的触发器 - 明确、具体
const goodTrigger = /deploy\s+to\s+(staging|production)/i

// ❌ 差的触发器 - 太宽泛
const badTrigger = /deploy/i

// ✅ 好的触发器 - 使用边界词
const goodTrigger2 = /\b(run|execute)\s+tests?\b/i

// ❌ 差的触发器 - 会误匹配
const badTrigger2 = /test/
```

### 2. 错误处理

```typescript
export const robustSkill: Skill = {
  name: "robust",
  description: "Skill with comprehensive error handling",

  trigger: /do\s+something/i,

  handler: async (input, context) => {
    try {
      // 验证输入
      if (!input || input.trim().length === 0) {
        throw new Error("Empty input")
      }

      // 执行操作
      const result = await performOperation(input)

      // 验证结果
      if (!result) {
        throw new Error("Operation returned empty result")
      }

      return {
        success: true,
        content: result
      }
    } catch (error) {
      // 分类错误
      if (error instanceof ValidationError) {
        return {
          success: false,
          content: `Validation error: ${error.message}`,
          data: { field: error.field }
        }
      }

      if (error instanceof NetworkError) {
        return {
          success: false,
          content: "Network error. Please check your connection.",
          data: { retryable: true }
        }
      }

      // 未知错误
      return {
        success: false,
        content: `Unexpected error: ${(error as Error).message}`,
        data: {
          error: (error as Error).stack,
          suggestion: "Please check the logs for more details"
        }
      }
    }
  }
}
```

### 3. 进度反馈

```typescript
export const progressSkill: Skill = {
  name: "progress",
  description: "Skill with progress updates",

  trigger: /long\s+running\s+task/i,

  handler: async (input, context) => {
    const steps = [
      "Initializing...",
      "Processing data...",
      "Generating results...",
      "Finalizing..."
    ]

    for (let i = 0; i < steps.length; i++) {
      // 发送进度事件
      context.events.emit("progress", {
        step: i + 1,
        total: steps.length,
        message: steps[i]
      })

      // 执行步骤
      await performStep(i)

      // 短暂延迟，让 UI 有时间更新
      await new Promise(resolve => setTimeout(resolve, 100))
    }

    return {
      success: true,
      content: "Task completed successfully!"
    }
  }
}
```

### 4. 可重用性

```typescript
// 创建可重用的 Skill 模板
function createTemplateSkill(
  name: string,
  trigger: RegExp,
  command: string
): Skill {
  return {
    name,
    description: `Execute ${name}`,
    category: SkillCategory.UTILITY,

    trigger,

    handler: async (input, context) => {
      // 从输入中提取参数
      const args = extractArgs(input, trigger)

      // 构建命令
      const fullCommand = interpolateCommand(command, args)

      // 执行命令
      const result = await context.tools.exec("bash", {
        command: fullCommand
      })

      return {
        success: result.success,
        content: result.success
          ? `${name} completed`
          : `${name} failed: ${result.error}`,
        data: result
      }
    }
  }
}

// 使用模板创建 Skills
export const lintSkill = createTemplateSkill(
  "lint",
  /run\s+lint/i,
  "npm run lint"
)

export const testSkill = createTemplateSkill(
  "test",
  /run\s+tests?/i,
  "npm test"
)

export const buildSkill = createTemplateSkill(
  "build",
  /run\s+build/i,
  "npm run build"
)
```

---

## 总结

✅ 你已经学会：
- 创建基本 Skills
- 实现高级功能（参数提取、多步骤工作流、条件触发）
- 使用 Skill 上下文
- 处理权限和错误
- 测试 Skills
- 遵循最佳实践

**下一步**：
- [调试技巧](./07-debugging.md)
- [工具系统](../04-subsystems/01-tool-system.md)
- [扩展系统](../04-subsystems/02-extension-system.md)

---

## 相关链接

- **Skills 系统**：`/LEARN/04-subsystems/05-skills-system.md`
- **编写扩展](./03-writing-extension.md)
- **添加工具](./05-adding-tool.md)
