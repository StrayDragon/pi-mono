# pi.cli 设置项参考

> 源码位置：`packages/coding-agent/src/core/settings-manager.ts`

所有设置均写入 `~/.config/pi/settings.json`（Linux/macOS）或等效配置目录。环境变量可覆盖部分设置。

---

## 模型与思考

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `defaultProvider` | `string \| undefined` | `undefined` | 默认 AI 提供商名称，如 `"anthropic"`、`"openai"` |
| `defaultModel` | `string \| undefined` | `undefined` | 默认模型 ID |
| `defaultThinkingLevel` | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh"` | `"medium"` | 默认思考/预算级别 |
| `hideThinkingBlock` | `boolean` | `false` | 是否在终端输出中隐藏思考块 |
| `thinkingBudgets` | `{ minimal?: number; low?: number; medium?: number; high?: number; } \| undefined` | `undefined` | 各思考级别的自定义 token 预算，覆盖提供商默认值 |
| `transport` | `"sse" \| "websocket" \| "auto"` | `"auto"` | 优先使用的传输协议（部分提供商支持多协议时生效） |

---

## 消息发送模式

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `steeringMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | 转向/中断消息如何发送给模型：`"all"` 一次性发送，`"one-at-a-time"` 逐条发送 |
| `followUpMode` | `"all" \| "one-at-a-time"` | `"one-at-a-time"` | 后续消息如何发送给模型（同上） |

---

## 模型切换

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enabledModels` | `string[] \| undefined` | `undefined` | Ctrl+P 切换模型的匹配模式（支持 glob 和 `provider/model:thinking` 简写格式） |

---

## UI 与显示

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `theme` | `string \| undefined` | `"dark"` | 主题名称，如 `"dark"`、`"light"` 或自定义主题文件路径 |
| `quietStartup` | `boolean` | `false` | 启动时隐藏欢迎头部信息 |
| `collapseChangelog` | `boolean` | `false` | 更新后显示精简版 changelog（而非完整版） |
| `enableInstallTelemetry` | `boolean` | `true` | 向 `https://pi.dev/api/report-install` 发送匿名安装/版本 ping |
| `doubleEscapeAction` | `"fork" \| "tree" \| "none"` | `"tree"` | 编辑器为空时连续按两次 Escape 的行为 |
| `treeFilterMode` | `"default" \| "no-tools" \| "user-only" \| "labeled-only" \| "all"` | `"default"` | 执行 `/tree` 时默认应用的过滤器 |
| `editorPaddingX` | `number` | `0`（范围 0–3） | TUI 输入编辑器的水平内边距 |
| `autocompleteMaxVisible` | `number` | `5`（范围 3–20） | 自动补全下拉菜单最大可见条目数 |
| `showHardwareCursor` | `boolean` | `false` | 在 IME 定位的同时显示终端原生光标（环境变量 `PI_HARDWARE_CURSOR=1` 也可启用） |

---

## 终端与图片

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `terminal.showImages` | `boolean` | `true` | 是否在终端中显示内联图片（需终端支持） |
| `terminal.imageWidthCells` | `number` | `60` | 内联图片在终端中的首选宽度（单位：单元格） |
| `terminal.clearOnShrink` | `boolean` | `false` | 内容收缩时清空空行（可能产生闪烁，环境变量 `PI_CLEAR_ON_SHRINK=1` 也可启用） |
| `terminal.showTerminalProgress` | `boolean` | `false` | 启用 OSC 9;4 终端进度指示 |
| `images.autoResize` | `boolean` | `true` | 自动将图片缩放至最大 2000×2000 以提高模型兼容性 |
| `images.blockImages` | `boolean` | `false` | 禁止将任何图片发送给 LLM 提供商 |

---

## 对话压缩

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `compaction.enabled` | `boolean` | `true` | 是否启用自动对话压缩（聊天过长时自动摘要） |
| `compaction.reserveTokens` | `number` | `16384` | 压缩时为 LLM 回复预留的 token 数量 |
| `compaction.keepRecentTokens` | `number` | `20000` | 压缩时保留的不经摘要的最近 token 数 |

