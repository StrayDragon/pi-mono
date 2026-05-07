# 团队集成 (Team Integration)

## 概述

本指南介绍如何在团队环境中集成和使用 pi-coding-agent，包括共享配置、标准化工作流、CI/CD 集成和最佳实践。

---

## 团队配置

### 共享配置文件

**`.pi/config.json`** (项目根目录):
```json
{
  "$schema": "https://raw.githubusercontent.com/mariozechner/pi-mono/main/packages/coding-agent/src/config/schema.json",

  // 统一的 Provider 配置
  "provider": "openai",
  "apiKey": "${PI_API_KEY}",  // 从环境变量读取
  "model": "gpt-4o",

  // 项目特定配置
  "maxTokens": 8000,
  "temperature": 0.7,

  // 代码风格
  "codeStyle": {
    "typescript": {
      "indent": 2,
      "semi": true,
      "singleQuote": false,
      "trailingComma": "es5"
    }
  },

  // 项目路径
  "paths": {
    "src": "./src",
    "tests": "./tests",
    "docs": "./docs"
  },

  // 团队约定
  "conventions": {
    "commitMessage": "conventional",
    "branchNaming": "feature/,bugfix/,hotfix/",
    "codeReview": "required"
  },

  // 上下文压缩
  "compaction": {
    "enabled": true,
    "threshold": 0.8,
    "minDistance": 10,
    "maxTokens": 6000
  }
}
```

### 环境变量管理

**`.env.example`**:
```bash
# LLM Provider API Keys
PI_API_KEY=sk-your-openai-key
PI_ANTHROPIC_KEY=sk-ant-your-anthropic-key

# Team Configuration
PI_TEAM_NAME=my-team
PI_PROJECT_NAME=my-project
PI_LOG_LEVEL=info

# Optional: Custom endpoint
PI_BASE_URL=https://api.openai.com/v1
```

**`.env`** (不提交到版本控制):
```bash
PI_API_KEY=sk-actual-key-here
PI_TEAM_NAME=my-team
```

**`.gitignore`**:
```
# 环境变量
.env
.env.local
.env.*.local

# PI 配置（敏感信息）
.pi/config.local.json

# PI 日志
.pi/logs/
*.log

# PI 会话
.pi/sessions/
.pi/cache/
```

---

## 标准化工作流

### 1. 提交消息规范

创建团队提交消息 Skill：

**`.pi/skills/commit.ts`**:
```typescript
export default {
  name: "conventional_commit",
  description: "Generate conventional commit messages",
  category: "git",

  trigger: /commit\s+(?:these\s+)?changes/i,

  handler: async (input, context) => {
    // 获取 Git 状态
    const gitStatus = await context.tools.exec("bash", {
      command: "git status --short"
    })

    if (!gitStatus.success) {
      return {
        success: false,
        content: "Failed to get git status"
      }
    }

    // 获取 diff
    const gitDiff = await context.tools.exec("bash", {
      command: "git diff --staged"
    })

    // 使用 LLM 生成提交消息
    const prompt = `
      Generate a conventional commit message for these changes:

      ${gitDiff.content}

      Format: <type>(<scope>): <description>

      Types: feat, fix, docs, style, refactor, test, chore
    `

    const result = await context.llm.chat([
      {
        role: "system",
        content: "You are a commit message generator. Follow conventional commits format."
      },
      {
        role: "user",
        content: prompt
      }
    ])

    // 提供提交命令
    return {
      success: true,
      content: `Generated commit message:\n\n${result.content}\n\nCommit with:\ngit commit -m "${result.content}"`
    }
  }
}
```

### 2. 代码审查工作流

**`.pi/skills/code-review.ts`**:
```typescript
export default {
  name: "team_code_review",
  description: "Perform team-standard code review",
  category: "development",

  trigger: /review\s+(?:this\s+)?(?:pr|pull\s+request|changes)/i,

  handler: async (input, context) => {
    // 获取团队审查标准
    const reviewStandards = await context.config.get("conventions.codeReviewStandards")

    // 获取修改的文件
    const gitStatus = await context.tools.exec("bash", {
      command: "git status --short"
    })

    const modifiedFiles = gitStatus.content
      .split("\n")
      .filter(line => line.trim())
      .map(line => line.substring(3))

    // 读取文件内容
    const fileContents = {}
    for (const file of modifiedFiles) {
      const result = await context.tools.exec("read", {
        filePath: file
      })
      if (result.success) {
        fileContents[file] = result.content
      }
    }

    // 使用团队标准进行审查
    const reviewPrompt = `
      Review these changes according to our team standards:

      Standards:
      ${JSON.stringify(reviewStandards, null, 2)}

      Files changed:
      ${JSON.stringify(Object.keys(fileContents))}

      For each file, check:
      1. Code style compliance
      2. Test coverage
      3. Documentation
      4. Security issues
      5. Performance concerns

      Provide specific, actionable feedback.
    `

    const result = await context.llm.chat([
      {
        role: "system",
        content: "You are a senior code reviewer. Be thorough but constructive."
      },
      {
        role: "user",
        content: reviewPrompt
      }
    ])

    return {
      success: true,
      content: result.content || "Review completed",
      data: {
        filesReviewed: Object.keys(fileContents),
        standards: reviewStandards
      }
    }
  }
}
```

