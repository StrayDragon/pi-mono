# 编写扩展 (Writing Extension)

## 概述

本文档指导开发者如何创建 pi-coding-agent 扩展。扩展是增强功能的强大方式，可以添加工具、技能、UI 组件和生命周期钩子。

---

## 扩展基础

### 什么是扩展

扩展是一个 TypeScript 模块，可以：
- 🔧 添加自定义工具
- 🎯 定义 Skills（触发式任务）
- 🎨 添加 UI 组件
- 🔌 挂载生命周期钩子
- 📦 打包和分发

### 扩展结构

```typescript
// my-extension.ts
import { defineExtension, Tool, Skill } from "@mariozechner/pi-coding-agent"

export default defineExtension({
  // 扩展元数据
  id: "my-extension",
  name: "My Extension",
  version: "1.0.0",
  description: "My first extension",

  // 添加工具
  tools: [
    // 工具定义...
  ],

  // 添加 Skills
  skills: [
    // Skill 定义...
  ],

  // 生命周期钩子
  hooks: {
    onLoad: async () => {
      console.log("Extension loaded!")
    }
  }
})
```

---

## 创建第一个扩展

### 步骤 1：初始化项目

```bash
# 创建扩展目录
mkdir ~/.pi/extensions/my-extension
cd ~/.pi/extensions/my-extension

# 初始化 npm 项目
npm init -y

# 安装依赖
npm install @mariozechner/pi-coding-agent

# 安装开发依赖
npm install -D typescript @types/node
```

### 步骤 2：配置 TypeScript

**tsconfig.json**:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 步骤 3：编写扩展

**src/index.ts**:
```typescript
import { defineExtension, Tool } from "@mariozechner/pi-coding-agent"

// 定义工具
const helloTool: Tool = {
  name: "hello",
  description: "Say hello to someone",
  parameters: {
    type: "object",
    properties: {
      name: {
        type: "string",
        description: "Name to greet"
      }
    },
    required: ["name"]
  },
  execute: async ({ name }) => {
    return {
      success: true,
      content: `Hello, ${name}!`
    }
  }
}

// 定义扩展
export default defineExtension({
  id: "hello-extension",
  name: "Hello Extension",
  version: "1.0.0",
  description: "A simple hello world extension",

  tools: [helloTool],

  hooks: {
    onLoad: async () => {
      console.log("Hello Extension loaded!")
    }
  }
})
```

### 步骤 4：构建和安装

**package.json**:
```json
{
  "name": "hello-extension",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  }
}
```

```bash
# 构建
npm run build

# pi 会自动发现 ~/.pi/extensions/ 下的扩展
pi
```

### 步骤 5：测试扩展

```bash
# 在 pi 中使用
You: Use the hello tool to greet Alice

Agent: I'll use the hello tool to greet Alice.

[Tool Call: hello]
{ "name": "Alice" }

[Tool Result]
Hello, Alice!

The tool successfully greeted Alice with the message "Hello, Alice!".
```

---

## 高级扩展功能

### 1. 多工具扩展

```typescript
import { defineExtension, Tool } from "@mariozechner/pi-coding-agent"

const weatherTool: Tool = {
  name: "weather",
  description: "Get weather for a location",
  parameters: {
    type: "object",
    properties: {
      location: {
        type: "string",
        description: "City name or ZIP code"
      },
      units: {
        type: "string",
        enum: ["celsius", "fahrenheit"],
        description: "Temperature units"
      }
    },
    required: ["location"]
  },
  execute: async ({ location, units = "celsius" }) => {
    // 调用天气 API
    const response = await fetch(
      `https://api.weather.com/${location}?units=${units}`
    )
    const data = await response.json()

    return {
      success: true,
      content: JSON.stringify(data)
    }
  }
}

