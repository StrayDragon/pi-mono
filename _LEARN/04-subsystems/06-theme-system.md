# Theme System

## 概述

pi-mono 的主题系统是一个强大而灵活的终端 UI 主题框架，支持：
- **JSON 主题定义**：通过 TypeBox Schema 验证的标准化格式
- **颜色值类型**：支持十六进制、变量引用和 256 色索引
- **自动颜色模式检测**：根据终端能力自动选择 truecolor 或 256color 模式
- **变量系统**：支持颜色变量引用和循环检测
- **热重载**：自定义主题文件修改时自动重新加载
- **HTML 导出**：支持将终端颜色转换为 CSS 用于 HTML 导出
- **语法高亮**：集成 cli-highlight 提供代码语法着色

**核心文件**：
- `/packages/coding-agent/src/modes/interactive/theme/theme.ts` (1142 行) - 主题系统核心
- `/packages/coding-agent/src/modes/interactive/theme/dark.json` - 深色主题
- `/packages/coding-agent/src/modes/interactive/theme/light.json` - 浅色主题

## 主题格式

### JSON Schema

主题文件遵循严格的 TypeBox Schema 验证：

```typescript
const ThemeJsonSchema = Type.Object({
  $schema: Type.Optional(Type.String()),  // JSON Schema URL
  name: Type.String(),                     // 主题名称
  vars: Type.Optional(Type.Record(Type.String(), ColorValueSchema)),
  colors: Type.Object({
    // 核心颜色 (10 个)
    accent: ColorValueSchema,
    border: ColorValueSchema,
    borderAccent: ColorValueSchema,
    borderMuted: ColorValueSchema,
    success: ColorValueSchema,
    error: ColorValueSchema,
    warning: ColorValueSchema,
    muted: ColorValueSchema,
    dim: ColorValueSchema,
    text: ColorValueSchema,
    thinkingText: ColorValueSchema,

    // 背景色 (7 个)
    selectedBg: ColorValueSchema,
    userMessageBg: ColorValueSchema,
    userMessageText: ColorValueSchema,
    customMessageBg: ColorValueSchema,
    customMessageText: ColorValueSchema,
    customMessageLabel: ColorValueSchema,
    toolPendingBg: ColorValueSchema,
    toolSuccessBg: ColorValueSchema,
    toolErrorBg: ColorValueSchema,
    toolTitle: ColorValueSchema,
    toolOutput: ColorValueSchema,

    // Markdown 颜色 (10 个)
    mdHeading: ColorValueSchema,
    mdLink: ColorValueSchema,
    mdLinkUrl: ColorValueSchema,
    mdCode: ColorValueSchema,
    mdCodeBlock: ColorValueSchema,
    mdCodeBlockBorder: ColorValueSchema,
    mdQuote: ColorValueSchema,
    mdQuoteBorder: ColorValueSchema,
    mdHr: ColorValueSchema,
    mdListBullet: ColorValueSchema,

    // 差异颜色 (3 个)
    toolDiffAdded: ColorValueSchema,
    toolDiffRemoved: ColorValueSchema,
    toolDiffContext: ColorValueSchema,

    // 语法高亮 (9 个)
    syntaxComment: ColorValueSchema,
    syntaxKeyword: ColorValueSchema,
    syntaxFunction: ColorValueSchema,
    syntaxVariable: ColorValueSchema,
    syntaxString: ColorValueSchema,
    syntaxNumber: ColorValueSchema,
    syntaxType: ColorValueSchema,
    syntaxOperator: ColorValueSchema,
    syntaxPunctuation: ColorValueSchema,

    // 思考级别边框 (6 个)
    thinkingOff: ColorValueSchema,
    thinkingMinimal: ColorValueSchema,
    thinkingLow: ColorValueSchema,
    thinkingMedium: ColorValueSchema,
    thinkingHigh: ColorValueSchema,
    thinkingXhigh: ColorValueSchema,

    // Bash 模式 (1 个)
    bashMode: ColorValueSchema,
  }),
  export: Type.Optional(Type.Object({
    pageBg: Type.Optional(ColorValueSchema),
    cardBg: Type.Optional(ColorValueSchema),
    infoBg: Type.Optional(ColorValueSchema),
  })),
});
```

