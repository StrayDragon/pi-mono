# 事件驱动架构

## 事件系统全景

Pi 的事件系统贯穿从 Agent 核心到 UI 渲染的整个链条。事件层层传递、逐级丰富，是解耦各子系统的核心机制。

```mermaid
graph TB
    subgraph "Layer 1: pi-ai"
        AE["AssistantMessageEvent<br/>start/text_delta/done/error"]
    end

    subgraph "Layer 2: pi-agent-core"
        AGE["AgentEvent<br/>agent_start/turn_start/message_*/tool_*"]
    end

    subgraph "Layer 3: pi-coding-agent"
        ASE["AgentSessionEvent<br/>继承 AgentEvent + 扩展事件"]
        EXE["ExtensionEvent<br/>30+ 事件类型"]
    end

    subgraph "Layer 4: UI"
        UIE["UI 更新<br/>组件创建/更新/销毁"]
    end

    AE -->|"封装为"| AGE
    AGE -->|"丰富为"| ASE
    ASE -->|"分发给"| EXE
    ASE -->|"驱动"| UIE
```

## Layer 1: AssistantMessageEvent (pi-ai)

LLM 供应商返回的流式事件，最底层的事件协议：

```mermaid
stateDiagram-v2
    [*] --> start
    start --> text_start
    start --> thinking_start
    start --> toolcall_start
    
    text_start --> text_delta
    text_delta --> text_delta
    text_delta --> text_end
    
    thinking_start --> thinking_delta
    thinking_delta --> thinking_delta
    thinking_delta --> thinking_end
    
    toolcall_start --> toolcall_delta
    toolcall_delta --> toolcall_delta
    toolcall_delta --> toolcall_end
    
    text_end --> text_start: 多个文本块
    text_end --> toolcall_start: 文本后工具
    thinking_end --> text_start: 思考后文本
    toolcall_end --> toolcall_start: 多个工具
    
    text_end --> done
    toolcall_end --> done
    text_end --> error
    toolcall_end --> error
    start --> error: 请求失败
```

| 事件 | 载荷 | 说明 |
|------|------|------|
| `start` | `{partial: AssistantMessage}` | 流开始，初始空消息 |
| `text_start` | `{contentIndex, partial}` | 文本块开始 |
| `text_delta` | `{contentIndex, delta, partial}` | 文本增量 |
| `text_end` | `{contentIndex, content, partial}` | 文本块完成 |
| `thinking_start/delta/end` | 同上结构 | 思考/推理块 |
| `toolcall_start/delta/end` | `{contentIndex, toolCall, partial}` | 工具调用块 |
| `done` | `{reason, message}` | 成功完成 |
| `error` | `{reason, error}` | 错误终止 |

## Layer 2: AgentEvent (pi-agent-core)

Agent 运行时事件，封装了完整的 turn/tool 生命周期：

```mermaid
stateDiagram-v2
    [*] --> agent_start
    
    agent_start --> turn_start
    
    state "Turn" as turn {
        turn_start2: turn_start
        turn_start2 --> message_start_user: 用户消息
        message_start_user --> message_end_user
        message_end_user --> message_start_assistant: 流式助手回复
        
        state "Assistant Streaming" as streaming {
            message_start_assistant2: message_start
            message_start_assistant2 --> message_update: 流式更新
            message_update --> message_update
            message_update --> message_end_assistant: message_end
        }
        
        message_end_assistant --> tool_execution_start: 有工具调用
        
        state "Tool Execution" as tool {
            tool_execution_start2: tool_execution_start
            tool_execution_start2 --> tool_execution_update
            tool_execution_update --> tool_execution_update
            tool_execution_update --> tool_execution_end
        }
        
        tool_execution_end --> message_start_toolresult
        message_start_toolresult --> message_end_toolresult
        
        message_end_toolresult --> turn_end
        message_end_assistant --> turn_end: 无工具调用
    }
    
    turn_end --> turn_start: 有更多工具/消息
    turn_end --> agent_end: 完成
    
    agent_end --> [*]
```

| 事件 | 时机 | 关键载荷 |
|------|------|---------|
| `agent_start` | 循环开始 | 无 |
| `agent_end` | 循环结束 | `messages: AgentMessage[]` |
| `turn_start` | 新一轮开始 | 无 |
| `turn_end` | 一轮结束 | `message, toolResults` |
| `message_start` | 消息开始 | `message: AgentMessage` |
| `message_update` | 助手流式更新 | `message, assistantMessageEvent` |
| `message_end` | 消息结束 | `message: AgentMessage` |
| `tool_execution_start` | 工具开始执行 | `toolCallId, toolName, args` |
| `tool_execution_update` | 工具部分输出 | `partialResult` |
| `tool_execution_end` | 工具执行完成 | `result, isError` |

## Layer 3: ExtensionEvent (pi-coding-agent)

扩展系统的事件是最丰富的层，包含 30+ 种事件类型：

