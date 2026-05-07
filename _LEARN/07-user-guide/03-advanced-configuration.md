# 高级配置 (Advanced Configuration)

## 概述

本指南深入介绍 pi-coding-agent 的高级配置选项，包括性能调优、安全设置、自定义主题、快捷键绑定等高级功能。

---

## 配置文件详解

### 完整配置 Schema

**`~/.pi/config.json`** 或项目根目录的 **`.pi/config.json`**:

```json
{
  "$schema": "https://raw.githubusercontent.com/mariozechner/pi-mono/main/packages/coding-agent/src/config/schema.json",

  // ===== 核心 LLM 配置 =====
  "provider": "openai",
  "apiKey": "sk-...",
  "model": "gpt-4o",
  "baseURL": "https://api.openai.com/v1",
  "timeout": 30000,

  // ===== LLM 参数 =====
  "maxTokens": 8000,
  "temperature": 0.7,
  "topP": 1.0,
  "frequencyPenalty": 0,
  "presencePenalty": 0,
  "stopSequences": [],

  // ===== 多 Provider 配置 =====
  "providers": {
    "openai": {
      "provider": "openai",
      "apiKey": "sk-...",
      "model": "gpt-4o"
    },
    "anthropic": {
      "provider": "anthropic",
      "apiKey": "sk-ant-...",
      "model": "claude-3-5-sonnet-20241022",
      "headers": {
        "anthropic-version": "2023-06-01"
      }
    },
    "ollama": {
      "provider": "ollama",
      "endpoint": "http://localhost:11434",
      "model": "llama3.1"
    }
  },

  // ===== 上下文管理 =====
  "context": {
    "maxHistory": 100,
    "maxTokens": 16000,
    "compressionThreshold": 0.8
  },

  // ===== 上下文压缩 =====
  "compaction": {
    "enabled": true,
    "threshold": 0.8,
    "minDistance": 10,
    "maxTokens": 6000,
    "provider": "openai",
    "model": "gpt-4o",
    "strategy": "smart"
  },

  // ===== 系统提示 =====
  "systemPrompt": {
    "role": "You are an expert programming assistant",
    "guidelines": [
      "Always write clean, maintainable code",
      "Include error handling",
      "Add comments for complex logic",
      "Follow best practices"
    ]
  },

  // ===== 主题 =====
  "theme": "default",
  "colorMode": "auto",
  "themeOverrides": {
    "primary": "#00ff00",
    "secondary": "#008800",
    "background": "#000000",
    "foreground": "#ffffff"
  },

  // ===== 快捷键 =====
  "keybindings": {
    "send": ["enter"],
    "cancel": ["ctrl+c"],
    "exit": ["ctrl+d"],
    "branch": ["ctrl+b"],
    "tree": ["ctrl+t"],
    "help": ["?"],
    "custom": {
      "save": {
        "keys": ["ctrl+s"],
        "action": "write_session"
      }
    }
  },

  // ===== 工具配置 =====
  "tools": {
    "enabled": ["read", "write", "edit", "bash", "git"],
    "disabled": [],
    "permissions": {
      "bash": {
        "allowDestructive": false,
        "allowedCommands": ["git", "npm", "ls", "cat"]
      },
      "write": {
        "confirmOverwrite": true
      }
    },
    "timeout": 5000
  },

  // ===== 会话管理 =====
  "sessions": {
    "directory": "~/.pi/sessions",
    "autoSave": true,
    "maxSessions": 100,
    "compression": true,
    "sync": {
      "enabled": false,
      "remote": "git@github.com:user/pi-sessions.git"
    }
  },

  // ===== 扩展配置 =====
  "extensions": {
    "directory": "~/.pi/extensions",
    "autoLoad": true,
    "disabled": [],
    "trusted": ["@mariozechner/*"]
  },

  // ===== 日志配置 =====
  "logging": {
    "level": "info",
    "directory": "~/.pi/logs",
    "maxFiles": 10,
    "maxSize": "10M",
    "format": "json"
  },

  // ===== 性能配置 =====
  "performance": {
    "cacheEnabled": true,
    "cacheSize": 1000,
    "parallelism": 4,
    "streaming": true,
    "bufferSize": 8192
  },

  // ===== 安全配置 =====
  "security": {
    "sandboxMode": false,
    "confirmDestructive": true,
    "maxFileSize": 10485760,
    "allowedDomains": ["github.com", "npmjs.org"],
    "dataRedaction": true
  },

  // ===== 网络配置 =====
  "network": {
    "proxy": "http://proxy.example.com:8080",
    "retryAttempts": 3,
    "retryDelay": 1000,
    "timeout": 30000
  },

  // ===== Git 集成 =====
  "git": {
    "autoCommit": false,
    "commitTemplate": "conventional",
    "branchNaming": "feature/,bugfix/,hotfix/",
    "prTemplate": ".github/PULL_REQUEST_TEMPLATE.md"
  },

  // ===== 代码风格 =====
  "codeStyle": {
    "typescript": {
      "indent": 2,
      "semi": true,
      "singleQuote": false,
      "trailingComma": "es5",
      "printWidth": 80
    },
    "python": {
      "indent": 4,
      "quotes": "double"
    }
  },

  // ===== 团队约定 =====
  "conventions": {
    "commitMessage": "conventional",
    "branchNaming": "feature/,bugfix/,hotfix/",
    "codeReview": "required",
    "testCoverage": 80
  },

  // ===== 实验性功能 =====
  "experimental": {
    "features": [],
    "beta": false
  }
}
```

