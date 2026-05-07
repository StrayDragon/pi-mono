# Keybinding System

## 概述

pi-mono 的快捷键系统是一个类型安全、高度可配置的终端 UI 快捷键管理框架，支持：
- **类型安全的键标识符**：编译时验证和自动补全
- **多修饰键组合**：支持 ctrl、shift、alt、super 及其组合
- **Kitty 键盘协议**：现代终端的增强键盘支持
- **向后兼容**：传统终端转义序列支持
- **冲突检测**：自动检测并报告快捷键冲突
- **用户配置**：通过 JSON 文件自定义快捷键
- **配置迁移**：自动迁移旧版快捷键配置

**核心文件**：
- `/packages/tui/src/keybindings.ts` (245 行) - TUI 快捷键管理器
- `/packages/tui/src/keys.ts` (600+ 行) - 键解析和类型定义
- `/packages/coding-agent/src/core/keybindings.ts` (371 行) - 应用快捷键定义

## 键标识符系统

### 类型安全的 KeyId

```typescript
type Letter = "a" | "b" | "c" | ... | "z";

type Digit = "0" | "1" | "2" | ... | "9";

type SymbolKey =
  | "`" | "-" | "=" | "[" | "]" | "\\" | ";" | "'"
  | "," | "." | "/" | "!" | "@" | "#" | "$" | "%"
  | "^" | "&" | "*" | "(" | ")" | "_" | "+" | "|"
  | "~" | "{" | "}" | ":" | "<" | ">" | "?";

type SpecialKey =
  | "escape" | "esc" | "enter" | "return" | "tab" | "space"
  | "backspace" | "delete" | "insert" | "clear" | "home" | "end"
  | "pageUp" | "pageDown" | "up" | "down" | "left" | "right"
  | "f1" | "f2" | ... | "f12";

type ModifierName = "ctrl" | "shift" | "alt" | "super";

type BaseKey = Letter | Digit | SymbolKey | SpecialKey;

// 递归类型生成所有修饰键组合
type ModifiedKeyId<Key extends string> = {
  [M in ModifierName]:
    `${M}+${Key}` |
    `${M}+${ModifiedKeyId<Key, Exclude<ModifierName, M>>}`;
}[ModifierName];

export type KeyId = BaseKey | ModifiedKeyId<BaseKey>;
```

**示例**：
```typescript
"ctrl+c"           // 有效
"ctrl+shift+p"     // 有效
"ctrl+alt+delete"  // 有效
"ctrl+shift+alt+super+x"  // 有效
"ctrl+z+shift"     // 无效（编译错误）
"unknow+key"       // 无效（编译错误）
```

### Key 辅助对象

提供类型安全的键标识符创建：

```typescript
export const Key = {
  // 特殊键
  escape: "escape" as const,
  enter: "enter" as const,
  tab: "tab" as const,
  // ... 更多特殊键

  // 符号键
  backtick: "`" as const,
  comma: "," as const,
  period: "." as const,
  // ... 更多符号键

  // 单修饰键
  ctrl: <K extends BaseKey>(key: K): `ctrl+${K}` => `ctrl+${key}`,
  shift: <K extends BaseKey>(key: K): `shift+${K}` => `shift+${key}`,
  alt: <K extends BaseKey>(key: K): `alt+${K}` => `alt+${key}`,
  super: <K extends BaseKey>(key: K): `super+${K}` => `super+${key}`,

  // 组合修饰键
  ctrlShift: <K extends BaseKey>(key: K): `ctrl+shift+${K}` => `ctrl+shift+${key}`,
  ctrlAlt: <K extends BaseKey>(key: K): `ctrl+alt+${K}` => `ctrl+alt+${key}`,
  shiftAlt: <K extends BaseKey>(key: K): `shift+alt+${K}` => `shift+alt+${key}`,
  // ... 更多组合
} as const;
```

**使用示例**：
```typescript
Key.ctrl("c")           // "ctrl+c"
Key.ctrlShift("p")      // "ctrl+shift+p"
Key.alt(Key.enter)      // "alt+enter"
```

## 快捷键定义

### 声明合并扩展

通过 TypeScript 模块声明合并扩展快捷键接口：

```typescript
// pi-tui 基础定义
export interface Keybindings {
  "tui.editor.cursorUp": true;
  "tui.editor.cursorDown": true;
  // ... 更多 TUI 快捷键
}

