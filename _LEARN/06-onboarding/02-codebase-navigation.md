# 代码库导航 (Codebase Navigation)

## 概述

本文档提供 pi-mono 代码库的详细导航指南，包括项目结构、关键文件、代码组织和查找技巧。

---

## Monorepo 结构

### 包依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                    pi-mono Monorepo                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  L4: 应用层 (Applications)                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │pi-coding-    │  │  pi-mom      │  │  pi-web-ui   │      │
│  │agent         │  │  (Slack)     │  │  (React)     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                 │                │                │
│         └─────────────────┼────────────────┘                │
│                           │                                 │
│  L3: 领域层 (Domain)                                        │
│  ┌──────────────────────────────────────────────────┐       │
│  │  Extensions, Skills, Templates, System Prompts    │       │
│  └──────────────────────────────────────────────────┘       │
│                           │                                 │
│  L2: Agent 运行时 (Agent Runtime)                           │
│  ┌──────────────────────────────────────────────────┐       │
│  │         pi-agent-core (Agent Loop, Events)        │       │
│  └──────────────────────────────────────────────────┘       │
│                           │                                 │
│  L1: 基础设施层 (Infrastructure)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   pi-ai      │  │   pi-tui     │  │   pi-db      │      │
│  │  (LLM API)   │  │  (Terminal)  │  │  (Storage)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 目录结构

```
pi-mono/
│
├── packages/                    # 核心包
│   ├── ai/                     # LLM 抽象层
│   ├── agent-core/             # Agent 运行时
│   ├── coding-agent/           # CLI 应用
│   ├── tui/                    # 终端 UI 库
│   ├── web-ui/                 # Web UI 组件
│   ├── mom/                    # Slack 机器人
│   └── pods/                   # GPU Pod 管理
│
├── extensions/                 # 内置扩展
│   ├── browser/               # 浏览器自动化
│   ├── database/              # 数据库工具
│   └── testing/               # 测试辅助
│
├── templates/                  # 项目模板
│   ├── node-ts/               # Node.js + TypeScript
│   └── python/                # Python 项目
│
├── tests/                      # 跨包测试
│   ├── integration/           # 集成测试
│   └── e2e/                   # 端到端测试
│
├── docs/                       # 文档
├── scripts/                    # 构建和工具脚本
├── .github/                    # GitHub 配置
│
├── package.json               # 根 package.json
├── pnpm-workspace.yaml        # pnpm 工作空间配置
├── tsconfig.json              # TypeScript 根配置
├── turbo.json                 # Turborepo 配置
└── README.md                  # 项目 README
```

---

## 关键文件索引

### 根目录配置

| 文件 | 用途 |
|------|------|
| `package.json` | 根包配置，定义 workspace 和脚本 |
| `pnpm-workspace.yaml` | pnpm workspace 配置 |
| `tsconfig.json` | TypeScript 基础配置 |
| `turbo.json` | Turborepo 构建配置 |
| `.eslintrc.js` | ESLint 配置 |
| `.prettierrc` | Prettier 配置 |
| `AGENTS.md` | AI Agent 贡献规则 |
| `CONTRIBUTING.md` | 贡献指南 |

### pi-ai 包

**路径**：`packages/ai/`

| 文件/目录 | 描述 | 行数 |
|----------|------|------|
| `src/types.ts` | 核心类型定义 | ~400 |
| `src/api-registry.ts` | Provider 注册表 | ~200 |
| `src/stream.ts` | 流式 API | ~150 |
| `src/models.ts` | 模型注册表 | ~300 |
| `src/providers/` | Provider 实现 | ~2000 |
| `src/providers/register-builtins.ts` | 内置 Provider 懒加载 | ~100 |

**关键类型**：
```typescript
// src/types.ts
interface LLMProvider {
  chat(messages: Message[], options: Options): AsyncGenerator<Chunk>
  embed(texts: string[]): Promise<number[][]>
}

interface Message {
  role: "user" | "assistant" | "system" | "tool"
  content: string
  toolCalls?: ToolCall[]
}
```

### pi-agent-core 包

**路径**：`packages/agent/`

| 文件/目录 | 描述 | 行数 |
|----------|------|------|
| `src/agent.ts` | Agent 类 | ~500 |
| `src/agent-loop.ts` | Agent Loop 核心逻辑 | ~800 |
| `src/types.ts` | AgentEvent 等核心类型 | ~300 |
| `src/events.ts` | EventStream 实现 | ~200 |
| `src/tools/` | 工具执行逻辑 | ~400 |