---

## 高级主题配置

### 自定义主题

创建自定义主题文件 **`~/.pi/themes/my-theme.json`**:

```json
{
  "name": "My Custom Theme",
  "description": "A dark theme with green accents",
  "colors": {
    "primary": "#00ff00",
    "secondary": "#00cc00",
    "success": "#00ff00",
    "warning": "#ffcc00",
    "error": "#ff0000",
    "info": "#00ccff",

    "background": {
      "default": "#0a0a0a",
      "dim": "#050505",
      "bright": "#101010"
    },

    "foreground": {
      "primary": "#ffffff",
      "secondary": "#cccccc",
      "muted": "#666666"
    },

    "border": {
      "default": "#333333",
      "focused": "#00ff00",
      "muted": "#1a1a1a"
    },

    "syntax": {
      "keyword": "#ff00ff",
      "string": "#00ff00",
      "comment": "#666666",
      "function": "#00ccff",
      "number": "#ffcc00",
      "operator": "#ffffff"
    },

    "ui": {
      "header": "#1a1a1a",
      "footer": "#1a1a1a",
      "panel": "#0f0f0f",
      "input": "#0a0a0a",
      "selection": "#1a3a1a"
    }
  },

  "styles": {
    "bold": true,
    "italic": false,
    "underline": false
  },

  "colorMode": "truecolor"
}
```

在配置中使用：

```json
{
  "theme": "my-theme",
  "themeOverrides": {
    "primary": "#00ff00",
    "background": "#0a0a0a"
  }
}
```

### 16 色主题（适用于受限终端）

```json
{
  "name": "16-color Terminal",
  "colors": {
    "primary": "green",
    "secondary": "brightGreen",
    "success": "green",
    "warning": "yellow",
    "error": "red",
    "info": "blue",

    "background": "black",
    "foreground": "white",
    "muted": "brightBlack",

    "border": "brightBlack",
    "focused": "green",

    "syntax": {
      "keyword": "magenta",
      "string": "green",
      "comment": "brightBlack",
      "function": "cyan",
      "number": "yellow",
      "operator": "white"
    }
  },

  "colorMode": "16color"
}
```

---

## 高级快捷键配置

### 模式绑定

```json
{
  "keybindings": {
    "normal": {
      "send": "enter",
      "newline": "alt+enter",
      "cancel": "ctrl+c",
      "exit": "ctrl+d"
    },

    "vim": {
      "send": "enter",
      "cancel": "ctrl+c",
      "exit": ":q",
      "save": ":w"
    },

    "emacs": {
      "send": "ctrl+j",
      "cancel": "ctrl+g",
      "exit": "ctrl+x ctrl+c",
      "save": "ctrl+x ctrl+s"
    }
  }
}
```

### 自定义快捷键动作