### 3. PR 模板生成

**`.pi/skills/pr-template.ts`**:
```typescript
export default {
  name: "pr_template",
  description: "Generate PR description from changes",
  category: "git",

  trigger: /create\s+pr\s+(?:description|template)/i,

  handler: async (input, context) => {
    // 获取当前分支
    const branchResult = await context.tools.exec("bash", {
      command: "git rev-parse --abbrev-ref HEAD"
    })

    const branch = branchResult.content?.trim() || "unknown"

    // 获取 commit 历史
    const commitsResult = await context.tools.exec("bash", {
      command: "git log main..HEAD --oneline"
    })

    // 获取 diff 统计
    const diffResult = await context.tools.exec("bash", {
      command: "git diff main...HEAD --stat"
    })

    // 生成 PR 描述
    const prPrompt = `
      Generate a pull request description based on:

      Branch: ${branch}
      Commits:
      ${commitsResult.content}

      Changed files:
      ${diffResult.content}

      Include:
      - Description
      - Type of change (feat/fix/refactor/docs/etc)
      - Related issues
      - Testing steps
      - Checklist
    `

    const result = await context.llm.chat([
      {
        role: "system",
        content: "Generate professional PR descriptions following the template."
      },
      {
        role: "user",
        content: prPrompt
      }
    ])

    return {
      success: true,
      content: `## Pull Request Description\n\n${result.content}`
    }
  }
}
```

---

## CI/CD 集成

### GitHub Actions 工作流

**`.github/workflows/pi-review.yml`**:
```yaml
name: PI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  pi-review:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install PI
        run: npm install -g @mariozechner/pi-coding-agent

      - name: Configure PI
        env:
          PI_API_KEY: ${{ secrets.PI_API_KEY }}
        run: |
          mkdir -p ~/.pi
          echo '{"provider":"openai","apiKey":"${PI_API_KEY}","model":"gpt-4o"}' > ~/.pi/config.json

      - name: Run PI Review
        id: review
        run: |
          REVIEW=$(pi --non-interactive "Review the changes in this PR" 2>&1)
          echo "result<<EOF" >> $GITHUB_OUTPUT
          echo "$REVIEW" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            const review = `${{ steps.review.outputs.result }}`
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🤖 PI Code Review\n\n${review}`
            })
```

### 自动化测试生成

**`.pi/skills/generate-tests.ts`**:
```typescript
export default {
  name: "generate_tests",
  description: "Generate tests for modified files",
  category: "testing",

  trigger: /generate\s+tests?(?:\s+for)?/i,

  handler: async (input, context) => {
    // 获取修改的源文件
    const gitStatus = await context.tools.exec("bash", {
      command: "git status --short"
    })

    const sourceFiles = gitStatus.content
      .split("\n")
      .filter(line => line.match(/\.(ts|js)$/))
      .map(line => line.substring(3))

    const generatedTests = []

    for (const file of sourceFiles) {
      // 读取源文件
      const sourceResult = await context.tools.exec("read", {
        filePath: file
      })

      if (!sourceResult.success) continue

      // 生成测试
      const testPrompt = `
        Generate comprehensive tests for this code:

        ${sourceResult.content}

        Use vitest. Include:
        - Unit tests for all functions
        - Edge cases
        - Error cases
        - Happy path tests
      `

      const testResult = await context.llm.chat([
        {
          role: "system",
          content: "Generate production-ready test code."
        },
        {
          role: "user",
          content: testPrompt
        }
      ])

      // 确定测试文件路径
      const testFile = file
        .replace("/src/", "/tests/")
        .replace(".ts", ".spec.ts")

      // 写入测试文件
      await context.tools.exec("write", {
        filePath: testFile,
        content: testResult.content || ""
      })

      generatedTests.push(testFile)
    }

    return {
      success: true,
      content: `Generated ${generatedTests.length} test files:\n${generatedTests.join("\n")}`
    }
  }
}
```