const forecastTool: Tool = {
  name: "forecast",
  description: "Get weather forecast",
  parameters: {
    type: "object",
    properties: {
      location: {
        type: "string",
        description: "City name or ZIP code"
      },
      days: {
        type: "number",
        description: "Number of days (1-7)",
        minimum: 1,
        maximum: 7
      }
    },
    required: ["location", "days"]
  },
  execute: async ({ location, days }) => {
    // 实现预报逻辑
    return {
      success: true,
      content: `Forecast for ${location} for ${days} days...`
    }
  }
}

export default defineExtension({
  id: "weather-extension",
  name: "Weather Extension",
  version: "1.0.0",
  description: "Weather tools",

  tools: [weatherTool, forecastTool]
})
```

### 2. 添加 Skills

```typescript
import { defineExtension, Skill } from "@mariozechner/pi-coding-agent"

const codeReviewSkill: Skill = {
  name: "code-review",
  description: "Perform code review on changes",
  trigger: /review\s+(this|changes|code)/i,

  handler: async (input, context) => {
    // 获取 Git 状态
    const gitStatus = await context.tools.exec("GitStatus")

    // 读取修改的文件
    const files = gitStatus.modified.map(async (file) => {
      const content = await context.tools.exec("Read", { filePath: file })
      return { file, content }
    })

    // 生成审查报告
    const review = await context.llm.chat([
      {
        role: "system",
        content: "You are a code reviewer. Analyze the changes for bugs, security issues, and improvements."
      },
      {
        role: "user",
        content: `Review these changes:\n${JSON.stringify(files)}`
      }
    ])

    return review.content
  }
}

export default defineExtension({
  id: "code-review-extension",
  name: "Code Review Extension",
  version: "1.0.0",

  skills: [codeReviewSkill]
})
```

### 3. 生命周期钩子

```typescript
import { defineExtension, ExtensionHooks } from "@mariozechner/pi-coding-agent"

const hooks: ExtensionHooks = {
  // 扩展加载时
  onLoad: async (context) => {
    console.log("Extension loaded!")
    // 初始化资源
    await context.store.set("initialized", true)
  },

  // 扩展卸载时
  onUnload: async (context) => {
    console.log("Extension unloaded!")
    // 清理资源
    await context.store.set("initialized", false)
  },

  // 收到消息时
  onMessage: async (message, context) => {
    // 记录所有消息
    await context.store.append("messages", message)
  },

  // 工具调用前
  beforeToolCall: async (tool, args, context) => {
    console.log(`Calling tool: ${tool.name}`)
  },

  // 工具调用后
  afterToolCall: async (tool, result, context) => {
    console.log(`Tool ${tool.name} returned:`, result)
  },

  // 上下文压缩时
  onCompaction: async (entries, context) => {
    // 自定义压缩逻辑
    return entries.filter(e => e.type !== "debug")
  }
}

export default defineExtension({
  id: "hooks-extension",
  name: "Hooks Extension",
  version: "1.0.0",
  hooks
})
```

### 4. 添加 UI 组件

```typescript
import { defineExtension, UIComponent } from "@mariozechner/pi-coding-agent"

const statusBarComponent: UIComponent = {
  type: "status-bar",
  position: "right",
  render: async (context) => {
    const gitStatus = await context.tools.exec("GitStatus")
    const branch = gitStatus.branch

    return `🌿 ${branch}`
  }
}

const panelComponent: UIComponent = {
  type: "panel",
  title: "My Panel",
  position: "right",
  render: async (context) => {
    return `
    ╔══════════════════╗
    ║  Custom Panel    ║
    ╠══════════════════╣
    ║                  ║
    ║  Panel Content   ║
    ║                  ║
    ╚══════════════════╝
    `
  }
}

export default defineExtension({
  id: "ui-extension",
  name: "UI Extension",
  version: "1.0.0",
  ui: {
    components: [statusBarComponent, panelComponent]
  }
})
```

### 5. 扩展间依赖

```typescript
import { defineExtension } from "@mariozechner/pi-coding-agent"