```json
{
  "keybindings": {
    "custom": {
      "save_session": {
        "keys": ["ctrl+s"],
        "action": "write_session",
        "params": {
          "path": "~/.pi/sessions/auto-save.json"
        }
      },

      "export_markdown": {
        "keys": ["ctrl+e"],
        "action": "export",
        "params": {
          "format": "markdown",
          "path": "~/exports/session-{timestamp}.md"
        }
      },

      "toggle_debug": {
        "keys": ["ctrl+shift+d"],
        "action": "toggle_debug"
      },

      "run_tests": {
        "keys": ["ctrl+t"],
        "action": "exec_tool",
        "params": {
          "tool": "bash",
          "command": "npm test"
        }
      }
    }
  }
}
```

---

## 性能调优

### 流式响应优化

```json
{
  "performance": {
    "streaming": {
      "enabled": true,
      "bufferSize": 8192,
      "flushInterval": 50,
      "chunkTimeout": 5000
    },

    "caching": {
      "enabled": true,
      "strategy": "lru",
      "maxSize": 1000,
      "ttl": 3600000
    },

    "parallelism": {
      "toolExecution": 4,
      "fileOperations": 2
    }
  }
}
```

### 内存管理

```json
{
  "memory": {
    "maxContextSize": 104857600,
    "maxCacheSize": 52428800,
    "compaction": {
      "aggressive": true,
      "targetRatio": 0.7
    },

    "gc": {
      "enabled": true,
      "interval": 300000,
      "threshold": 0.8
    }
  }
}
```

### 网络优化

```json
{
  "network": {
    "keepAlive": true,
    "keepAliveMsecs": 1000,
    "maxSockets": 100,
    "maxFreeSockets": 10,

    "timeout": {
      "socket": 30000,
      "response": 60000,
      "keepAlive": 60000
    },

    "compression": {
      "enabled": true,
      "threshold": 1024
    }
  }
}
```

---

## 安全配置

### 沙箱模式

```json
{
  "security": {
    "sandboxMode": true,
    "allowedOperations": [
      "read",
      "write",
      "bash:safe"
    ],

    "sandboxRules": {
      "bash": {
        "allowedCommands": [
          "git",
          "npm",
          "node",
          "python",
          "ls",
          "cat",
          "grep"
        ],
        "blockedCommands": [
          "rm",
          "sudo",
          "chmod",
          "chown"
        ],
        "maxExecutionTime": 10000
      },

      "write": {
        "allowedPaths": [
          "./src",
          "./tests",
          "./docs"
        ],
        "blockedPaths": [
          "/etc",
          "/usr",
          "/bin",
          "/sbin"
        ],
        "maxFileSize": 1048576
      },

      "network": {
        "allowedDomains": [
          "api.github.com",
          "registry.npmjs.org",
          "*.example.com"
        ],
        "blockedDomains": [
          "*.malicious.com"
        ]
      }
    }
  }
}
```

### 数据脱敏

```json
{
  "security": {
    "dataRedaction": {
      "enabled": true,
      "patterns": [
        {
          "name": "email",
          "pattern": "\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b",
          "replacement": "[REDACTED_EMAIL]"
        },
        {
          "name": "api_key",
          "pattern": "sk-[a-zA-Z0-9]{48}",
          "replacement": "[REDACTED_KEY]"
        },
        {
          "name": "token",
          "pattern": "Bearer [a-zA-Z0-9\\-._~+/]+=*",
          "replacement": "Bearer [REDACTED]"
        },
        {
          "name": "password",
          "pattern": "(?i)password['\"\\s]*[:=]['\"\\s]*[^'\"\\s]+",
          "replacement": "password=[REDACTED]"
        }
      ]
    }
  }
}
```

### 审计日志

```json
{
  "security": {
    "audit": {
      "enabled": true,
      "logFile": "~/.pi/logs/audit.jsonl",
      "events": [
        "tool_execution",
        "file_write",
        "bash_command",
        "api_request",
        "config_change"
      ],
      "format": "json",
      "rotation": {
        "enabled": true,
        "maxFiles": 30,
        "maxSize": "100M"
      }
    }
  }
}
```

---

## 多模型配置

### 模型路由策略

