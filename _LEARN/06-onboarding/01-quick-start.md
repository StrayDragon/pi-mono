# 快速上手 (Quick Start)

## 概述

本指南帮助新开发者在 30 分钟内设置开发环境、构建项目并运行 pi-coding-agent。

---

## 前置要求

### 必需条件

- **Node.js**: >= 18.0.0
- **npm**: >= 9.0.0
- **Git**: >= 2.30.0

### 推荐条件

- **pnpm**: 更快的包管理器
- **tmux**: 终端复用器（用于测试 TUI）
- **现代终端**: 支持 truecolor（Kitty、iTerm2、Windows Terminal）

### 验证环境

```bash
node --version  # v18.x.x 或更高
npm --version   # 9.x.x 或更高
git --version   # 2.x.x 或更高
```

---

## 安装步骤

### 方式 1：从 npm 安装（推荐用于使用）

```bash
# 全局安装
npm install -g @mariozechner/pi-coding-agent

# 验证安装
pi --version
# 输出: pi-coding-agent v0.70.2
```

### 方式 2：从源码运行（推荐用于开发）

```bash
# 1. 克隆仓库
git clone https://github.com/mariozechner/pi-mono.git
cd pi-mono

# 2. 安装依赖
npm install

# 3. 构建所有包
npm run build

# 4. 启动开发模式
npm run start
```

---

## 首次配置

### 配置向导

```bash
pi
# 首次运行会自动启动配置向导
```

**配置向导步骤**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Welcome to pi-coding-agent!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

? Choose your LLM Provider:
  > OpenAI
    Anthropic
    Ollama (Local)
    Google
    AWS Bedrock
    Other

? Enter your API Key:
  sk-..................................

? Select default model:
  > gpt-4o
    gpt-4o-mini
    claude-3.5-sonnet
    claude-3-haiku

? Choose color mode:
  > Auto-detect
    Truecolor
    256-color

✓ Configuration saved to ~/.pi/config.json
```

### 手动配置

创建配置文件 `~/.pi/config.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/mariozechner/pi-mono/main/packages/coding-agent/src/config/schema.json",
  "provider": "openai",
  "apiKey": "sk-your-api-key",
  "model": "gpt-4o",
  "theme": "default",
  "maxTokens": 8000,
  "temperature": 0.7,
  "compaction": {
    "enabled": true,
    "threshold": 0.8,
    "minDistance": 10
  }
}
```

### 使用 Ollama（本地模型）

```bash
# 安装 Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# 下载模型
ollama pull llama3.1
ollama pull codellama

# 配置 pi 使用 Ollama
export PI_PROVIDER=ollama
export PI_MODEL=llama3.1
pi
```

---

## 第一次对话

### 启动交互式会话

```bash
pi
```

**界面示例**：

```
┌───────────────────────────────────────────────────────────────┐
│  pi-coding-agent v0.70.2                    [OpenAI · gpt-4o]  │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  💡 Tip: Press ? for help, Ctrl+C to cancel, Ctrl+D to exit  │
│                                                               │
│  ───────────────────────────────────────────────────────────  │
│                                                               │
│  You: What can you help me with?                              │
│                                                               │
│  Agent: I can help you with:                                  │
│                                                               │
│  • Writing and refactoring code                               │
│  • Debugging and troubleshooting                              │
│  • File operations (read, write, edit)                        │
│  • Git operations (status, commit, push)                      │
│  • Running shell commands                                     │
│  • Web search and fetching                                    │
│  • And much more!                                             │
│                                                               │
│  Try asking me to create a function, refactor some code,      │
│  or explain a concept.                                        │
│                                                               │
│  ───────────────────────────────────────────────────────────  │
│                                                               │
│  > │                                                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 基本操作

**输入消息**：
```
You: Create a TypeScript function to validate email addresses

Agent: [生成代码...]

function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

// Usage examples
console.log(isValidEmail("user@example.com"))  // true
console.log(isValidEmail("invalid"))          // false

Need me to add more validation rules or create tests?
```

**键盘快捷键**：
- `Enter` - 发送消息
- `Ctrl+C` - 取消当前生成
- `Ctrl+D` - 退出程序
- `?` - 显示帮助
- `Ctrl+B` - 创建会话分支
- `Ctrl+T` - 显示会话树

---

## 核心功能演示

### 1. 文件操作

```bash
You: Read package.json and tell me the dependencies

[Agent 使用 Read 工具读取 package.json]

Agent: Found the following dependencies:

• typescript: ^5.3.0
• @types/node: ^20.0.0
• ink: ^4.0.0

Want me to check for outdated packages?
```

### 2. Git 操作

```bash
You: What's the current git status?

[Agent 使用 GitStatus 工具]

Agent: Repository status:

Modified files:
• src/main.ts (3 lines changed)
• src/utils.ts (12 lines added)

Untracked files:
• test.spec.ts

Branch: main (ahead of origin by 2 commits)

Shall I create a commit with these changes?
```

### 3. 代码重构

```bash
You: Refactor this function to be more readable

[Agent 使用 Edit 工具修改代码]

Agent: I've refactored the function with these improvements:

✓ Extracted magic numbers to constants
✓ Split complex logic into smaller functions
✓ Added JSDoc comments
✓ Improved variable names

Changes applied to src/calculator.ts:23-45

Want me to run the tests to verify?
```

### 4. Shell 命令