### 颜色值类型

```typescript
type ColorValue =
  | string   // 十六进制 "#ff0000" 或变量引用 "primary" 或空字符串 ""
  | number;  // 256 色索引 0-255
```

**示例**：
```json
{
  "vars": {
    "cyan": "#00d7ff",
    "blue": "#5f87ff",
    "accent": "#8abeb7"
  },
  "colors": {
    "accent": "accent",           // 变量引用
    "border": "blue",             // 变量引用
    "mdHeading": "#f0c674",       // 十六进制
    "syntaxKeyword": 42           // 256 色索引
  }
}
```

## 主题发现与加载

### 主题来源

主题从三个来源加载：

1. **内置主题** (`getBuiltinThemes()`)
   - 位置：`getThemesDir()` - `packages/coding-agent/src/modes/interactive/theme/`
   - 包括：`dark.json`, `light.json`
   - 缓存在 `BUILTIN_THEMES` 全局变量中

2. **自定义主题** (Custom Themes)
   - 位置：`getCustomThemesDir()` - `~/.pi/agent/themes/`
   - 扫描所有 `.json` 文件

3. **注册主题** (Registered Themes)
   - 通过 `setRegisteredThemes()` 注册的内存主题实例
   - 用于扩展或程序化创建主题

```typescript
// 获取所有可用主题
export function getAvailableThemes(): string[] {
  const themes = new Set<string>(Object.keys(getBuiltinThemes()));
  const customThemesDir = getCustomThemesDir();
  if (fs.existsSync(customThemesDir)) {
    const files = fs.readdirSync(customThemesDir);
    for (const file of files) {
      if (file.endsWith(".json")) {
        themes.add(file.slice(0, -5));
      }
    }
  }
  for (const name of registeredThemes.keys()) {
    themes.add(name);
  }
  return Array.from(themes).sort();
}
```

### 验证流程

```typescript
function parseThemeJson(label: string, json: unknown): ThemeJson {
  if (!validateThemeJson.Check(json)) {
    const errors = Array.from(validateThemeJson.Errors(json));
    const missingColors = new Set<string>();
    const otherErrors: string[] = [];

    for (const error of errors) {
      if (error.keyword === "required" && error.instancePath === "/colors") {
        const requiredProperties = (error.params as { requiredProperties?: string[] }).requiredProperties;
        for (const requiredProperty of requiredProperties ?? []) {
          missingColors.add(requiredProperty);
        }
        continue;
      }
      const path = error.instancePath || "/";
      otherErrors.push(`  - ${path}: ${error.message}`);
    }

    let errorMessage = `Invalid theme "${label}":\n`;
    if (missingColors.size > 0) {
      errorMessage += "\nMissing required color tokens:\n";
      errorMessage += Array.from(missingColors)
        .sort()
        .map((color) => `  - ${color}`)
        .join("\n");
    }
    if (otherErrors.length > 0) {
      errorMessage += `\n\nOther errors:\n${otherErrors.join("\n")}`;
    }

    throw new Error(errorMessage);
  }

  return json as ThemeJson;
}
```

**验证失败示例输出**：
```
Invalid theme "my-theme":

Missing required color tokens:
  - accent
  - border
  - mdHeading

Other errors:
  - /vars/0: Expected string but got number
```

## 变量解析系统

### 变量引用机制

颜色值可以引用 `vars` 中定义的变量：

```typescript
function resolveVarRefs(
  value: ColorValue,
  vars: Record<string, ColorValue>,
  visited = new Set<string>(),
): string | number {
  // 直接值（数字、空字符串、十六进制）直接返回
  if (typeof value === "number" || value === "" || value.startsWith("#")) {
    return value;
  }

  // 循环引用检测
  if (visited.has(value)) {
    throw new Error(`Circular variable reference detected: ${value}`);
  }

  // 变量不存在
  if (!(value in vars)) {
    throw new Error(`Variable reference not found: ${value}`);
  }

  // 递归解析
  visited.add(value);
  return resolveVarRefs(vars[value], vars, visited);
}
```