```mermaid
graph TB
    subgraph "资源事件"
        RD["resources_discover"]
    end

    subgraph "会话事件"
        SS["session_start"]
        SBS["session_before_switch"]
        SBF["session_before_fork"]
        SBC["session_before_compact"]
        SC["session_compact"]
        SBT["session_before_tree"]
        ST["session_tree"]
        SSD["session_shutdown"]
    end

    subgraph "Agent 事件"
        CTX["context"]
        BPR["before_provider_request"]
        APR["after_provider_response"]
        BAS["before_agent_start"]
        AS["agent_start"]
        AE["agent_end"]
        TS["turn_start"]
        TE["turn_end"]
    end

    subgraph "消息事件"
        MS["message_start"]
        MU["message_update"]
        ME["message_end"]
    end

    subgraph "工具事件"
        TES["tool_execution_start"]
        TEU["tool_execution_update"]
        TEE["tool_execution_end"]
        TC["tool_call"]
        TR["tool_result"]
    end

    subgraph "输入事件"
        INP["input"]
        UB["user_bash"]
    end

    subgraph "模型事件"
        MSE["model_select"]
        TLS["thinking_level_select"]
    end
```

### 事件分类

#### 可拦截事件（返回结果影响行为）

| 事件 | 返回结果 | 拦截能力 |
|------|---------|---------|
| `input` | `action: "handled"` | 完全接管输入处理 |
| `input` | `action: "transform"` | 修改输入文本 |
| `tool_call` | `{block: true}` | 阻止工具执行 |
| `tool_result` | `{content, isError}` | 修改工具结果 |
| `context` | `{messages}` | 替换上下文消息 |
| `before_provider_request` | payload | 替换请求负载 |
| `before_agent_start` | `{systemPrompt}` | 替换系统提示 |
| `message_end` | `{message}` | 替换最终消息 |
| `session_before_switch` | `{cancel: true}` | 取消会话切换 |
| `session_before_fork` | `{cancel: true}` | 取消分支 |
| `session_before_compact` | `{cancel: true}` | 取消压缩 |
| `session_before_tree` | `{cancel: true}` | 取消树导航 |
| `user_bash` | `{operations, result}` | 替换 bash 执行 |

#### 通知事件（只读观察）

| 事件 | 用途 |
|------|------|
| `session_start` | 会话初始化后的设置 |
| `session_compact` | 压缩完成后的通知 |
| `session_tree` | 树导航完成后的通知 |
| `session_shutdown` | 清理资源 |
| `agent_start/end` | Agent 运行状态追踪 |
| `turn_start/end` | 轮次追踪 |
| `message_start/update` | UI 渲染驱动 |
| `tool_execution_*` | 工具执行追踪 |
| `model_select` | 模型变更追踪 |
| `thinking_level_select` | 思考级别变更追踪 |

## 事件分发机制

```mermaid
sequenceDiagram
    participant Source as 事件源
    participant Runner as ExtensionRunner
    participant Ext1 as 扩展 1
    participant Ext2 as 扩展 2
    participant Core as 核心处理

    Source->>Runner: dispatchEvent(event)
    Runner->>Ext1: handler(event, ctx)
    Ext1->>Runner: result1
    Runner->>Ext2: handler(event, ctx)
    Ext2->>Runner: result2
    Runner->>Runner: 合并结果
    Runner->>Core: 返回合并后的结果
```

### 结果合并策略

| 事件类型 | 合并策略 |
|---------|---------|
| `tool_call` | 任一 `block: true` 则阻止 |
| `tool_result` | 最后一个非空结果生效 |
| `context` | 最后一个非空 messages 生效 |
| `before_agent_start` | systemPrompt 链式替换 |
| `input` | 第一个 `handled` 或最后一个 `transform` |
| `session_before_*` | 任一 `cancel: true` 则取消 |

## 事件与 UI 的映射

```mermaid
graph LR
    subgraph "事件"
        E1["agent_start"]
        E2["message_start<br/>(assistant)"]
        E3["message_update"]
        E4["tool_execution_start"]
        E5["tool_execution_update"]
        E6["tool_execution_end"]
        E7["message_end"]
        E8["agent_end"]
    end

    subgraph "UI 操作"
        U1["显示加载动画"]
        U2["创建 AssistantMessageComponent"]
        U3["增量渲染 Markdown"]
        U4["创建 ToolExecutionComponent"]
        U5["更新工具输出"]
        U6["完成工具显示"]
        U7["完成消息"]
        U8["移除加载, 恢复编辑器焦点"]
    end

    E1 --> U1
    E2 --> U2
    E3 --> U3
    E4 --> U4
    E5 --> U5
    E6 --> U6
    E7 --> U7
    E8 --> U8
```

## 钩子系统 (AgentHarness)

AgentHarness 层的钩子提供了比扩展事件更底层的控制点：

```mermaid
graph TB
    subgraph "AgentHarness 钩子"
        H1["beforeToolCall<br/>工具执行前"]
        H2["afterToolCall<br/>工具执行后"]
        H3["shouldStopAfterTurn<br/>是否停止"]
        H4["prepareNextTurn<br/>准备下一轮"]
        H5["getSteeringMessages<br/>获取 steering 消息"]
        H6["getFollowUpMessages<br/>获取 follow-up 消息"]
        H7["convertToLlm<br/>消息转换"]
        H8["transformContext<br/>上下文变换"]
        H9["getApiKey<br/>动态 API Key"]
    end

    subgraph "调用时机"
        T1["工具参数验证后"]
        T2["工具执行完成后"]
        T3["turn_end 后"]
        T4["准备下次 LLM 调用前"]
        T5["每轮结束时"]
        T6["Agent 即将停止时"]
        T7["每次 LLM 调用前"]
        T8["convertToLlm 前"]
        T9["每次 LLM 调用前"]
    end

    H1 --- T1
    H2 --- T2
    H3 --- T3
    H4 --- T4
    H5 --- T5
    H6 --- T6
    H7 --- T7
    H8 --- T8
    H9 --- T9
```
