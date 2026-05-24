# 供应商/模型系统

Pi 的模型系统位于 `packages/coding-agent/src/core/model-registry.ts` 和 `model-resolver.ts`，管理 **Provider（谁提供）× Api（什么协议）** 二维模型空间，并负责 API key 解析、OAuth 认证和模型选择。

## 双轴模型

```mermaid
graph TB
    subgraph "Provider 轴 — 谁"
        P1["anthropic"]
        P2["openai"]
        P3["github-copilot"]
        P4["openai-codex"]
        P5["custom-proxy"]
    end

    subgraph "Api 轴 — 怎么通信"
        A1["anthropic-messages"]
        A2["openai-responses"]
        A3["openai-completions"]
        A4["google-generative-ai"]
    end

    P1 --> A1
    P2 --> A2
    P3 --> A3
    P4 --> A2
    P5 --> A1

    MODEL["Model = provider + id + api + baseUrl + ..."]
    P1 --> MODEL
    A1 --> MODEL
```

- **Provider**：供应商标识（如 `anthropic`、`openai`），决定 API key 来源和 OAuth 流程
- **Api**：协议类型（如 `anthropic-messages`、`openai-responses`），决定请求/响应格式和 stream 实现
- 每个 `Model<Api>` 绑定一个 provider 和一个 api，可覆盖 baseUrl、headers、compat 等

```mermaid
classDiagram
    class Model {
        +string provider
        +string id
        +string name
        +Api api
        +string baseUrl
        +boolean reasoning
        +input[] input
        +cost cost
        +number contextWindow
        +number maxTokens
        +compat compat
    }

    class ModelRegistry {
        +AuthStorage authStorage
        +find(provider, id)
        +getAvailable()
        +getApiKeyAndHeaders(model)
        +registerProvider(name, config)
    }

    class AuthStorage {
        +getApiKey(provider)
        +login(provider, callbacks)
    }

    ModelRegistry --> AuthStorage
    ModelRegistry --> Model
```

## ModelRegistry 架构

```mermaid
graph TB
    subgraph "数据源"
        BI["pi-ai 内置模型<br/>models.generated.ts"]
        MJ["~/.pi/agent/models.json"]
        EXT["扩展 registerProvider()"]
    end

    subgraph "ModelRegistry"
        LOAD["loadModels()"]
        MERGE["mergeCustomModels()"]
        OVERRIDE["applyModelOverride()"]
        OAUTH_MOD["OAuth modifyModels()"]
        MODELS["models: Model[]"]
        PRC["providerRequestConfigs"]
        REG["registeredProviders"]
    end

    subgraph "运行时"
        FIND["find(provider, id)"]
        AVAIL["getAvailable()"]
        AUTH["getApiKeyAndHeaders()"]
    end

    BI --> LOAD
    MJ --> LOAD
    EXT --> REG --> LOAD
    LOAD --> MERGE --> OVERRIDE --> OAUTH_MOD --> MODELS
    LOAD --> PRC
    MODELS --> FIND
    MODELS --> AVAIL
    PRC --> AUTH
```

### 内置模型

来自 `@earendil-works/pi-ai` 的 `getProviders()` + `getModels(provider)`，数据生成于 `packages/ai/src/models.generated.ts`（由 `generate-models.ts` 脚本维护）。

### 自定义模型 (models.json)

路径：`~/.pi/agent/models.json`

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "apiKey": "ollama",
      "api": "openai-completions",
      "models": [
        {
          "id": "llama3",
          "name": "Llama 3",
          "reasoning": false,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 8192,
          "maxTokens": 4096
        }
      ]
    }
  }
}
```

支持：

- **完整 provider 定义**（baseUrl + api + models）
- **provider 级 override**（仅覆盖 baseUrl/headers/compat）
- **modelOverrides**（按 model id 部分覆盖内置模型属性）
- JSON 注释和 trailing comma（加载时 strip）

自定义模型与内置模型按 `provider + id` 合并，**custom 优先**。

## API Key 解析链

```mermaid
flowchart TD
    REQ["请求模型"] --> MR["ModelRegistry.getApiKeyAndHeaders()"]
    MR --> AS["AuthStorage.getApiKey()"]

    AS --> R1{"runtime override?<br/>--api-key"}
    R1 -->|是| KEY["返回 API key"]
    R1 -->|否| R2{"auth.json<br/>api_key?"}
    R2 -->|是| RESOLVE["resolveConfigValue()"]
    R2 -->|否| R3{"auth.json<br/>oauth?"}
    R3 -->|是| OAUTH["getOAuthApiKey()<br/>proper-lockfile 刷新"]
    R3 -->|否| R4{"环境变量?<br/>getEnvApiKey()"}
    R4 -->|是| KEY
    R4 -->|否| R5{"fallbackResolver?<br/>models.json apiKey"}
    R5 -->|是| KEY
    R5 -->|否| FAIL["无 key"]

    RESOLVE --> KEY
    OAUTH --> KEY

    MR --> PC["providerRequestConfigs.apiKey"]
    PC --> ENV{"env var name?"}
    ENV -->|是| KEY

    KEY --> HEADERS["合并 headers<br/>provider + model"]
    HEADERS --> OUT["ResolvedRequestAuth"]
