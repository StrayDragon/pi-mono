# 功能全景图

## 功能总览

```mermaid
mindmap
  root((Pi 功能))
    核心编码
      文件读取 read
      命令执行 bash
      精确编辑 edit
      文件创建 write
      代码搜索 grep
      文件查找 find
      目录列表 ls
    多供应商 LLM
      Anthropic Claude
      OpenAI GPT/o
      Google Gemini
      Mistral
      Amazon Bedrock
      DeepSeek
      xAI Grok
      30+ 供应商
    交互界面
      差分渲染 TUI
      Markdown 渲染
      语法高亮
      内联图片 Kitty
      多行编辑器
      自动补全
    会话管理
      JSONL 持久化
      树状分支
      上下文压缩
      HTML 导出
      会话恢复
    可扩展性
      TypeScript 扩展
      自定义工具
      自定义供应商
      Slash 命令
      快捷键注册
      UI 组件定制
    多种模式
      Interactive TUI
      Print 模式
      RPC JSON-L
      SDK 编程接口
    安全与质量
      精确版本锁定
      Shrinkwrap
      npm audit
      贡献者门控
```

## 按领域分类

### 一、LLM 集成

#### 支持的 API 协议（9 种）

| API 协议 | 对应供应商 | 特性 |
|----------|-----------|------|
| `anthropic-messages` | Anthropic, Fireworks | 流式工具调用, 扩展思考 |
| `openai-completions` | OpenAI, OpenRouter, DeepSeek, Groq, xAI | 最广泛兼容 |
| `openai-responses` | OpenAI 原生 | Responses API |
| `openai-codex-responses` | OpenAI Codex | 云端沙箱执行 |
| `azure-openai-responses` | Azure OpenAI | 企业级 |
| `google-generative-ai` | Google AI Studio | Gemini 系列 |
| `google-vertex` | Google Cloud Vertex | 企业级 |
| `mistral-conversations` | Mistral AI | Mistral 系列 |
| `bedrock-converse-stream` | AWS Bedrock | 多模型接入 |

#### 支持的供应商（30+）

```mermaid
graph TB
    subgraph 一线供应商
        A1["Anthropic"]
        A2["OpenAI"]
        A3["Google"]
        A4["Mistral"]
    end

    subgraph 云平台
        B1["Amazon Bedrock"]
        B2["Azure OpenAI"]
        B3["Google Vertex"]
    end

    subgraph 聚合平台
        C1["OpenRouter"]
        C2["Vercel AI Gateway"]
        C3["Cloudflare"]
        C4["Fireworks"]
        C5["Together"]
    end

    subgraph 专业供应商
        D1["DeepSeek"]
        D2["xAI / Grok"]
        D3["Groq"]
        D4["Cerebras"]
        D5["HuggingFace"]
        D6["Kimi Coding"]
    end

    subgraph 区域供应商
        E1["MiniMax"]
        E2["Moonshot AI"]
        E3["Xiaomi"]
    end
```

#### 模型能力

| 能力 | 说明 |
|------|------|
| 工具调用 | 所有模型必须支持（入选门槛） |
| 推理思考 | 可调节 minimal/low/medium/high/xhigh |
| 图片输入 | 支持视觉理解的模型 |
| 提示缓存 | 减少重复 token 成本 |
| Token 追踪 | 输入/输出/缓存 token 计数和成本 |
| 跨供应商切换 | 同会话内切换，自动消息格式转换 |

### 二、工具系统

```mermaid
graph TB
    subgraph 默认启用
        R["read<br/>读取文件"]
        B["bash<br/>执行命令"]
        E["edit<br/>编辑文件"]
        W["write<br/>创建文件"]
    end

    subgraph 可选启用
        G["grep<br/>正则搜索"]
        F["find<br/>文件查找"]
        L["ls<br/>目录列表"]
    end

    subgraph 扩展工具
        X["自定义工具<br/>via registerTool()"]
    end

    style R fill:#c8e6c9
    style B fill:#c8e6c9
    style E fill:#c8e6c9
    style W fill:#c8e6c9
    style G fill:#fff9c4
    style F fill:#fff9c4
    style L fill:#fff9c4
    style X fill:#e1f5fe
```

#### 工具详情

| 工具 | 参数 | 输出 | 安全控制 |
|------|------|------|---------|
| `read` | `file_path`, `offset`, `limit` | 文件内容（含行号） | 路径验证 |
| `bash` | `command`, `timeout`, `description` | stdout/stderr | spawn hook, 超时 |
| `edit` | `file_path`, `old_string`, `new_string` | diff 预览 | 文件锁队列 |
| `write` | `file_path`, `content` | 成功/失败 | 文件锁队列 |
| `grep` | `pattern`, `path`, `include` | 匹配行和上下文 | 路径限制 |
| `find` | `pattern`, `path`, `type` | 文件路径列表 | 路径限制 |
| `ls` | `path`, `ignore` | 目录树 | 路径限制 |

