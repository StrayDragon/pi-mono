# 设计哲学与取舍

Pi 是一个面向编码场景的多包 monorepo。其设计目标不是功能最大化，而是在**可扩展性、类型安全、供应链安全**与**运行时极简**之间取得平衡。本文档说明核心原则、关键取舍，以及这些决策如何体现在代码结构中。

## 核心原则

### 1. Stream-first，Errors-in-band

LLM 响应以**异步事件流**为首要抽象，而非一次性 Promise。错误与完成信号都作为流内事件传递，消费者通过 `for await` 或 `.result()` 统一处理。

```mermaid
sequenceDiagram
    participant Caller as 调用方
    participant Stream as AssistantMessageEventStream
    participant Provider as ApiProvider

    Caller->>Provider: stream(model, context)
    Provider->>Stream: push(start)
    loop 增量输出
        Provider->>Stream: push(text_delta / thinking_delta)
        Stream-->>Caller: yield event
    end
    alt 成功
        Provider->>Stream: push(done)
        Stream-->>Caller: result() → AssistantMessage
    else 失败
        Provider->>Stream: push(error)
        Stream-->>Caller: result() → AssistantMessage (stopReason=error)
    end
```

**含义：**

- UI 可以逐 token 渲染，无需等待完整响应
- 取消（AbortSignal）在流层面统一生效
- 错误不是异常抛出的唯一路径；`error` 事件携带结构化诊断信息
- 源码：`packages/ai/src/utils/event-stream.ts`、`packages/ai/src/providers/*.ts`

### 2. 依赖方向向下

四层架构严格单向依赖：

```mermaid
graph TD
    subgraph L4["Layer 4: 交互"]
        TUI["pi-tui"]
        Modes["interactive / print / rpc"]
    end

    subgraph L3["Layer 3: 应用"]
        CA["pi-coding-agent"]
    end

    subgraph L2["Layer 2: 运行时"]
        AC["pi-agent-core"]
    end

    subgraph L1["Layer 1: LLM 抽象"]
        AI["pi-ai"]
    end

    Modes --> CA
    CA --> AC
    CA --> AI
    CA --> TUI
    AC --> AI

    AI -.->|"禁止"| AC
    TUI -.->|"禁止"| AI
    AC -.->|"禁止"| CA

    style AI fill:#e8f5e9
    style AC fill:#fff3e0
    style CA fill:#e1f5fe
    style TUI fill:#fce4ec
```

**含义：**

- `pi-tui` 是纯 UI 库，不感知 Agent 或 LLM
- `pi-ai` 不依赖 Agent 运行时
- 上层通过接口消费下层能力，而非反向耦合

### 3. 接口隔离

每层只暴露最小必要接口：

| 层 | 对外接口 | 上层用法 |
|----|---------|---------|
| pi-ai | `stream()`、`getModel()`、`registerApiProvider()` | 获取流、查模型 |
| pi-agent-core | `Agent`、`AgentHarness`、`AgentEvent` | 实例化 + 订阅 |
| pi-tui | `Component`、`TUI`、`Editor` | 实现组件、挂载 |
| pi-coding-agent | `createAgentSession()`、`ExtensionAPI` | SDK 编程 |

```mermaid
graph LR
    subgraph 上层
        SDK["createAgentSession()"]
        EXT["ExtensionAPI"]
    end

    subgraph 隔离边界
        AS["AgentSession"]
        AG["Agent"]
        SF["StreamFn"]
    end

    subgraph 下层
        AI["pi-ai stream()"]
    end

    SDK --> AS
    EXT --> AS
    AS --> AG
    AG --> SF
    SF --> AI
```

### 4. 传输抽象（StreamFn 注入）

Agent 循环不直接调用 LLM API，而是通过可注入的 `StreamFn`：

```mermaid
graph TB
    LOOP["AgentLoop"]
    SF{"StreamFn 策略"}
    DIRECT["streamSimple() 直连"]
    PROXY["streamProxy() 代理"]
    MOCK["faux provider 测试"]
    CUSTOM["扩展自定义 stream"]

    LOOP --> SF
    SF --> DIRECT
    SF --> PROXY
    SF --> MOCK
    SF --> CUSTOM
```

**用途：**

- 生产：直连供应商 API
- 测试：注入 faux provider，无需真实 API Key
- 扩展：包装 payload、添加 header、路由到自定义网关