**关键类**：
```typescript
// src/agent.ts
class Agent {
  constructor(
    private llm: LLMProvider,
    private tools: ToolRegistry,
    private events: EventStream<AgentEvent>
  ) { }

  async chat(message: string): Promise<Response>
  async stream(message: string): AsyncGenerator<Chunk>
}

// src/agent-loop.ts
class AgentLoop {
  async run(messages: Message[]): Promise<void>
  private async handleToolCalls(toolCalls: ToolCall[]): Promise<void>
  private async checkForFollowUp(): Promise<boolean>
}
```

### pi-coding-agent 包

**路径**：`packages/coding-agent/`

| 文件/目录 | 描述 | 行数 |
|----------|------|------|
| `src/index.ts` | CLI 入口点 | ~100 |
| `src/core/agent-session.ts` | AgentSession 类 | ~800 |
| `src/core/session-manager.ts` | 会话管理器 | ~400 |
| `src/core/extensions/` | 扩展系统 | ~2000 |
| `src/core/tools/` | 内置工具定义 | ~1000 |
| `src/core/compaction/` | 上下文压缩 | ~1500 |
| `src/modes/interactive/` | 交互模式 | ~1500 |
| `src/config/config.ts` | 配置管理 | ~300 |

### pi-tui 包

**路径**：`packages/tui/`

| 文件/目录 | 描述 | 行数 |
|----------|------|------|
| `src/tui.ts` | TUI 核心与差分渲染 | ~600 |
| `src/components/` | UI 组件 | ~2000 |
| `src/keybindings.ts` | 快捷键系统 | ~250 |
| `src/keys.ts` | 类型安全按键 ID | ~150 |
| `src/theme/` | 主题系统 | ~400 |

---

## 代码查找技巧

### 使用 grep 查找代码

```bash
# 查找函数定义
grep -r "function streamSimple" packages/

# 查找类定义
grep -r "class AgentLoop" packages/

# 查找接口定义
grep -r "interface LLMProvider" packages/

# 查找工具定义
grep -r "name: \"read\"" packages/coding-agent/src/tools/

# 递归查找子目录
find packages -name "*.ts" -type f | xargs grep "pattern"
```

### 使用 ripgrep (更快)

```bash
# 安装 ripgrep
npm install -g ripgrep

# 使用 rg 查找
rg "streamSimple" packages/
rg "class Agent" packages/ --type ts

# 查找并显示上下文
rg "AgentLoop" packages/ -C 3

# 只显示文件名
rg "interface.*Provider" packages/ --files-with-matches
```

### 使用 IDE 功能

**VS Code**：
- `Ctrl+Shift+F` - 全局搜索
- `Ctrl+T` - 符号搜索
- `F12` - 跳转到定义
- `Shift+F12` - 查找引用

**JetBrains IDEs**：
- `Ctrl+Shift+N` - 查找文件
- `Ctrl+Alt+Shift+N` - 查找符号
- `Ctrl+Click` - 跳转到定义
- `Alt+F7` - 查找使用

---

## 代码组织模式

### 1. 按功能分层

```
src/
├── core/              # 核心逻辑
│   ├── agent.ts
│   ├── session.ts
│   └── tools/
├── modes/             # 运行模式
│   ├── interactive/
│   └── non-interactive/
├── config/            # 配置
├── utils/             # 工具函数
└── types/             # 类型定义
```

### 2. 按领域划分

```
src/
├── agent/             # Agent 相关
│   ├── agent.ts
│   ├── agent-loop.ts
│   └── events.ts
├── llm/               # LLM 相关
│   ├── providers/
│   ├── models.ts
│   └── stream.ts
└── tools/             # 工具相关
    ├── registry.ts
    ├── executor.ts
    └── built-in/
```

### 3. 导出模式

**barrel exports** (`index.ts`):
```typescript
// packages/ai/src/index.ts
export * from "./types"
export * from "./api-registry"
export * from "./stream"
export * from "./models"
```

**使用 barrel exports**:
```typescript
// 简洁导入
import { LLMProvider, Message, APIRegistry } from "@mariozechner/pi-ai"

// 而非
import { LLMProvider } from "@mariozechner/pi-ai/types"
import { APIRegistry } from "@mariozechner/pi-ai/api-registry"
```

---

## 关键代码路径

### 对话流程

```
用户输入
  ↓
packages/coding-agent/src/modes/interactive/interactive-mode.ts
  ↓ 接收输入
packages/coding-agent/src/core/agent-session.ts
  ↓ 创建消息
packages/agent/src/agent.ts
  ↓ 调用 LLM
packages/ai/src/stream.ts
  ↓ 流式响应
packages/tui/src/tui.ts
  ↓ 渲染输出
显示给用户
```

### 工具执行流程