### 三、会话管理

```mermaid
graph TB
    subgraph 会话树
        Root["会话根节点"]
        M1["用户消息 1"]
        A1["助手回复 1"]
        M2["用户消息 2"]
        A2["助手回复 2"]
        Fork["分支点"]
        B1["分支 A"]
        B2["分支 B"]
        Comp["压缩节点"]

        Root --> M1 --> A1 --> M2 --> A2
        A2 --> Fork
        Fork --> B1
        Fork --> B2
        A1 --> Comp
    end

    subgraph 操作
        Create["创建会话"]
        Resume["恢复会话"]
        ForkOp["分支 fork"]
        Nav["树导航"]
        Compact["压缩"]
        Export["HTML 导出"]
    end
```

#### 会话条目类型

| 类型 | 说明 | 在 LLM 上下文中 |
|------|------|-----------------|
| `message` | 用户/助手/工具结果消息 | 是 |
| `model_change` | 模型切换记录 | 否（元数据） |
| `thinking_level_change` | 思考级别变更 | 否（元数据） |
| `compaction` | 上下文压缩摘要 | 是（替换旧消息） |
| `branch_summary` | 分支摘要 | 是（注入上下文） |
| `custom` | 扩展自定义数据 | 否 |
| `custom_message` | 扩展注入的上下文 | 是 |
| `label` | 用户标记 | 否 |
| `session_info` | 会话元数据 | 否 |

### 四、扩展系统

```mermaid
graph TB
    subgraph 扩展能力
        EV["事件订阅<br/>30+ 事件类型"]
        TL["注册工具"]
        CM["Slash 命令"]
        SC["快捷键"]
        UI["UI 组件"]
        PR["自定义供应商"]
        AC["自动补全"]
        ED["自定义编辑器"]
    end

    subgraph 事件生命周期
        SS["session_start"]
        BA["before_agent_start"]
        AS["agent_start"]
        TS["turn_start"]
        TC["tool_call"]
        TR["tool_result"]
        TE["turn_end"]
        AE["agent_end"]
        SD["session_shutdown"]
    end

    SS --> BA --> AS --> TS --> TC --> TR --> TE --> AE --> SD
```

#### 扩展发现路径

| 路径 | 优先级 | 说明 |
|------|--------|------|
| CLI `--extensions` | 最高 | 命令行指定 |
| `.pi/extensions/` | 项目级 | 项目内扩展 |
| `~/.pi/agent/extensions/` | 用户级 | 全局扩展 |
| settings `extensions[]` | 配置级 | 设置文件引用 |

### 五、技能系统

```mermaid
flowchart TD
    A["Agent 发现需要技能"] --> B["读取 SKILL.md"]
    B --> C["注入技能内容<br/>到用户消息"]
    C --> D["Agent 按技能指引行动"]

    E["技能发现路径"]
    E --> F["~/.pi/agent/skills/"]
    E --> G[".pi/skills/"]
    E --> H["~/.agents/skills/"]
    E --> I["扩展动态添加"]
```

### 六、配置系统

```mermaid
graph TB
    subgraph 配置层次
        G["全局配置<br/>~/.pi/agent/"]
        P["项目配置<br/>.pi/"]
        C["CLI 参数"]
        E["环境变量"]
    end

    E --> C --> P --> G

    subgraph 配置文件
        S["settings.json"]
        A["auth.json"]
        M["models.json"]
        K["keybindings.json"]
        T["themes/*.json"]
    end
```

| 配置项 | 全局 | 项目 | CLI | 说明 |
|--------|------|------|-----|------|
| 默认模型 | `settings.json` | `settings.json` | `--model` | 启动时使用的模型 |
| API Key | `auth.json` / 环境变量 | - | `--api-key` | LLM 认证 |
| 主题 | `settings.json` | `settings.json` | - | UI 主题 |
| 快捷键 | `keybindings.json` | - | - | 键绑定覆盖 |
| 扩展 | `settings.json` | `settings.json` | `--extensions` | 扩展列表 |
| 工具 | - | - | `--tools` | 启用的工具 |
| 思考级别 | `settings.json` | - | `--thinking` | 默认推理级别 |

### 七、安全特性

| 特性 | 实现 |
|------|------|
| 精确依赖版本 | `save-exact=true` + 手动 review |
| Shrinkwrap | 发布包锁定完整依赖树 |
| npm audit | 定时 CI 扫描 |
| 贡献者门控 | 新贡献者 issue/PR 自动关闭 |
| 生命周期脚本 | `--ignore-scripts` 默认 |
| 最小发布龄期 | `min-release-age=2` |
| Lockfile 提交保护 | 预提交钩子检查 |