---

## 团队协作

### 1. 共会话协议

创建会话分享机制：

**`.pi/extensions/session-share/index.ts`**:
```typescript
import { defineExtension } from "@mariozechner/pi-coding-agent"

export default defineExtension({
  id: "session-share",
  name: "Session Sharing",
  version: "1.0.0",

  hooks: {
    onMessage: async (message, context) => {
      // 检测分享命令
      if (message.content.startsWith("/share")) {
        const sessionData = {
          id: context.session.id,
          messages: context.session.messages,
          timestamp: new Date().toISOString(),
          author: process.env.USER || "unknown"
        }

        // 导出到共享目录
        const sharedDir = ".pi/sessions/shared/"
        const filename = `session-${Date.now()}.json`

        await context.tools.exec("write", {
          filePath: sharedDir + filename,
          content: JSON.stringify(sessionData, null, 2)
        })

        return `Session shared: ${filename}`
      }
    }
  }
})
```

### 2. 团队知识库

**`.pi/skills/knowledge-base.ts`**:
```typescript
export default {
  name: "knowledge_base",
  description: "Search and use team knowledge base",
  category: "utility",

  trigger: /(?:search|lookup)\s+(?:knowledge|docs|kb)\s+(.+)/i,

  handler: async (input, context) => {
    const query = input.match(/(?:search|lookup)\s+(?:knowledge|docs|kb)\s+(.+)/i)?.[1]

    if (!query) {
      return {
        success: false,
        content: "Please provide a search query"
      }
    }

    // 搜索团队文档
    const searchResult = await context.tools.exec("bash", {
      command: `grep -r "${query}" docs/ .pi/knowledge/ --include="*.md" -l`
    })

    if (!searchResult.success) {
      return {
        success: false,
        content: "Knowledge base search failed"
      }
    }

    const files = searchResult.content?.split("\n").filter(Boolean) || []

    // 读取相关文件
    const relevantContent = []
    for (const file of files.slice(0, 5)) {  // 最多 5 个文件
      const result = await context.tools.exec("read", {
        filePath: file
      })

      if (result.success) {
        relevantContent.push({
          file,
          content: result.content
        })
      }
    }

    // 使用 LLM 总结
    const summaryPrompt = `
      Query: ${query}

      Relevant documents found:
      ${JSON.stringify(relevantContent.map(d => d.file))}

      Please summarize the most relevant information for this query.
    `

    const summaryResult = await context.llm.chat([
      {
        role: "system",
        content: "You are a knowledge base assistant. Provide concise, relevant answers."
      },
      {
        role: "user",
        content: summaryPrompt
      }
    ])

    return {
      success: true,
      content: summaryResult.content || "No relevant information found",
      data: {
        query,
        filesFound: files,
        sources: relevantContent
      }
    }
  }
}
```

### 3. 代码规范检查

**`.pi/skills/style-check.ts`**:
```typescript
export default {
  name: "style_check",
  description: "Check code against team style guide",
  category: "development",

  trigger: /check\s+(?:style|formatting|code\s+style)/i,

  handler: async (input, context) => {
    // 获取团队代码风格配置
    const styleConfig = await context.config.get("codeStyle.typescript")

    // 获取修改的文件
    const gitStatus = await context.tools.exec("bash", {
      command: "git status --short"
    })

    const files = gitStatus.content
      .split("\n")
      .filter(line => line.match(/\.(ts|tsx)$/))
      .map(line => line.substring(3))

    const issues = []

    for (const file of files) {
      const contentResult = await context.tools.exec("read", {
        filePath: file
      })

      if (!contentResult.success) continue

      const content = contentResult.content as string

      // 检查缩进
      const lines = content.split("\n")
      lines.forEach((line, index) => {
        const leadingWhitespace = line.match(/^\s*/)?.[0].length || 0

        if (leadingWhitespace % styleConfig.indent !== 0 && line.trim()) {
          issues.push({
            file,
            line: index + 1,
            issue: "Incorrect indentation",
            expected: `Multiples of ${styleConfig.indent}`,
            actual: leadingWhitespace
          })
        }
      })

      // 检查分号
      if (styleConfig.semi) {
        const missingSemi = content.match(/^[^/\s]*[^;{}]\s*$/gm)
        if (missingSemi) {
          issues.push({
            file,
            issue: "Missing semicolons",
            count: missingSemi.length
          })
        }
      }

      // 检查引号
      const singleQuotes = (content.match(/'/g) || []).length
      const doubleQuotes = (content.match(/"/g) || []).length

      if (styleConfig.singleQuote && doubleQuotes > singleQuotes) {
        issues.push({
          file,
          issue: "Should use single quotes instead of double quotes"
        })
      }
    }

    if (issues.length === 0) {
      return {
        success: true,
        content: "✓ All style checks passed!"
      }
    }

    // 格式化问题报告
    const report = issues.map(issue =>
      `- **${issue.file}**: ${issue.issue}`
    ).join("\n")

    return {
      success: false,
      content: `Style issues found:\n\n${report}`,
      data: { issues }
    }
  }
}
```