```
LLM 返回工具调用
  ↓
packages/agent/src/agent-loop.ts
  ↓ 解析工具调用
packages/coding-agent/src/core/tools/executor.ts
  ↓ 查找工具
packages/coding-agent/src/core/tools/registry.ts
  ↓ 执行工具
工具.execute()
  ↓ 返回结果
packages/agent/src/agent-loop.ts
  ↓ 发送回 LLM
下一个 LLM 请求
```

### 扩展加载流程

```
启动时
  ↓
packages/coding-agent/src/core/extensions/loader.ts
  ↓ 扫描目录
~/.pi/extensions/
  ↓ 加载扩展
extension.default
  ↓ 注册钩子
packages/coding-agent/src/core/extensions/hooks.ts
  ↓ 触发 onLoad
扩展可用
```

---

## 包间依赖关系

### 依赖图

```
pi-coding-agent
  ├── pi-agent-core (必须)
  ├── pi-ai (必须)
  ├── pi-tui (必须)
  └── pi-db (可选)

pi-agent-core
  └── pi-ai (必须)

pi-mom
  ├── pi-agent-core (必须)
  ├── pi-ai (必须)
  └── pi-coding-agent (继承工具)

pi-web-ui
  ├── pi-agent-core (必须)
  └── pi-ai (必须)

pi-tui
  └── 无外部依赖（基础包）
```

### 查看 package.json 依赖

```bash
# 查看特定包的依赖
cat packages/coding-agent/package.json | jq .dependencies

# 查看所有包的依赖关系
find packages -name "package.json" | xargs grep -A 5 '"dependencies"'

# 检查循环依赖
npx madge --circular --extensions ts packages/
```

---

## 开发工作流

### 修改代码并测试

```bash
# 1. 修改代码
vim packages/ai/src/providers/openai.ts

# 2. 构建
npm run build

# 3. 运行
npm run start

# 4. 或使用开发模式（自动重新编译）
npm run dev
```

### 添加新功能

```bash
# 1. 创建功能分支
git checkout -b feature/my-feature

# 2. 修改代码
vim packages/coding-agent/src/core/tools/my-tool.ts

# 3. 添加测试
vim tests/unit/my-tool.spec.ts

# 4. 运行测试
npm test

# 5. 构建并验证
npm run build
npm run start

# 6. 提交
git add .
git commit -m "feat: add my-tool"
```

### 调试代码

```bash
# 1. 启用调试模式
PI_DEBUG=1 npm run dev

# 2. 或使用 Node.js 调试器
node --inspect-brk packages/coding-agent/dist/index.js

# 3. 在 VS Code 中调试
# 创建 .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug pi",
      "program": "${workspaceFolder}/packages/coding-agent/dist/index.js",
      "cwd": "${workspaceFolder}",
      "outFiles": ["${workspaceFolder}/packages/*/dist/**/*.js"]
    }
  ]
}
```

---

## 代码风格和约定

### 命名约定

**文件名**：
- `kebab-case.ts` - 文件和目录
- `kebab-case.spec.ts` - 测试文件
- `kebab-case.types.ts` - 类型定义文件

**变量/函数**：
- `camelCase` - 变量和函数
- `PascalCase` - 类和接口
- `SCREAMING_SNAKE_CASE` - 常量

**类型/接口**：
```typescript
// 接口：PascalCase，无 I 前缀
interface LLMProvider { }

// 类型别名：PascalCase
type MessageRole = "user" | "assistant"

// 泛型：T 前缀
interface Result<T> { }
```

### 导入顺序

```typescript
// 1. Node.js 内置模块
import { readFile } from "fs"
import { join } from "path"

// 2. 外部依赖
import { z } from "zod"
import { EventEmitter } from "events"

// 3. 内部包（@mariozechner）
import { Agent } from "@mariozechner/pi-agent-core"
import { TUI } from "@mariozechner/pi-tui"

// 4. 相对导入
import { myUtil } from "./utils"
import { MyType } from "./types"
```

### 注释风格

```typescript
/**
 * Multi-line documentation comment
 * describing a function or class.
 *
 * @param input - The input value
 * @returns The processed output
 * @throws {Error} When input is invalid
 */
function process(input: string): string {
  // Single-line comment for implementation details
  return input.trim()
}

// TODO: Future work
// FIXME: Known issue
// HACK: Temporary workaround
```

---

## 总结

✅ 你已经学会：
- 理解 Monorepo 结构
- 定位关键文件
- 查找代码模式
- 理解代码流程
- 遵循代码风格

**下一步**：
- [编写扩展](./03-writing-extension.md)
- [添加 Provider](./04-adding-provider.md)
- [添加工具](./05-adding-tool.md)

---

## 相关链接

- **快速上手**：`/LEARN/06-onboarding/01-quick-start.md`
- **编写扩展**：`/LEARN/06-onboarding/03-writing-extension.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
