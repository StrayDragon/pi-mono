# pi-tui 源码深度分析

> `@earendil-works/pi-tui` — 差分渲染终端 UI 框架

## 包概览

pi-tui 是一个**独立的终端 UI 库**，完全不依赖 AI 或 Agent。核心特性是差分渲染——只更新变化的行。

```mermaid
graph TB
    subgraph "pi-tui"
        TUI_CLS["TUI 类<br/>渲染引擎"]
        COMP["Component 接口<br/>渲染单元"]
        TERM["Terminal 接口<br/>终端抽象"]
        KEYS["Keys 模块<br/>输入解析"]
        KB["KeybindingsManager<br/>快捷键"]
    end

    subgraph "组件库"
        TEXT["Text"]
        MD["Markdown"]
        EDITOR["Editor"]
        SELECT["SelectList"]
        INPUT["Input"]
        LOADER["Loader"]
        IMAGE["Image"]
        BOX["Box"]
    end

    TUI_CLS --> COMP
    TUI_CLS --> TERM
    TUI_CLS --> KEYS
    COMP --> TEXT & MD & EDITOR & SELECT & INPUT & LOADER & IMAGE & BOX
```

## 文件结构

```
packages/tui/src/
├── index.ts                # 公共导出
├── tui.ts                  # TUI 类 (~1200 行, 核心)
├── terminal.ts             # Terminal 接口 + ProcessTerminal
├── terminal-image.ts       # Kitty/iTerm2 图片协议
├── keys.ts                 # 键盘输入解析
├── keybindings.ts          # 快捷键管理器
├── stdin-buffer.ts         # stdin 输入缓冲
├── utils.ts                # 宽度计算, ANSI 处理
├── autocomplete.ts         # 自动补全
├── fuzzy.ts                # 模糊匹配
├── editor-component.ts     # 编辑器接口
├── undo-stack.ts           # 撤销栈
├── kill-ring.ts            # kill ring (Emacs 风格)
├── native-modifiers.ts     # 原生修饰键检测
└── components/
    ├── box.ts              # 带背景的容器
    ├── cancellable-loader.ts
    ├── editor.ts           # 多行编辑器 (~1500 行)
    ├── image.ts            # 内联图片
    ├── input.ts            # 单行输入
    ├── loader.ts           # 加载动画
    ├── markdown.ts         # Markdown 渲染器
    ├── select-list.ts      # 选择列表
    ├── settings-list.ts    # 设置列表
    ├── spacer.ts           # 空白
    ├── text.ts             # 多行文本
    └── truncated-text.ts   # 截断文本
```

## 核心概念: Component

```mermaid
classDiagram
    class Component {
        <<interface>>
        +render(width: number): string[]
        +handleInput?(data: string): void
        +wantsKeyRelease?: boolean
        +invalidate(): void
    }

    class Container {
        -children: Component[]
        +addChild(component)
        +removeChild(component)
        +render(width): string[]
    }

    class TUI {
        +terminal: Terminal
        -previousLines: string[]
        -focusedComponent: Component
        -overlayStack: Overlay[]
        +addChild(component)
        +setFocus(component)
        +showOverlay(component, options)
        +requestRender()
        +start()
        +stop()
    }

    Component <|-- Container
    Container <|-- TUI
```

每个 Component 的 `render(width)` 方法返回 `string[]`——每个字符串是一行终端输出（含 ANSI 转义码）。

## 差分渲染引擎 (tui.ts)

### 渲染管线