```

### AuthStorage 优先级

`AuthStorage.getApiKey()` 完整链：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | Runtime override | CLI `--api-key` |
| 2 | auth.json (api_key) | 持久化 API key |
| 3 | auth.json (oauth) | OAuth token，过期自动刷新 |
| 4 | 环境变量 | `getEnvApiKey(provider)` |
| 5 | Fallback resolver | models.json 中的 apiKey 字段 |

`ModelRegistry.getApiKeyAndHeaders()` 额外检查 `providerRequestConfigs` 中的 apiKey（可引用 env var 名）。

### AuthStorage 与 proper-lockfile

OAuth token 刷新使用 **proper-lockfile** 防止多实例竞态：

```mermaid
sequenceDiagram
    participant P1 as Pi 实例 A
    participant P2 as Pi 实例 B
    participant LOCK as auth.json lock
    participant API as OAuth Provider

    P1->>LOCK: acquire lock
    P1->>API: refreshToken()
    P2->>LOCK: acquire lock (等待)
    P1->>LOCK: release + 写入新 cred
    P2->>LOCK: acquire
    P2->>P2: 发现 token 已刷新，直接使用
```

- 同步操作：`lockSync` + 重试
- 异步刷新：`lock()` + stale 检测 + `onCompromised` 回调
- 文件权限：`auth.json` mode `0600`，目录 `0700`

## OAuth 流程

内置 OAuth 支持（via `@earendil-works/pi-ai/oauth`）：

| Provider | ID | 用途 |
|----------|-----|------|
| Anthropic | `anthropic` | Claude 订阅/API |
| GitHub Copilot | `github-copilot` | Copilot 模型 |
| OpenAI Codex | `openai-codex` | Codex 订阅 |

```mermaid
sequenceDiagram
    participant User
    participant TUI as LoginDialog
    participant AS as AuthStorage
    participant OAuth as OAuthProvider
    participant Browser

    User->>TUI: /login → 选择 provider
    TUI->>OAuth: provider.login(callbacks)
    OAuth->>Browser: 打开授权 URL / device code
    Browser->>OAuth: 授权完成
    OAuth-->>TUI: OAuthCredentials
    TUI->>AS: set(provider, { type: oauth, ... })
    AS->>AS: 写入 auth.json (locked)

    Note over AS,OAuth: 后续请求
    AS->>OAuth: getApiKey(credentials)
    alt token 过期
        AS->>OAuth: refreshToken() (locked)
        OAuth-->>AS: 新 credentials
    end
```

扩展可通过 `registerProvider()` 注册自定义 OAuth：

```typescript
pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",
    login(callbacks) { ... },
    refreshToken(credentials) { ... },
    getApiKey(credentials) { return credentials.access; },
    modifyModels(models, credentials) { ... },
  },
});
```

## 扩展 Provider 注册

```mermaid
flowchart TD
    EXT["扩展 registerProvider()"] --> PHASE{"bindCore 前?"}
    PHASE -->|是| QUEUE["pendingProviderRegistrations"]
    PHASE -->|否| DIRECT["ModelRegistry.registerProvider()"]
    QUEUE --> FLUSH["bindCore() flush"]
    FLUSH --> DIRECT

    DIRECT --> VALIDATE["validateProviderConfig()"]
    VALIDATE --> APPLY["applyProviderConfig()"]
    APPLY --> OAUTH_REG["registerOAuthProvider()"]
    APPLY --> API_REG["registerApiProvider()"]
    APPLY --> MODELS["替换/添加 models"]
    APPLY --> URL["覆盖 baseUrl/headers"]
```

三种注册模式：

1. **完整 provider**：`models` + `baseUrl` + `apiKey`/`oauth` → 替换该 provider 全部模型
2. **URL 覆盖**：仅 `baseUrl` → 修改已有模型的 endpoint
3. **OAuth 注册**：`oauth` 对象 → 启用 `/login` 支持

`unregisterProvider()` 移除扩展注册的 provider 并 `refresh()` 恢复内置模型。

## 模型解析顺序

启动时 `findInitialModel()` 按优先级选择模型：

```mermaid
flowchart TD
    START["启动"] --> C1{"CLI --provider + --model?"}
    C1 -->|是| CLI["resolveCliModel()"]
    C1 -->|否| C2{"scoped models 且非 continue?"}
    C2 -->|是| SCOPED["scopedModels[0]"]
    C2 -->|否| C3{"settings default?"}
    C3 -->|是| SETTINGS["defaultProvider/defaultModelId"]
    C3 -->|否| C4["getAvailable() 首个有 key 的模型"]
    C4 --> C5["known provider 默认模型"]
    C5 --> C6["availableModels[0]"]

    CLI --> DONE["InitialModelResult"]
    SCOPED --> DONE
    SETTINGS --> DONE
    C6 --> DONE