---

## 分支摘要

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `branchSummary.reserveTokens` | `number` | `16384` | 分支摘要 prompt 和回复预留的 token 数 |
| `branchSummary.skipPrompt` | `boolean` | `false` | 执行 `/tree` 导航时跳过"是否摘要？"确认，默认不摘要 |

---

## 重试

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `retry.enabled` | `boolean` | `true` | 是否启用 Agent 级自动重试（处理临时错误） |
| `retry.maxRetries` | `number` | `3` | Agent 级最大重试次数 |
| `retry.baseDelayMs` | `number` | `2000` | 指数退避基础延迟（毫秒），依次为 2s、4s、8s |
| `retry.provider.timeoutMs` | `number \| undefined` | `undefined`（SDK 默认） | 提供商/SDK 请求超时（毫秒） |
| `retry.provider.maxRetries` | `number \| undefined` | `undefined`（SDK 默认） | 提供商/SDK 重试次数 |
| `retry.provider.maxRetryDelayMs` | `number` | `60000` | 服务器请求的最大重试延迟上限（毫秒），`0` 取消上限 |

---

## Shell

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `shellPath` | `string \| undefined` | `undefined` | 自定义 shell 路径（如 Windows 上使用 Cygwin） |
| `shellCommandPrefix` | `string \| undefined` | `undefined` | 每个 bash 命令前添加的前缀（如 `"shopt -s expand_aliases"`） |
| `npmCommand` | `string[] \| undefined` | `undefined` | 自定义 npm 命令（argv 风格，如 `["mise", "exec", "node@20", "--", "npm"]`） |

---

## 会话

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `sessionDir` | `string \| undefined` | `undefined` | 自定义会话存储目录（支持绝对路径、相对路径和 `~`） |

---

## 资源加载

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `packages` | `PackageSource[]` | `[]` | npm/git 包来源，加载其中的扩展、技能、提示词和主题 |
| `extensions` | `string[]` | `[]` | 本地扩展文件路径（支持 glob 及排除语法） |
| `skills` | `string[]` | `[]` | 本地技能文件路径（支持 glob 及排除语法） |
| `prompts` | `string[]` | `[]` | 本地提示词模板路径（支持 glob 及排除语法） |
| `themes` | `string[]` | `[]` | 本地主题文件路径（支持 glob 及排除语法） |
| `enableSkillCommands` | `boolean` | `true` | 将技能注册为 `/skill:name` 斜杠命令 |

---

## Markdown 输出

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `markdown.codeBlockIndent` | `string` | `"  "`（两个空格） | 输出中代码块的缩进字符串 |

---

## 警告

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `warnings.anthropicExtraUsage` | `boolean` | `true` | 当 Anthropic 订阅认证可能产生付费额外用量时显示警告 |

---

## 内部元数据（用户通常不直接修改）

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `lastChangelogVersion` | `string \| undefined` | `undefined` | 记录上次向用户展示的 changelog 版本（由程序管理） |

---

## 环境变量覆盖

以下环境变量可覆盖对应设置或全局行为：

| 环境变量 | 覆盖内容 |
|----------|----------|
| `PI_CODING_AGENT_DIR` | 配置文件目录 |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖 `sessionDir` |
| `PI_CLEAR_ON_SHRINK=1` | 启用 `terminal.clearOnShrink` |
| `PI_HARDWARE_CURSOR=1` | 启用 `showHardwareCursor` |
| `PI_TELEMETRY=1` 或 `PI_TELEMETRY=0` | 覆盖 `enableInstallTelemetry` |
| `PI_SKIP_VERSION_CHECK=1` | 禁用版本检查 |
| `PI_OFFLINE=1` | 禁用所有启动时网络请求 |
| `PI_PACKAGE_DIR` | 覆盖包解析目录 |
| `PI_SHARE_VIEWER_URL` | 覆盖分享预览 URL |