```bash
You: Run the test suite

[Agent 使用 Bash 工具执行命令]

Agent: Running tests...

✓ test/unit/utils.spec.ts (12/12 passed)
✓ test/integration/api.spec.ts (8/8 passed)
✓ test/e2e/workflow.spec.ts (5/5 passed)

Total: 25 tests passed in 2.3s

All tests passing! 🎉
```

---

## 开发模式

### 运行开发版本

```bash
# 在 pi-mono 目录下
npm run dev

# 或使用 ts-node 直接运行
npx ts-node packages/coding-agent/src/index.ts
```

### 热重载

```bash
# 启动监视模式
npm run watch

# 代码更改会自动重新编译
```

### 调试模式

```bash
# 启用详细日志
PI_DEBUG=1 npm run dev

# 或在代码中设置断点
# packages/coding-agent/src/index.ts
debugger;
```

---

## 项目结构导航

### 关键目录

```
pi-mono/
├── packages/
│   ├── ai/                 # LLM 抽象层
│   │   └── src/
│   │       ├── types.ts    # 核心类型定义
│   │       ├── api-registry.ts  # Provider 注册
│   │       └── providers/  # 各 Provider 实现
│   │
│   ├── agent-core/         # Agent 运行时
│   │   └── src/
│   │       ├── agent.ts    # Agent 类
│   │       └── agent-loop.ts  # 主循环
│   │
│   ├── coding-agent/       # CLI 应用
│   │   └── src/
│   │       ├── core/       # 核心逻辑
│   │       ├── tools/      # 内置工具
│   │       └── modes/      # 交互模式
│   │
│   └── tui/                # 终端 UI
│       └── src/
│           ├── tui.ts      # TUI 核心
│           └── components/ # UI 组件
│
├── extensions/             # 内置扩展
├── templates/              # 项目模板
└── docs/                   # 文档
```

### 查找文件

```bash
# 查找特定函数
grep -r "function streamSimple" packages/

# 查找类型定义
grep -r "interface LLMProvider" packages/

# 查找工具实现
find packages/coding-agent/src -name "*.ts" | xargs grep "class.*Tool"
```

---

## 常见任务

### 添加新的 Provider

```typescript
// packages/ai/src/providers/my-provider.ts
import { LLMProvider, Message, Chunk } from "../types"

export class MyProvider implements LLMProvider {
  async *chat(messages: Message[], options: Options): AsyncGenerator<Chunk> {
    // 实现聊天逻辑
  }

  async embed(texts: string[]): Promise<number[][]> {
    // 实现嵌入逻辑
  }
}

// 注册 Provider
// packages/ai/src/api-registry.ts
registry.register("my-provider", (config) => new MyProvider(config))
```

### 创建自定义工具

```typescript
// ~/.pi/extensions/my-tool.ts
import { Tool } from "@mariozechner/pi-coding-agent"

export default {
  name: "my-tool",
  description: "My custom tool",
  parameters: {
    type: "object",
    properties: {
      input: { type: "string", description: "Input value" }
    },
    required: ["input"]
  },
  execute: async ({ input }) => {
    return {
      success: true,
      content: `Processed: ${input}`
    }
  }
} satisfies Tool
```

### 编写测试

```typescript
// tests/unit/my-test.spec.ts
import { describe, it, expect } from "vitest"

describe("My Function", () => {
  it("should work correctly", () => {
    const result = myFunction("test")
    expect(result).toBe("expected")
  })
})
```

---

## 故障排除

### 常见问题

**问题 1：API Key 无效**
```bash
# 检查配置
cat ~/.pi/config.json

# 重新配置
pi --config
```

**问题 2：模块未找到**
```bash
# 清理并重新安装
rm -rf node_modules package-lock.json
npm install
npm run build
```

**问题 3：TUI 显示异常**
```bash
# 检查终端颜色支持
echo $COLORTERM

# 切换到 256 色模式
export PI_COLOR_MODE=256
pi
```

**问题 4：权限错误**
```bash
# 检查文件权限
ls -la ~/.pi/

# 修复权限
chmod -R 755 ~/.pi/
```

### 获取帮助

```bash
# 内置帮助
pi --help

# 查看日志
tail -f ~/.pi/logs/pi.log

# GitHub Issues
https://github.com/mariozechner/pi-mono/issues

# Discord 社区
https://discord.gg/pi-coding-agent
```

---

## 下一步

现在你已经成功设置并运行了 pi-coding-agent，可以继续探索：

1. **开发者上手指南**：
   - [代码库导航](./02-codebase-navigation.md)
   - [编写扩展](./03-writing-extension.md)
   - [添加工具](./05-adding-tool.md)

2. **深入理解**：
   - [架构概览](../02-architecture/01-architecture-overview.md)
   - [设计模式](../05-patterns/01-design-philosophy.md)

3. **实践项目**：
   - 创建你的第一个扩展
   - 贡献给开源项目
   - 分享你的使用经验

---

## 总结

✅ 你已经学会：
- 安装和配置 pi-coding-agent
- 运行第一次对话
- 使用核心功能（文件操作、Git、Shell）
- 启动开发模式
- 导航项目结构

**预计时间**：30 分钟

**下一步**：选择一个感兴趣的方向深入探索！

---

## 相关链接

- **代码库导航**：`/LEARN/06-onboarding/02-codebase-navigation.md`
- **编写扩展**：`/LEARN/06-onboarding/03-writing-extension.md`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
