# 技能系统

技能系统实现 [Agent Skills 标准](https://agentskills.io)，为特定任务提供可复用的 Markdown 指令。核心实现位于 `packages/coding-agent/src/core/skills.ts`，由 `ResourceLoader` 聚合多路径来源，经 `formatSkillsForPrompt()` 注入系统提示，并在 TUI 中以专用组件渲染。

---

## 架构概览

```mermaid
flowchart TB
    subgraph 发现层
        PM[PackageManager.resolve]
        RL[ResourceLoader.reload]
        EXT[extensions: resources_discover]
        CLI[CLI --skills]
    end

    subgraph 加载层
        LS[loadSkills / loadSkillsFromDir]
        VAL[名称与描述校验]
        MAP[skillMap 去重与冲突检测]
    end

    subgraph 运行时
        SP[buildSystemPrompt]
        FSF[formatSkillsForPrompt]
        AG[Agent 会话]
        READ[read 工具]
        TUI[SkillInvocationMessageComponent]
    end

    PM --> RL
    EXT --> RL
    CLI --> RL
    RL --> LS
    LS --> VAL --> MAP
    MAP --> SP
    SP --> FSF --> AG
    AG -->|模型选择技能| READ
    AG -->|/skill:name| EXP[expandSkillCommand]
    EXP -->|XML skill 块| TUI
    READ -->|读取 SKILL.md| AG
```

---

## SKILL.md 格式

每个技能是一个 Markdown 文件，**必须**包含 YAML frontmatter 与正文指令。

```yaml
---
name: my-skill              # 可选；默认取 SKILL.md 所在目录名
description: 简短说明用途   # 必需；最长 1024 字符
disable-model-invocation: false  # 可选；true 时不出现在系统提示中
---

# 技能正文

在此编写任务专用指令、步骤、示例等。
```

| 字段 | 说明 |
|------|------|
| `name` | 小写 `a-z`、数字、连字符；最长 64 字符；不能以 `-` 开头/结尾；不能含 `--` |
| `description` | 必填；供模型判断何时加载该技能 |
| `disable-model-invocation` | 为 `true` 时仅可通过 `/skill:name` 显式调用，不会写入 `<available_skills>` |

### 目录发现规则

`loadSkillsFromDir()` 按以下规则扫描：

1. 若目录内存在 `SKILL.md`，视为技能根目录，**不再**向下递归
2. 否则扫描根目录下直接 `.md` 文件
3. 递归进入子目录查找 `SKILL.md`
4. 尊重 `.gitignore` / `.ignore` / `.fdignore`，跳过 `node_modules`

```mermaid
flowchart TD
    START[进入目录 dir] --> HAS{存在 SKILL.md?}
    HAS -->|是| LOAD[loadSkillFromFile]
    HAS -->|否| SCAN[遍历条目]
    SCAN --> DOT{以 . 开头?}
    DOT -->|是| SKIP1[跳过]
    DOT -->|否| NM{node_modules?}
    NM -->|是| SKIP2[跳过]
    NM -->|否| DIR{是目录?}
    DIR -->|是| RECUR[递归 loadSkillsFromDirInternal]
    DIR -->|否| MD{根目录 .md 文件?}
    MD -->|是| LOAD
    MD -->|否| SKIP3[跳过]
    LOAD --> END[返回 Skill]
```

---

## 发现路径

技能由 `PackageManager` 与 `ResourceLoader` 统一解析，最终传入 `loadSkills()` 的 `skillPaths`。

| 来源 | 路径 | 作用域 |
|------|------|--------|
| 用户默认 | `~/.pi/agent/skills/` | 全局用户技能 |
| 项目默认 | `.pi/skills/`（相对 cwd） | 项目级技能 |
| Agent Skills 用户 | `~/.agents/skills/` | 跨工具共享的用户技能 |
| Agent Skills 项目 | 自 cwd 向上至 git 根（或文件系统根）的 `.agents/skills/` | 项目祖先链上的技能 |
| CLI | `--skills <path>` | 显式追加路径（文件或目录） |
| 扩展 | `resources_discover` 事件返回的 `skillPaths` | 扩展动态注册 |

### 优先级与冲突

- 同名技能：**先加载者胜出**（写入 `skillMap` 的条目保留）
- 后加载同名技能产生 `collision` 诊断，不会覆盖
- 符号链接指向同一物理文件会被 `canonicalizePath` 去重

```mermaid
flowchart LR
    subgraph 默认路径
        U1[~/.pi/agent/skills]
        P1[.pi/skills]
        U2[~/.agents/skills]
        P2[祖先 .agents/skills]
    end

    subgraph 显式路径
        CLI[--skills]
        EXT[extension skillPaths]
    end

    U1 --> MERGE[loadSkills skillMap]
    P1 --> MERGE
    U2 --> MERGE
    P2 --> MERGE
    CLI --> MERGE
    EXT --> MERGE
    MERGE --> OUT[Skill 列表 + diagnostics]
```

`ResourceLoader` 调用 `loadSkills({ includeDefaults: false })`，因为默认目录已由 `PackageManager.resolve()` 展开为具体 `skillPaths`。

---

## 系统提示中的技能列表

`formatSkillsForPrompt()` 将可见技能格式化为 XML，追加到系统提示末尾（需启用 `read` 工具）：

```xml
<available_skills>
  <skill>
    <name>example</name>
    <description>...</description>
    <location>/abs/path/to/SKILL.md</location>
  </skill>
</available_skills>
```

要点：

- `disableModelInvocation === true` 的技能**不会**出现在此列表
- 提示中说明：任务匹配描述时使用 `read` 工具加载 `<location>` 文件
- 技能内相对路径应相对于 `SKILL.md` 所在目录（`baseDir`）解析

调用链：`buildSystemPrompt()` → `formatSkillsForPrompt(skills)` → 模型可见的 `<available_skills>`。

---

## 技能调用流程

存在两条主要路径：**模型主动调用**与**用户显式命令**。

```mermaid
sequenceDiagram
    participant U as 用户
    participant E as 编辑器 / AgentSession
    participant M as 模型
    participant R as read 工具
    participant FS as SKILL.md

    Note over U,FS: 路径 A：模型主动调用
    M->>M: 系统提示见 available_skills
    M->>R: read(location)
    R->>FS: 读取文件
    R-->>M: 技能正文
    M->>M: 按技能指令执行

    Note over U,FS: 路径 B：用户 /skill:name
    U->>E: /skill:my-skill 附加参数
    E->>E: expandSkillCommand()
    E->>FS: readFileSync(filePath)
    E->>E: 构造 XML skill 块
    E->>M: user message 含 skill 块 + 可选 userMessage
    M->>M: 基于已注入内容继续
```

### XML 包装格式

显式调用或扩展展开后，用户消息形如：

```xml
<skill name="my-skill" location="/path/to/SKILL.md">
References are relative to /path/to/skill/dir.

（SKILL.md 正文，不含 frontmatter）
</skill>

（可选：用户附加说明）
```

`parseSkillBlock()` 用正则解析该结构，供 TUI 拆分渲染。

### `/skill:name` 命令

`AgentSession.expandSkillCommand()`：

1. 识别 `/skill:` 前缀
2. 在已加载技能中按 `name` 查找
3. 读取 `SKILL.md`，剥离 frontmatter
4. 包装为上述 XML 块；未知技能则原样透传

`disable-model-invocation: true` 的技能仍可通过 `/skill:name` 使用。

---

## TUI 渲染：SkillInvocationMessageComponent

位置：`packages/coding-agent/src/modes/interactive/components/skill-invocation-message.ts`

当用户消息含 skill XML 块时，`interactive-mode.ts` 调用 `parseSkillBlock()`，用 `SkillInvocationMessageComponent` 渲染（与普通 `UserMessageComponent` 分离）。

| 状态 | 显示 |
|------|------|
| 折叠（默认） | `[skill] 技能名 (Ctrl+O to expand)` |
| 展开 | `[skill]` 标签 + Markdown 渲染的技能名与正文 |

样式：

- 背景：`theme.bg("customMessageBg")`
- 标签：`customMessageLabel`
- 正文：`customMessageText`
- 与自定义消息（custom message）视觉一致

若 XML 块后还有 `userMessage`，单独渲染为普通用户消息气泡。

```mermaid
stateDiagram-v2
    [*] --> Collapsed: 解析 skill 块
    Collapsed --> Expanded: Ctrl+O / toolOutputExpanded
    Expanded --> Collapsed: 再次切换
    Collapsed --> [*]
    Expanded --> [*]

    state Collapsed {
        [*] --> ShowLine
        ShowLine: 单行 [skill] name
    }

    state Expanded {
        [*] --> ShowMD
        ShowMD: Markdown 标题 + 全文
    }
```

---

## 扩展集成

### resources_discover

扩展可监听 `resources_discover`，返回额外 `skillPaths`：

```typescript
pi.on("resources_discover", () => ({
  skillPaths: ["/path/to/extra/skills"],
}));
```

`AgentSession` 在 reload 时合并这些路径并重新 `loadSkills`。

### 斜杠命令自动注册

已加载技能自动暴露为 `/skill:<name>`，出现在命令补全中（`source: "skill"`）。

---

## 关键类型

```typescript
interface Skill {
  name: string;
  description: string;
  filePath: string;      // SKILL.md 绝对路径
  baseDir: string;       // 技能目录（相对路径解析基准）
  sourceInfo: SourceInfo;
  disableModelInvocation: boolean;
}

interface SkillFrontmatter {
  name?: string;
  description?: string;
  "disable-model-invocation"?: boolean;
}
```

---

## 相关源文件

| 文件 | 职责 |
|------|------|
| `packages/coding-agent/src/core/skills.ts` | 加载、校验、格式化 |
| `packages/coding-agent/src/core/resource-loader.ts` | 路径聚合与 reload |
| `packages/coding-agent/src/core/package-manager.ts` | `.pi/` 与 `.agents/` 自动发现 |
| `packages/coding-agent/src/core/system-prompt.ts` | 注入技能列表 |
| `packages/coding-agent/src/core/agent-session.ts` | `/skill:` 展开、parseSkillBlock |
| `packages/coding-agent/src/modes/interactive/components/skill-invocation-message.ts` | TUI 组件 |
