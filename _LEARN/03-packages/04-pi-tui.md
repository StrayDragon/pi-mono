# pi-tui 包深度分析

> 终端 UI 库：差分渲染、组件系统、快捷键管理

---

## 1. 包概览

### 1.1 定位

**pi-tui** 是 pi-mono 架构中的 **L4: Presentation Layer**，一个功能完整的终端 UI 库。

**核心职责**：
- **差分渲染**：仅更新变化区域，提升性能
- **组件系统**：可组合的 UI 组件
- **快捷键管理**：灵活的键盘绑定系统
- **终端兼容**：支持多种终端协议（Kitty、iTerm2）
- **文本处理**：换行、ANSI 代码、东亚字符支持

**依赖**：无（零外部依赖）

**被依赖**：
- `@mariozechner/pi-coding-agent` - 交互模式 UI
- `@mariozechner/pi-web-ui` - Web UI 组件

### 1.2 目录结构

```
packages/tui/src/
├── index.ts              # 主要导出文件
├── tui.ts                # 核心 TUI 类（差分渲染）
├── terminal.ts           # 终端接口抽象
├── keybindings.ts        # 键盘绑定系统
├── keys.ts               # 键盘输入处理
├── stdin-buffer.ts       # 输入缓冲
├── kill-ring.ts          # Emacs 风格 kill ring
├── undo-stack.ts         # 撤销栈
├── autocomplete.ts       # 自动补全系统
├── fuzzy.ts              # 模糊匹配算法
├── terminal-image.ts     # 终端图像支持
├── utils.ts              # 工具函数
├── editor-component.ts   # 编辑器组件接口
└── components/           # 组件库
    ├── box.ts            # 容器组件
    ├── text.ts           # 文本组件
    ├── editor.ts         # 文本编辑器（2000+ 行）
    ├── input.ts          # 输入框
    ├── select-list.ts    # 选择列表
    ├── markdown.ts       # Markdown 渲染器
    ├── image.ts          # 图像组件
    ├── loader.ts         # 加载器
    ├── cancellable-loader.ts  # 可取消加载器
    ├── spacer.ts         # 间隔组件
    ├── truncated-text.ts # 截断文本
    └── settings-list.ts  # 设置列表
```

### 1.3 关键文件

| 文件 | 行数 | 核心功能 |
|------|------|---------|
| `tui.ts` | 800+ | TUI 核心、差分渲染 |
| `components/editor.ts` | 2000+ | Emacs 风格编辑器 |
| `components/markdown.ts` | 800+ | Markdown 渲染 |
| `keybindings.ts` | 400+ | 快捷键系统 |
| `terminal-image.ts` | 500+ | 图像协议支持 |

---

## 2. 核心架构

### 2.1 组件接口

```typescript
// 基础组件接口
export interface Component {
    // 使组件失效，触发重新渲染
    invalidate(): void;

    // 渲染组件
    render(width: number): string[];
}

// 可聚焦组件接口
export interface Focusable {
    focused: boolean;
    handleInput?(data: string): void;
}

// 容器接口
export interface Container extends Component {
    // 添加子组件
    add(component: Component, options?: AddOptions): void;

    // 移除子组件
    remove(component: Component): void;

    // 聚焦子组件
    focus(component: Focusable): void;
}
```

### 2.2 TUI 核心类

**源文件**：`/packages/tui/src/tui.ts`

```typescript
export class TUI implements Container {
    // 终端尺寸
    width: number;
    height: number;

    // 子组件
    private children: Array<{
        component: Component;
        options?: AddOptions;
    }> = [];

    // 渲染缓存
    private renderCache = new Map<string, string[]>();

    // 硬件光标位置
    private hardwareCursor = { row: 0, col: 0 };

    // 终端能力
    capabilities: TerminalCapabilities;

    // 构造函数
    constructor(options?: TUIOptions);

    // 渲染请求
    requestRender(): void;

    // 添加子组件
    add(component: Component, options?: AddOptions): void;

    // 移除子组件
    remove(component: Component): void;

    // 聚焦组件
    focus(component: Focusable): void;

    // 启动 TUI
    async start(): Promise<void>;

    // 停止 TUI
    async stop(): Promise<void>;

    // 渲染（差分更新）
    private render(): void;
}
```

### 2.3 差分渲染实现