// pi-coding-agent 扩展
declare module "@mariozechner/pi-tui" {
  interface Keybindings extends AppKeybindings {}
}

export interface AppKeybindings {
  "app.interrupt": true;
  "app.clear": true;
  // ... 更多应用快捷键
}
```

### 快捷键定义结构

```typescript
export interface KeybindingDefinition {
  defaultKeys: KeyId | KeyId[];  // 默认键绑定
  description?: string;          // 人类可读描述
}

export type KeybindingDefinitions = Record<string, KeybindingDefinition>;

export const KEYBINDINGS = {
  "app.interrupt": {
    defaultKeys: "escape",
    description: "Cancel or abort"
  },
  "app.clear": {
    defaultKeys: "ctrl+c",
    description: "Clear editor"
  },
  "app.model.cycleForward": {
    defaultKeys: ["ctrl+p", "ctrl+shift+p"],  // 多个快捷键
    description: "Cycle to next model"
  },
} as const satisfies KeybindingDefinitions;
```

### TUI 默认快捷键

**编辑器导航**：
```typescript
"tui.editor.cursorUp": { defaultKeys: "up" },
"tui.editor.cursorDown": { defaultKeys: "down" },
"tui.editor.cursorLeft": { defaultKeys: ["left", "ctrl+b"] },
"tui.editor.cursorRight": { defaultKeys: ["right", "ctrl+f"] },
"tui.editor.cursorWordLeft": { defaultKeys: ["alt+left", "ctrl+left", "alt+b"] },
"tui.editor.cursorWordRight": { defaultKeys: ["alt+right", "ctrl+right", "alt+f"] },
"tui.editor.cursorLineStart": { defaultKeys: ["home", "ctrl+a"] },
"tui.editor.cursorLineEnd": { defaultKeys: ["end", "ctrl+e"] },
```

**编辑器操作**：
```typescript
"tui.editor.deleteCharBackward": { defaultKeys: "backspace" },
"tui.editor.deleteCharForward": { defaultKeys: ["delete", "ctrl+d"] },
"tui.editor.deleteWordBackward": { defaultKeys: ["ctrl+w", "alt+backspace"] },
"tui.editor.deleteWordForward": { defaultKeys: ["alt+d", "alt+delete"] },
"tui.editor.yank": { defaultKeys: "ctrl+y" },
"tui.editor.undo": { defaultKeys: "ctrl+-" },
```

**通用选择**：
```typescript
"tui.select.up": { defaultKeys: "up" },
"tui.select.down": { defaultKeys: "down" },
"tui.select.confirm": { defaultKeys: "enter" },
"tui.select.cancel": { defaultKeys: ["escape", "ctrl+c"] },
```

### 应用快捷键

**会话管理**：
```typescript
"app.session.new": { defaultKeys: [] },
"app.session.tree": { defaultKeys: [] },
"app.session.fork": { defaultKeys: [] },
"app.session.resume": { defaultKeys: [] },
"app.session.rename": { defaultKeys: "ctrl+r" },
"app.session.delete": { defaultKeys: "ctrl+d" },
```

**模型管理**：
```typescript
"app.model.cycleForward": { defaultKeys: "ctrl+p" },
"app.model.cycleBackward": { defaultKeys: "shift+ctrl+p" },
"app.model.select": { defaultKeys: "ctrl+l" },
"app.models.save": { defaultKeys: "ctrl+s" },
"app.models.enableAll": { defaultKeys: "ctrl+a" },
```

**树导航**：
```typescript
"app.tree.foldOrUp": { defaultKeys: ["ctrl+left", "alt+left"] },
"app.tree.unfoldOrDown": { defaultKeys: ["ctrl+right", "alt+right"] },
"app.tree.editLabel": { defaultKeys: "shift+l" },
"app.tree.filter.cycleForward": { defaultKeys: "ctrl+o" },
```

## 快捷键管理器

### KeybindingsManager 类

```typescript
export class KeybindingsManager {
  private definitions: KeybindingDefinitions;
  private userBindings: KeybindingsConfig;
  private keysById = new Map<Keybinding, KeyId[]>();
  private conflicts: KeybindingConflict[] = [];

