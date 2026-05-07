# 添加工具 (Adding Tool)

## 概述

本文档指导开发者如何为 pi-coding-agent 创建自定义工具。工具是 Agent 与外部世界交互的接口，使其能够执行文件操作、运行命令、调用 API 等。

---

## 工具基础

### Tool 接口

```typescript
// packages/coding-agent/src/core/tools/types.ts
interface Tool {
  // 基本信息
  name: string                          // 工具名称（唯一）
  description: string                   // 工具描述

  // 参数定义
  parameters?: ToolParameterSchema      // JSON Schema

  // 执行逻辑
  execute?(args: unknown): Promise<ToolResult>

  // 权限和限制
  permissions?: ToolPermissions         // 权限要求
  rateLimit?: RateLimit                 // 速率限制
}

interface ToolResult {
  success: boolean
  content?: string                      // 返回内容
  data?: unknown                        // 结构化数据
  error?: string                        // 错误信息
  metadata?: Record<string, unknown>    // 元数据
}

interface ToolParameterSchema {
  type: "object" | "string" | "number" | "boolean" | "array"
  properties?: Record<string, ToolParameterSchema>
  required?: string[]
  enum?: string[]
  description?: string
}
```

### 工具注册

```typescript
// packages/coding-agent/src/core/tools/registry.ts
class ToolRegistry {
  private tools = new Map<string, Tool>()

  register(tool: Tool): void {
    if (this.tools.has(tool.name)) {
      throw new Error(`Tool already registered: ${tool.name}`)
    }
    this.tools.set(tool.name, tool)
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name)
  }

  list(): Tool[] {
    return Array.from(this.tools.values())
  }
}

const registry = new ToolRegistry()
registry.register(myTool)
```

---

## 创建简单工具

### 步骤 1：定义工具

```typescript
// my-tool.ts
import { Tool } from "@mariozechner/pi-coding-agent"

export const helloTool: Tool = {
  name: "hello",
  description: "Say hello to someone",

  parameters: {
    type: "object",
    properties: {
      name: {
        type: "string",
        description: "Name to greet"
      },
      enthusiastic: {
        type: "boolean",
        description: "Add enthusiasm",
        default: false
      }
    },
    required: ["name"]
  },

  execute: async (args) => {
    // 类型断言
    const { name, enthusiastic = false } = args as {
      name: string
      enthusiastic?: boolean
    }

    try {
      const greeting = enthusiastic
        ? `Hello, ${name}!!! 🎉`
        : `Hello, ${name}.`

      return {
        success: true,
        content: greeting
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

### 步骤 2：注册工具

```typescript
// packages/coding-agent/src/core/tools/index.ts
import { helloTool } from "./my-tool"

export function registerBuiltinTools(registry: ToolRegistry) {
  // ... 其他工具

  registry.register(helloTool)
}
```

### 步骤 3：测试工具

```bash
# 在 pi 中使用
pi

You: Use the hello tool to greet Alice

Agent: I'll use the hello tool to greet Alice.

[Tool Call: hello]
{
  "name": "Alice",
  "enthusiastic": false
}