**示例**：
```json
{
  "vars": {
    "primary": "#ff0000",
    "secondary": "primary",    // 引用另一个变量
    "accent": "secondary"      // 链式引用
  },
  "colors": {
    "border": "accent"         // 解析为 #ff0000
  }
}
```

**循环引用检测**：
```json
{
  "vars": {
    "a": "b",
    "b": "a"  // 错误：Circular variable reference detected: a
  }
}
```

## 颜色模式检测

### 检测逻辑

系统自动检测终端颜色支持能力：

```typescript
function detectColorMode(): ColorMode {
  const colorterm = process.env.COLORTERM;
  if (colorterm === "truecolor" || colorterm === "24bit") {
    return "truecolor";
  }

  // Windows Terminal 支持 truecolor
  if (process.env.WT_SESSION) {
    return "truecolor";
  }

  const term = process.env.TERM || "";

  // 有限终端降级到 256color
  if (term === "dumb" || term === "" || term === "linux") {
    return "256color";
  }

  // Terminal.app 不支持 truecolor
  if (process.env.TERM_PROGRAM === "Apple_Terminal") {
    return "256color";
  }

  // GNU screen 需要显式 COLORTERM=truecolor
  if (term === "screen" || term.startsWith("screen-") || term.startsWith("screen.")) {
    return "256color";
  }

  // 默认 assume truecolor（现代终端）
  return "truecolor";
}
```

### 环境变量优先级

| 环境变量 | 值 | 模式 |
|---------|-----|------|
| `COLORTERM` | `truecolor`, `24bit` | truecolor |
| `WT_SESSION` | (any) | truecolor |
| `TERM_PROGRAM` | `Apple_Terminal` | 256color |
| `TERM` | `dumb`, `""`, `linux` | 256color |
| `TERM` | `screen*` | 256color |
| (其他) | - | truecolor |

## 颜色转换系统

### Truecolor 模式

直接使用 24-bit RGB 颜色：