```mermaid
flowchart TD
    REQ["requestRender()"] --> THROTTLE["节流: 16ms 最小间隔"]
    THROTTLE --> RENDER["render(width)"]
    RENDER --> CHILDREN["遍历子组件<br/>收集所有行"]
    CHILDREN --> OVERLAY["compositeOverlays()<br/>叠加覆盖层"]
    OVERLAY --> CURSOR["extractCursorPosition()<br/>提取光标位置"]
    CURSOR --> RESET["applyLineResets()<br/>添加行尾重置"]
    RESET --> DIFF{"差分策略选择"}
    
    DIFF -->|"首次渲染"| FULL["全量绘制"]
    DIFF -->|"宽度变化"| FULL
    DIFF -->|"窗口缩小"| CLEAR["清屏 + 全量绘制"]
    DIFF -->|"其他"| PARTIAL["逐行对比差分"]
    
    PARTIAL --> CMP["对比 newLines vs previousLines"]
    CMP -->|"不同"| UPDATE["移动光标 + 清行 + 重写"]
    CMP -->|"相同"| SKIP["跳过"]
    
    FULL & CLEAR & PARTIAL --> WRITE["terminal.write(output)"]
    WRITE --> STORE["previousLines = newLines"]
```

### 差分对比核心逻辑

```mermaid
graph TB
    subgraph "差分渲染"
        LINE["逐行对比"]
        LINE --> SAME{"newLine === prevLine?"}
        SAME -->|"是"| SKIP["跳过该行"]
        SAME -->|"否"| MOVE["\\x1b[row;1H<br/>移动光标到该行"]
        MOVE --> CLEAR_LINE["\\x1b[2K<br/>清除该行"]
        CLEAR_LINE --> WRITE_LINE["写入新行内容"]
    end
    
    subgraph "额外行处理"
        MORE["新行数 > 旧行数"] --> APPEND["追加新行"]
        LESS["新行数 < 旧行数"] --> ERASE["擦除多余行"]
    end
```

### 覆盖层 (Overlay) 系统

```mermaid
graph TB
    subgraph "叠加层栈"
        BASE["基础内容 (子组件)"]
        O1["Overlay 1 (模型选择器)"]
        O2["Overlay 2 (确认对话框)"]
    end

    subgraph "定位"
        ANCHOR["anchor: 'bottom' | 'top'"]
        PERCENT["widthPercent / heightPercent"]
        OFFSET["offsetX / offsetY"]
    end

    BASE --> COMPOSITE["compositeOverlays()"]
    O1 --> COMPOSITE
    O2 --> COMPOSITE
    COMPOSITE --> FINAL["最终渲染行"]
```

覆盖层渲染到独立层，然后按 Z 序合成到基础内容上。支持居中、锚定、百分比尺寸。

## 输入处理链

```mermaid
flowchart TD
    STDIN["stdin raw 数据"] --> BUFFER["StdinBuffer<br/>拆分完整序列"]
    BUFFER --> TUI_INPUT["TUI.handleInput()"]
    TUI_INPUT --> LISTENERS["inputListeners<br/>(全局拦截)"]
    LISTENERS -->|"未消费"| DEBUG{"调试键?<br/>Shift+Ctrl+D"}
    DEBUG -->|"是"| DEBUG_MODE["切换调试模式"]
    DEBUG -->|"否"| OVERLAY_CHECK{"有覆盖层?"}
    OVERLAY_CHECK -->|"是"| OVERLAY_INPUT["覆盖层组件.handleInput()"]
    OVERLAY_CHECK -->|"否"| FOCUS{"有焦点组件?"}
    FOCUS -->|"是"| FOCUSED["focusedComponent.handleInput()"]
    FOCUS -->|"否"| DROP["丢弃"]
    
    FOCUSED --> RERENDER["requestRender()"]
```

### Kitty 键盘协议

pi-tui 支持 Kitty 键盘协议（更精确的键检测）：

```mermaid
graph LR
    STDIN["原始输入"] --> DETECT{"协议检测"}
    DETECT -->|"Kitty"| KITTY["解析 CSI u 序列<br/>精确修饰键"]
    DETECT -->|"传统"| LEGACY["解析传统转义序列<br/>有歧义"]
    KITTY --> KEY["Key 对象"]
    LEGACY --> KEY
    KEY --> KB["KeybindingsManager.matches()"]
```

## 组件库

### Editor 组件 (~1500 行)

