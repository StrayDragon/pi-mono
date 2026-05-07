# pi.cli 模型配置参考

> 源码核心文件：
> - `packages/ai/src/models.generated.ts` — 内置模型列表（自动生成）
> - `packages/ai/src/models.ts` — 模型类型与 Provider 定义
> - `packages/coding-agent/src/core/model-registry.ts` — 模型注册表
> - `packages/coding-agent/src/core/model-resolver.ts` — 默认模型解析
> - `packages/ai/src/providers/` — 各 Provider 的 API 调用实现
> - `packages/ai/src/providers/transform-messages.ts` — 消息格式转换

---

## 1. 模型文件

自定义模型和 Provider 覆盖写在 `~/.pi/agent/models.json`：

```json
{
  "providers": {
    "<provider-id>": {
      "name": "Display Name",
      "baseUrl": "https://api.example.com",
      "apiKey": "!env:MY_API_KEY",
      "api": "openai-completions",
      "headers": { "X-Custom": "!env:HEADER_VAL" },
      "authHeader": false,
      "compat": { ... },
      "models": [ ... ],
      "modelOverrides": {
        "<model-id>": { ... }
      }
    }
  }
}
```

其中 `!env:VAR_NAME` 表示从环境变量读取，避免明文写入 key。

---

## 2. 支持的 Providers（31 个）

| Provider ID | API 类型 | 说明 |
|-------------|---------|------|
| `anthropic` | `anthropic-messages` | Anthropic 官方 API |
| `openai` | `openai-completions` | OpenAI 官方 API |
| `azure-openai-responses` | `azure-openai-responses` | Azure OpenAI Responses API |
| `openai-codex` | `openai-codex-responses` | OpenAI Codex |
| `github-copilot` | `openai-completions` | GitHub Copilot |
| `google` | `google-generative-ai` | Google Gemini |
| `google-vertex` | `google-vertex` | Google Vertex AI |
| `deepseek` | `openai-completions` | DeepSeek |
| `amazon-bedrock` | `bedrock-converse-stream` | Amazon Bedrock（Node only） |
| `xai` | `openai-completions` | xAI Grok |
| `groq` | `openai-completions` | Groq |
| `cerebras` | `openai-completions` | Cerebras |
| `openrouter` | `openai-completions` | OpenRouter |
| `vercel-ai-gateway` | `openai-completions` | Vercel AI Gateway |
| `zai` | `openai-completions` | ZAI |
| `mistral` | `mistral-conversations` | Mistral |
| `minimax` | `openai-completions` | MiniMax |
| `minimax-cn` | `openai-completions` | MiniMax（中国） |
| `moonshotai` | `openai-completions` | Moonshot AI |
| `moonshotai-cn` | `openai-completions` | Moonshot AI（中国） |
| `huggingface` | `openai-completions` | Hugging Face Inference |
| `fireworks` | `openai-completions` | Fireworks AI |
| `opencode` | `openai-completions` | OpenCode Zen |
| `opencode-go` | `openai-completions` | OpenCode Go |
| `kimi-coding` | `openai-completions` | Kimi For Coding |
| `cloudflare-workers-ai` | `openai-completions` | Cloudflare Workers AI |
| `cloudflare-ai-gateway` | `openai-completions` | Cloudflare AI Gateway |
| `xiaomi` | `openai-completions` | Xiaomi MiMo |
| `xiaomi-token-plan-cn` | `openai-completions` | Xiaomi MiMo（中国） |
| `xiaomi-token-plan-ams` | `openai-completions` | Xiaomi MiMo（阿姆斯特丹） |
| `xiaomi-token-plan-sgp` | `openai-completions` | Xiaomi MiMo（新加坡） |

---

## 3. 默认模型

每个 Provider 的默认模型（`model-resolver.ts`）：