  constructor(
    definitions: KeybindingDefinitions,
    userBindings: KeybindingsConfig = {}
  ) {
    this.definitions = definitions;
    this.userBindings = userBindings;
    this.rebuild();
  }

  // 检查输入是否匹配快捷键
  matches(data: string, keybinding: Keybinding): boolean {
    const keys = this.keysById.get(keybinding) ?? [];
    for (const key of keys) {
      if (matchesKey(data, key)) return true;
    }
    return false;
  }

  // 获取快捷键的键绑定
  getKeys(keybinding: Keybinding): KeyId[] {
    return [...(this.keysById.get(keybinding) ?? [])];
  }

  // 获取快捷键定义
  getDefinition(keybinding: Keybinding): KeybindingDefinition {
    return this.definitions[keybinding];
  }

  // 获取所有冲突
  getConflicts(): KeybindingConflict[] {
    return this.conflicts.map(conflict => ({
      ...conflict,
      keybindings: [...conflict.keybindings]
    }));
  }

  // 更新用户配置
  setUserBindings(userBindings: KeybindingsConfig): void {
    this.userBindings = userBindings;
    this.rebuild();
  }

  // 获取解析后的绑定（默认 + 用户）
  getResolvedBindings(): KeybindingsConfig {
    const resolved: KeybindingsConfig = {};
    for (const id of Object.keys(this.definitions)) {
      const keys = this.keysById.get(id as Keybinding) ?? [];
      resolved[id] = keys.length === 1 ? keys[0]! : [...keys];
    }
    return resolved;
  }
}
```

### 构建和冲突检测

```typescript
private rebuild(): void {
  this.keysById.clear();
  this.conflicts = [];

  // 收集用户键声明
  const userClaims = new Map<KeyId, Set<Keybinding>>();
  for (const [keybinding, keys] of Object.entries(this.userBindings)) {
    if (!(keybinding in this.definitions)) continue;
    for (const key of normalizeKeys(keys)) {
      const claimants = userClaims.get(key) ?? new Set<Keybinding>();
      claimants.add(keybinding as Keybinding);
      userClaims.set(key, claimants);
    }
  }

  // 检测冲突（一个键被多个操作声明）
  for (const [key, keybindings] of userClaims) {
    if (keybindings.size > 1) {
      this.conflicts.push({
        key,
        keybindings: [...keybindings]
      });
    }
  }

  // 构建键映射
  for (const [id, definition] of Object.entries(this.definitions)) {
    const userKeys = this.userBindings[id];
    const keys = userKeys === undefined
      ? normalizeKeys(definition.defaultKeys)
      : normalizeKeys(userKeys);
    this.keysById.set(id as Keybinding, keys);
  }
}
```

**冲突示例**：
```json
// keybindings.json
{
  "app.clear": "ctrl+c",
  "tui.select.cancel": "ctrl+c"  // 冲突！
}
```

```typescript
// 冲突报告
[
  {
    key: "ctrl+c",
    keybindings: ["app.clear", "tui.select.cancel"]
  }
]
```

## 用户配置

### 配置文件

用户快捷键配置存储在 `~/.pi/agent/keybindings.json`：

```json
{
  "app.clear": "ctrl+c",
  "app.model.cycleForward": ["ctrl+p", "ctrl+shift+p"],
  "tui.editor.cursorLeft": "ctrl+b"
}
```

### 配置加载

```typescript
export class KeybindingsManager extends TuiKeybindingsManager {
  private configPath: string | undefined;

  static create(agentDir: string = getAgentDir()): KeybindingsManager {
    const configPath = join(agentDir, "keybindings.json");
    const userBindings = KeybindingsManager.loadFromFile(configPath);
    return new KeybindingsManager(userBindings, configPath);
  }

  private static loadFromFile(path: string): KeybindingsConfig {
    const rawConfig = loadRawConfig(path);
    if (!rawConfig) return {};
    return toKeybindingsConfig(
      migrateKeybindingsConfig(rawConfig).config
    );
  }