export default defineExtension({
  id: "dependent-extension",
  name: "Dependent Extension",
  version: "1.0.0",

  // 声明依赖
  dependencies: {
    extensions: ["base-extension"],
    packages: {
      "axios": "^1.6.0",
      "zod": "^3.0.0"
    }
  },

  hooks: {
    onLoad: async (context) => {
      // 访问依赖的扩展
      const baseExt = context.extensions.get("base-extension")

      // 使用依赖的包
      const axios = await import("axios")
      const response = await axios.get("https://api.example.com")
    }
  }
})
```

---

## 扩展 API 参考

### defineExtension

```typescript
interface ExtensionDefinition {
  // 元数据
  id: string                    // 唯一标识符
  name: string                  // 显示名称
  version: string               // 语义版本
  description?: string          // 描述

  // 能力
  tools?: Tool[]                // 添加的工具
  skills?: Skill[]              // 添加的技能
  templates?: Template[]        // 添加的模板
  ui?: UIConfig                 // UI 组件

  // 依赖
  dependencies?: {
    extensions?: string[]       // 依赖的扩展
    packages?: Record<string, string>  // 依赖的包
  }

  // 生命周期
  hooks?: ExtensionHooks        // 生命周期钩子
}

function defineExtension(def: ExtensionDefinition): Extension
```

### ExtensionContext

```typescript
interface ExtensionContext {
  // 访问其他扩展
  extensions: {
    get(id: string): Extension | undefined
    list(): Extension[]
  }

  // 工具执行
  tools: {
    exec(name: string, args: unknown): Promise<ToolResult>
    register(tool: Tool): void
    unregister(name: string): void
  }

  // LLM 访问
  llm: {
    chat(messages: Message[]): Promise<Response>
    stream(messages: Message[]): AsyncGenerator<Chunk>
  }

  // 存储和配置
  store: ExtensionStore
  config: ExtensionConfig

  // 事件
  events: EventStream

  // 日志
  logger: {
    debug(message: string): void
    info(message: string): void
    warn(message: string): void
    error(message: string): void
  }
}
```

### Tool 接口

```typescript
interface Tool {
  name: string                          // 工具名称
  description: string                   // 工具描述
  parameters?: ToolParameterSchema      // 参数 Schema
  execute?: (args: unknown) => Promise<ToolResult>
}

interface ToolParameterSchema {
  type: "object" | "string" | "number" | "boolean" | "array"
  properties?: Record<string, ToolParameterSchema>
  required?: string[]
  enum?: string[]
  minimum?: number
  maximum?: number
  description?: string
}

interface ToolResult {
  success: boolean
  content?: string
  error?: string
  metadata?: Record<string, unknown>
}
```

### Skill 接口

```typescript
interface Skill {
  name: string                          // 技能名称
  description: string                   // 技能描述
  trigger: RegExp | string              // 触发条件
  handler: (input: string, context: ExtensionContext) => Promise<string>
}
```

---

## 扩展最佳实践

### 1. 错误处理

```typescript
const safeTool: Tool = {
  name: "safe-tool",
  description: "Tool with error handling",
  execute: async (args) => {
    try {
      // 验证输入
      if (!args || typeof args !== "object") {
        throw new Error("Invalid arguments")
      }

      // 执行逻辑
      const result = await doSomething(args)

      return {
        success: true,
        content: result
      }
    } catch (error) {
      // 返回错误而非抛出
      return {
        success: false,
        error: error.message,
        content: null
      }
    }
  }
}
```

### 2. 参数验证

```typescript
import { z } from "zod"

const schema = z.object({
  url: z.string().url(),
  method: z.enum(["GET", "POST", "PUT", "DELETE"]),
  body: z.object({}).optional()
})