编辑器是最复杂的组件，支持：

```mermaid
graph TB
    subgraph "Editor 功能"
        WRAP["自动换行"]
        CURSOR_NAV["光标导航"]
        SELECT_TEXT["文本选择"]
        UNDO["撤销/重做"]
        KILL["Kill ring (Ctrl+K/Y)"]
        PASTE["括号粘贴模式"]
        AC["自动补全"]
        SCROLL["滚动"]
        IME["IME 光标定位"]
    end

    subgraph "渲染"
        LINES["行缓存"]
        GUTTER["行号槽"]
        BORDER["边框"]
        HIGHLIGHT["选中高亮"]
    end
```

### Markdown 组件

使用 `marked` 库解析 Markdown，然后通过主题函数渲染为 ANSI 彩色输出：

```mermaid
graph LR
    MD["Markdown 文本"] --> PARSE["marked.parse()"]
    PARSE --> TOKENS["Token 流"]
    TOKENS --> RENDER["渲染为 ANSI"]
    RENDER --> THEME["应用 MarkdownTheme"]
    THEME --> LINES["string[]"]
```

主题接口（由调用方提供）：

```typescript
interface MarkdownTheme {
   heading: (text: string) => string;
   link: (text: string) => string;
   code: (text: string) => string;
   codeBlock: (text: string) => string;
   highlightCode?: (code: string, lang?: string) => string[];
   // ...更多
}
```

### SelectList 组件

```mermaid
graph TB
    subgraph "SelectList"
        ITEMS["items: {label, value}[]"]
        FILTER["可选过滤"]
        SCROLL_VIEW["滚动视图"]
        HIGHLIGHT_ITEM["当前选中高亮"]
        KB_NAV["键盘导航<br/>(up/down/enter/esc)"]
    end
```

## 宽度计算 (utils.ts)

终端宽度计算是 TUI 的基础难题：

```mermaid
graph LR
    TEXT["字符串"] --> STRIP["移除 ANSI 转义码"]
    STRIP --> WIDTH["逐字符计算宽度"]
    WIDTH --> CJK{"CJK / Emoji?"}
    CJK -->|"是"| W2["宽度 = 2"]
    CJK -->|"否"| W1["宽度 = 1"]
    
    TEXT2["字符串"] --> TRUNC["truncateToWidth(str, maxWidth)"]
    TRUNC --> SCAN["扫描直到宽度达限"]
    SCAN --> ELLIPSIS["添加省略号"]
    
    TEXT3["长字符串"] --> WRAP_TEXT["wrapTextWithAnsi(str, width)"]
    WRAP_TEXT --> PRESERVE["保留跨行 ANSI 状态"]
```

`visibleWidth()` 是性能热点——在每次渲染时对每行调用。使用 `get-east-asian-width` 库处理 CJK 和 emoji 的宽度。

## 与 coding-agent 的集成

pi-tui **不感知** Agent 的存在。coding-agent 通过以下方式桥接：

```mermaid
graph TB
    subgraph "pi-tui 提供"
        TUI_CLS2["TUI + Container"]
        COMP2["Component 接口"]
        EDITOR2["Editor + EditorComponent"]
        THEME_IF["Theme 接口"]
        KB2["Keybindings"]
    end

    subgraph "coding-agent 提供"
        CUSTOM_EDITOR["CustomEditor<br/>扩展 Editor"]
        APP_KB["App Keybindings<br/>声明合并"]
        DARK_THEME["dark.json / light.json"]
        MSG_COMP["AssistantMessageComponent"]
        TOOL_COMP["ToolExecutionComponent"]
        FOOTER_COMP["FooterComponent"]
    end

    TUI_CLS2 --> CUSTOM_EDITOR
    COMP2 --> MSG_COMP & TOOL_COMP & FOOTER_COMP
    EDITOR2 --> CUSTOM_EDITOR
    THEME_IF --> DARK_THEME
    KB2 --> APP_KB
```
