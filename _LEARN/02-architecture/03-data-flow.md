# 数据流与消息流

## 端到端数据流

```mermaid
flowchart TB
    subgraph "用户输入"
        UI["TUI 编辑器 / stdin / RPC"]
    end

    subgraph "输入处理"
        IP["InputEvent 分发"]
        SK["技能检测 & 加载"]
        IMG["图片附件处理"]
        SP["系统提示构建"]
    end

    subgraph "Agent 循环"
        CTX["构建 AgentContext"]
        TF["transformContext()<br/>上下文裁剪"]
        CONV["convertToLlm()<br/>AgentMessage → Message"]
        LLM_CTX["构建 LLM Context<br/>{systemPrompt, messages, tools}"]
    end

    subgraph "LLM 调用"
        RESOLVE["解析模型 & API Key"]
        STREAM["stream() / streamSimple()"]
        PROVIDER["供应商实现<br/>(Anthropic/OpenAI/...)"]
        SSE["SSE/WebSocket 流"]
    end

    subgraph "响应处理"
        EVENTS["AssistantMessageEvent 流"]
        PARTIAL["部分消息更新"]
        TOOL_CALLS["提取工具调用"]
    end

    subgraph "工具执行"
        VALIDATE["参数验证 (TypeBox)"]
        BEFORE["beforeToolCall 钩子"]
        EXEC["工具执行"]
        AFTER["afterToolCall 钩子"]
        RESULT["工具结果"]
    end

    subgraph "持久化"
        JSONL["JSONL 写入"]
        SESSION["会话树更新"]
    end

    UI --> IP --> SK --> SP
    IMG --> SP
    SP --> CTX --> TF --> CONV --> LLM_CTX
    LLM_CTX --> RESOLVE --> STREAM --> PROVIDER --> SSE
    SSE --> EVENTS --> PARTIAL --> TOOL_CALLS
    TOOL_CALLS --> VALIDATE --> BEFORE --> EXEC --> AFTER --> RESULT
    RESULT --> CTX
    PARTIAL --> JSONL
    RESULT --> JSONL
```

## 消息生命周期

### 用户消息流

```mermaid
sequenceDiagram
    participant Editor as TUI 编辑器
    participant IM as InteractiveMode
    participant AS as AgentSession
    participant Ext as ExtensionRunner
    participant Agent as Agent
    participant Loop as AgentLoop

    Editor->>IM: onSubmit(text)
    IM->>IM: 检查 slash 命令
    IM->>IM: 检查 bash 前缀 (!/!!)
    IM->>AS: prompt(text, images)
    AS->>Ext: emit("input", {text, images})
    Note over Ext: 扩展可 transform/handle
    AS->>Ext: emit("before_agent_start", {prompt, systemPrompt})
    Note over Ext: 扩展可修改系统提示
    AS->>Agent: prompt([userMessage])
    Agent->>Loop: agentLoop(prompts, context, config)
    Loop->>Loop: emit("agent_start")
    Loop->>Loop: emit("message_start", userMessage)
    Loop->>Loop: emit("message_end", userMessage)
```

### 助手消息流

```mermaid
sequenceDiagram
    participant Loop as AgentLoop
    participant Stream as streamSimple()
    participant Provider as 供应商
    participant AS as AgentSession
    participant TUI as TUI

    Loop->>Loop: transformContext(messages)
    Loop->>Loop: convertToLlm(messages)
    Loop->>Stream: stream(model, llmContext, options)
    Stream->>Provider: HTTP/WebSocket 请求
    Provider-->>Stream: SSE 事件流
    
    Stream-->>Loop: {type: "start", partial}
    Loop->>AS: emit("message_start")
    AS->>TUI: 创建 AssistantMessageComponent
    
    loop 流式更新
        Stream-->>Loop: {type: "text_delta", delta}
        Loop->>AS: emit("message_update")
        AS->>TUI: 增量渲染 Markdown
    end
    
    Stream-->>Loop: {type: "toolcall_end", toolCall}
    Loop->>AS: emit("message_update")
    
    Stream-->>Loop: {type: "done", message}
    Loop->>AS: emit("message_end")
    AS->>AS: 写入 JSONL
```

### 工具调用流

```mermaid
sequenceDiagram
    participant Loop as AgentLoop
    participant Ext as ExtensionRunner
    participant Tool as 工具实现
    participant FS as 文件系统/Shell
    participant TUI as TUI

    Note over Loop: 从助手消息提取 toolCalls
    
    Loop->>Loop: prepareToolCall()
    Note over Loop: 查找工具, 验证参数
    
    Loop->>Ext: beforeToolCall({toolCall, args})
    Note over Ext: 扩展可 block
    
    alt 工具被阻止
        Loop->>Loop: 返回错误结果
    else 允许执行
        Loop->>Loop: emit("tool_execution_start")
        Loop->>TUI: 显示工具开始
        Loop->>Tool: execute(id, params, signal, onUpdate)
        
        loop 部分更新
            Tool->>Loop: onUpdate(partialResult)
            Loop->>TUI: 显示部分输出
        end
        
        Tool->>FS: 实际 I/O 操作
        FS->>Tool: 结果
        Tool->>Loop: return result
    end
    
    Loop->>Ext: afterToolCall({toolCall, result})
    Note over Ext: 扩展可修改结果
    
    Loop->>Loop: emit("tool_execution_end")
    Loop->>Loop: 构建 ToolResultMessage
    Loop->>Loop: emit("message_start/end", toolResult)
```