源码：`packages/agent/src/types.ts`（`StreamFn`）、`packages/agent/src/agent-loop.ts`

### 5. 事件驱动解耦

扩展、会话、UI 之间通过事件总线通信，而非直接调用：

```mermaid
flowchart LR
    subgraph 生产者
        AS["AgentSession"]
        AG["Agent"]
        RL["ResourceLoader"]
    end

    subgraph 事件总线
        EB["EventBus / ExtensionRunner"]
    end

    subgraph 消费者
        EXT["扩展 pi.on()"]
        UI["InteractiveMode"]
        SM["SessionManager"]
    end

    AS --> EB
    AG --> EB
    RL --> EB
    EB --> EXT
    EB --> UI
    EB --> SM
```

**典型事件：** `session_start`、`tool_call`、`before_agent_start`、`resources_discover`

源码：`packages/coding-agent/src/core/extensions/runner.ts`、`packages/coding-agent/src/core/event-bus.ts`

---

## 关键取舍

### Monorepo 锁步版本 vs 独立版本

```mermaid
graph LR
    subgraph 当前方案["锁步版本 (chosen)"]
        V["0.12.x 统一版本"]
        AI["@earendil-works/pi-ai"]
        AC["@earendil-works/pi-agent-core"]
        CA["@earendil-works/pi-coding-agent"]
        TU["@earendil-works/pi-tui"]
        V --> AI & AC & CA & TU
    end

    subgraph 替代方案["独立版本 (rejected)"]
        V1["pi-ai 1.2"]
        V2["agent 0.8"]
        V3["coding-agent 2.0"]
        V1 -.->|"版本矩阵爆炸"| V2
        V2 -.->|"兼容性测试成本"| V3
    end
```

| 维度 | 锁步版本 | 独立版本 |
|------|---------|---------|
| 发布复杂度 | 低：一次 bump 全部包 | 高：需维护兼容矩阵 |
| 内部 API 演进 | 可同步 breaking change | 需 semver 约束 |
| 用户安装 | 版本号一致，不易混装 | 可能装到不兼容组合 |

**决策：** 所有包共享同一版本号，`npm run release:patch/minor` 统一发布。

### Erasable TS 语法 vs 完整 TypeScript

```mermaid
flowchart TB
    TS["TypeScript 源码"]
    STRIP["Node strip-only 模式"]
    JS["纯 JS 输出（无 emit 构造）"]

    TS --> STRIP --> JS

    subgraph 允许
        A1["interface / type"]
        A2["泛型"]
        A3["declare module 合并"]
    end

    subgraph 禁止
        B1["enum"]
        B2["namespace"]
        B3["参数属性"]
        B4["import = / export ="]
    end
```

**原因：** Bun 二进制与 `tsx` 运行时直接执行 TS，不需要 tsc emit。禁止需 JS 输出的语法，保证 strip 后行为一致。

**影响：** 用显式字段 + 构造函数赋值替代 parameter properties；用 union + const 替代 enum。

规则来源：`AGENTS.md` → Code Quality

### JSONL 追加写入 vs 数据库

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant SM as SessionManager
    participant File as session.jsonl

    User->>SM: 新消息 / 分支 / 压缩
    SM->>File: appendEntry(JSON line)
    Note over File: 只追加，不修改历史行

    User->>SM: 加载会话
    SM->>File: 顺序读取全部行
    SM->>SM: 重建消息树 + 分支