```

| 优先级 | 来源 | 条件 |
|--------|------|------|
| 1 | CLI | `--provider` + `--model` 或 `--model provider/pattern` |
| 2 | Scoped models | `--models` 或 settings `enabledModels`，非 resume/continue |
| 3 | Settings 默认 | `defaultProvider` + `defaultModelId` |
| 4 | 可用模型 | 第一个已配置 auth 的模型 |
| 5 | 已知默认 | 各 provider 的 defaultModelPerProvider |
| 6 | 兜底 | availableModels[0] |

### Continue/Resume 时的模型恢复

```mermaid
flowchart TD
    RESUME["--continue / --resume"] --> CTX["buildSessionContext().model"]
    CTX --> RESTORE["restoreModelFromSession()"]
    RESTORE --> FOUND{"model 存在且有 auth?"}
    FOUND -->|是| USE["使用会话模型"]
    FOUND -->|否| FALLBACK["回退到 currentModel 或 availableModels[0]"]
```

Resume 时跳过 scoped models 优先级，优先恢复会话中记录的 model。

### Scoped Models

`--models "anthropic/*,openai/gpt-*"` 解析为 `ScopedModel[]`：

- 支持 glob（`anthropic/*`）和模糊匹配（`*sonnet*`）
- 可选 thinking level 后缀：`model:high`
- 模型切换（cycle）仅在 scoped 列表内循环
- scoped 列表过滤掉无 auth 的模型

## 模型注册表架构图

```mermaid
graph LR
    subgraph "ModelRegistry 内部状态"
        M["models[]"]
        PRC["providerRequestConfigs<br/>apiKey, headers, authHeader"]
        MRH["modelRequestHeaders"]
        RP["registeredProviders<br/>扩展注册"]
    end

    subgraph "AuthStorage"
        DATA["auth.json data"]
        RUNTIME["runtimeOverrides"]
        FALLBACK["fallbackResolver"]
    end

    subgraph "pi-ai"
        GETM["getModels()"]
        GETP["getProviders()"]
        STREAM["streamSimple()"]
        OAUTH_P["OAuthProviders"]
    end

    GETM --> M
    GETP --> M
    RP --> M
    DATA --> PRC
    RUNTIME --> PRC
    FALLBACK --> PRC
    OAUTH_P --> M
    M --> STREAM
```

## API Key 解析详图

```mermaid
graph TB
    subgraph "getApiKeyAndHeaders(model)"
        direction TB
        K1["authStorage.getApiKey(provider)"]
        K2["providerConfig.apiKey → resolveConfigValueOrThrow"]
        K3["合并 model.headers + provider.headers"]
        K4["authHeader → Bearer token"]
    end

    subgraph "getApiKey(provider) in AuthStorage"
        direction TB
        A1["runtimeOverrides"]
        A2["auth.json api_key"]
        A3["auth.json oauth → refresh"]
        A4["process.env"]
        A5["fallbackResolver"]
    end

    A1 --> A2 --> A3 --> A4 --> A5
    K1 --> MERGE["apiKey = K1 ?? K2"]
    A5 -.-> K1
    MERGE --> K3 --> K4
```

### apiKey 值类型

models.json 中 `apiKey` 可以是：

- 字面量 key 字符串
- 环境变量名（如 `"ANTHROPIC_API_KEY"`）
- 命令引用（`!"cmd args"`）— 运行时执行获取

## 关键源文件

| 文件 | 职责 |
|------|------|
| `model-registry.ts` | ModelRegistry、models.json 加载、Provider 注册 |
| `model-resolver.ts` | CLI/scoped/初始模型解析 |
| `auth-storage.ts` | auth.json 读写、OAuth 刷新、锁 |
| `resolve-config-value.ts` | env/command 配置值解析 |
| `packages/ai/src/models.generated.ts` | 内置模型数据 |
| `extensions/types.ts` | ProviderConfig 类型 |

## CLI 相关

| 标志 | 作用 |
|------|------|
| `--provider` / `--model` | 指定模型 |
| `--models` | scoped 模型列表（glob） |
| `--api-key` | runtime API key override |
| `--list-models` | 列出可用模型 |
| `/login` | OAuth 登录流程 |