| Provider ID | 默认模型 ID |
|-------------|------------|
| `anthropic` | `claude-opus-4-7` |
| `openai` | `gpt-5.4` |
| `azure-openai-responses` | `gpt-5.4` |
| `openai-codex` | `gpt-5.5` |
| `deepseek` | `deepseek-v4-pro` |
| `google` | `gemini-3.1-pro-preview` |
| `google-vertex` | `gemini-3.1-pro-preview` |
| `github-copilot` | `gpt-5.4` |
| `openrouter` | `moonshotai/kimi-k2.6` |
| `vercel-ai-gateway` | `zai/glm-5.1` |
| `xai` | `grok-4.20-0309-reasoning` |
| `groq` | `openai/gpt-oss-120b` |
| `cerebras` | `zai-glm-4.7` |
| `zai` | `glm-5.1` |
| `mistral` | `devstral-medium-latest` |
| `minimax` / `minimax-cn` | `MiniMax-M2.7` |
| `moonshotai` / `moonshotai-cn` | `kimi-k2.6` |
| `huggingface` | `moonshotai/Kimi-K2.6` |
| `fireworks` | `accounts/fireworks/models/kimi-k2p6` |
| `opencode` / `opencode-go` | `kimi-k2.6` |
| `kimi-coding` | `kimi-for-coding` |
| `cloudflare-workers-ai` | `@cf/moonshotai/kimi-k2.6` |
| `cloudflare-ai-gateway` | `workers-ai/@cf/moonshotai/kimi-k2.6` |
| `xiaomi` / `*-cn` / `*-ams` / `*-sgp` | `mimo-v2.5-pro` |
| `amazon-bedrock` | `us.anthropic.claude-opus-4-6-v1` |

---

## 4. 环境变量与 API Key

| Provider ID | 环境变量 |
|-------------|---------|
| `anthropic` | `ANTHROPIC_API_KEY`（或 `ANTHROPIC_OAUTH_TOKEN`，后者优先级更高） |
| `openai` | `OPENAI_API_KEY` |
| `azure-openai-responses` | `AZURE_OPENAI_API_KEY` |
| `deepseek` | `DEEPSEEK_API_KEY` |
| `google` | `GEMINI_API_KEY` |
| `google-vertex` | `GOOGLE_CLOUD_API_KEY`（或通过 `gcloud auth application-default login` 使用 ADC） |
| `groq` | `GROQ_API_KEY` |
| `cerebras` | `CEREBRAS_API_KEY` |
| `xai` | `XAI_API_KEY` |
| `openrouter` | `OPENROUTER_API_KEY` |
| `vercel-ai-gateway` | `AI_GATEWAY_API_KEY` |
| `zai` | `ZAI_API_KEY` |
| `mistral` | `MISTRAL_API_KEY` |
| `minimax` | `MINIMAX_API_KEY` |
| `minimax-cn` | `MINIMAX_CN_API_KEY` |
| `moonshotai` | `MOONSHOT_API_KEY` |
| `moonshotai-cn` | `MOONSHOT_API_KEY` |
| `huggingface` | `HF_TOKEN` |
| `fireworks` | `FIREWORKS_API_KEY` |
| `opencode` / `opencode-go` | `OPENCODE_API_KEY` |
| `kimi-coding` | `KIMI_API_KEY` |
| `cloudflare-workers-ai` | `CLOUDFLARE_API_KEY` |
| `cloudflare-ai-gateway` | `CLOUDFLARE_API_KEY` |
| `xiaomi` | `XIAOMI_API_KEY` |
| `xiaomi-token-plan-cn` | `XIAOMI_TOKEN_PLAN_CN_API_KEY` |
| `xiaomi-token-plan-ams` | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` |
| `xiaomi-token-plan-sgp` | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` |
| `github-copilot` | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN` |
| `amazon-bedrock` | `AWS_PROFILE` 或 `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`，`AWS_REGION` |

---

## 5. API 调用详解

### 5.1 Anthropic Provider

**实际请求示例**（`packages/ai/src/providers/anthropic.ts`）：

```http
POST https://api.anthropic.com/v1/messages
Headers:
  x-api-key: <ANTHROPIC_API_KEY>
  anthropic-version: 2023-06-01
  anthropic-beta: interleaved-thinking-2025-05-14
  anthropic-beta: fine-grained-tool-streaming-2025-05-14
 anthropic-dangerous-direct-browser-access: true

Body (JSON):
{
  "model": "claude-opus-4-7",
  "messages": [
    { "role": "user", "content": [...] },
    { "role": "assistant", "content": [...] },
    { "role": "user", "content": [...] }
  ],
  "system": [
    { "type": "text", "text": "<system-prompt>" }
  ],
  "max_tokens": 8192,
  "stream": true,
  "thinking": {
    "type": "adaptive",
    "effort": "medium"
  },
  "tools": [
    {
      "name": "Bash",
      "description": "...",
      "input_schema": { "type": "object", "properties": {...} }
    }
  ],
  "metadata": { "user_id": "<session-id>" }
}
```

> **备注**：Anthropic 原生支持 `thinking` 块。启用思考时，响应为流式，其中 `message_start` → `content_block_start(type=thinking)` → `content_block_delta`（思考内容）→ `content_block_stop` → `content_block_start(type=text)` → `content_block_delta`（回复文本）→ `message_stop`。工具调用通过 `content_block_start(type=tool_use)` 和 `content_block_delta` 流式返回。`fine-grained-tool-streaming-2025-05-14` beta 支持工具结果流式回传。

### 5.2 OpenAI-Compatible Provider（通用）

**实际请求示例**（`packages/ai/src/providers/openai-completions.ts`）：

```http
POST https://api.openai.com/v1/chat/completions
Headers:
  Authorization: Bearer <OPENAI_API_KEY>
  Content-Type: application/json

