# 终端用户指南 (End User Guide)

## 概述

本指南面向终端用户，介绍如何使用 pi-coding-agent 提高编程效率。无需深入技术细节，只需跟随示例操作即可快速上手。

---

## 安装指南

### 方式 1：npm 安装（推荐）

```bash
# 全局安装
npm install -g @mariozechner/pi-coding-agent

# 验证安装
pi --version
# 输出: pi-coding-agent v0.70.2

# 启动
pi
```

### 方式 2：使用 npx（无需安装）

```bash
# 直接运行
npx @mariozechner/pi-coding-agent

# 或使用别名
alias pi='npx @mariozechner/pi-coding-agent'
pi
```

### 方式 3：从源码运行

```bash
# 克隆仓库
git clone https://github.com/mariozechner/pi-mono.git
cd pi-mono

# 安装依赖
npm install

# 构建
npm run build

# 运行
npm run start
```

---

## 初次配置

### 自动配置向导

首次运行 `pi` 时，会自动启动配置向导：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Welcome to pi-coding-agent!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

? Choose your LLM Provider:
  ▸ OpenAI (GPT-4o, GPT-4o-mini)
    Anthropic (Claude 3.5 Sonnet, Claude 3 Haiku)
    Ollama (Local LLMs)
    Google (Gemini Pro)
    AWS Bedrock
    Azure OpenAI
    Custom

? Enter your API Key:
  sk-....................................................

? Select default model:
  ▸ gpt-4o (推荐)
    gpt-4o-mini (快速，成本低)
    gpt-4-turbo

? Choose color mode:
  ▸ Auto-detect
    Truecolor
    256-color

✓ Configuration saved to ~/.pi/config.json
```

### 手动配置

如需手动配置或修改配置，编辑 `~/.pi/config.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/mariozechner/pi-mono/main/packages/coding-agent/src/config/schema.json",
  "provider": "openai",
  "apiKey": "sk-your-api-key-here",
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

### 配置多个 Provider

```json
{
  "provider": "openai",
  "apiKey": "sk-openai-key",
  "model": "gpt-4o",
  "providers": {
    "anthropic": {
      "provider": "anthropic",
      "apiKey": "sk-ant-anthropic-key",
      "model": "claude-3-5-sonnet-20241022"
    },
    "ollama": {
      "provider": "ollama",
      "endpoint": "http://localhost:11434",
      "model": "llama3.1"
    }
  }
}
```

---

## 基础使用

### 启动会话

```bash
# 启动交互式会话
pi

# 指定工作目录
pi /path/to/project

# 使用特定配置
pi --config ~/.pi/config-work.json

# 启用调试模式
pi --debug
```

### 界面导航

```
┌───────────────────────────────────────────────────────────────┐
│  pi-coding-agent v0.70.2                    [OpenAI · gpt-4o]  │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  You: Create a TypeScript function to validate email          │
│                                                               │
│  Agent: I'll create an email validation function for you.     │
│                                                               │
│  function isValidEmail(email: string): boolean {              │
│    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/           │
│    return emailRegex.test(email)                              │
│  }                                                             │
│                                                               │
│  ───────────────────────────────────────────────────────────  │
│                                                               │
│  > │                                                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 键盘快捷键

| 快捷键 | 功能 |
|--------|------|
| `Enter` | 发送消息 |
| `Ctrl+C` | 取消当前生成 |
| `Ctrl+D` | 退出程序 |
| `Ctrl+B` | 创建会话分支 |
| `Ctrl+T` | 显示会话树 |
| `Ctrl+R` | 重新生成响应 |
| `Ctrl+U` | 清空输入 |
| `Ctrl+W` | 删除前一个词 |
| `Ctrl+A` | 移到行首 |
| `Ctrl+E` | 移到行尾 |
| `?` | 显示帮助 |
| `Tab` | 自动补全 |
| `↑/↓` | 浏览历史 |

---

## 常见任务

### 1. 创建新功能

**任务**：创建一个 TypeScript 函数来计算数组平均值

```bash
You: Create a TypeScript function to calculate the average of an array of numbers

Agent: Here's a TypeScript function to calculate the average:

function calculateAverage(numbers: number[]): number {
  if (numbers.length === 0) {
    throw new Error("Cannot calculate average of empty array")
  }

  const sum = numbers.reduce((acc, num) => acc + num, 0)
  return sum / numbers.length
}

// Usage examples:
const scores = [85, 90, 78, 92, 88]
console.log(calculateAverage(scores))  // 86.6
```

### 2. 读取文件

**任务**：查看 package.json 中的依赖

```bash
You: Read package.json and list all dependencies

Agent: [使用 Read 工具]

Found the following dependencies in package.json:

Dependencies:
• typescript: ^5.3.0
• @types/node: ^20.0.0
• vitest: ^1.0.0

DevDependencies:
• eslint: ^8.55.0
• prettier: ^3.1.0
```

### 3. 修改代码

**任务**：将函数改为使用箭头函数

```bash
You: Convert the calculateAverage function to an arrow function

Agent: [使用 Edit 工具修改代码]

I've converted the function to an arrow function:

export const calculateAverage = (numbers: number[]): number => {
  if (numbers.length === 0) {
    throw new Error("Cannot calculate average of empty array")
  }

  const sum = numbers.reduce((acc, num) => acc + num, 0)
  return sum / numbers.length
}
```

### 4. 运行测试

**任务**：运行项目中的测试套件

```bash
You: Run all tests

Agent: [使用 Bash 工具执行命令]

Running test suite...

✓ src/utils/math.spec.ts (5/5 passed)
✓ src/utils/string.spec.ts (8/8 passed)
✓ src/core/agent.spec.ts (12/12 passed)

Total: 25 tests passed in 2.3s

All tests passing! 🎉
```

### 5. Git 操作

**任务**：查看当前 Git 状态并创建提交

```bash
You: Show git status and commit changes with message "Add email validation"

Agent: [使用 GitStatus 和 Bash 工具]

Git Status:
Modified files:
• src/utils/email-validator.ts (new)
• src/utils/email-validator.spec.ts (new)

Untracked files:
• examples/email-validation.ts

Branch: main

Creating commit...

git add src/utils/email-validator.ts src/utils/email-validator.spec.ts
git commit -m "Add email validation"

✓ Committed successfully: Add email validation

Commit hash: a1b2c3d
```

### 6. 代码审查

**任务**：审查最近的代码更改

```bash
You: Review the recent changes

Agent: [使用 GitDiff 和 Read 工具]

Reviewing recent changes...

📝 Changes in src/utils/email-validator.ts:

✓ Good:
  • Clear function name
  • Proper type annotations
  • Includes error handling

⚠️ Suggestions:
  • Consider adding more validation rules
  • Add JSDoc comments for better documentation
  • Consider international email support

Overall: The implementation is solid. Consider the suggestions above for enhancement.
```

### 7. 调试代码

**任务**：帮助调试一个 bug

```bash
You: I'm getting "Cannot read property 'x' of undefined" in my function

Agent: Let me help you debug this issue. First, let me see the code:

[读取代码]

The error occurs because the function doesn't validate if the input object exists before accessing its properties. Here's the fix:

function processValue(input: { x: number } | undefined): number {
  if (!input) {
    throw new Error("Input is required")
  }

  return input.x * 2
}

Or with optional chaining:

function processValue(input?: { x: number }): number {
  return input?.x ?? 0 * 2
}
```

---

## 会话管理

### 创建分支

```bash
# 按 Ctrl+B 创建分支
# 或直接输入
You: Create a branch to explore alternative implementations

Agent: ✓ Created new branch: "branch-2"

You are now on branch-2. Changes here won't affect the main conversation.
```

### 切换分支

```bash
You: Switch to branch-1

Agent: [按 Ctrl+T 查看会话树]

Session Tree:
root
├── branch-1 (active)
└── branch-2

✓ Switched to branch-1

You: [现在在 branch-1 的上下文中]
```

### 导出会话

```bash
You: Export this conversation to a markdown file

Agent: [使用 Write 工具]

✓ Conversation exported to: ~/conversations/session-2024-01-15.md
```

---

## 技巧和窍门

### 1. 使用精确的指令

❌ **差**：
```
Make the code better
```

✅ **好**：
```
Refactor the calculateAverage function to use reduce() instead of a for loop
```

### 2. 分步处理复杂任务

```bash
You: I need to add user authentication. Let's break this down:

1. First, create a User type with email and password fields
2. Then create a validateUser function
3. Finally, create an authenticate function

Let's start with step 1
```

### 3. 引用具体代码

```bash
You: Look at src/utils/email-validator.ts line 15-20 and explain the regex pattern

Agent: The regex pattern `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` works as follows:

• `^` - Start of string
• `[^\s@]+` - One or more non-whitespace, non-@ characters
• `@` - Literal @ symbol
• `[^\s@]+` - One or more non-whitespace, non-@ characters (domain)
• `\.` - Literal dot
• `[^\s@]+` - One or more non-whitespace, non-@ characters (TLD)
• `$` - End of string

This ensures basic email format validation.
```

### 4. 利用上下文

```bash
# pi 会记住之前的对话
You: Now that we have the email validation, add a function to validate phone numbers using the same pattern

Agent: I'll create a similar function for phone validation:

function isValidPhone(phone: string): boolean {
  const phoneRegex = /^\d{3}-\d{3}-\d{4}$/
  return phoneRegex.test(phone)
}
```

### 5. 批量操作

```bash
You: For all .spec.ts files in src/utils, add a describe block at the top

Agent: I'll update all test files:

[批量编辑多个文件]

✓ Updated 5 test files with describe blocks
```

---

## 最佳实践

### 1. 版本控制

```bash
# 在进行大规模更改前，询问 Agent
You: Before we refactor, let's make sure we have a clean git state

Agent: [检查 Git 状态]

Git status is clean with no uncommitted changes. Safe to proceed with refactoring.
```

### 2. 测试驱动开发

```bash
You: Write tests first for the email validation function, then implement it

Agent: I'll create the tests first:

[创建测试文件]

Now let's implement the function to make these tests pass:

[实现函数]
```

### 3. 代码审查

```bash
You: Review this code for potential security issues

Agent: [分析代码]

🔒 Security Review:

Issues found:
1. ⚠️  SQL injection risk in line 45
2. ⚠️  Unvalidated user input in line 23
3. ✓ Proper error handling

Recommendations:
• Use parameterized queries
• Validate and sanitize input
```

### 4. 文档优先

```bash
You: Write JSDoc comments for this function

Agent: I'll add comprehensive JSDoc documentation:

/**
 * Calculates the average of an array of numbers.
 *
 * @param numbers - An array of numeric values
 * @returns The arithmetic mean of the input values
 * @throws {Error} If the input array is empty
 *
 * @example
 * ```ts
 * calculateAverage([1, 2, 3, 4, 5]) // returns 3
 * ```
 */
function calculateAverage(numbers: number[]): number {
  // ...
}
```

---

## 故障排除

### 问题 1：API 连接失败

**错误**：
```
Error: API request failed: Failed to fetch
```

**解决方案**：
```bash
# 1. 检查网络连接
ping api.openai.com

# 2. 验证 API Key
cat ~/.pi/config.json | grep apiKey

# 3. 尝试使用备用 endpoint
# 在 config.json 中设置:
{
  "baseURL": "https://api.openai.com/v1",
  "timeout": 30000
}
```

### 问题 2：内存不足

**错误**：
```
JavaScript heap out of memory
```

**解决方案**：
```bash
# 增加 Node.js 内存限制
export NODE_OPTIONS="--max-old-space-size=4096"
pi

# 或降低上下文压缩阈值
# 在 config.json 中:
{
  "compaction": {
    "threshold": 0.7,
    "maxTokens": 4000
  }
}
```

### 问题 3：响应速度慢

**解决方案**：
```bash
# 1. 使用更快的模型
{
  "model": "gpt-4o-mini"  // 比 gpt-4o 快 3-5 倍
}

# 2. 降低 maxTokens
{
  "maxTokens": 2000
}

# 3. 启用上下文压缩
{
  "compaction": {
    "enabled": true
  }
}
```

---

## 进阶使用

### 自定义技能

创建 `~/.pi/skills/` 目录并添加自定义技能：

```typescript
// ~/.pi/skills/deploy.ts
export default {
  name: "deploy",
  description: "Deploy to production",
  trigger: /deploy\s+to\s+production/i,

  handler: async (input, context) => {
    const result = await context.tools.exec("bash", {
      command: "npm run deploy:prod"
    })

    return {
      success: result.success,
      content: result.success
        ? "Deployed to production! 🚀"
        : `Deployment failed: ${result.error}`
    }
  }
}
```

### 自定义工具

```typescript
// ~/.pi/tools/weather.ts
export default {
  name: "weather",
  description: "Get weather for a location",

  parameters: {
    type: "object",
    properties: {
      location: {
        type: "string",
        description: "City name or ZIP code"
      }
    },
    required: ["location"]
  },

  execute: async ({ location }) => {
    const response = await fetch(
      `https://api.weather.com/${location}`
    )
    const data = await response.json()

    return {
      success: true,
      content: `Weather in ${location}: ${data.temp}°C`
    }
  }
}
```

---

## 常用命令参考

```bash
# 启动选项
pi --help                    # 显示帮助
pi --version                 # 显示版本
pi --config <file>           # 使用特定配置
pi --debug                   # 启用调试
pi --no-color                # 禁用颜色
pi --theme <name>            # 使用特定主题

# 会话管理
pi                           # 启动新会话
pi --resume                  # 恢复上次会话
pi --branch <name>           # 创建/切换分支

# 工具
pi --list-tools              # 列出所有工具
pi --list-skills             # 列出所有技能
pi --list-extensions         # 列出所有扩展

# 配置
pi --config-wizard           # 重新运行配置向导
pi --show-config             # 显示当前配置
pi --reset-config            # 重置配置
```

---

## 资源和帮助

### 获取帮助

- **内置帮助**：在 pi 中按 `?` 或输入 `help`
- **GitHub Issues**：https://github.com/mariozechner/pi-mono/issues
- **Discord 社区**：https://discord.gg/pi-coding-agent
- **文档**：https://docs.pi-ai.dev

### 学习资源

- **快速上手**：`/LEARN/06-onboarding/01-quick-start.md`
- **代码库导航**：`/LEARN/06-onboarding/02-codebase-navigation.md`
- **编写扩展**：`/LEARN/06-onboarding/03-writing-extension.md`

---

## 总结

✅ 你已经学会：
- 安装和配置 pi-coding-agent
- 使用界面和键盘快捷键
- 执行常见编程任务
- 管理会话和分支
- 应用最佳实践
- 解决常见问题

**下一步**：
- [团队集成](./02-team-integration.md)
- [高级配置](./03-advanced-configuration.md)

---

## 相关链接

- **开发者指南**：`/LEARN/06-onboarding/`
- **架构概览**：`/LEARN/02-architecture/01-architecture-overview.md`
- **扩展系统**：`/LEARN/04-subsystems/02-extension-system.md`