---

## 最佳实践

### 1. 版本控制

```bash
# .pi/ 目录结构
.pi/
├── config.json              # 团队配置（提交）
├── config.local.json        # 个人配置（不提交）
├── skills/                  # 团队 Skills（提交）
│   ├── commit.ts
│   ├── review.ts
│   └── pr-template.ts
├── extensions/              # 团队扩展（提交）
│   └── session-share/
└── sessions/                # 会话（不提交）
    └── .gitkeep
```

**`.gitignore`** 更新：
```
# 个人配置
.pi/config.local.json

# 会话和缓存
.pi/sessions/
.pi/cache/
.pi/logs/

# 但保留团队文件
!.pi/skills/
!.pi/extensions/
```

### 2. 文档标准化

创建团队文档模板：

**`.pi/templates/README.md`**:
```markdown
# Project Documentation Template

## Overview
<!-- Brief project description -->

## Setup
\`\`\`bash
npm install
npm run dev
\`\`\`

## Architecture
<!-- System architecture description -->

## Team Conventions
<!-- Coding standards, commit conventions, etc. -->

## PI Configuration
<!-- How to use PI with this project -->

## Troubleshooting
<!-- Common issues and solutions -->
```

### 3. 知识共享

设置定期知识同步会议：

```typescript
// .pi/skills/daily-sync.ts
export default {
  name: "daily_sync",
  description: "Generate daily sync summary",
  category: "utility",

  trigger: /daily\s+sync|standup/i,

  handler: async (input, context) => {
    // 获取今天的 commits
    const todayCommits = await context.tools.exec("bash", {
      command: "git log --since='1 day ago' --pretty=format:'%h %s' --author=$(git config user.name)"
    })

    // 获取未解决的问题
    const issues = await context.tools.exec("bash", {
      command: "gh issue list --state open --limit 5"
    })

    // 生成同步摘要
    const syncPrompt = `
      Generate a daily sync summary from:

      Recent commits:
      ${todayCommits.content}

      Open issues:
      ${issues.content}

      Include:
      - What was accomplished
      - What's in progress
      - Blockers
      - Next steps
    `

    const result = await context.llm.chat([
      {
        role: "system",
        content: "Generate concise standup summaries."
      },
      {
        role: "user",
        content: syncPrompt
      }
    ])

    return {
      success: true,
      content: `## Daily Sync\n\n${result.content}`
    }
  }
}
```

### 4. 安全实践

**敏感数据处理**：
```typescript
// .pi/skills/secure-redact.ts
export default {
  name: "secure_redact",
  description: "Redact sensitive data before sharing",
  category: "security",

  trigger: /redact\s+(?:sensitive|secrets)/i,

  handler: async (input, context) => {
    // 获取当前缓冲区内容
    const content = await context.tools.exec("get_buffer", {})

    if (!content.success) {
      return {
        success: false,
        content: "Failed to get content"
      }
    }

    // 定义敏感模式
    const sensitivePatterns = [
      { pattern: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g, replacement: "[EMAIL]" },
      { pattern: /\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b/g, replacement: "[CARD]" },
      { pattern: /sk-[a-zA-Z0-9]{48}/g, replacement: "[API_KEY]" },
      { pattern: /Bearer [a-zA-Z0-9\-._~+/]+=*/g, replacement: "[TOKEN]" }
    ]

    let redactedContent = content.content as string

    // 应用脱敏规则
    for (const { pattern, replacement } of sensitivePatterns) {
      redactedContent = redactedContent.replace(pattern, replacement)
    }

    // 保存脱敏版本
    await context.tools.exec("write", {
      filePath: ".pi/redacted-output.txt",
      content: redactedContent
    })

    return {
      success: true,
      content: "Sensitive data redacted. Safe to share.",
      data: {
        redactedFile: ".pi/redacted-output.txt"
      }
    }
  }
}
```

---

## 性能优化

### 1. 共享缓存

```typescript
// .pi/extensions/shared-cache/index.ts
import { defineExtension } from "@mariozechner/pi-coding-agent"