```json
{
  "modelRouting": {
    "strategy": "cost",
    "rules": [
      {
        "condition": "task == 'code_generation'",
        "model": "gpt-4o",
        "reason": "Best for complex code generation"
      },
      {
        "condition": "task == 'simple_qa'",
        "model": "gpt-4o-mini",
        "reason": "Faster and cheaper for simple tasks"
      },
      {
        "condition": "task == 'summarization'",
        "model": "claude-3-haiku",
        "reason": "Good at summarization, cost-effective"
      }
    ],

    "fallback": {
      "enabled": true,
      "model": "gpt-4o-mini",
      "threshold": 3
    }
  }
}
```

### 成本优化配置

```json
{
  "costOptimization": {
    "enabled": true,
    "budget": {
      "daily": 10.0,
      "monthly": 200.0
    },

    "alerts": {
      "threshold": 0.8,
      "action": "warn"
    },

    "preferredModels": {
      "default": "gpt-4o-mini",
      "complex": "gpt-4o",
      "simple": "gpt-4o-mini"
    },

    "caching": {
      "enabled": true,
      "ttl": 86400
    }
  }
}
```

---

## 插件和扩展配置

### 扩展加载策略

```json
{
  "extensions": {
    "loadStrategy": "lazy",
    "priority": [
      "team-extensions",
      "user-extensions",
      "builtin-extensions"
    ],

    "dependencies": {
      "autoResolve": true,
      "versionConflicts": "warn"
    },

    "sandbox": {
      "enabled": false,
      "allowNetwork": false
    }
  }
}
```

### 扩展权限配置

```json
{
  "extensions": {
    "permissions": {
      "default": {
        "tools": ["read", "write"],
        "network": false,
        "filesystem": "./workspace"
      },

      "trusted": {
        "@mariozechner/*": {
          "tools": ["*"],
          "network": true,
          "filesystem": "*"
        }
      },

      "untrusted": {
        "tools": ["read"],
        "network": false,
        "filesystem": "./readonly"
      }
    }
  }
}
```

---

## Git 集成配置

### 智能 Git 操作

```json
{
  "git": {
    "autoDetect": true,
    "smartCommit": true,
    "commitTemplate": {
      "type": "conventional",
      "scopes": ["feat", "fix", "docs", "style", "refactor", "test", "chore"],
      "requireBody": false,
      "requireFooter": false
    },

    "branchProtection": {
      "enabled": true,
      "protected": ["main", "master", "develop"],
      "requireClean": true
    },

    "prIntegration": {
      "enabled": true,
      "autoCreate": false,
      "template": ".github/PULL_REQUEST_TEMPLATE.md",
      "labels": ["auto-generated", "pi-assisted"]
    }
  }
}
```

### Git Hook 集成

```json
{
  "git": {
    "hooks": {
      "pre-commit": {
        "enabled": true,
        "commands": [
          "npm run lint",
          "npm run test:unit"
        ]
      },

      "pre-push": {
        "enabled": true,
        "commands": [
          "npm run test:integration",
          "npm run build"
        ]
      },

      "commit-msg": {
        "enabled": true,
        "validate": "conventional"
      }
    }
  }
}
```

---

## 高级系统提示配置

### 分层系统提示

```json
{
  "systemPrompt": {
    "core": {
      "role": "You are an expert programming assistant",
      "version": "1.0.0"
    },

    "personality": {
      "tone": "professional but friendly",
      "style": "concise and practical",
      "language": "TypeScript and JavaScript focused"
    },

    "capabilities": [
      "Code generation and refactoring",
      "Debugging and troubleshooting",
      "Code review and optimization",
      "Testing and documentation"
    ],

    "guidelines": [
      "Always write clean, maintainable code",
      "Include error handling",
      "Add comments for complex logic",
      "Follow best practices",
      "Consider performance implications"
    ],

    "constraints": [
      "Never write insecure code",
      "Always validate user input",
      "Avoid unnecessary complexity",
      "Prefer standard library solutions"
    ],

    "projectSpecific": {
      "framework": "React",
      "styling": "Tailwind CSS",
      "testing": "Vitest",
      "buildTool": "Vite"
    },

    "teamConventions": {
      "commitStyle": "conventional",
      "codeStyle": "Airbnb",
      "branchNaming": "feature/,bugfix/,hotfix/",
      "prTemplate": ".github/PULL_REQUEST_TEMPLATE.md"
    }
  }
}
```