Body (JSON):
{
  "model": "gpt-5.4",
  "messages": [
    { "role": "system", "content": "<system-prompt>" },
    { "role": "user", "content": [...] },
    { "role": "assistant", "content": [...] },
    { "role": "tool", "tool_call_id": "...", "content": "..." }
  ],
  "stream": true,
  "stream_options": { "include_usage": true },
  "max_completion_tokens": 8192,
  "temperature": 1.0,
  "tools": [...],
  "tool_choice": "auto"
}
```

> **备注**：不同兼容层（OpenAI、OpenRouter、DeepSeek、ZAI、Qwen）通过 `thinkingFormat` 决定思考参数格式。消息中的 `role` 映射：Anthropic 的 `toolResult` → OpenAI 的 `tool`，Anthropic 的 `user` 直接映射。`store: false` 默认关闭（除非 Provider 设置了 `supportsStore: true`）。Provider 通过 `maxTokensField` 决定使用 `max_tokens` 还是 `max_completion_tokens`。

### 5.3 OpenRouter Provider

**实际请求示例**：

```http
POST https://openrouter.ai/api/v1/chat/completions
Headers:
  Authorization: Bearer <OPENROUTER_API_KEY>
  HTTP-Referer: https://pi.dev
  X-OpenRouter-Title: pi
  X-OpenRouter-Categories: cli-agent

Body (JSON):
{
  "model": "moonshotai/kimi-k2.6",
  "messages": [...],
  "reasoning": {
    "effort": "medium"
  },
  "provider": {
    "order": ["OpenRouter"],
    "max_price": { "completion": 0.0 }
  }
}
```

> **备注**：OpenRouter 的 `reasoning` 参数为嵌套对象格式（不同于 OpenAI 的平铺 `reasoning_effort`）。支持 `openRouterRouting` 配置中的 `order`/`only`/`ignore`/`max_price` 等路由策略。

### 5.4 DeepSeek Provider

**实际请求示例**：

```http
POST https://api.deepseek.com/v1/chat/completions
Headers:
  Authorization: Bearer <DEEPSEEK_API_KEY>

Body (JSON):
{
  "model": "deepseek-v4-pro",
  "messages": [...],
  "thinking": { "type": "enabled" },
  "reasoning_effort": "medium"
}
```

> **备注**：DeepSeek 使用 `thinking` 块与 `reasoning_effort` 组合参数。`thinking.type` 控制是否启用，`reasoning_effort` 控制思考量。

### 5.5 ZAI / Qwen Provider

**实际请求示例**：

```http
POST <zai-base-url>/chat/completions
Headers:
  Authorization: Bearer <ZAI_API_KEY>

Body (JSON):
{
  "model": "glm-5.1",
  "messages": [...],
  "enable_thinking": true
}
```

> **备注**：ZAI 和 Qwen 系列使用顶层 `enable_thinking: boolean` 字段，不使用嵌套的 `reasoning` 对象。

### 5.6 Google Vertex AI Provider

**实际请求示例**：

```http
POST https://<region>-aiplatform.googleapis.com/v1/projects/<project>/locations/<region>/publishers/google/models/gemini-3.1-pro-preview:predict
Headers:
  Authorization: Bearer <ADC token via gcloud>
  Content-Type: application/json

Body (JSON):
{
  "instances": [{
    "contents": [...],
    "systemInstruction": { ... }
  }],
  "parameters": {
    "thinkingConfig": { "thinkingBudget": "<budget>" },
    "maxOutputTokens": 8192
  }
}
```

> **备注**：Vertex 使用 Google Cloud 的 `predict` 端点，认证通过 Application Default Credentials（`gcloud auth application-default login`）。不支持 `google-generative-ai` REST API 格式。

---

## 6. 消息结构

### 6.1 内部消息类型

```typescript
// 用户消息
interface UserMessage {
  role: "user";
  content: string | Content[];
  timestamp: number;
}