```typescript
private render(): void {
    // 1. 渲染所有子组件
    const lines: string[] = [];
    for (const { component, options } of this.children) {
        const componentLines = component.render(options?.width ?? this.width);
        lines.push(...componentLines);
    }

    // 2. 应用叠加层
    const compositeLines = this.compositeOverlays(
        lines,
        this.width,
        this.height
    );

    // 3. 计算差异
    const diff = this.computeDiff(this.lastLines, compositeLines);

    // 4. 输出差异
    this.outputDiff(diff);

    // 5. 保存当前帧
    this.lastLines = compositeLines;
}

private computeDiff(prev: string[], curr: string[]): Diff[] {
    const diff: Diff[] = [];

    for (let i = 0; i < Math.max(prev.length, curr.length); i++) {
        if (prev[i] !== curr[i]) {
            diff.push({
                line: i,
                text: curr[i] ?? "",
            });
        }
    }

    return diff;
}

private outputDiff(diff: Diff[]): void {
    for (const { line, text } of diff) {
        // 移动光标到指定行
        process.stdout.write(`\x1b[${line + 1};0H`);

        // 清除行
        process.stdout.write("\x1b[2K");

        // 输出新内容
        process.stdout.write(text);
    }

    // 移动光标到硬件位置
    if (this.hardwareCursor.row > 0) {
        process.stdout.write(
            `\x1b[${this.hardwareCursor.row + 1};${this.hardwareCursor.col + 1}H`
        );
    }
}
```

---

## 3. 组件系统

### 3.1 Box 容器组件

**源文件**：`/packages/tui/src/components/box.ts`

```typescript
export class Box implements Component {
    private children: Component[] = [];
    private paddingX: number = 0;
    private paddingY: number = 0;
    private bgFn?: (text: string) => string;
    private cache?: RenderCache;

    constructor(options?: BoxOptions) {
        this.paddingX = options?.paddingX ?? 0;
        this.paddingY = options?.paddingY ?? 0;
        this.bgFn = options?.bg;
    }

    add(component: Component): void {
        this.children.push(component);
        this.invalidate();
    }

    render(width: number): string[] {
        // 1. 渲染子组件
        const childWidth = width - 2 * this.paddingX;
        const childLines: string[] = [];

        for (const child of this.children) {
            const lines = child.render(childWidth);
            childLines.push(...lines);
        }

        // 2. 应用内边距
        const paddingLine = " ".repeat(width);
        const lines: string[] = [];

        // 上下内边距
        for (let i = 0; i < this.paddingY; i++) {
            lines.push(paddingLine);
        }

        // 子组件行
        for (const line of childLines) {
            const paddedLine =
                " ".repeat(this.paddingX) +
                line.padEnd(width - 2 * this.paddingX) +
                " ".repeat(this.paddingX);

            lines.push(this.bgFn ? this.bgFn(paddedLine) : paddedLine);
        }

        // 下内边距
        for (let i = 0; i < this.paddingY; i++) {
            lines.push(paddingLine);
        }

        return lines;
    }

    invalidate(): void {
        this.cache = undefined;
    }

    private matchCache(
        width: number,
        childLines: string[],
        bgSample: string
    ): boolean {
        if (!this.cache) return false;

        return (
            this.cache.width === width &&
            this.cache.childLines.length === childLines.length &&
            this.cache.bgSample === bgSample
        );
    }
}
```

### 3.2 Text 文本组件

**源文件**：`/packages/tui/src/components/text.ts`

```typescript
export class Text implements Component {
    private text: string;
    private cachedText?: string;
    private cachedWidth?: number;
    private cachedLines?: string[];

    constructor(text: string) {
        this.text = text;
    }

    setText(text: string): void {
        this.text = text;
        this.invalidate();
    }

    render(width: number): string[] {
        // 检查缓存
        if (
            this.cachedText === this.text &&
            this.cachedWidth === width &&
            this.cachedLines
        ) {
            return this.cachedLines;
        }

        // 规范化文本
        const normalizedText = this.text
            .replace(/\t/g, "    ")  // 制表符转空格
            .replace(/\r\n/g, "\n")  // Windows 换行
            .replace(/\r/g, "\n");   // 旧 Mac 换行

        // 计算内容宽度（考虑 ANSI 代码）
        const contentWidth = width - stripAnsi(normalizedText).length + normalizedText.length;

        // 换行（保留 ANSI 代码）
        const wrappedLines = wrapTextWithAnsi(normalizedText, contentWidth);

        // 填充到宽度
        const lines = wrappedLines.map((line) =>
            line.padEnd(visibleWidth(line))
        );

        // 缓存结果
        this.cachedText = this.text;
        this.cachedWidth = width;
        this.cachedLines = lines;

        return lines;
    }

    invalidate(): void {
        this.cachedText = undefined;
        this.cachedWidth = undefined;
        this.cachedLines = undefined;
    }
}
```

### 3.3 Editor 编辑器组件

**源文件**：`/packages/tui/src/components/editor.ts`

```typescript
export class Editor implements Component, Focusable {
    private state: EditorState = {
        lines: [""],
        cursorLine: 0,
        cursorCol: 0,
        scrollLine: 0,
        scrollCol: 0,
    };

    private killRing = new KillRing();
    private undoStack = new UndoStack<EditorState>();
    private options: EditorOptions;

    constructor(options?: EditorOptions) {
        this.options = options ?? {};
    }

    focused = false;

    render(width: number): string[] {
        const { height = 20 } = this.options;
        const lines: string[] = [];

        // 计算可见范围
        const visibleLines = Math.min(height, this.state.lines.length);
        const startLine = this.state.scrollLine;
        const endLine = startLine + visibleLines;

        // 渲染可见行
        for (let i = startLine; i < endLine; i++) {
            const line = this.state.lines[i] ?? "";
            const displayLine = this.renderLine(line, width);
            lines.push(displayLine);
        }

        // 渲染状态栏
        if (this.options.statusBar) {
            const statusLine = this.renderStatusBar(width);
            lines.push(statusLine);
        }

        return lines;
    }

    private renderLine(line: string, width: number): string {
        // 计算滚动偏移
        const scrollCol = this.state.scrollCol;
        const maxCol = scrollCol + width;
        const visibleText = line.slice(scrollCol, maxCol);

        // 填充到宽度
        return visibleText.padEnd(width);
    }

    private renderStatusBar(width: number): string {
        const { cursorLine, cursorCol } = this.state;
        const line = cursorLine + 1;
        const col = cursorCol + 1;
        const total = this.state.lines.length;

        const status = `Line ${line}/${total}, Col ${col}`;
        return status.padEnd(width);
    }

    handleInput(data: string): void {
        // 保存撤销状态
        this.undoStack.push({ ...this.state });

        // 处理输入
        if (data === "\r") {
            // 回车：换行
            this.handleEnter();
        } else if (data === "\x7f") {
            // 退格：删除字符
            this.handleBackspace();
        } else if (data.startsWith("\x1b")) {
            // 转义序列：特殊键
            this.handleEscapeSequence(data);
        } else {
            // 普通字符：插入
            this.handleInsert(data);
        }

        this.invalidate();
    }

    private handleEnter(): void {
        const currentLine = this.state.lines[this.state.cursorLine];
        const beforeCursor = currentLine.slice(0, this.state.cursorCol);
        const afterCursor = currentLine.slice(this.state.cursorCol);

        // 分割行
        this.state.lines.splice(this.state.cursorLine, 1, beforeCursor, afterCursor);

        // 移动光标
        this.state.cursorLine++;
        this.state.cursorCol = 0;

        // 调整滚动
        this.adjustScroll();
    }

    private handleBackspace(): void {
        const currentLine = this.state.lines[this.state.cursorLine];

        if (this.state.cursorCol > 0) {
            // 删除当前行字符
            const beforeCursor = currentLine.slice(0, this.state.cursorCol - 1);
            const afterCursor = currentLine.slice(this.state.cursorCol);
            this.state.lines[this.state.cursorLine] = beforeCursor + afterCursor;
            this.state.cursorCol--;
        } else if (this.state.cursorLine > 0) {
            // 合并到上一行
            const prevLine = this.state.lines[this.state.cursorLine - 1];
            const newCol = prevLine.length;
            this.state.lines[this.state.cursorLine - 1] = prevLine + currentLine;
            this.state.lines.splice(this.state.cursorLine, 1);
            this.state.cursorLine--;
            this.state.cursorCol = newCol;
        }

        this.adjustScroll();
    }

    private handleInsert(text: string): void {
        const currentLine = this.state.lines[this.state.cursorLine];
        const beforeCursor = currentLine.slice(0, this.state.cursorCol);
        const afterCursor = currentLine.slice(this.state.cursorCol);

        this.state.lines[this.state.cursorLine] = beforeCursor + text + afterCursor;
        this.state.cursorCol += text.length;

        this.adjustScroll();
    }

    private adjustScroll(): void {
        const { height = 20 } = this.options;
        const width = this.options.width ?? 80;

        // 垂直滚动
        if (this.state.cursorLine < this.state.scrollLine) {
            this.state.scrollLine = this.state.cursorLine;
        } else if (this.state.cursorLine >= this.state.scrollLine + height) {
            this.state.scrollLine = this.state.cursorLine - height + 1;
        }

        // 水平滚动
        if (this.state.cursorCol < this.state.scrollCol) {
            this.state.scrollCol = this.state.cursorCol;
        } else if (this.state.cursorCol >= this.state.scrollCol + width) {
            this.state.scrollCol = this.state.cursorCol - width + 1;
        }
    }

    invalidate(): void {
        // 触发重新渲染
        if (this.options.onInvalidate) {
            this.options.onInvalidate();
        }
    }
}
```

### 3.4 Input 输入组件

**源文件**：`/packages/tui/src/components/input.ts`

```typescript
export class Input implements Component, Focusable {
    private value: string = "";
    private cursor: number = 0;
    private placeholder: string = "";
    private killRing = new KillRing();
    private undoStack = new UndoStack<InputState>();
    private pasteBuffer: string = "";
    private isInPaste: boolean = false;

    focused = false;

    render(width: number): string[] {
        const displayValue = this.value || this.placeholder;
        const visibleLength = visibleWidth(displayValue);

        // 截断到宽度
        const truncated = truncateToWidth(displayValue, width);

        // 渲染光标
        let display = truncated;
        if (this.focused) {
            const beforeCursor = display.slice(0, this.cursor);
            const afterCursor = display.slice(this.cursor);
            display = beforeCursor + "\x1b[7m \x1b[0m" + afterCursor;
        }

        return [display.padEnd(width)];
    }

    handleInput(data: string): void {
        // 括号粘贴模式检测
        if (data === "\x1b[200~") {
            this.isInPaste = true;
            this.pasteBuffer = "";
            return;
        }

        if (this.isInPaste) {
            if (data === "\x1b[201~") {
                // 结束粘贴
                this.isInPaste = false;
                this.insert(this.pasteBuffer);
                this.pasteBuffer = "";
            } else {
                this.pasteBuffer += data;
            }
            return;
        }

        // 保存撤销状态
        this.undoStack.push({ value: this.value, cursor: this.cursor });

        // 处理输入
        if (data === "\r") {
            // 回车：触发提交
            if (this.options.onSubmit) {
                this.options.onSubmit(this.value);
            }
        } else if (data === "\x7f") {
            // 退格
            this.value = this.value.slice(0, this.cursor - 1) + this.value.slice(this.cursor);
            this.cursor = Math.max(0, this.cursor - 1);
        } else if (data.startsWith("\x1b")) {
            // 转义序列
            this.handleEscapeSequence(data);
        } else if (!data.startsWith("\x01")) {
            // 普通字符（Ctrl+A 等已处理）
            this.insert(data);
        }

        this.invalidate();
    }

    private insert(text: string): void {
        const before = this.value.slice(0, this.cursor);
        const after = this.value.slice(this.cursor);
        this.value = before + text + after;
        this.cursor += text.length;
    }

    invalidate(): void {
        if (this.options.onInvalidate) {
            this.options.onInvalidate();
        }
    }
}
```

### 3.5 SelectList 选择列表组件

**源文件**：`/packages/tui/src/components/select-list.ts`

```typescript
export class SelectList implements Component, Focusable {
    private items: SelectItem[] = [];
    private filteredItems: SelectItem[] = [];
    private selectedIndex: number = 0;
    private query: string = "";
    private inputComponent: Input;

    focused = false;

    constructor(items: SelectItem[]) {
        this.items = items;
        this.filteredItems = items;

        // 创建搜索输入框
        this.inputComponent = new Input();
        this.inputComponent.options = {
            onSubmit: (value) => {
                this.selectItem(this.filteredItems[this.selectedIndex]);
            },
        };
    }

    render(width: number): string[] {
        const lines: string[] = [];

        // 渲染输入框
        lines.push(...this.inputComponent.render(width));

        // 计算列表布局
        const { primaryWidth, secondaryWidth } = this.calculateColumnWidths(width);

        // 渲染列表项
        for (let i = 0; i < Math.min(this.filteredItems.length, 10); i++) {
            const item = this.filteredItems[i];
            const isSelected = i === this.selectedIndex;
            const prefix = isSelected ? "> " : "  ";

            const primary = truncateToWidth(item.primary, primaryWidth);
            const secondary = truncateToWidth(item.secondary ?? "", secondaryWidth);

            const line = `${prefix}${primary.padEnd(primaryWidth)}${secondary}`;
            lines.push(isSelected ? `\x1b[7m${line}\x1b[0m` : line);
        }

        return lines;
    }

    handleInput(data: string): void {
        // 转发到输入框
        this.inputComponent.handleInput(data);

        // 更新搜索查询
        this.query = this.inputComponent.value;
        this.filteredItems = fuzzyFilter(this.items, this.query, (item) =>
            `${item.primary} ${item.secondary ?? ""}`.toLowerCase()
        );

        this.invalidate();
    }

    private calculateColumnWidths(width: number): {
        primaryWidth: number;
        secondaryWidth: number;
    } {
        // 计算主列和次列的最佳宽度
        const maxPrimaryWidth = Math.max(
            ...this.filteredItems.map((item) => visibleWidth(item.primary))
        );
        const maxSecondaryWidth = Math.max(
            ...this.filteredItems.map((item) => visibleWidth(item.secondary ?? ""))
        );

        const availableWidth = width - 4; // 减去前缀和间距
        const primaryWidth = Math.min(maxPrimaryWidth, availableWidth * 0.6);
        const secondaryWidth = availableWidth - primaryWidth;

        return { primaryWidth, secondaryWidth };
    }
}
```

---

## 4. 快捷键系统

### 4.1 快捷键定义

**源文件**：`/packages/tui/src/keybindings.ts`

```typescript
export interface KeybindingDefinition {
    defaultKeys: string | string[];
    description: string;
    conflictResolution?: "override" | "ignore" | "warn";
}

export const TUI_KEYBINDINGS: Record<string, KeybindingDefinition> = {
    // 光标移动
    "tui.editor.cursorUp": {
        defaultKeys: "up",
        description: "Move cursor up",
    },
    "tui.editor.cursorDown": {
        defaultKeys: "down",
        description: "Move cursor down",
    },
    "tui.editor.cursorLeft": {
        defaultKeys: "left",
        description: "Move cursor left",
    },
    "tui.editor.cursorRight": {
        defaultKeys: "right",
        description: "Move cursor right",
    },

    // 编辑操作
    "tui.editor.killLine": {
        defaultKeys: "Ctrl+K",
        description: "Kill to end of line",
    },
    "tui.editor.yank": {
        defaultKeys: "Ctrl+Y",
        description: "Yank (paste)",
    },
    "tui.editor.undo": {
        defaultKeys: ["Ctrl+_", "Ctrl+/"],
        description: "Undo",
    },

    // TUI 操作
    "tui.abort": {
        defaultKeys: "Ctrl+C",
        description: "Abort operation",
    },
    "tui.submit": {
        defaultKeys: "enter",
        description: "Submit input",
    },
};
```

### 4.2 快捷键管理器

```typescript
export class KeybindingManager {
    private bindings: Map<string, string[]> = new Map();
    private reverseBindings: Map<string, string> = new Map();

    constructor() {
        this.loadDefaults();
    }

    private loadDefaults(): void {
        for (const [action, definition] of Object.entries(TUI_KEYBINDINGS)) {
            const keys = Array.isArray(definition.defaultKeys)
                ? definition.defaultKeys
                : [definition.defaultKeys];

            this.bindings.set(action, keys);

            for (const key of keys) {
                this.reverseBindings.set(key, action);
            }
        }
    }

    // 设置快捷键
    setKeybinding(action: string, keys: string | string[]): void {
        const keyArray = Array.isArray(keys) ? keys : [keys];

        // 移除旧绑定
        const oldKeys = this.bindings.get(action) ?? [];
        for (const key of oldKeys) {
            this.reverseBindings.delete(key);
        }

        // 添加新绑定
        this.bindings.set(action, keyArray);
        for (const key of keyArray) {
            this.reverseBindings.set(key, action);
        }
    }

    // 获取动作
    getAction(key: string): string | undefined {
        return this.reverseBindings.get(key);
    }

    // 获取快捷键
    getKeys(action: string): string[] {
        return this.bindings.get(action) ?? [];
    }

    // 解析按键序列
    parseKey(data: string): string {
        if (data === "\r") return "enter";
        if (data === "\x7f") return "backspace";
        if (data === "\x1b") return "escape";
        if (data === "\x01") return "Ctrl+A";
        if (data === "\x03") return "Ctrl+C";
        if (data === "\x0b") return "Ctrl+K";
        if (data === "\x19") return "Ctrl+Y";

        // 转义序列
        if (data.startsWith("\x1b[")) {
            const seq = data.slice(2);
            if (seq === "A") return "up";
            if (seq === "B") return "down";
            if (seq === "C") return "right";
            if (seq === "D") return "left";
        }

        return data;
    }
}
```

---

## 5. 终端图像支持

### 5.1 Kitty 图像协议

**源文件**：`/packages/tui/src/terminal-image.ts`

```typescript
export interface ImageDimensions {
    width: number;
    height: number;
    rows: number;
    cols: number;
}

export function encodeKitty(
    base64Data: string,
    dimensions: ImageDimensions,
    options?: { imageId?: number; placement?: "inline" | "above" }
): string {
    const { width, height } = dimensions;
    const imageId = options?.imageId ?? 1;
    const placement = options?.placement ?? "inline";

    // 构建 Kitty 协议命令
    const cmd = [
        "\x1b_G",  // 开始
        `a=T`,     // 传输格式：base64
        `f=100`,   // 图像格式：PNG/JPEG
        `i=${imageId}`,  // 图像 ID
        placement === "inline" ? "" : `p=1`,  // 放置模式
        `;${base64Data}`,  // 数据
        "\x1b\\",  // 结束
    ].join("");

    return cmd;
}

export function renderImage(
    base64Data: string,
    dimensions: ImageDimensions,
    options?: { maxWidthCells?: number; imageId?: number }
): string[] {
    const { cols, rows } = dimensions;
    const maxWidth = options?.maxWidthCells ?? cols;

    // 编码图像
    const imageCode = encodeKitty(base64Data, dimensions, {
        imageId: options?.imageId,
    });

    // 创建占位行
    const lines: string[] = [];

    for (let i = 0; i < rows; i++) {
        if (i === 0) {
            // 第一行包含图像数据
            lines.push(imageCode + "\x1b[${rows}D");  // 下移 rows 行
        } else {
            // 后续行为空
            lines.push("");
        }
    }

    return lines;
}
```

### 5.2 终端能力检测

```typescript
export interface TerminalCapabilities {
    images: boolean;
    hyperlinks: boolean;
    sixel: boolean;
}

export async function detectCapabilities(): Promise<TerminalCapabilities> {
    const term = process.env.TERM ?? "";
    const termProgram = process.env.TERM_PROGRAM ?? "";

    return {
        images:
            term.includes("kitty") ||
            termProgram === "iTerm.app" ||
            process.env.KITTY_WINDOW_ID !== undefined,

        hyperlinks:
            term.includes("kitty") ||
            termProgram === "iTerm.app" ||
            term.includes("xterm"),

        sixel: term.includes("sixel") || term.includes("mlterm"),
    };
}
```

---

## 6. 文本处理工具

### 6.1 可见宽度计算

```typescript
export function visibleWidth(str: string): number {
    let width = 0;

    for (const char of str) {
        // 跳过 ANSI 代码
        if (char === "\x1b") {
            const end = str.indexOf("m", str.indexOf(char));
            if (end !== -1) {
                continue;
            }
        }

        // 计算字符宽度
        const code = char.codePointAt(0) ?? 0;

        if (code < 32 || (code >= 0x7f && code < 0xa0)) {
            // 控制字符：0 宽度
            continue;
        } else if (code >= 0x1100 && isWideCharacter(code)) {
            // 东亚宽字符：2 宽度
            width += 2;
        } else if (code >= 0x1f300 && code <= 0x1f9ff) {
            // Emoji：2 宽度
            width += 2;
        } else {
            // 其他字符：1 宽度
            width += 1;
        }
    }

    return width;
}

function isWideCharacter(code: number): boolean {
    // 东亚宽字符范围
    return (
        (code >= 0x1100 && code <= 0x115f) ||
        (code >= 0x2e80 && code <= 0xa4cf) ||
        (code >= 0xac00 && code <= 0xd7a3) ||
        (code >= 0xf900 && code <= 0xfaff) ||
        (code >= 0xfe10 && code <= 0xfe19) ||
        (code >= 0xfe30 && code <= 0xfe6f) ||
        (code >= 0xff00 && code <= 0xff60) ||
        (code >= 0xffe0 && code <= 0xffe6) ||
        (code >= 0x20000 && code <= 0x2fffd) ||
        (code >= 0x30000 && code <= 0x3fffd)
    );
}
```

### 6.2 保留 ANSI 的换行

```typescript
export function wrapTextWithAnsi(text: string, width: number): string[] {
    const lines: string[] = [];
    let currentLine = "";
    let currentWidth = 0;
    let inAnsi = false;

    for (let i = 0; i < text.length; i++) {
        const char = text[i];

        // 检测 ANSI 转义序列
        if (char === "\x1b") {
            const end = text.indexOf("m", i);
            if (end !== -1) {
                const ansi = text.slice(i, end + 1);
                currentLine += ansi;
                i = end;
                continue;
            }
        }

        // 处理换行符
        if (char === "\n") {
            lines.push(currentLine);
            currentLine = "";
            currentWidth = 0;
            continue;
        }

        // 计算字符宽度
        const charWidth = visibleWidth(char);

        // 检查是否需要换行
        if (currentWidth + charWidth > width) {
            lines.push(currentLine);
            currentLine = char;
            currentWidth = charWidth;
        } else {
            currentLine += char;
            currentWidth += charWidth;
        }
    }

    if (currentLine) {
        lines.push(currentLine);
    }

    return lines;
}
```

---

## 7. 使用示例

### 7.1 基础使用

```typescript
import { TUI, Box, Text } from "@mariozechner/pi-tui";

// 创建 TUI
const tui = new TUI();

// 创建组件
const box = new Box({ paddingX: 2, paddingY: 1 });
const text = new Text("Hello, World!");

box.add(text);
tui.add(box, { width: "100%" });

// 启动
await tui.start();
```

### 7.2 输入框

```typescript
import { TUI, Input, Box } from "@mariozechner/pi-tui";

const tui = new TUI();

const input = new Input();
input.options = {
    placeholder: "Enter your message...",
    onSubmit: (value) => {
        console.log("Submitted:", value);
        tui.stop();
    },
};

const box = new Box();
box.add(input);
tui.add(box, { height: 3 });

await tui.start();
```

### 7.3 选择列表

```typescript
import { TUI, SelectList, Box } from "@mariozechner/pi-tui";

const tui = new TUI();

const list = new SelectList([
    { primary: "Option 1", secondary: "Description 1" },
    { primary: "Option 2", secondary: "Description 2" },
    { primary: "Option 3", secondary: "Description 3" },
]);

list.options = {
    onSelect: (item) => {
        console.log("Selected:", item);
        tui.stop();
    },
};

const box = new Box();
box.add(list);
tui.add(box, { height: 12 });

await tui.start();
```

---

## 8. 总结

pi-tui 包是一个功能完整的终端 UI 库：

**核心特性**：
1. **差分渲染**：仅更新变化区域
2. **组件系统**：可组合的 UI 组件
3. **快捷键管理**：灵活的键盘绑定
4. **终端兼容**：支持多种终端协议
5. **文本处理**：完整的换行和 ANSI 支持

**设计优势**：
- **零依赖**：无外部依赖
- **高性能**：差分渲染和智能缓存
- **可扩展**：插件化架构
- **类型安全**：完整的 TypeScript 类型

**适用场景**：
- CLI 工具
- 终端应用
- 交互式脚本
- 开发工具

---

**相关文档**：
- [架构概览](../02-architecture/01-architecture-overview.md)
- [核心数据流](../02-architecture/03-data-flow.md)
- [pi-coding-agent 包分析](./03-pi-coding-agent.md)