### 动态系统提示

```typescript
// 高级：根据项目动态生成系统提示
const generateSystemPrompt = async (project: Project) => {
  const packageJson = await readPackageJson()

  return {
    role: "You are a programming assistant for this project",

    project: {
      name: packageJson.name,
      type: detectProjectType(packageJson),
      dependencies: Object.keys(packageJson.dependencies),
      scripts: Object.keys(packageJson.scripts)
    },

    conventions: await loadTeamConventions(),
    architecture: await loadArchitectureDocs(),
    patterns: await loadDesignPatterns()
  }
}
```

---

## 配置验证

### 使用 Schema 验证

```bash
# 验证配置文件
pi --validate-config

# 输出示例
✓ Configuration is valid
✓ Provider: openai
✓ Model: gpt-4o
✓ All required fields present
⚠️  Warning: API key contains 'sk-' prefix (good)
```

### 配置调试

```bash
# 显示当前配置
pi --show-config

# 显示特定配置
pi --show-config provider
pi --show-config theme
pi --show-config keybindings

# 测试配置
pi --test-config

# 比较配置
pi --diff-config ~/.pi/config.json .pi/config.json
```

---

## 配置管理最佳实践

### 1. 环境分离

```bash
# 开发环境
.pi/config.development.json

# 测试环境
.pi/config.testing.json

# 生产环境
.pi/config.production.json

# 使用环境变量切换
export PI_ENV=production
pi --config ~/.pi/config.${PI_ENV}.json
```

### 2. 配置继承

**`base-config.json`**:
```json
{
  "provider": "openai",
  "model": "gpt-4o",
  "theme": "default"
}
```

**`team-config.json`**:
```json
{
  "extends": "./base-config.json",
  "tools": {
    "enabled": ["read", "write", "bash", "git"]
  }
}
```

### 3. 配置版本控制

```bash
# .pi/ 目录结构
.pi/
├── config.json.template      # 配置模板（提交）
├── config.local.json          # 本地配置（不提交）
├── config.team.json           # 团队配置（提交）
└── .gitkeep
```

**`.gitignore`**:
```
.pi/config.local.json
.pi/config.*.local.json
```

### 4. 配置迁移

维护配置迁移脚本：

```typescript
// scripts/migrate-config.ts
import { readFileSync, writeFileSync } from "fs"
import { join } from "path"

const migrations = [
  {
    version: "0.70.0",
    migrate: (config: any) => {
      // 添加新字段
      if (!config.performance) {
        config.performance = {
          cacheEnabled: true,
          streaming: true
        }
      }

      // 重命名字段
      if (config.maxHistory) {
        config.context = config.context || {}
        config.context.maxHistory = config.maxHistory
        delete config.maxHistory
      }

      return config
    }
  }
]

function migrateConfig(configPath: string) {
  const config = JSON.parse(readFileSync(configPath, "utf-8"))
  const currentVersion = config.version || "0.0.0"

  for (const migration of migrations) {
    if (currentVersion < migration.version) {
      console.log(`Migrating to ${migration.version}...`)
      migration.migrate(config)
      config.version = migration.version
    }
  }

  writeFileSync(configPath, JSON.stringify(config, null, 2))
  console.log("Configuration migrated successfully")
}
```

---

## 总结

✅ 你已经学会：
- 理解完整配置 Schema
- 自定义主题和快捷键
- 性能调优和内存管理
- 安全配置和数据脱敏
- 多模型配置和成本优化
- 插件和扩展配置
- Git 集成配置
- 系统提示配置
- 配置验证和管理

**配置清单**：
- ✓ 设置合适的 Provider 和模型
- ✓ 配置主题和快捷键
- ✓ 优化性能和内存使用
- ✓ 启用安全功能
- ✓ 配置团队约定
- ✓ 验证配置文件

**相关资源**：
- **终端用户指南**：`/LEARN/07-user-guide/01-end-user-guide.md`
- **团队集成**：`/LEARN/07-user-guide/02-team-integration.md`
- **扩展系统**：`/LEARN/04-subsystems/02-extension-system.md`