// 助手消息
interface AssistantMessage {
  role: "assistant";
  content: Content[]; // TextContent | ThinkingContent | ToolCall[]
  timestamp: number;
}

// 工具结果
interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: Content[]; // TextContent | ImageContent[]
  isError: boolean;
  timestamp: number;
}

// Content 类型
type TextContent   = { type: "text";    text: string; textSignature?: string };
type ImageContent  = { type: "image";   data: string; mimeType: string };
type ThinkingContent = { type: "thinking"; thinking: string; thinkingSignature?: string; redacted?: boolean };
type ToolCall      = { type: "toolCall"; id: string; name: string; arguments: Record<string, unknown>; thoughtSignature?: string };
```

### 6.2 API 消息转换

内部消息通过 `transform-messages.ts` 转换为各 Provider 的格式：

| 内部类型 | Anthropic | OpenAI | OpenRouter |
|---------|-----------|--------|------------|
| `user` | `role: "user"` | `role: "user"` | `role: "user"` |
| `assistant` + `ToolCall[]` | 转为 `content_block` 列表 | 转为 `assistant` + `tool` 消息 | 同 OpenAI |
| `toolResult` | 转为 `user` role 含结果 | `role: "tool"` | 同 OpenAI |
| `ThinkingContent` | 保持原样（API 原生支持） | 通过 `thinkingFormat` 映射 | 通过 `reasoning` 对象 |
| Agent 特有类型（`bashExecution` 等） | 通过 `convertToLlm()` 转为 `user` 文本消息 | 同 | 同 |

---

## 7. 思考级别与 Token 预算

### 7.1 级别定义

| 级别 | 默认 Token 预算 | 说明 |
|------|--------------|------|
| `off` | — | 禁用思考 |
| `minimal` | 1024 | 极少思考 |
| `low` | 2048 | 少量思考 |
| `medium` | 8192 | 中等思考（默认） |
| `high` | 16384 | 深度思考 |
| `xhigh` | 由 `thinkingBudgets.high` 或模型上限决定 | 极高思考 |

可通过 `settings.json` 中的 `thinkingBudgets` 覆盖：
```json
{ "thinkingBudgets": { "medium": 16384, "high": 32768 } }
```

### 7.2 思考参数格式（按 Provider）

| Provider | 参数格式 | 示例 |
|----------|---------|------|
| Anthropic | `thinking: { type: "adaptive", effort: "medium" }` | Opus 4.6/4.7 使用自适应 |
| Anthropic（旧模型） | `thinking: { type: "enabled", budget_tokens: 8192 }` | 指定固定预算 |
| OpenAI | `reasoning_effort: "medium"` | 顶层参数 |
| OpenRouter | `reasoning: { effort: "medium" }` | 嵌套对象 |
| DeepSeek | `thinking: { type: "enabled" }, reasoning_effort: "medium"` | 两者组合 |
| ZAI / Qwen | `enable_thinking: true` | 布尔开关 |
| Qwen Chat Template | `chat_template_kwargs.enable_thinking: true` | 特殊模板参数 |

### 7.3 Anthropic 额外参数

```typescript
thinking: {
  type: "adaptive" | "enabled" | "disabled",
  effort?: "low" | "medium" | "high" | "xhigh" | "max",  // type=adaptive 时
  budget_tokens?: number,                                   // type=enabled 时
  display?: "summarized" | "omitted"                       // 思考块显示方式
}
```

> **备注**：`thinkingDisplay: "summarized"`（默认）将思考摘要显示给用户；`"omitted"` 完全隐藏。`interleaved-thinking-2025-05-14` beta 使思考内容可以与工具调用交错流式传输。

---

## 8. Provider Compat 配置

在 `models.json` 中为 Provider 或单个模型指定兼容行为：

### 8.1 OpenAI-Compatible 兼容选项

```json
{
  "compat": {
    "supportsStore": false,
    "supportsDeveloperRole": true,
    "supportsReasoningEffort": false,
    "supportsStrictMode": true,
    "thinkingFormat": "openrouter",
    "cacheControlFormat": "anthropic",
    "requiresToolResultName": true,
    "requiresAssistantAfterToolResult": true,
    "requiresThinkingAsText": false,
    "maxTokensField": "max_completion_tokens",
    "openRouterRouting": {
      "order": ["OpenRouter"],
      "only": ["google"],
      "ignore": ["openai/gpt-3.5"],
      "max_price": { "completion": 0.0 }
    },
    "vercelGatewayRouting": {
      "order": ["OpenAI"],
      "only": ["*"]
    }
  }
}
```

| 选项 | 类型 | 说明 |
|------|------|------|
| `supportsStore` | `boolean` | Provider 是否支持 `store` 参数 |
| `supportsDeveloperRole` | `boolean` | 是否支持 `role: "developer"`（vs `system`） |
| `supportsReasoningEffort` | `boolean` | 是否支持 `reasoning_effort` 参数 |
| `supportsStrictMode` | `boolean` | 是否支持严格模式 |
| `thinkingFormat` | `string` | 思考参数格式：`openai`、`openrouter`、`deepseek`、`zai`、`qwen`、`qwen-chat-template` |
| `cacheControlFormat` | `string` | 目前仅支持 `"anthropic"` |
| `requiresToolResultName` | `boolean` | 工具结果消息是否需要工具名 |
| `requiresAssistantAfterToolResult` | `boolean` | 工具结果后是否强制跟 assistant 消息 |
| `requiresThinkingAsText` | `boolean` | 是否将思考转为 `<thinking>` 文本标签 |
| `maxTokensField` | `string` | `max_completion_tokens` 或 `max_tokens` |
| `openRouterRouting` | `object` | OpenRouter 路由策略 |
| `vercelGatewayRouting` | `object` | Vercel Gateway 路由策略 |

### 8.2 OpenRouter 路由策略

```json
{
  "allow_fallbacks": true,
  "require_parameters": false,
  "data_collection": "deny",
  "order": ["google", "anthropic"],
  "only": ["google/gemini-*"],
  "ignore": ["openai/gpt-3.5-turbo"],
  "quantizations": ["int4", "fp8"],
  "sort": { "by": "price", "partition": "openai" },
  "max_price": {
    "prompt": 0.001,
    "completion": 0.01
  },
  "preferred_max_latency": { "p50": 10 }
}
```

---

## 9. 单模型覆盖

在 `models.json` 的 `modelOverrides` 中覆盖单个模型的参数：

```json
{
  "providers": {
    "openrouter": {
      "models": [
        {
          "id": "custom-model",
          "name": "My Custom Model",
          "contextWindow": 200000,
          "maxTokens": 32000,
          "reasoning": true,
          "cost": { "input": 0.1, "output": 0.5, "cacheRead": 0.01 }
        }
      ],
      "modelOverrides": {
        "moonshotai/kimi-k2.6": {
          "reasoning": true,
          "thinkingLevelMap": { "high": null }
        }
      }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `contextWindow` | `number` | 上下文窗口大小（token） |
| `maxTokens` | `number` | 单次生成最大 token 数 |
| `reasoning` | `boolean` | 是否支持思考/RM |
| `cost` | `object` | 价格：`input`、`output`、`cacheRead`、`cacheWrite`（$/M tokens） |
| `thinkingLevelMap` | `object` | 级别映射：`{ "high": null }` 表示禁用 `high` 级别 |
| `input` | `string[]` | 支持的输入类型：`["text"]` 或 `["text", "image"]` |

---

## 10. CLI 使用方式

```bash
# 指定 Provider 和模型
pi --provider anthropic --model claude-opus-4-7

# 指定模型（使用默认 Provider）
pi --model gpt-5.4

# 指定带思考级别的模型
pi --model gpt-5.4:high

# 指定 Provider:模型:思考级别
pi --model anthropic/claude-opus-4-7:medium

# 思考级别（覆盖模型默认值）
pi --thinking high

# Ctrl+P 切换模型（从 enabledModels 中循环）
pi --models "anthropic/*,openai/gpt-5.4"

# 列出模型（支持搜索过滤）
pi --list-models
pi --list-models anthropic
pi --list-models "claude-sonnet"

# 内联 API Key
pi --provider openai --api-key sk-xxx --model gpt-5.4
```

---

## 11. Provider API 类型速查

| API 类型 | 请求格式 | 适用 Provider |
|---------|---------|--------------|
| `anthropic-messages` | Anthropic `messages.create` API | `anthropic` |
| `openai-completions` | OpenAI `chat.completions` API | `openai`、`deepseek`、`groq`、`cerebras`、`openrouter`、`fireworks` 等 20+ Provider |
| `azure-openai-responses` | Azure OpenAI Responses API | `azure-openai-responses` |
| `openai-codex-responses` | OpenAI Codex Responses API | `openai-codex` |
| `google-generative-ai` | Google Gemini REST API | `google` |
| `google-vertex` | Vertex AI `predict` API | `google-vertex` |
| `bedrock-converse-stream` | AWS Bedrock Converse API | `amazon-bedrock` |
| `mistral-conversations` | Mistral `/v1/conversations` API | `mistral` |

> **注意**：内置 Provider 均为动态导入（lazy-load），Bedrock 仅 Node 端可用。

---

## 12. 自定义模型字段 → API 请求参数完整映射

> 适用于 `~/.pi/agent/models.json` 中 `models[]` 和 `modelOverrides` 下定义的字段。

### 12.1 字段总览

| models.json 字段 | 类型 | **是否发送至 API** | 用途 |
|-----------------|------|:--------------:|------|
| `id` | `string` | ✅ 是 | 映射为 HTTP body 的 `model` 字段 |
| `name` | `string` | ❌ 否 | 仅用于 CLI 显示，不参与请求 |
| `api` | `Api` 类型字符串 | ❌ 否（控制派发） | 决定使用哪个 Provider 的 stream 函数 |
| `baseUrl` | `string` (URL) | ❌ 否（控制请求目标） | 设置 SDK client 的 `baseURL`，决定请求发往何处 |
| `reasoning` | `boolean` | ❌ 否（控制行为） | 启用思考功能的总开关，`false` 时强制关闭思考 |
| `thinkingLevelMap` | `Partial<Record<ThinkingLevel, string \| null>>` | ❌ 否（控制映射） | 将 pi 思考级别映射为 Provider 特定值 |
| `input` | `("text" \| "image")[]` | ❌ 否（控制预处理） | `["text"]` 时自动将消息中的图片替换为占位文本 |
| `cost` | `{ input, output, cacheRead, cacheWrite }` ($/M) | ❌ 否 | 仅用于本地费用计算，不发往 API |
| `contextWindow` | `number` | ❌ 否 | 仅用于本地上下文大小验证，不发往 API |
| `maxTokens` | `number` | ✅ 是 | 映射为 `max_tokens` / `max_completion_tokens` |
| `headers` | `Record<string, string>` | ✅ 是 | 合并至 HTTP 请求头 |
| `compat` | `OpenAICompletionsCompat` 等 | ❌ 否（控制行为） | 决定请求参数命名、格式和路由策略 |
| `modelOverrides` | `Record<string, Partial<Model>>` | ✅ 是（按模型覆盖） | 对已存在的模型进行部分字段覆盖 |

---

### 12.2 完整请求流程（以自定义 OpenAI-Compatible Provider 为例）

给定以下 `models.json` 配置：

```json
{
  "providers": {
    "my-provider": {
      "baseUrl": "https://api.example.com/v1",
      "apiKey": "!env:MY_API_KEY",
      "api": "openai-completions",
      "models": [{
        "id": "my-model-1",
        "name": "My Custom Model",
        "contextWindow": 128000,
        "maxTokens": 8192,
        "reasoning": true,
        "thinkingLevelMap": { "off": "none", "high": null },
        "input": ["text"],
        "cost": { "input": 0.5, "output": 1.5, "cacheRead": 0, "cacheWrite": 0 }
      }],
      "modelOverrides": {
        "my-model-1": {
          "maxTokens": 16384
        }
      }
    }
  }
}
```

执行：`pi --provider my-provider --model my-model-1 --thinking high`

#### Step 1：模型注册表解析

`model-registry.ts` 按 `provider + id` 查找，找到后合并 `modelOverrides`：

```typescript
// 最终解析出的 Model 对象：
{
  id: "my-model-1",
  name: "My Custom Model",           // 来自 models[0].name
  api: "openai-completions",
  provider: "my-provider",
  baseUrl: "https://api.example.com/v1",
  reasoning: true,
  thinkingLevelMap: { off: "none", high: null },
  input: ["text"],
  cost: { input: 0.5, output: 1.5, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 16384,                // 被 modelOverrides 覆盖为 16384
  headers: undefined,
  compat: undefined
}
```

#### Step 2：派发至 Provider

`api: "openai-completions"` → 查 `api-registry.ts` → 路由至 `streamSimpleOpenAICompletions`（`openai-completions.ts`）

#### Step 3：`buildBaseOptions` 计算 maxTokens

```typescript
// simple-options.ts
maxTokens: options?.maxTokens
  ?? (model.maxTokens > 0 ? Math.min(model.maxTokens, 32000) : undefined)
// → Math.min(16384, 32000) = 16384
```

#### Step 4：`clampThinkingLevel` 处理思考级别

```typescript
// models.ts
getSupportedThinkingLevels(model)
// thinkingLevelMap.off = "none" → 支持 off
// thinkingLevelMap.high = null  → 标记为不支持，过滤掉
// → 支持级别: ["off", "minimal", "low", "medium"]（不含 "high" 和 "xhigh"）

clampThinkingLevel(model, "high")
// 发现 "high" 不在支持列表，向下查找最近的："medium"
```

> **关键**：`null` 表示该级别**不支持**，不会发送 null 参数，而是整个级别被降级。

#### Step 5：HTTP 请求体构建

```json
{
  "model": "my-model-1",
  "messages": [...],
  "stream": true,
  "stream_options": { "include_usage": true },
  "max_completion_tokens": 16384,
  "reasoning_effort": "medium",
  "store": false
}
```

| 模型字段 | 影响 | 值 |
|---------|------|-----|
| `model.id` | → `model` | `"my-model-1"` |
| `maxTokens`（被 overrides 为 16384） | → `max_completion_tokens` | `16384` |
| `thinkingLevelMap.high = null` → clamp 到 `medium` | → `reasoning_effort` | `"medium"` |
| `baseUrl` | 设置请求目标 URL | `https://api.example.com/v1/chat/completions` |
| `input: ["text"]` | 图片预处理为占位符 | 不发往 API |
| `cost` | 本地计费 | 不发往 API |
| `contextWindow` | 上下文大小校验 | 不发往 API |

---

### 12.3 `thinkingLevelMap` 的具体行为

#### 映射原理

```typescript
// models.ts
const EXTENDED_THINKING_LEVELS = ["off", "minimal", "low", "medium", "high", "xhigh"];

function getSupportedThinkingLevels(model) {
  if (!model.reasoning) return ["off"];
  return EXTENDED_THINKING_LEVELS.filter((level) => {
    const mapped = model.thinkingLevelMap?.[level];
    if (mapped === null) return false;  // null = 该级别不支持
    if (level === "xhigh") return mapped !== undefined;
    return true;
  });
}
```

#### `thinkingLevelMap` 值的三种情况

| 映射值 | 效果 | 示例 |
|-------|------|------|
| `undefined` | 使用 pi 内置默认值 | `{ "high": undefined }` → 高思考发 `reasoning_effort: "high"` |
| `"low"` 等具体字符串 | 映射为该字符串发送 | `{ "high": "medium" }` → 高思考发 `reasoning_effort: "medium"` |
| `null` | **该级别被禁用**，自动降级 | `{ "high": null }` → 用户选 high → clamp 到 medium |

#### 不同 Provider 的实际发送效果

**OpenAI-Compatible（`thinkingFormat: "openai"`）**：
```json
{ "reasoning_effort": "medium" }
```

**OpenRouter（`thinkingFormat: "openrouter"`）**：
```json
{ "reasoning": { "effort": "medium" } }
```

**DeepSeek（`thinkingFormat: "deepseek"`）**：
```json
{ "thinking": { "type": "enabled" }, "reasoning_effort": "medium" }
```

**ZAI / Qwen（`thinkingFormat: "zai"`）**：
```json
{ "enable_thinking": true }
```

**Anthropic（原生思考）**：
```json
{ "thinking": { "type": "adaptive", "effort": "medium" } }
```
（若模型不支持自适应，fallback 到 `{ type: "enabled", budget_tokens: <thinkingBudget> }`）

---

### 12.4 `compat` 字段对请求的精确影响

#### `maxTokensField`

```typescript
// openai-completions.ts
if (options?.maxTokens) {
  if (compat.maxTokensField === "max_tokens") {
    (params as any).max_tokens = options.maxTokens;
  } else {
    params.max_completion_tokens = options.maxTokens;
  }
}
```

| `maxTokensField` | 发送的字段 |
|-----------------|----------|
| `"max_tokens"` | `max_tokens: 16384` |
| `"max_completion_tokens"`（默认） | `max_completion_tokens: 16384` |
| 未设置 | `max_completion_tokens: 16384`（默认行为） |

#### `thinkingFormat`

| `thinkingFormat` | 思考参数结构 |
|-----------------|-----------|
| `"openai"` | `reasoning_effort: "medium"` |
| `"openrouter"` | `reasoning: { effort: "medium" }` |
| `"deepseek"` | `thinking: { type: "enabled" }, reasoning_effort: "medium"` |
| `"zai"` | `enable_thinking: true` |
| `"qwen"` | `enable_thinking: true` |
| `"qwen-chat-template"` | `chat_template_kwargs: { enable_thinking: true }` |

#### `supportsStore`

```typescript
if (compat.supportsStore) {
  params.store = true;
} else {
  params.store = false;  // 大多数 Provider 的默认值
}
```

#### `requiresThinkingAsText`

若为 `true`，思考内容不作为独立 block，而是被包裹为 `<thinking>...</thinking>` 文本标签混入 assistant 消息。

#### `openRouterRouting`

完整透传至 HTTP body 的 `provider` 字段：

```json
{
  "model": "openrouter/auto",
  "provider": {
    "order": ["Google", "Anthropic"],
    "only": ["google/gemini-*"],
    "ignore": ["openai/gpt-3.5-turbo"],
    "max_price": { "completion": 0.001 }
  }
}
```

---

### 12.5 `modelOverrides` 的精确行为

`modelOverrides` 是对**已存在于注册表中的模型**的部分字段覆盖。合并顺序：

```
内置模型定义  ←  models.json provider.models[]  ←  models.json provider.modelOverrides[modelId]
```

**示例**：覆盖内置 OpenRouter 模型

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "moonshotai/kimi-k2.6": {
          "maxTokens": 32768,
          "reasoning": true,
          "thinkingLevelMap": { "off": "none" }
        }
      }
    }
  }
}
```

执行 `pi --model openrouter/moonshotai/kimi-k2.6 --thinking off`：

```json
{
  "model": "moonshotai/kimi-k2.6",
  "reasoning_effort": "none",
  "max_completion_tokens": 32768
}
```

> **注意**：`modelOverrides` 中的 `thinkingLevelMap.off = "none"` 只覆盖了 `off` 的映射，其余级别仍使用内置的 `thinkingLevelMap`。

---

### 12.6 `input: ["text"]` vs `["text", "image"]` 的实际差异

```typescript
// transform-messages.ts
function downgradeUnsupportedImages(messages, model) {
  if (model.input.includes("image")) return messages;
  // 替换图片为占位符
  return messages.map((msg) => {
    if (msg.role === "user" && Array.isArray(msg.content)) {
      return {
        ...msg,
        content: msg.content.map((block) => {
          if (block.type === "image") {
            return { type: "text", text: "(image omitted: model does not support images)" };
          }
          return block;
        }),
      };
    }
    return msg;
  });
}
```

**效果**：当 `input: ["text"]` 时，任何发送给模型的图片内容都被替换为文本占位符，而不是报错或丢弃整条消息。这发生在 API 调用之前，不影响请求参数结构（始终是 `messages[]` 格式）。

---

### 12.7 完整字段 → 请求参数对照表

以 `openai-completions` API 为例：

| models.json 字段 | 解析后的 Model 字段 | HTTP 请求体参数 | 示例值 |
|----------------|-----------------|--------------|-------|
| `id` | `model.id` | `model` | `"my-model-1"` |
| `baseUrl` | `model.baseUrl` | 请求目标 URL | `https://api.example.com/v1/chat/completions` |
| `maxTokens`（或 `modelOverrides`） | `options.maxTokens` | `max_completion_tokens` 或 `max_tokens` | `16384` |
| `reasoning: true` | `model.reasoning` | 决定是否发送 `reasoning_effort` 等 | — |
| `thinkingLevelMap` | 影响 `reasoningEffort` 值 | `reasoning_effort` 或 `reasoning.effort` 等 | `"medium"` |
| `headers` | 合并至 SDK 请求头 | HTTP Header | `X-Custom: value` |
| `apiKey` | 注入 SDK client | `Authorization: Bearer <key>` | — |
| `compat.maxTokensField` | 影响字段名 | `max_tokens` vs `max_completion_tokens` | — |
| `compat.thinkingFormat` | 影响思考参数结构 | `reasoning_effort` / `reasoning {}` 等 | — |
| `compat.openRouterRouting` | 透传 | `provider` 字段 | — |
| `input: ["text"]` | 预处理消息 | **不发送图片** | 图片 → 占位符 |
| `contextWindow` | 仅本地使用 | **不发往 API** | — |
| `cost` | 仅本地计费 | **不发往 API** | — |
| `name` | 仅显示 | **不发往 API** | — |