## 消息格式转换

### AgentMessage → LLM Message

```mermaid
graph LR
    subgraph "AgentMessage 层"
        UM["UserMessage<br/>role: user"]
        AM["AssistantMessage<br/>role: assistant"]
        TRM["ToolResultMessage<br/>role: toolResult"]
        BE["BashExecutionMessage<br/>role: bashExecution"]
        CS["CompactionSummary<br/>role: compactionSummary"]
        BS["BranchSummary<br/>role: branchSummary"]
        CM["CustomMessage<br/>role: custom"]
    end

    subgraph "convertToLlm()"
        CONV["转换逻辑"]
    end

    subgraph "LLM Message 层"
        LUM["UserMessage"]
        LAM["AssistantMessage"]
        LTRM["ToolResultMessage"]
    end

    UM -->|"直通"| CONV
    AM -->|"直通"| CONV
    TRM -->|"直通"| CONV
    BE -->|"转为 user text"| CONV
    CS -->|"转为 user text"| CONV
    BS -->|"转为 user text"| CONV
    CM -->|"按 display 配置"| CONV

    CONV --> LUM
    CONV --> LAM
    CONV --> LTRM
```

### 跨供应商消息转换 (transform-messages.ts)

当在同一会话中切换模型供应商时，`pi-ai` 层自动转换消息格式：

```mermaid
graph TB
    subgraph "转换操作"
        IMG["图片降级<br/>非视觉模型"]
        THINK["思考块转换<br/>→ &lt;thinking&gt; 文本"]
        TOOLID["工具调用 ID 规范化"]
        SIGN["签名清理"]
    end

    subgraph "场景"
        S1["Anthropic → OpenAI"]
        S2["OpenAI → Google"]
        S3["视觉模型 → 文本模型"]
    end

    S1 --> THINK
    S1 --> TOOLID
    S2 --> THINK
    S2 --> SIGN
    S3 --> IMG
```

## 数据持久化流

```mermaid
flowchart TB
    subgraph "运行时"
        MSG["消息事件"]
        MC["模型变更"]
        TLC["思考级别变更"]
        COMP["压缩事件"]
    end

    subgraph "SessionManager"
        APPEND["append(entry)"]
        TREE["更新树索引"]
        LEAF["移动 leaf 指针"]
    end

    subgraph "JsonlSessionStorage"
        WRITE["追加写入 JSONL"]
        FILE["~/.pi/agent/sessions/<cwd-encoded>/xxx.jsonl"]
    end

    MSG --> APPEND
    MC --> APPEND
    TLC --> APPEND
    COMP --> APPEND
    APPEND --> TREE --> LEAF --> WRITE --> FILE
```

### JSONL 条目格式

```json
{"id":"abc123","parentId":"def456","type":"message","timestamp":1717000000000,"data":{"role":"user","content":"修复这个 bug"}}
{"id":"ghi789","parentId":"abc123","type":"message","timestamp":1717000001000,"data":{"role":"assistant","content":[{"type":"text","text":"好的，让我查看一下"}]}}
{"id":"jkl012","parentId":"ghi789","type":"model_change","timestamp":1717000002000,"data":{"provider":"anthropic","model":"claude-sonnet-4-20250514"}}
```

## 上下文构建流程

从会话树路径构建 LLM 上下文：

```mermaid
flowchart TD
    TREE["会话树"] --> PATH["提取根到叶路径"]
    PATH --> CHECK{"路径中有压缩节点?"}
    
    CHECK -->|"是"| COMP["使用压缩摘要<br/>+ 压缩后的消息"]
    CHECK -->|"否"| ALL["使用所有消息"]
    
    COMP --> BUILD["构建 SessionContext"]
    ALL --> BUILD
    
    BUILD --> SC["SessionContext<br/>{messages, thinkingLevel, model}"]
    SC --> TRANSFORM["transformContext()<br/>应用上下文窗口管理"]
    TRANSFORM --> CONVERT["convertToLlm()<br/>过滤非 LLM 消息"]
    CONVERT --> FINAL["最终 LLM Context"]
```

## API Key 解析流程

```mermaid
flowchart TD
    START["需要 API Key"] --> CHECK1{"auth.json 中有?"}
    CHECK1 -->|"是"| OAUTH["检查 OAuth token 有效期"]
    OAUTH --> EXPIRED{"过期?"}
    EXPIRED -->|"是"| REFRESH["refreshToken()"]
    REFRESH --> USE
    EXPIRED -->|"否"| USE["使用 token"]
    CHECK1 -->|"否"| CHECK2{"环境变量中有?"}
    CHECK2 -->|"是"| USE2["使用环境变量值"]
    CHECK2 -->|"否"| CHECK3{"models.json 中配置?"}
    CHECK3 -->|"是"| USE3["使用配置的 key"]
    CHECK3 -->|"否"| CHECK4{"getApiKey() 动态解析?"}
    CHECK4 -->|"是"| USE4["使用动态值"]
    CHECK4 -->|"否"| FAIL["无可用 Key"]
```