const validatedTool: Tool = {
  name: "validated-tool",
  description: "Tool with parameter validation",
  parameters: {
    type: "object",
    properties: {
      url: { type: "string", description: "URL to request" },
      method: { type: "string", enum: ["GET", "POST", "PUT", "DELETE"] },
      body: { type: "object", description: "Request body" }
    },
    required: ["url", "method"]
  },
  execute: async (args) => {
    // 使用 Zod 验证
    const validated = schema.parse(args)

    // 执行逻辑
    const response = await fetch(validated.url, {
      method: validated.method,
      body: JSON.stringify(validated.body)
    })

    return {
      success: true,
      content: await response.text()
    }
  }
}
```

### 3. 异步操作

```typescript
const asyncTool: Tool = {
  name: "async-tool",
  description: "Tool with async operations",
  execute: async (args) => {
    // 使用 Promise.all 并行执行
    const [result1, result2] = await Promise.all([
      asyncOperation1(args),
      asyncOperation2(args)
    ])

    // 使用 for...of 串行执行
    const results = []
    for (const item of args.items) {
      const result = await processItem(item)
      results.push(result)
    }

    return {
      success: true,
      content: JSON.stringify(results)
    }
  }
}
```

### 4. 资源清理

```typescript
export default defineExtension({
  id: "resource-extension",
  name: "Resource Extension",
  version: "1.0.0",

  hooks: {
    onLoad: async (context) => {
      // 创建资源
      const watcher = fs.watch("./config", () => {
        context.logger.info("Config changed")
      })

      // 保存引用以便清理
      context.store.set("watcher", watcher)
    },

    onUnload: async (context) => {
      // 清理资源
      const watcher = await context.store.get("watcher")
      if (watcher) {
        watcher.close()
      }
    }
  }
})
```

---

## 扩展调试

### 启用调试日志

```bash
# 启用扩展调试
PI_EXT_DEBUG=1 pi

# 或在代码中设置
context.logger.debug("Debug message")
```

### 使用 VS Code 调试

**.vscode/launch.json**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to pi",
      "port": 9229,
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}

```bash
# 启动 pi 并启用调试
node --inspect-brk $(which pi)
```

### 单元测试

```typescript
// test/extension.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import myExtension from "../src/index"

describe("My Extension", () => {
  let context: MockExtensionContext

  beforeEach(() => {
    context = createMockContext()
  })

  it("should load successfully", async () => {
    await myExtension.hooks?.onLoad?.(context)
    expect(context.store.get("initialized")).toBe(true)
  })

  it("should execute tool", async () => {
    const tool = myExtension.tools?.[0]
    const result = await tool.execute?.({ test: "value" })

    expect(result.success).toBe(true)
    expect(result.content).toBeDefined()
  })
})
```

---

## 扩展发布

### 准备发布

```bash
# 1. 更新版本
npm version patch

# 2. 构建
npm run build

# 3. 测试
npm test

# 4. 发布到 npm
npm publish
```

### 分发扩展

**方式 1：npm 包**
```bash
# 用户安装
npm install -g my-extension
```

**方式 2：Git 仓库**
```bash
# 用户从 Git 安装
pi ext install https://github.com/user/my-extension
```

**方式 3：本地文件**
```bash
# 用户复制到扩展目录
cp -r my-extension ~/.pi/extensions/
```

---

## 总结

✅ 你已经学会：
- 创建基本扩展
- 添加工具和 Skills
- 使用生命周期钩子
- 添加 UI 组件
- 处理错误和验证
- 调试和测试
- 发布扩展

**下一步**：
- [添加 Provider](./04-adding-provider.md)
- [添加工具](./05-adding-tool.md)
- [创建 Skill](./06-creating-skill.md)

---

## 相关链接

- **扩展系统**：`/LEARN/04-subsystems/02-extension-system.md`
- **工具系统**：`/LEARN/04-subsystems/01-tool-system.md`
- **Skills 系统**：`/LEARN/04-subsystems/05-skills-system.md`
