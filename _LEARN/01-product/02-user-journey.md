# 用户旅程与使用场景

## 用户旅程总览

```mermaid
journey
    title Pi 用户的典型一天
    section 安装与设置
      npm install -g pi-coding-agent: 5: 用户
      配置 API Key: 4: 用户
      首次启动 pi: 5: 用户
    section 日常使用
      打开终端, 进入项目目录: 5: 用户
      输入 pi 启动交互模式: 5: 用户
      描述编码任务: 5: 用户
      审查 Pi 的修改: 4: 用户, Pi
      接受或要求修正: 4: 用户
    section 高级操作
      切换模型: 4: 用户
      使用 slash 命令: 4: 用户
      查看会话历史: 3: 用户
      导出 HTML 报告: 3: 用户
```

## 首次使用流程

```mermaid
flowchart TD
    A["安装 Pi"] --> B{"有 API Key?"}
    B -->|"是"| C["设置环境变量<br/>ANTHROPIC_API_KEY"]
    B -->|"否"| D["pi login<br/>OAuth 登录"]
    C --> E["cd 项目目录"]
    D --> E
    E --> F["运行 pi"]
    F --> G["选择模型"]
    G --> H["开始对话"]
    H --> I["Pi 读取代码<br/>理解上下文"]
    I --> J["Pi 执行任务<br/>读取/编辑/运行"]
    J --> K{"满意?"}
    K -->|"是"| L["继续下一个任务"]
    K -->|"否"| M["提供反馈<br/>Pi 迭代"]
    M --> J
```

## 核心使用场景

### 场景 1：Bug 修复

```mermaid
sequenceDiagram
    participant U as 用户
    participant P as Pi Agent
    participant LLM as LLM 模型
    participant FS as 文件系统

    U->>P: "修复 login 页面的表单验证 bug"
    P->>LLM: 发送用户请求 + 系统提示
    LLM->>P: 调用 grep 工具搜索
    P->>FS: grep "login" "validation"
    FS->>P: 搜索结果
    P->>LLM: 返回搜索结果
    LLM->>P: 调用 read 工具读取文件
    P->>FS: read src/pages/login.ts
    FS->>P: 文件内容
    P->>LLM: 返回文件内容
    LLM->>P: 调用 edit 工具修复代码
    P->>FS: edit src/pages/login.ts
    FS->>P: 编辑成功
    P->>LLM: 返回结果
    LLM->>P: 调用 bash 运行测试
    P->>FS: bash "npm test"
    FS->>P: 测试通过
    LLM->>P: 汇总修复结果
    P->>U: 显示修复详情和测试结果
```

### 场景 2：代码重构

用户请求 Pi 重构一个模块：

1. **分析阶段**：Pi 用 `read` 和 `grep` 理解现有代码结构
2. **规划阶段**：Pi 制定重构方案并告知用户
3. **执行阶段**：Pi 逐文件 `edit`，保持增量修改
4. **验证阶段**：Pi 用 `bash` 运行类型检查和测试

### 场景 3：新功能开发

```mermaid
flowchart LR
    A["用户描述需求"] --> B["Pi 分析<br/>现有代码"]
    B --> C["Pi 创建<br/>新文件"]
    C --> D["Pi 编辑<br/>相关文件"]
    D --> E["Pi 运行<br/>测试/检查"]
    E --> F{"通过?"}
    F -->|"否"| G["Pi 修复"]
    G --> E
    F -->|"是"| H["用户审查"]
```

### 场景 4：脚本化使用（Print 模式）

```bash
# 单次提问
pi -p "解释这个函数的作用" < src/complex.ts

# 管道使用
cat error.log | pi -p "分析这个错误日志"

# JSON 输出
pi -p "列出所有 TODO" --json
```

### 场景 5：IDE 集成（RPC 模式）

```mermaid
sequenceDiagram
    participant IDE as IDE 插件
    participant RPC as Pi RPC 模式
    participant Agent as Agent 核心

    IDE->>RPC: JSONL: {"type":"prompt","text":"..."}
    RPC->>Agent: 转发到 Agent
    Agent->>RPC: 事件流
    RPC->>IDE: JSONL: {"type":"message_start",...}
    RPC->>IDE: JSONL: {"type":"message_update",...}
    RPC->>IDE: JSONL: {"type":"tool_execution",...}
    RPC->>IDE: JSONL: {"type":"message_end",...}
```

## 交互模式详解

### Interactive 模式（主模式）

```mermaid
graph TB
    subgraph TUI界面
        Header["头部: 快捷键提示"]
        Chat["聊天区: 消息 + 工具输出"]
        Editor["编辑器: 多行输入"]
        Footer["底栏: 模型 / Token / 思考级别"]
    end

    subgraph 用户操作
        Type["输入提示词"]
        Shortcut["快捷键操作"]
        Slash["Slash 命令"]
    end

    Type --> Editor
    Shortcut --> Header
    Slash --> Editor

    subgraph 快捷键
        Esc["Esc: 中断"]
        CtrlD["Ctrl+D: 退出"]
        CtrlR["Ctrl+R: 切换模型"]
        CtrlK["Ctrl+K: 压缩上下文"]
    end
```

### 关键用户交互

| 操作 | 快捷键/命令 | 说明 |
|------|------------|------|
| 发送消息 | Enter | 提交当前编辑器内容 |
| 换行 | Shift+Enter | 在编辑器中换行 |
| 中断 | Escape | 停止当前 Agent 运行 |
| 退出 | Ctrl+D | 退出 Pi |
| 切换模型 | Ctrl+R | 打开模型选择器 |
| 调整思考级别 | Ctrl+T | 循环切换推理深度 |
| 压缩上下文 | Ctrl+K | 触发上下文压缩 |
| 会话树 | Ctrl+L | 浏览会话分支 |
| 新会话 | Ctrl+N | 开始新会话 |
| 运行 bash | `!cmd` | 在编辑器中直接执行命令 |
| Slash 命令 | `/command` | 调用注册的命令 |

## 用户心智模型

```mermaid
graph TB
    subgraph 用户视角
        U["我"] -->|"对话"| P["Pi = 我的编码助手"]
        P -->|"读写我的代码"| C["我的项目"]
        P -->|"使用各种 AI"| M["AI 模型"]
        P -->|"记住上下文"| S["会话历史"]
    end

    subgraph 实际发生的事
        P2["Pi Agent"]
        P2 -->|"AgentLoop"| L["LLM 调用"]
        P2 -->|"工具执行"| T["read/bash/edit/write"]
        P2 -->|"JSONL"| S2["会话持久化"]
        P2 -->|"扩展事件"| E["扩展系统"]
    end
```

## 使用频率矩阵

```mermaid
quadrantChart
    title 功能使用频率 vs 复杂度
    x-axis "简单" --> "复杂"
    y-axis "偶尔" --> "每天"
    quadrant-1 高频复杂
    quadrant-2 高频简单
    quadrant-3 低频简单
    quadrant-4 低频复杂
    "日常编码对话": [0.3, 0.95]
    "Bug 修复": [0.4, 0.85]
    "代码审查": [0.35, 0.6]
    "切换模型": [0.15, 0.5]
    "会话分支": [0.55, 0.2]
    "编写扩展": [0.8, 0.1]
    "自定义供应商": [0.75, 0.08]
    "RPC 集成": [0.85, 0.05]
    "上下文压缩": [0.4, 0.3]
```