```typescript
function hexToRgb(hex: string): { r: number; g: number; b: number } {
  const cleaned = hex.replace("#", "");
  if (cleaned.length !== 6) {
    throw new Error(`Invalid hex color: ${hex}`);
  }
  const r = parseInt(cleaned.substring(0, 2), 16);
  const g = parseInt(cleaned.substring(2, 4), 16);
  const b = parseInt(cleaned.substring(4, 6), 16);
  return { r, g, b };
}

function fgAnsi(color: string | number, mode: ColorMode): string {
  if (color === "") return "\x1b[39m";  // 重置
  if (typeof color === "number") return `\x1b[38;5;${color}m`;
  if (color.startsWith("#")) {
    if (mode === "truecolor") {
      const { r, g, b } = hexToRgb(color);
      return `\x1b[38;2;${r};${g};${b}m`;  // ANSI 24-bit
    }
  }
}
```

**ANSI 转义码**：
- 前景色：`\x1b[38;2;R;G;Bm`
- 背景色：`\x1b[48;2;R;G;Bm`

### 256 色模式

将十六进制转换为 256 色索引：

```typescript
// 6x6x6 色立方体通道值
const CUBE_VALUES = [0, 95, 135, 175, 215, 255];

// 灰度渐变值 (索引 232-255)
const GRAY_VALUES = Array.from({ length: 24 }, (_, i) => 8 + i * 10);

function rgbTo256(r: number, g: number, b: number): number {
  // 在 6x6x6 立方体中查找最接近的颜色
  const rIdx = findClosestCubeIndex(r);
  const gIdx = findClosestCubeIndex(g);
  const bIdx = findClosestCubeIndex(b);
  const cubeIndex = 16 + 36 * rIdx + 6 * gIdx + bIdx;

  // 查找最接近的灰度
  const gray = Math.round(0.299 * r + 0.587 * g + 0.114 * b);
  const grayIdx = findClosestGrayIndex(gray);
  const grayIndex = 232 + grayIdx;

  // 如果颜色有明显的饱和度（色调重要），优先使用立方体
  const maxC = Math.max(r, g, b);
  const minC = Math.min(r, g, b);
  const spread = maxC - minC;

  // 只有在颜色接近中性（spread < 10）且灰度更接近时才使用灰度
  if (spread < 10 && grayDist < cubeDist) {
    return grayIndex;
  }

  return cubeIndex;
}
```

**ANSI 256 色索引分配**：
- 索引 0-15：基本颜色
- 索引 16-231：6x6x6 色立方体（216 色）
- 索引 232-255：灰度渐变（24 色）

## 主题实例

### Theme 类

```typescript
export class Theme {
  readonly name?: string;
  readonly sourcePath?: string;
  sourceInfo?: SourceInfo;
  private fgColors: Map<ThemeColor, string>;
  private bgColors: Map<ThemeBg, string>;
  private mode: ColorMode;

  constructor(
    fgColors: Record<ThemeColor, string | number>,
    bgColors: Record<ThemeBg, string | number>,
    mode: ColorMode,
    options: { name?: string; sourcePath?: string; sourceInfo?: SourceInfo } = {},
  ) {
    this.name = options.name;
    this.sourcePath = options.sourcePath;
    this.mode = mode;
    this.fgColors = new Map();
    for (const [key, value] of Object.entries(fgColors) as [ThemeColor, string | number][]) {
      this.fgColors.set(key, fgAnsi(value, mode));
    }
    this.bgColors = new Map();
    for (const [key, value] of Object.entries(bgColors) as [ThemeBg, string | number][]) {
      this.bgColors.set(key, bgAnsi(value, mode));
    }
  }

  // 前景色（自动重置）
  fg(color: ThemeColor, text: string): string {
    const ansi = this.fgColors.get(color);
    if (!ansi) throw new Error(`Unknown theme color: ${color}`);
    return `${ansi}${text}\x1b[39m`;
  }

  // 背景色（自动重置）
  bg(color: ThemeBg, text: string): string {
    const ansi = this.bgColors.get(color);
    if (!ansi) throw new Error(`Unknown theme background color: ${color}`);
    return `${ansi}${text}\x1b[49m`;
  }

  // 文本样式（基于 chalk）
  bold(text: string): string {
    return chalk.bold(text);
  }
  italic(text: string): string {
    return chalk.italic(text);
  }
  underline(text: string): string {
    return chalk.underline(text);
  }
  strikethrough(text: string): string {
    return chalk.strikethrough(text);
  }
  inverse(text: string): string {
    return chalk.inverse(text);
  }

  // 思考级别边框色
  getThinkingBorderColor(level: "off" | "minimal" | "low" | "medium" | "high" | "xhigh"): (str: string) => string {
    switch (level) {
      case "off": return (str: string) => this.fg("thinkingOff", str);
      case "minimal": return (str: string) => this.fg("thinkingMinimal", str);
      case "low": return (str: string) => this.fg("thinkingLow", str);
      case "medium": return (str: string) => this.fg("thinkingMedium", str);
      case "high": return (str: string) => this.fg("thinkingHigh", str);
      case "xhigh": return (str: string) => this.fg("thinkingXhigh", str);
      default: return (str: string) => this.fg("thinkingOff", str);
    }
  }
}
```

### 全局主题实例

使用 `globalThis` 确保跨模块加载器（tsx + jiti）共享：

```typescript
const THEME_KEY = Symbol.for("@mariozechner/pi-coding-agent:theme");

export const theme: Theme = new Proxy({} as Theme, {
  get(_target, prop) {
    const t = (globalThis as Record<symbol, Theme>)[THEME_KEY];
    if (!t) throw new Error("Theme not initialized. Call initTheme() first.");
    return (t as unknown as Record<string | symbol, unknown>)[prop];
  },
});

function setGlobalTheme(t: Theme): void {
  (globalThis as Record<symbol, Theme>)[THEME_KEY] = t;
}
```

## 热重载系统

### 文件监视器

自定义主题支持文件监视和自动重载：

```typescript
let themeWatcher: fs.FSWatcher | undefined;
let themeReloadTimer: NodeJS.Timeout | undefined;

function startThemeWatcher(): void {
  stopThemeWatcher();

  // 只监视自定义主题
  if (!currentThemeName || currentThemeName === "dark" || currentThemeName === "light") {
    return;
  }

  const customThemesDir = getCustomThemesDir();
  const watchedThemeName = currentThemeName;
  const watchedFileName = `${watchedThemeName}.json`;
  const themeFile = path.join(customThemesDir, watchedFileName);

  if (!fs.existsSync(themeFile)) {
    return;
  }

  const scheduleReload = () => {
    if (themeReloadTimer) {
      clearTimeout(themeReloadTimer);
    }
    themeReloadTimer = setTimeout(() => {
      themeReloadTimer = undefined;

      // 忽略切换主题后的过时定时器
      if (currentThemeName !== watchedThemeName) {
        return;
      }

      // 文件临时缺失时保持最后加载的主题
      if (!fs.existsSync(themeFile)) {
        return;
      }

      try {
        const reloadedTheme = loadThemeFromPath(themeFile);
        registeredThemes.set(watchedThemeName, reloadedTheme);
        setGlobalTheme(reloadedTheme);
        if (onThemeChangeCallback) {
          onThemeChangeCallback();
        }
      } catch (_error) {
        // 忽略错误（文件编辑时可能处于无效状态）
      }
    }, 100);  // 100ms 防抖
  };

  themeWatcher = watchWithErrorHandler(
    customThemesDir,
    (_eventType, filename) => {
      if (currentThemeName !== watchedThemeName) {
        return;
      }
      if (!filename || filename !== watchedFileName) {
        return;
      }
      scheduleReload();
    },
    () => {
      closeWatcher(themeWatcher);
      themeWatcher = undefined;
    },
  ) ?? undefined;
}
```

### 热重载特性

- **防抖延迟**：100ms 防抖避免频繁重载
- **状态检查**：只在当前主题匹配时重载
- **容错处理**：文件编辑时忽略无效状态
- **回调通知**：重载后触发 `onThemeChange` 回调

## 语法高亮集成

### CLI Highlight 集成

使用 `cli-highlight` 库提供语法高亮：

```typescript
import { highlight, supportsLanguage } from "cli-highlight";

function buildCliHighlightTheme(t: Theme): CliHighlightTheme {
  return {
    keyword: (s: string) => t.fg("syntaxKeyword", s),
    built_in: (s: string) => t.fg("syntaxType", s),
    literal: (s: string) => t.fg("syntaxNumber", s),
    number: (s: string) => t.fg("syntaxNumber", s),
    string: (s: string) => t.fg("syntaxString", s),
    comment: (s: string) => t.fg("syntaxComment", s),
    function: (s: string) => t.fg("syntaxFunction", s),
    title: (s: string) => t.fg("syntaxFunction", s),
    class: (s: string) => t.fg("syntaxType", s),
    type: (s: string) => t.fg("syntaxType", s),
    attr: (s: string) => t.fg("syntaxVariable", s),
    variable: (s: string) => t.fg("syntaxVariable", s),
    params: (s: string) => t.fg("syntaxVariable", s),
    operator: (s: string) => t.fg("syntaxOperator", s),
    punctuation: (s: string) => t.fg("syntaxPunctuation", s),
  };
}

export function highlightCode(code: string, lang?: string): string[] {
  const validLang = lang && supportsLanguage(lang) ? lang : undefined;
  if (!validLang) {
    return code.split("\n").map((line) => theme.fg("mdCodeBlock", line));
  }
  const opts = {
    language: validLang,
    ignoreIllegals: true,
    theme: getCliHighlightTheme(theme),
  };
  try {
    return highlight(code, opts).split("\n");
  } catch {
    return code.split("\n");
  }
}
```

### 支持的语言

```typescript
const extToLang: Record<string, string> = {
  ts: "typescript", tsx: "typescript",
  js: "javascript", jsx: "javascript",
  py: "python", rb: "ruby", rs: "rust", go: "go",
  java: "java", kt: "kotlin", swift: "swift",
  c: "c", h: "c", cpp: "cpp", cc: "cpp", cxx: "cpp",
  cs: "csharp", php: "php",
  sh: "bash", bash: "bash", zsh: "bash",
  sql: "sql", html: "html", css: "css",
  json: "json", yaml: "yaml", yml: "yaml",
  md: "markdown", dockerfile: "dockerfile",
  lua: "lua", perl: "perl", r: "r",
  scala: "scala", clj: "clojure",
  ex: "elixir", exs: "elixir", erl: "erlang",
  hs: "haskell", ml: "ocaml", vim: "vim",
  graphql: "graphql", proto: "protobuf",
  tf: "hcl", hcl: "hcl",
};
```

## HTML 导出支持

### 颜色转换

将终端颜色转换为 CSS 兼容的十六进制：

```typescript
function ansi256ToHex(index: number): string {
  // 基本颜色 (0-15)
  const basicColors = [
    "#000000", "#800000", "#008000", "#808000",
    "#000080", "#800080", "#008080", "#c0c0c0",
    "#808080", "#ff0000", "#00ff00", "#ffff00",
    "#0000ff", "#ff00ff", "#00ffff", "#ffffff",
  ];
  if (index < 16) {
    return basicColors[index];
  }

  // 色立方体 (16-231)
  if (index < 232) {
    const cubeIndex = index - 16;
    const r = Math.floor(cubeIndex / 36);
    const g = Math.floor((cubeIndex % 36) / 6);
    const b = cubeIndex % 6;
    const toHex = (n: number) => (n === 0 ? 0 : 55 + n * 40).toString(16).padStart(2, "0");
    return `#${toHex(r)}${toHex(g)}${toHex(b)}`;
  }

  // 灰度 (232-255)
  const gray = 8 + (index - 232) * 10;
  const grayHex = gray.toString(16).padStart(2, "0");
  return `#${grayHex}${grayHex}${grayHex}`;
}

export function getResolvedThemeColors(themeName?: string): Record<string, string> {
  const name = themeName ?? currentThemeName ?? getDefaultTheme();
  const isLight = name === "light";
  const themeJson = loadThemeJson(name);
  const resolved = resolveThemeColors(themeJson.colors, themeJson.vars);

  const defaultText = isLight ? "#000000" : "#e5e5e7";
  const cssColors: Record<string, string> = {};

  for (const [key, value] of Object.entries(resolved)) {
    if (typeof value === "number") {
      cssColors[key] = ansi256ToHex(value);
    } else if (value === "") {
      cssColors[key] = defaultText;
    } else {
      cssColors[key] = value;
    }
  }
  return cssColors;
}
```

### 导出颜色配置

主题可以指定 HTML 导出的专用颜色：

```json
{
  "colors": { ... },
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

```typescript
export function getThemeExportColors(themeName?: string): {
  pageBg?: string;
  cardBg?: string;
  infoBg?: string;
} {
  const name = themeName ?? currentThemeName ?? getDefaultTheme();
  try {
    const themeJson = loadThemeJson(name);
    const exportSection = themeJson.export;
    if (!exportSection) return {};

    const vars = themeJson.vars ?? {};
    const resolve = (value: ColorValue | undefined): string | undefined => {
      if (value === undefined) return undefined;
      const resolved = resolveVarRefs(value, vars);
      if (typeof resolved === "number") return ansi256ToHex(resolved);
      if (resolved === "") return undefined;
      return resolved;
    };

    return {
      pageBg: resolve(exportSection.pageBg),
      cardBg: resolve(exportSection.cardBg),
      infoBg: resolve(exportSection.infoBg),
    };
  } catch {
    return {};
  }
}
```

## TUI 集成

### 主题适配器

为不同 TUI 组件提供主题适配：

```typescript
export function getMarkdownTheme(): MarkdownTheme {
  return {
    heading: (text: string) => theme.fg("mdHeading", text),
    link: (text: string) => theme.fg("mdLink", text),
    linkUrl: (text: string) => theme.fg("mdLinkUrl", text),
    code: (text: string) => theme.fg("mdCode", text),
    codeBlock: (text: string) => theme.fg("mdCodeBlock", text),
    codeBlockBorder: (text: string) => theme.fg("mdCodeBlockBorder", text),
    quote: (text: string) => theme.fg("mdQuote", text),
    quoteBorder: (text: string) => theme.fg("mdQuoteBorder", text),
    hr: (text: string) => theme.fg("mdHr", text),
    listBullet: (text: string) => theme.fg("mdListBullet", text),
    bold: (text: string) => theme.bold(text),
    italic: (text: string) => theme.italic(text),
    underline: (text: string) => theme.underline(text),
    strikethrough: (text: string) => chalk.strikethrough(text),
    highlightCode: (code: string, lang?: string): string[] => {
      return highlightCode(code, lang);
    },
  };
}

export function getSelectListTheme(): SelectListTheme {
  return {
    selectedPrefix: (text: string) => theme.fg("accent", text),
    selectedText: (text: string) => theme.fg("accent", text),
    description: (text: string) => theme.fg("muted", text),
    scrollInfo: (text: string) => theme.fg("muted", text),
    noMatch: (text: string) => theme.fg("muted", text),
  };
}

export function getEditorTheme(): EditorTheme {
  return {
    borderColor: (text: string) => theme.fg("borderMuted", text),
    selectList: getSelectListTheme(),
  };
}
```

## 最佳实践

### 创建自定义主题

1. **从内置主题复制**：
```bash
cp ~/.pi/themes/dark.json ~/.pi/themes/my-theme.json
```

2. **修改必要字段**：
```json
{
  "$schema": "https://raw.githubusercontent.com/badlogic/pi-mono/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": { ... },
  "colors": { ... }
}
```

3. **测试主题**：
```bash
pi config set theme my-theme
```

### 颜色设计建议

1. **使用变量系统**：定义语义化变量
```json
{
  "vars": {
    "primary": "#ff0000",
    "secondary": "#00ff00",
    "muted": "#808080"
  },
  "colors": {
    "accent": "primary",
    "border": "secondary",
    "dim": "muted"
  }
}
```

2. **考虑可访问性**：确保足够的对比度
```json
{
  "colors": {
    "text": "",        // 使用终端默认色
    "mdCode": "#b5bd68",  // 与背景有足够对比
    "syntaxComment": "#6A9955"  // 明显但不过于突出
  }
}
```

3. **遵循语义命名**：
- `accent`：主要强调色
- `border`：边框颜色
- `success/error/warning`：状态指示
- `muted/dim`：次要内容
- `md*`：Markdown 元素
- `syntax*`：语法高亮

### 调试主题

使用测试脚本验证主题：

```typescript
import { loadThemeFromPath } from "./theme";

const theme = loadThemeFromPath("~/.pi/themes/my-theme.json");
console.log(theme.fg("accent", "Accent text"));
console.log(theme.bg("selectedBg", "Selected background"));
```

## 生命周期图

[MermaidChart:./_LEARN/docs/mmd/theme-system-lifecycle.mmd]

## 参考资源

- **主题源文件**：`/packages/coding-agent/src/modes/interactive/theme/theme.ts:1`
- **深色主题**：`/packages/coding-agent/src/modes/interactive/theme/dark.json`
- **浅色主题**：`/packages/coding-agent/src/modes/interactive/theme/light.json`
- **配置路径**：`/packages/coding-agent/src/config.ts:339`
- **主题选择器**：`/packages/coding-agent/src/modes/interactive/components/theme-selector.ts:1`