[Tool Result]
Hello, Alice.
```

---

## 高级工具功能

### 1. 文件操作工具

```typescript
export const searchFilesTool: Tool = {
  name: "search_files",
  description: "Search for files matching a pattern",

  parameters: {
    type: "object",
    properties: {
      pattern: {
        type: "string",
        description: "File pattern (e.g., '*.ts', 'test*.spec.js')"
      },
      path: {
        type: "string",
        description: "Directory to search (default: current directory)"
      },
      recursive: {
        type: "boolean",
        description: "Search recursively",
        default: true
      }
    },
    required: ["pattern"]
  },

  permissions: {
    readFile: true
  },

  execute: async (args) => {
    const { pattern, path = ".", recursive = true } = args as {
      pattern: string
      path?: string
      recursive?: boolean
    }

    try {
      const glob = new RegExp(
        pattern.replace(/\*/g, ".*").replace(/\?/g, ".")
      )

      const files: string[] = []

      const searchDir = async (dir: string) => {
        const entries = await fs.readdir(dir, { withFileTypes: true })

        for (const entry of entries) {
          const fullPath = join(dir, entry.name)

          if (entry.isDirectory() && recursive) {
            await searchDir(fullPath)
          } else if (entry.isFile() && glob.test(entry.name)) {
            files.push(fullPath)
          }
        }
      }

      await searchDir(path)

      return {
        success: true,
        content: `Found ${files.length} files:\n${files.join("\n")}`,
        data: { files }
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

### 2. HTTP 请求工具

```typescript
export const httpGetTool: Tool = {
  name: "http_get",
  description: "Make HTTP GET request",

  parameters: {
    type: "object",
    properties: {
      url: {
        type: "string",
        description: "URL to request"
      },
      headers: {
        type: "object",
        description: "HTTP headers"
      },
      timeout: {
        type: "number",
        description: "Request timeout in milliseconds",
        default: 10000
      }
    },
    required: ["url"]
  },

  permissions: {
    network: true
  },

  rateLimit: {
    maxCalls: 10,
    period: 60000  // 10 calls per minute
  },

  execute: async (args) => {
    const { url, headers = {}, timeout = 10000 } = args as {
      url: string
      headers?: Record<string, string>
      timeout?: number
    }

    try {
      const controller = new AbortController()
      const timeoutId = setTimeout(() => controller.abort(), timeout)

      const response = await fetch(url, {
        method: "GET",
        headers,
        signal: controller.signal
      })

      clearTimeout(timeoutId)

      const text = await response.text()

      return {
        success: response.ok,
        content: text,
        metadata: {
          status: response.status,
          headers: Object.fromEntries(response.headers.entries())
        }
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

### 3. Git 操作工具

```typescript
export const gitCommitTool: Tool = {
  name: "git_commit",
  description: "Create a Git commit",

  parameters: {
    type: "object",
    properties: {
      message: {
        type: "string",
        description: "Commit message"
      },
      files: {
        type: "array",
        items: { type: "string" },
        description: "Files to commit (default: all)"
      },
      amend: {
        type: "boolean",
        description: "Amend previous commit",
        default: false
      }
    },
    required: ["message"]
  },

  permissions: {
    git: true,
    writeFile: true
  },

  execute: async (args, context) => {
    const { message, files, amend = false } = args as {
      message: string
      files?: string[]
      amend?: boolean
    }

    try {
      // 添加文件
      if (files && files.length > 0) {
        await context.tools.exec("bash", {
          command: `git add ${files.join(" ")}`
        })
      } else {
        await context.tools.exec("bash", {
          command: "git add -A"
        })
      }

      // 创建提交
      const amendFlag = amend ? "--amend" : ""
      const result = await context.tools.exec("bash", {
        command: `git commit ${amendFlag} -m "${message}"`
      })

      return {
        success: true,
        content: result.content || "Committed successfully"
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

### 4. 数据库操作工具

```typescript
export const queryDatabaseTool: Tool = {
  name: "query_database",
  description: "Execute SQL query on database",

  parameters: {
    type: "object",
    properties: {
      connection: {
        type: "string",
        description: "Database connection string or name"
      },
      query: {
        type: "string",
        description: "SQL query to execute"
      },
      params: {
        type: "array",
        items: { type: "string" },
        description: "Query parameters"
      }
    },
    required: ["connection", "query"]
  },

  permissions: {
    database: true
  },

  execute: async (args) => {
    const { connection, query, params = [] } = args as {
      connection: string
      query: string
      params?: unknown[]
    }

    try {
      // 获取数据库连接
      const db = getDatabaseConnection(connection)

      // 执行查询
      const result = await db.query(query, params)

      return {
        success: true,
        content: JSON.stringify(result, null, 2),
        data: result
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

---

## 工具组合

### 复合工具

```typescript
export const deployAppTool: Tool = {
  name: "deploy_app",
  description: "Deploy application to production",

  parameters: {
    type: "object",
    properties: {
      app: {
        type: "string",
        description: "Application name"
      },
      environment: {
        type: "string",
        enum: ["staging", "production"],
        description: "Target environment"
      },
      version: {
        type: "string",
        description: "Version to deploy"
      }
    },
    required: ["app", "environment", "version"]
  },

  execute: async (args, context) => {
    const { app, environment, version } = args as {
      app: string
      environment: "staging" | "production"
      version: string
    }

    try {
      // 1. 运行测试
      const testResult = await context.tools.exec("bash", {
        command: `npm test -- --coverage`
      })

      if (!testResult.success) {
        throw new Error("Tests failed")
      }

      // 2. 构建应用
      const buildResult = await context.tools.exec("bash", {
        command: `npm run build`
      })

      if (!buildResult.success) {
        throw new Error("Build failed")
      }

      // 3. 创建 Git tag
      await context.tools.exec("bash", {
        command: `git tag -a v${version} -m "Release v${version}"`
      })

      // 4. 推送到远程
      await context.tools.exec("bash", {
        command: `git push origin v${version}`
      })

      // 5. 部署
      const deployResult = await context.tools.exec("bash", {
        command: `kubectl set image deployment/${app} ${app}=${version} -n ${environment}`
      })

      return {
        success: true,
        content: `Deployed ${app} v${version} to ${environment}`,
        metadata: {
          testResult: testResult.content,
          buildResult: buildResult.content,
          deployResult: deployResult.content
        }
      }
    } catch (error) {
      return {
        success: false,
        error: (error as Error).message
      }
    }
  }
}
```

---

## 工具权限

### 权限定义

```typescript
interface ToolPermissions {
  // 文件操作
  readFile?: boolean
  writeFile?: boolean
  deleteFile?: boolean

  // 网络操作
  network?: boolean
  domain?: string[]  // 允许的域名列表

  // 系统操作
  executeCommand?: boolean
  git?: boolean

  // 数据库
  database?: boolean

  // 危险操作
  destructive?: boolean  // 需要额外确认
}

// 权限检查
class ToolExecutor {
  async execute(tool: Tool, args: unknown): Promise<ToolResult> {
    // 检查权限
    if (tool.permissions) {
      const hasPermission = await this.checkPermissions(tool.permissions)

      if (!hasPermission) {
        return {
          success: false,
          error: "Permission denied"
        }
      }
    }

    // 执行工具
    return tool.execute!(args, this.context)
  }

  private async checkPermissions(permissions: ToolPermissions): Promise<boolean> {
    // 检查用户权限配置
    const userPermissions = this.getUserPermissions()

    if (permissions.network && !userPermissions.allowNetwork) {
      return false
    }

    if (permissions.destructive && !userPermissions.allowDestructive) {
      return false
    }

    return true
  }
}
```

---

## 工具测试

### 单元测试

```typescript
// tests/tools/my-tool.test.ts
import { describe, it, expect } from "vitest"
import { helloTool } from "@/tools/my-tool"

describe("hello tool", () => {
  it("should greet successfully", async () => {
    const result = await helloTool.execute!({
      name: "Alice"
    })

    expect(result.success).toBe(true)
    expect(result.content).toBe("Hello, Alice.")
  })

  it("should greet enthusiastically", async () => {
    const result = await helloTool.execute!({
      name: "Bob",
      enthusiastic: true
    })

    expect(result.success).toBe(true)
    expect(result.content).toContain("!!!")
  })

  it("should handle missing name", async () => {
    const result = await helloTool.execute!({})

    expect(result.success).toBe(false)
    expect(result.error).toBeDefined()
  })
})
```

### 集成测试

```typescript
// tests/integration/tools.test.ts
import { describe, it, expect } from "vitest"
import { ToolRegistry } from "@/tools/registry"
import { searchFilesTool } from "@/tools/search-files"

describe("tool integration", () => {
  it("should search files in test directory", async () => {
    const registry = new ToolRegistry()
    registry.register(searchFilesTool)

    const tool = registry.get("search_files")!
    const result = await tool.execute!({
      pattern: "*.test.ts",
      path: "./tests",
      recursive: true
    })

    expect(result.success).toBe(true)
    expect(result.data?.files).toBeInstanceOf(Array)
    expect(result.data?.files.length).toBeGreaterThan(0)
  })
})
```

---

## 工具最佳实践

### 1. 输入验证

```typescript
export const validatedTool: Tool = {
  name: "validated_tool",
  description: "Tool with input validation",

  parameters: {
    type: "object",
    properties: {
      email: {
        type: "string",
        format: "email",
        description: "Email address"
      },
      age: {
        type: "number",
        minimum: 0,
        maximum: 150,
        description: "Age in years"
      }
    },
    required: ["email", "age"]
  },

  execute: async (args) => {
    // 使用 Zod 验证
    const schema = z.object({
      email: z.string().email(),
      age: z.number().min(0).max(150)
    })

    try {
      const validated = schema.parse(args)

      // 执行逻辑
      return {
        success: true,
        content: `Validated: ${validated.email}, ${validated.age}`
      }
    } catch (error) {
      return {
        success: false,
        error: "Validation failed",
        data: (error as z.ZodError).errors
      }
    }
  }
}
```

### 2. 错误处理

```typescript
export const robustTool: Tool = {
  name: "robust_tool",
  description: "Tool with comprehensive error handling",

  execute: async (args) => {
    try {
      // 验证输入
      if (!args || typeof args !== "object") {
        throw new Error("Invalid arguments")
      }

      // 执行逻辑
      const result = await someOperation(args)

      return {
        success: true,
        content: result
      }
    } catch (error) {
      // 分类错误
      if (error instanceof ValidationError) {
        return {
          success: false,
          error: "Validation error",
          data: error.details
        }
      }

      if (error instanceof NetworkError) {
        return {
          success: false,
          error: "Network error",
          metadata: { retryable: true }
        }
      }

      // 未知错误
      return {
        success: false,
        error: "Internal error",
        metadata: {
          message: (error as Error).message,
          stack: (error as Error).stack
        }
      }
    }
  }
}
```

### 3. 幂等性

```typescript
export const idempotentTool: Tool = {
  name: "idempotent_tool",
  description: "Tool that can be safely called multiple times",

  execute: async (args) => {
    const { id, data } = args as {
      id: string
      data: unknown
    }

    // 检查是否已存在
    const existing = await getRecord(id)

    if (existing) {
      return {
        success: true,
        content: "Already exists",
        data: existing
      }
    }

    // 创建新记录
    const result = await createRecord(id, data)

    return {
      success: true,
      content: "Created",
      data: result
    }
  }
}
```

---

## 总结

✅ 你已经学会：
- 创建基本工具
- 实现高级功能（文件操作、HTTP、Git、数据库）
- 组合多个工具
- 处理权限和错误
- 测试工具
- 遵循最佳实践

**下一步**：
- [创建 Skill](./06-creating-skill.md)
- [调试技巧](./07-debugging.md)
- [扩展系统](./03-writing-extension.md)

---

## 相关链接

- **工具系统**：`/LEARN/04-subsystems/01-tool-system.md`
- **扩展系统**：`/LEARN/04-subsystems/02-extension-system.md`
- **Agent Loop**：`/LEARN/03-packages/02-pi-agent-core.md`