```

| 维度 | JSONL | 数据库 |
|------|-------|--------|
| 部署 | 零依赖，文件即会话 | 需 DB 服务 |
| 可审计 | git diff 友好 | 需导出工具 |
| 分支/压缩 | 追加 entry 类型即可 | 需 schema 迁移 |
| 并发 | 单写者假设 | 原生支持 |

**决策：** 会话持久化为 `~/.pi/agent/sessions/*.jsonl`，entry 类型包括 `message`、`branch`、`compaction`、`custom`。

源码：`packages/agent/src/harness/session/jsonl-storage.ts`、`packages/coding-agent/src/core/session-manager.ts`

### jiti 动态加载 vs 预编译扩展

```mermaid
graph TB
    subgraph 开发/Node["Node.js 开发模式"]
        J["jiti 运行时编译"]
        ALIAS["模块 alias 解析"]
        EXT_TS[".pi/extensions/*.ts"]
        EXT_TS --> J --> ALIAS
    end

    subgraph Bun二进制["Bun 二进制发布"]
        VM["virtualModules 静态打包"]
        EXT_TS2["扩展 TS"]
        EXT_TS2 --> VM
    end

    subgraph 若预编译["预编译方案 (未采用)"]
        BUILD["扩展需先 build"]
        PUB["发布 compiled JS"]
    end
```

| 维度 | jiti 动态加载 | 预编译 |
|------|-------------|--------|
| 开发体验 | 改扩展即生效 | 需 rebuild |
| 依赖 | 扩展可 import typebox 等 | 需声明 peer deps |
| 二进制 | virtualModules 预打包核心依赖 | 同样可行 |
| 安全 | 需审查扩展代码 | 同样需审查 |

**决策：** 扩展以 TS 源文件形式存在，jiti 在 Node 模式加载；Bun 二进制通过 `virtualModules` 提供 `@earendil-works/pi-*` 等包。

源码：`packages/coding-agent/src/core/extensions/loader.ts`

---

## 极简主义：核心最小，扩展承载

```mermaid
mindmap
  root((Pi 核心))
    内置能力
      read / bash / edit / write
      会话 JSONL
      模型目录
      TUI 交互
    扩展承载
      自定义工具
      自定义 Provider
      UI 组件/命令
      压缩策略
      主题/快捷键
      技能发现
```

**原则：**

- 核心包只保留编码 Agent 的通用路径
- 团队/个人定制走 `.pi/extensions/` 或 `~/.pi/agent/extensions/`
- 不在核心中添加一次性功能

示例：`packages/coding-agent/examples/extensions/` 含 70+ 扩展示例，核心不内置这些行为。

---

## 供应链安全哲学

```mermaid
flowchart TB
    subgraph 安装时
        CI["npm ci --ignore-scripts"]
        LOCK["package-lock.json 精确 pin"]
        SW["coding-agent shrinkwrap 独立校验"]
    end

    subgraph 开发时
        PRE["pre-commit 阻止意外 lockfile 变更"]
        PIN["直接依赖精确版本"]
        REV["lifecycle script 白名单审查"]
    end

    subgraph 发布时
        PUB["npm publish WebAuthn 2FA"]
        SMOKE["release:local 冒烟测试"]
    end

    CI --> LOCK
    LOCK --> SW
    PRE --> PIN
    PIN --> REV
    PUB --> SMOKE
```

**要点：**

1. **默认不跑 lifecycle scripts**：`npm install --ignore-scripts`，防止 postinstall 恶意代码
2. **锁文件即代码**：直接依赖 pinned 到精确版本；pre-commit 需 `PI_ALLOW_LOCKFILE_CHANGE=1` 才允许改 lockfile
3. **shrinkwrap 独立校验**：`packages/coding-agent/npm-shrinkwrap.json` 单独生成与检查
4. **新依赖 lifecycle 需显式 allowlist**：`scripts/generate-coding-agent-shrinkwrap.mjs`
5. **扩展是信任边界**：扩展以用户身份运行，可访问文件系统和网络；文档强调只安装可信扩展

规则来源：`AGENTS.md` → Dependency and Install Security

---

## 设计决策全景

```mermaid
graph TB
    subgraph 运行时
        SF["Stream-first"]
        ED["Event-driven"]
        TA["Transport abstraction"]
    end

    subgraph 架构
        DD["Dependency down"]
        IS["Interface segregation"]
        MIN["Core minimalism"]
    end

    subgraph 工程
        LS["Lockstep versioning"]
        ET["Erasable TS"]
        JL["JSONL sessions"]
        JT["jiti extensions"]
    end

    subgraph 安全
        SC["Supply chain hardening"]
        EXT_T["Extension trust boundary"]
    end

    SF --> UI["流式 UI"]
    ED --> EXT["可插拔扩展"]
    TA --> TEST["可测试/mock"]
    DD --> MAINT["可维护分层"]
    MIN --> EXT
    SC --> CI["安全 CI 流程"]
```

---

## 延伸阅读

- [整体架构](../02-architecture/01-architecture-overview.md)
- [事件系统](../02-architecture/04-event-system.md)
- [扩展系统](../04-subsystems/02-extension-system.md)
- [设计模式索引](./02-patterns-catalog.md)