  reload(): void {
    if (!this.configPath) return;
    this.setUserBindings(
      KeybindingsManager.loadFromFile(this.configPath)
    );
  }
}
```

### 配置解析

```typescript
function toKeybindingsConfig(value: unknown): KeybindingsConfig {
  if (!isRecord(value)) return {};

  const config: KeybindingsConfig = {};
  for (const [key, binding] of Object.entries(value)) {
    if (typeof binding === "string") {
      config[key] = binding as KeyId;
    } else if (Array.isArray(binding) &&
               binding.every(entry => typeof entry === "string")) {
      config[key] = binding as KeyId[];
    }
  }
  return config;
}
```

## 配置迁移

### 旧版名称映射

系统自动迁移旧版快捷键名称到新命名空间：

```typescript
const KEYBINDING_NAME_MIGRATIONS = {
  // 旧名称 → 新名称
  cursorUp: "tui.editor.cursorUp",
  cursorDown: "tui.editor.cursorDown",
  cursorLeft: "tui.editor.cursorLeft",
  cursorRight: "tui.editor.cursorRight",
  deleteCharBackward: "tui.editor.deleteCharBackward",
  deleteCharForward: "tui.editor.deleteCharForward",
  newLine: "tui.input.newLine",
  submit: "tui.input.submit",
  selectUp: "tui.select.up",
  selectDown: "tui.select.down",
  interrupt: "app.interrupt",
  clear: "app.clear",
  exit: "app.exit",
  cycleModelForward: "app.model.cycleForward",
  cycleThinkingLevel: "app.thinking.cycle",
  // ... 更多映射
} as const;
```

### 迁移逻辑

```typescript
function migrateKeybindingsConfig(rawConfig: Record<string, unknown>): {
  config: Record<string, unknown>;
  migrated: boolean;
} {
  const config: Record<string, unknown> = {};
  let migrated = false;

  for (const [key, value] of Object.entries(rawConfig)) {
    // 检查是否是旧名称
    const nextKey = isLegacyKeybindingName(key)
      ? KEYBINDING_NAME_MIGRATIONS[key]
      : key;

    if (nextKey !== key) {
      migrated = true;
    }

    // 如果新键已存在，跳过（用户已手动配置）
    if (key !== nextKey && Object.hasOwn(rawConfig, nextKey)) {
      migrated = true;
      continue;
    }

    config[nextKey] = value;
  }

  return {
    config: orderKeybindingsConfig(config),
    migrated
  };
}
```

**迁移示例**：
```json
// 旧配置
{
  "cursorUp": "k",
  "clear": "ctrl+c"
}

// 迁移后
{
  "tui.editor.cursorUp": "k",
  "app.clear": "ctrl+c"
}
```

## 键解析

### Kitty 键盘协议

支持现代终端的增强键盘协议：

```typescript
let _kittyProtocolActive = false;

export function setKittyProtocolActive(active: boolean): void {
  _kittyProtocolActive = active;
}

export function isKittyProtocolActive(): boolean {
  return _kittyProtocolActive;
}
```

**Kitty 协议特性**：
- 精确的修饰键状态（shift、alt、ctrl、super）
- 区分大小写字母
- 支持所有符号键
- 可靠的小键盘处理

### 修饰键掩码

```typescript
const MODIFIERS = {
  shift: 1,
  alt: 2,
  ctrl: 4,
  super: 8,
} as const;