export default defineExtension({
  id: "shared-cache",
  name: "Shared Cache",
  version: "1.0.0",

  hooks: {
    onLoad: async (context) => {
      // 初始化共享缓存目录
      const cacheDir = ".pi/cache/shared"

      await context.tools.exec("bash", {
        command: `mkdir -p ${cacheDir}`
      })

      context.logger.info(`Shared cache initialized: ${cacheDir}`)
    }
  }
})
```

### 2. 并行处理

```typescript
// 在团队工作流中并行执行任务
export const parallelWorkflowSkill = {
  name: "parallel_workflow",
  description: "Run team tasks in parallel",
  category: "utility",

  trigger: /run\s+parallel\s+tasks/i,

  handler: async (input, context) => {
    const tasks = [
      context.tools.exec("bash", { command: "npm run lint" }),
      context.tools.exec("bash", { command: "npm run test:unit" }),
      context.tools.exec("bash", { command: "npm run type-check" })
    ]

    const results = await Promise.all(tasks)

    return {
      success: results.every(r => r.success),
      content: `Parallel workflow completed: ${results.filter(r => r.success).length}/${results.length} passed`,
      data: { results }
    }
  }
}
```

---

## 监控和分析

### 1. 使用统计

**`.pi/extensions/usage-stats/index.ts`**:
```typescript
import { defineExtension } from "@mariozechner/pi-coding-agent"

export default defineExtension({
  id: "usage-stats",
  name: "Usage Statistics",
  version: "1.0.0",

  hooks: {
    onMessage: async (message, context) => {
      // 记录使用统计
      const stats = {
        timestamp: new Date().toISOString(),
        user: process.env.USER,
        message: message.content.substring(0, 100),
        session: context.session.id
      }

      // 追加到统计文件
      await context.tools.exec("bash", {
        command: `echo '${JSON.stringify(stats)}' >> .pi/stats/usage.log`
      })
    }
  }
})
```

### 2. 团队洞察

```typescript
// .pi/skills/team-insights.ts
export default {
  name: "team_insights",
  description: "Generate team usage insights",
  category: "analytics",

  trigger: /team\s+insights|usage\s+report/i,

  handler: async (input, context) => {
    // 读取使用日志
    const logResult = await context.tools.exec("bash", {
      command: "cat .pi/stats/usage.log | tail -100"
    })

    if (!logResult.success) {
      return {
        success: false,
        content: "No usage data available"
      }
    }

    const logs = logResult.content
      .split("\n")
      .filter(Boolean)
      .map(line => JSON.parse(line))

    // 生成洞察
    const insightsPrompt = `
      Generate team insights from this usage data:

      ${JSON.stringify(logs, null, 2)}

      Include:
      - Most active users
      - Common tasks
      - Peak usage times
      - Recommendations
    `

    const result = await context.llm.chat([
      {
        role: "system",
        content: "Generate actionable team insights."
      },
      {
        role: "user",
        content: insightsPrompt
      }
    ])

    return {
      success: true,
      content: result.content || "Insights generated"
    }
  }
}
```

---

## 总结

✅ 你已经学会：
- 设置团队配置和环境
- 标准化工作流程
- 集成 CI/CD
- 实现团队协作功能
- 应用最佳实践
- 监控和分析使用情况

**团队集成清单**：
- ✓ 创建共享配置文件
- ✓ 设置环境变量管理
- ✓ 实现标准工作流 Skills
- ✓ 配置 CI/CD 集成
- ✓ 建立代码审查流程
- ✓ 设置知识共享机制
- ✓ 实现使用监控

**下一步**：
- [高级配置](./03-advanced-configuration.md)
- [编写扩展](../06-onboarding/03-writing-extension.md)
- [创建 Skill](../06-onboarding/06-creating-skill.md)

---

## 相关链接

- **终端用户指南**：`/LEARN/07-user-guide/01-end-user-guide.md`
- **高级配置**：`/LEARN/07-user-guide/03-advanced-configuration.md`
- **扩展系统**：`/LEARN/04-subsystems/02-extension-system.md`