const LOCK_MASK = 64 + 128;  // Caps Lock + Num Lock
```

### 键匹配

```typescript
export function matchesKey(data: string, keyId: KeyId): boolean {
  const parsed = parseKey(data);
  if (!parsed) return false;

  const { key, modifiers } = parseKeyId(keyId);

  // 检查修饰键匹配
  if (modifiers.ctrl !== (parsed.modifiers & 4)) return false;
  if (modifiers.shift !== (parsed.modifiers & 1)) return false;
  if (modifiers.alt !== (parsed.modifiers & 2)) return false;
  if (modifiers.super !== (parsed.modifiers & 8)) return false;

  // 检查键匹配
  return parsed.key === key;
}
```

### 键解析流程

```typescript
export function parseKey(data: string): ParsedKey | undefined {
  // 1. 尝试 Kitty 协议
  if (_kittyProtocolActive) {
    const kitty = parseKittyEscapeSequence(data);
    if (kitty) return kitty;
  }

  // 2. 尝试传统转义序列
  const legacy = parseLegacyEscapeSequence(data);
  if (legacy) return legacy;

  // 3. 尝试单个字符
  if (data.length === 1) {
    return parseSingleCharacter(data);
  }

  return undefined;
}
```

**传统转义序列示例**：
```
输入        → 解析结果
"\x1b[A"    → { key: "up", modifiers: 0 }
"\x1b[1;3A" → { key: "up", modifiers: 2 } (Alt+Up)
"\x1b[B"    → { key: "down", modifiers: 0 }
"a"         → { key: "a", modifiers: 0 }
"\x01"      → { key: "a", modifiers: 4 } (Ctrl+A)
```

**Kitty 协议示例**：
```
输入                      → 解析结果
"\x1b[96;5u"            → { key: "f12", modifiers: 4 } (Ctrl+F12)
"\x1b[97;3u"            → { key: "a", modifiers: 2 } (Alt+A)
"\x1b[48;5u"            → { key: "0", modifiers: 4 } (Ctrl+0)
```

## 平台差异

### Windows 特殊处理

某些快捷键在 Windows 上有不同的默认值：

```typescript
"app.suspend": {
  defaultKeys: process.platform === "win32" ? [] : "ctrl+z",
  description: "Suspend to background"
},
"app.clipboard.pasteImage": {
  defaultKeys: process.platform === "win32" ? "alt+v" : "ctrl+v",
  description: "Paste image from clipboard"
}
```

### Ctrl+符号键冲突

某些 Ctrl+符号键组合与 ASCII 控制码重叠：

```
Ctrl+[  → ESC (ASCII 27)
Ctrl+J  → Line Feed (ASCII 10)
Ctrl+M  → Carriage Return (ASCII 13)
Ctrl+I  → Tab (ASCII 9)
```

**解决方案**：
- 使用 Ctrl+Shift 组合避免冲突
- 例如：`Ctrl+Shift+]` 而不是 `Ctrl+]`

## 最佳实践

### 快捷键设计原则

1. **遵循惯例**：
   - `Ctrl+C`：复制/取消
   - `Ctrl+V`：粘贴
   - `Ctrl+Z`：撤销
   - `Ctrl+S`：保存
   - `Ctrl+F`：查找

2. **避免冲突**：
   - 不要使用系统级快捷键（如 `Alt+Tab`）
   - 避免与应用常用快捷键冲突

3. **提供备选**：
   ```typescript
   "tui.editor.cursorLeft": {
     defaultKeys: ["left", "ctrl+b"],  // 箭头键 + Emacs 风格
   }
   ```

4. **考虑平台差异**：
   ```typescript
   "app.suspend": {
     defaultKeys: process.platform === "win32" ? [] : "ctrl+z"
   }
   ```

### 自定义快捷键

1. **创建配置文件**：
```bash
# ~/.pi/agent/keybindings.json
{
  "app.model.cycleForward": "ctrl+m",
  "app.clear": "ctrl+k"
}
```

2. **验证配置**：
   - 重启应用或重新加载配置
   - 检查冲突报告

3. **测试快捷键**：
   - 在实际使用场景中测试
   - 确保没有意外覆盖重要快捷键

### 调试快捷键

```typescript
import { getKeybindings, parseKey } from "@mariozechner/pi-tui";

// 获取快捷键管理器
const kb = getKeybindings();

// 获取操作的键绑定
console.log(kb.getKeys("app.clear"));  // ["ctrl+c"]

// 检查输入是否匹配
console.log(kb.matches("\x03", "app.clear"));  // true (Ctrl+C)

// 解析键输入
const parsed = parseKey("\x1b[A");
console.log(parsed);  // { key: "up", modifiers: 0 }

// 检查冲突
console.log(kb.getConflicts());  // []
```

## 生命周期图

[MermaidChart:./_LEARN/docs/mmd/keybinding-system-lifecycle.mmd]

## 参考资源

- **TUI 快捷键**：`/packages/tui/src/keybindings.ts:1`
- **键解析**：`/packages/tui/src/keys.ts:1`
- **应用快捷键**：`/packages/coding-agent/src/core/keybindings.ts:1`
- **快捷键文档**：`/packages/coding-agent/docs/keybindings.md`
- **迁移测试**：`/packages/coding-agent/test/keybindings-migration.test.ts:1`

## 扩展阅读

- [Kitty Keyboard Protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)
- [Terminal Escape Sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)
- [Emacs Key Bindings](https://www.gnu.org/software/emacs/manual/html_node/emacs/Key-Bindings.html)
