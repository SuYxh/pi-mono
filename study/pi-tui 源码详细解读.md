# @mariozechner/pi-tui 源码详细解读

> 本文档是对 `packages/tui` 包的源码级深度解读，涵盖渲染引擎、组件系统、输入处理、按键解析、快捷键系统、编辑器组件、Markdown 渲染、自动补全、Kill Ring / Undo Stack 等核心模块。

---

## 一、包概览

| 属性 | 值 |
|---|---|
| 包名 | `@mariozechner/pi-tui` |
| 定位 | 自研的终端 UI 框架，为 pi（Coding Agent）提供完整的交互界面 |
| 入口 | `src/index.ts` → `dist/index.js` |
| 运行环境 | Node.js >= 20（仅终端，不支持浏览器） |
| 依赖 | `marked`（Markdown 解析）、`string-width`（Unicode 宽度计算） |
| 测试框架 | `node --test`（Node.js 内置测试运行器） |

### 1.1 设计哲学

pi-tui **不是**一个通用 TUI 框架（如 Ink/Blessed），而是为 pi 这个特定应用量身定制的：

1. **行级渲染**：组件的 `render(width)` 返回 `string[]`（每个元素是一行终端输出），上层负责换行和填充
2. **差分输出**：TUI 引擎维护前后帧的行数组，只重绘变化的行
3. **CSI 2026 同步输出**：使用 `\x1bP=1s` / `\x1bP=2s` 包裹输出，避免画面撕裂
4. **组件极简接口**：`Component` 只需实现 `render(width): string[]`，`Focusable` 额外实现 `handleInput(data)`

### 1.2 目录结构

```
packages/tui/
├── src/
│   ├── index.ts                # 公共导出
│   ├── tui.ts                  # 核心渲染引擎（TUI 类）
│   ├── terminal.ts             # 终端抽象层（Terminal 类）
│   ├── keys.ts                 # 按键解析（CSI/SS3/Meta/Kitty 协议）
│   ├── keybindings.ts          # 快捷键注册中心（KeybindingsManager）
│   ├── stdin-buffer.ts         # stdin 输入缓冲（StdinBuffer）
│   ├── utils.ts                # 工具函数（visibleWidth, wrapTextWithAnsi 等）
│   ├── editor-component.ts     # EditorComponent 接口（扩展点）
│   ├── autocomplete.ts         # 自动补全系统
│   ├── fuzzy.ts                # 模糊匹配算法
│   ├── kill-ring.ts            # Emacs Kill Ring
│   ├── undo-stack.ts           # 通用 Undo 栈
│   ├── terminal-image.ts       # 终端图片协议（Kitty/iTerm2）
│   └── components/
│       ├── editor.ts           # 多行编辑器（2196 行，最大组件）
│       ├── input.ts            # 单行输入框
│       ├── markdown.ts         # Markdown 渲染器
│       ├── select-list.ts      # 选择列表
│       └── box.ts              # 容器盒子
├── test/                       # node --test 测试
├── package.json
└── tsconfig.json
```

---

## 二、核心渲染引擎 (`src/tui.ts`)

### 2.1 Component 接口

```typescript
export interface Component {
    render(width: number): string[];
    invalidate?(): void;
}
```

所有 UI 组件的基础契约。`render(width)` 接收可用宽度，返回渲染好的行数组（带 ANSI 转义码的字符串）。`invalidate()` 可选，用于清除缓存、强制重渲染。

### 2.2 Focusable 接口

```typescript
export interface Focusable {
    focused: boolean;
    handleInput(data: string): void;
}
```

可接收输入的组件。TUI 维护一个 `focusStack`，栈顶的 Focusable 接收所有按键输入。

### 2.3 CURSOR_MARKER

```typescript
export const CURSOR_MARKER = "\x1b[?25h";
```

零宽度标记，嵌入在渲染输出中。TUI 渲染时会：
1. 扫描所有行，找到包含 `CURSOR_MARKER` 的行
2. 计算该标记在行中的列位置
3. 将硬件光标移动到对应位置（用于 IME 输入法定位）

### 2.4 TUI 类核心

```typescript
export class TUI {
    terminal: Terminal;
    private components: Component[] = [];
    private focusStack: Focusable[] = [];
    private previousLines: string[] = [];
    private renderScheduled = false;

    // 核心方法
    requestRender(): void;        // 请求下一帧渲染（去重）
    addComponent(c: Component): void;
    removeComponent(c: Component): void;
    pushFocus(f: Focusable): void;
    popFocus(f: Focusable): void;
}
```

### 2.5 渲染管线

TUI 的渲染采用 **请求-调度-执行** 三阶段：

```
组件状态变化
    │
    ├─ tui.requestRender()
    │    │
    │    └─ this.renderScheduled = true
    │       queueMicrotask(() => this.render())   // 去抖动：微任务级合并
    │
    └─ this.render()
         │
         ├─ 1. 收集所有组件的渲染输出
         │     for (const component of this.components) {
         │         lines.push(...component.render(terminal.cols));
         │     }
         │
         ├─ 2. 差分对比
         │     for (let i = 0; i < lines.length; i++) {
         │         if (lines[i] !== this.previousLines[i]) {
         │             changedLines.push(i);
         │         }
         │     }
         │
         ├─ 3. CSI 2026 同步输出包裹
         │     output += "\x1bP=1s";  // 开始同步更新
         │     // ... 输出变化的行 ...
         │     output += "\x1bP=2s";  // 结束同步更新
         │
         ├─ 4. 只重绘变化的行
         │     for (const lineIndex of changedLines) {
         │         moveCursor(lineIndex);
         │         write(lines[lineIndex]);
         │     }
         │
         ├─ 5. 光标定位
         │     // 扫描 CURSOR_MARKER，计算列位置
         │     // 移动硬件光标到标记位置
         │
         └─ 6. 保存当前帧
              this.previousLines = lines;
```

**差分渲染的性能收益**：对于 pi 的典型场景（用户输入、LLM 流式响应），每帧只有少量行变化。差分渲染将终端写入量从 O(全屏) 降到 O(变化行数)。

**CSI 2026 同步输出**：`\x1bP=1s` 告诉终端 "接下来的输出是一帧的一部分，先缓冲不要显示"，`\x1bP=2s` 告诉终端 "这帧完成了，现在显示"。这避免了用户看到半渲染的画面。

### 2.6 输入处理管线

```
stdin
  │
  ├─ StdinBuffer 缓冲
  │    │
  │    ├─ data event（完整的按键序列）
  │    │    │
  │    │    └─ TUI.handleInput(data)
  │    │         │
  │    │         ├─ focusStack.length > 0?
  │    │         │    └─ focusStack[top].handleInput(data)
  │    │         │
  │    │         └─ 否则：全局按键处理
  │    │
  │    └─ paste event（完整的粘贴文本）
  │         └─ 直接传给焦点组件
  │
  └─ Terminal.onResize()
       └─ TUI.requestRender()
```

### 2.7 焦点管理

```typescript
// 推入焦点
pushFocus(focusable: Focusable): void {
    // 将之前的焦点组件的 focused 设为 false
    const prev = this.focusStack[this.focusStack.length - 1];
    if (prev) prev.focused = false;
    // 新组件入栈并获取焦点
    this.focusStack.push(focusable);
    focusable.focused = true;
    this.requestRender();
}

// 弹出焦点
popFocus(focusable: Focusable): void {
    // 从栈中移除指定组件
    const index = this.focusStack.indexOf(focusable);
    if (index !== -1) {
        this.focusStack.splice(index, 1);
        focusable.focused = false;
    }
    // 恢复栈顶组件的焦点
    const next = this.focusStack[this.focusStack.length - 1];
    if (next) next.focused = true;
    this.requestRender();
}
```

**栈式焦点**的应用场景：
- 正常状态：Editor 在栈底
- 弹出自动补全列表：SelectList 入栈（EditorComponent 内部管理）
- 弹出确认对话框：Dialog 入栈
- 关闭对话框：Dialog 出栈，焦点自动回到 SelectList 或 Editor

---

## 三、终端抽象层 (`src/terminal.ts`)

### 3.1 Terminal 类

```typescript
export class Terminal {
    rows: number;
    cols: number;
    readonly stdin: NodeJS.ReadStream;
    readonly stdout: NodeJS.WriteStream;

    enableRawMode(): void;     // 进入原始模式（关闭行缓冲、回显）
    disableRawMode(): void;    // 恢复正常模式
    enableMouseTracking(): void;
    disableMouseTracking(): void;
    enableBracketedPaste(): void;
    disableBracketedPaste(): void;
    write(data: string): void;
    hideCursor(): void;
    showCursor(): void;
    moveCursorTo(row: number, col: number): void;
    clearLine(): void;
    clearScreen(): void;
    queryTerminalVersion(): void;      // XTVERSION 查询
    queryCellDimensions(): void;       // 像素尺寸查询（用于图片渲染）
}
```

**原始模式 (Raw Mode)**：标准终端是行缓冲的（按 Enter 才发送），进入原始模式后，每个按键都立即发送到 stdin。这是 TUI 应用的基础。

### 3.2 终端能力检测

```typescript
// terminal-image.ts
export function detectCapabilities(): TerminalCapabilities {
    if (process.env.KITTY_WINDOW_ID || termProgram === "kitty")
        return { images: "kitty", trueColor: true, hyperlinks: true };
    if (termProgram === "ghostty" || process.env.GHOSTTY_RESOURCES_DIR)
        return { images: "kitty", trueColor: true, hyperlinks: true };
    if (termProgram === "wezterm")
        return { images: "iterm2", trueColor: true, hyperlinks: true };
    // ... VS Code, iTerm2, Hyper, etc.
}
```

pi-tui 支持两种图片协议：
- **Kitty Graphics Protocol**：Kitty、Ghostty 原生支持
- **iTerm2 Inline Images**：WezTerm、iTerm2 支持
- 其他终端：不显示图片

---

## 四、按键解析 (`src/keys.ts`)

### 4.1 问题

终端输入不是简单的字符——按键通过转义序列编码：

| 按键 | 原始字节 |
|---|---|
| `a` | `0x61` |
| `Enter` | `0x0D` |
| `Ctrl+C` | `0x03` |
| `Up` | `\x1b[A` |
| `Shift+Enter` | `\x1b[13;2~` (xterm) 或 `\x1b[13;2u` (Kitty) |
| `Ctrl+Alt+Delete` | `\x1b[3;7~` |
| `Alt+a` | `\x1b\x61` (Meta) 或 `\x1b[97;3u` (Kitty CSI-u) |

### 4.2 KeyId 标识符

```typescript
export type KeyId = string;
// 格式: "[modifier+]keyname"
// 修饰符: ctrl, alt, shift, meta
// 键名: a-z, 0-9, enter, escape, up, down, left, right, backspace, delete, tab, ...

// 例子:
// "ctrl+c"
// "shift+enter"
// "alt+backspace"
// "ctrl+alt+]"
```

### 4.3 matchesKey 函数

这是按键匹配的核心函数：

```typescript
export function matchesKey(data: string, keyId: KeyId): boolean {
    // 1. 解析 keyId → 期望的修饰符 + 键名
    const parts = keyId.split("+");
    const keyName = parts.pop()!;
    const wantCtrl = parts.includes("ctrl");
    const wantAlt = parts.includes("alt");
    const wantShift = parts.includes("shift");

    // 2. 解析 data → 实际的修饰符 + 键码
    const parsed = parseKeySequence(data);

    // 3. 比较
    return parsed.ctrl === wantCtrl
        && parsed.alt === wantAlt
        && parsed.shift === wantShift
        && parsed.key === keyName;
}
```

### 4.4 按键序列解析

`parseKeySequence()` 处理 6 种编码格式：

**1. 控制字符 (C0)**
```
\x01 → Ctrl+A     (code - 0x40 = 'A')
\x03 → Ctrl+C
\x1A → Ctrl+Z
```

**2. CSI 标准序列**
```
\x1b[A     → Up
\x1b[B     → Down
\x1b[1;2A  → Shift+Up       (参数2=修饰符, 2=Shift)
\x1b[1;5A  → Ctrl+Up        (5=Ctrl)
\x1b[1;3A  → Alt+Up         (3=Alt)
\x1b[1;7A  → Ctrl+Alt+Up    (7=Ctrl+Alt)
```

修饰符解码表（修饰符参数 = 1 + 位掩码）：
| 参数值 | 位掩码 | 修饰符 |
|---|---|---|
| 2 | 1 | Shift |
| 3 | 2 | Alt |
| 4 | 3 | Shift+Alt |
| 5 | 4 | Ctrl |
| 6 | 5 | Ctrl+Shift |
| 7 | 6 | Ctrl+Alt |
| 8 | 7 | Ctrl+Alt+Shift |

**3. Kitty CSI-u 协议**
```
\x1b[97u     → a                (codepoint 97 = 'a')
\x1b[97;2u   → Shift+a          (修饰符同上)
\x1b[13u     → Enter            (codepoint 13)
\x1b[13;2u   → Shift+Enter
\x1b[97:65u  → a with Shift='A' (97=base, 65=shifted)
```

**4. SS3 序列**
```
\x1bOA → Up（部分终端在 Application 模式下使用）
\x1bOP → F1
\x1bOQ → F2
```

**5. Meta 序列**
```
\x1ba  → Alt+a   (ESC + 字符 = Meta/Alt 修饰)
\x1b\r → Alt+Enter
```

**6. xterm 功能键**
```
\x1b[3~   → Delete
\x1b[5~   → PageUp
\x1b[6~   → PageDown
\x1b[2;5~ → Ctrl+Insert
```

### 4.5 Kitty CSI-u 可打印字符解码

```typescript
export function decodeKittyPrintable(data: string): string | undefined {
    // CSI-u: \x1b[<codepoint>u
    const match = data.match(/^\x1b\[(\d+)u$/);
    if (!match) return undefined;

    const codepoint = parseInt(match[1], 10);
    // 只解码可打印字符（>= 32，排除功能键映射范围）
    if (codepoint >= 32 && codepoint < 0x110000) {
        return String.fromCodePoint(codepoint);
    }
    return undefined;
}
```

这解决了一个关键问题：当终端启用 Kitty 协议时，即使普通的 `a` 键也会以 `\x1b[97u` 发送。Editor 组件在处理完所有快捷键匹配后，会尝试 `decodeKittyPrintable` 来提取可打印字符。

---

## 五、输入缓冲 (`src/stdin-buffer.ts`)

### 5.1 问题

Node.js 的 `stdin.on("data")` 事件可能将一个完整的转义序列拆成多个 chunk：

```
鼠标点击 \x1b[<35;20;5m 可能到达为：
  Event 1: "\x1b"
  Event 2: "[<35"
  Event 3: ";20;5m"
```

如果不缓冲，`\x1b` 会被误判为 Escape 键。

### 5.2 StdinBuffer 实现

```typescript
export class StdinBuffer extends EventEmitter<StdinBufferEventMap> {
    private buffer: string = "";
    private timeout: ReturnType<typeof setTimeout> | null = null;
    private readonly timeoutMs: number;       // 默认 10ms
    private pasteMode: boolean = false;
    private pasteBuffer: string = "";
}
```

**核心流程**：

```
stdin chunk 到达
    │
    ├─ 检查是否在粘贴模式（Bracketed Paste）
    │    ├─ 是：累积到 pasteBuffer，等待 \x1b[201~
    │    └─ 否：检查是否包含 \x1b[200~（粘贴开始）
    │
    ├─ 累积到 buffer
    │
    ├─ extractCompleteSequences(buffer)
    │    │
    │    ├─ 逐字节扫描，对每个位置调用 isCompleteSequence()
    │    │
    │    ├─ 完整序列 → 放入 sequences 数组
    │    └─ 不完整序列 → 留在 remainder
    │
    ├─ 发射所有完整序列的 "data" 事件
    │
    └─ 如果有 remainder
         └─ 设置 10ms 超时后 flush（保底：即使后续没有数据也能发出）
```

**Bracketed Paste Mode**：终端在粘贴开始时发送 `\x1b[200~`，结束时发送 `\x1b[201~`。StdinBuffer 检测到开始标记后进入 pasteMode，将所有数据累积到 pasteBuffer，直到检测到结束标记，然后一次性发射 `paste` 事件。

### 5.3 序列完整性判断

```typescript
function isCompleteSequence(data: string): "complete" | "incomplete" | "not-escape" {
    if (!data.startsWith(ESC)) return "not-escape";
    if (data.length === 1) return "incomplete";

    // CSI: ESC [ ... (终止符 0x40-0x7E)
    if (data.startsWith("\x1b[")) return isCompleteCsiSequence(data);
    // OSC: ESC ] ... (ST 或 BEL 终止)
    if (data.startsWith("\x1b]")) return isCompleteOscSequence(data);
    // DCS: ESC P ... ESC \
    if (data.startsWith("\x1bP")) return isCompleteDcsSequence(data);
    // APC: ESC _ ... ESC \  (Kitty 图片响应)
    if (data.startsWith("\x1b_")) return isCompleteApcSequence(data);
    // SS3: ESC O + 1 char
    if (data.startsWith("\x1bO")) return data.length >= 3 ? "complete" : "incomplete";
    // Meta: ESC + 1 char
    if (data.length >= 2) return "complete";

    return "complete";
}
```

---

## 六、快捷键系统 (`src/keybindings.ts`)

### 6.1 可配置的快捷键架构

```
KeybindingDefinition          // 定义：{ defaultKeys, description }
    ↓
KeybindingsManager            // 管理器：合并默认+用户配置
    ↓
getKeybindings().matches()    // 运行时匹配
```

### 6.2 内置快捷键定义

```typescript
export const TUI_KEYBINDINGS = {
    // 光标移动
    "tui.editor.cursorUp":        { defaultKeys: "up" },
    "tui.editor.cursorDown":      { defaultKeys: "down" },
    "tui.editor.cursorLeft":      { defaultKeys: ["left", "ctrl+b"] },
    "tui.editor.cursorRight":     { defaultKeys: ["right", "ctrl+f"] },
    "tui.editor.cursorWordLeft":  { defaultKeys: ["alt+left", "ctrl+left", "alt+b"] },
    "tui.editor.cursorWordRight": { defaultKeys: ["alt+right", "ctrl+right", "alt+f"] },
    "tui.editor.cursorLineStart": { defaultKeys: ["home", "ctrl+a"] },
    "tui.editor.cursorLineEnd":   { defaultKeys: ["end", "ctrl+e"] },
    "tui.editor.jumpForward":     { defaultKeys: "ctrl+]" },
    "tui.editor.jumpBackward":    { defaultKeys: "ctrl+alt+]" },

    // 删除操作
    "tui.editor.deleteCharBackward":  { defaultKeys: "backspace" },
    "tui.editor.deleteCharForward":   { defaultKeys: ["delete", "ctrl+d"] },
    "tui.editor.deleteWordBackward":  { defaultKeys: ["ctrl+w", "alt+backspace"] },
    "tui.editor.deleteWordForward":   { defaultKeys: ["alt+d", "alt+delete"] },
    "tui.editor.deleteToLineStart":   { defaultKeys: "ctrl+u" },
    "tui.editor.deleteToLineEnd":     { defaultKeys: "ctrl+k" },

    // Kill/Yank (Emacs 风格)
    "tui.editor.yank":    { defaultKeys: "ctrl+y" },
    "tui.editor.yankPop": { defaultKeys: "alt+y" },
    "tui.editor.undo":    { defaultKeys: "ctrl+-" },

    // 输入
    "tui.input.newLine": { defaultKeys: "shift+enter" },
    "tui.input.submit":  { defaultKeys: "enter" },
    "tui.input.tab":     { defaultKeys: "tab" },

    // 选择列表
    "tui.select.confirm": { defaultKeys: "enter" },
    "tui.select.cancel":  { defaultKeys: ["escape", "ctrl+c"] },
};
```

### 6.3 KeybindingsManager

```typescript
export class KeybindingsManager {
    private definitions: KeybindingDefinitions;  // 所有快捷键定义（含默认值）
    private userBindings: KeybindingsConfig;     // 用户自定义覆盖
    private keysById = new Map<Keybinding, KeyId[]>();  // 合并后的映射
    private conflicts: KeybindingConflict[];     // 冲突检测结果

    matches(data: string, keybinding: Keybinding): boolean {
        const keys = this.keysById.get(keybinding) ?? [];
        for (const key of keys) {
            if (matchesKey(data, key)) return true;
        }
        return false;
    }
}
```

**合并策略**：用户配置完全覆盖默认值（不是追加）。如果用户为 `tui.editor.cursorLeft` 配置了 `["left"]`，那么默认的 `ctrl+b` 就不再生效。

**冲突检测**：如果用户配置中两个不同的 keybinding 使用了相同的 KeyId，会记录到 `conflicts` 数组。

### 6.4 Declaration Merging 扩展点

```typescript
export interface Keybindings {
    "tui.editor.cursorUp": true;
    "tui.editor.cursorDown": true;
    // ...
}
```

下游包（如 `pi-agent-core`、`pi-coding-agent`）可以通过 TypeScript 声明合并添加新的快捷键 ID：

```typescript
// 在 coding-agent 中
declare module "@mariozechner/pi-tui" {
    interface Keybindings {
        "app.toggleThinking": true;
        "app.interrupt": true;
        "app.openEditor": true;
    }
}
```

这样 `getKeybindings().matches(data, "app.toggleThinking")` 也能获得类型检查。

---

## 七、多行编辑器组件 (`src/components/editor.ts`)

这是 pi-tui 最大、最复杂的组件（2196 行），实现了完整的终端文本编辑器。

### 7.1 状态模型

```typescript
interface EditorState {
    lines: string[];       // 逻辑行数组
    cursorLine: number;    // 光标所在逻辑行
    cursorCol: number;     // 光标所在列（字符偏移，非视觉列）
}
```

关键区分：**逻辑行** vs **视觉行**。一个逻辑行如果超过终端宽度，会被自动换行为多个视觉行。光标位置始终用逻辑坐标表示。

### 7.2 Word Wrap 算法

编辑器使用自定义的 `wordWrapLine()` 函数（而非简单截断）：

```typescript
export function wordWrapLine(line: string, maxWidth: number, preSegmented?): TextChunk[] {
    // 1. 基于 Intl.Segmenter 遍历每个 grapheme
    // 2. 遇到换行点（空格后的非空格字符）时记录 wrapOppIndex
    // 3. 当累积宽度超过 maxWidth 时：
    //    a. 优先回退到最近的 wrapOppIndex（词边界换行）
    //    b. 无词边界则在当前位置强制断行
    // 4. 返回 TextChunk[]，每个 chunk 记录 { text, startIndex, endIndex }
}
```

**TextChunk** 记录了视觉文本到原始逻辑行的映射关系，这对于光标在视觉行和逻辑行之间转换至关重要。

### 7.3 Paste Marker 机制

对于大段粘贴（>10 行或 >1000 字符），编辑器不会直接插入文本，而是：

```typescript
// 存储实际内容
this.pastes.set(pasteId, filteredText);

// 插入紧凑标记
const marker = `[paste #1 +123 lines]`;
this.insertTextAtCursorInternal(marker);
```

标记在编辑器中显示为原子单元（通过 `segmentWithMarkers()` 实现），光标移动、删除都将其视为一个整体。提交时 `expandPasteMarkers()` 将标记替换回原始文本。

**segmentWithMarkers** 的实现：
```typescript
function segmentWithMarkers(text: string, validIds: Set<number>): Iterable<Intl.SegmentData> {
    // 1. 如果文本中没有 "[paste #" → 直接用 Intl.Segmenter
    // 2. 正则找到所有 [paste #N ...] 标记
    // 3. 遍历 Intl.Segmenter 输出，将属于标记范围内的多个 grapheme 合并为一个
    // 结果：paste marker 在 segmenter 眼中是单个 grapheme
}
```

### 7.4 垂直光标移动 — Sticky Column

当光标在不同长度的行之间上下移动时，需要"记住"用户的目标列位置：

```
第1行: Hello World!          光标在 W (col=6)
第2行: Hi                    ← 按 Down：光标到 col=2（行尾）
第3行: Hello Beautiful World! ← 再按 Down：光标恢复到 col=6（记住的列）
```

实现在 `computeVerticalMoveColumn()` 中，有一个完整的决策表（7 种场景）：

| P | S | T | U | 场景 | 设置 Preferred | 移动到 |
|---|---|---|---|---|---|---|
| 0 | * | 0 | - | 首次导航，目标行够长 | null | 当前列 |
| 0 | * | 1 | - | 首次导航，目标行较短 | 当前列 | 目标行尾 |
| 1 | 0 | 0 | 0 | 被截断后，目标行能容纳 preferred | null | preferred |
| 1 | 0 | 0 | 1 | 被截断后，目标行仍不够长 | 保持 | 目标行尾 |
| 1 | 0 | 1 | - | 被截断后，目标行更短 | 保持 | 目标行尾 |
| 1 | 1 | 0 | - | 重包装后，目标行够长 | null | 当前列 |
| 1 | 1 | 1 | - | 重包装后，目标行较短 | 当前列 | 目标行尾 |

其中 P=preferredVisualCol是否设置, S=光标是否在行中(非行尾), T=目标行比当前列短, U=目标行比preferred短。

### 7.5 Emacs 风格的文本操作

#### Kill Ring (`kill-ring.ts`)

```typescript
export class KillRing {
    private ring: string[] = [];

    push(text: string, opts: { prepend: boolean; accumulate?: boolean }): void {
        if (opts.accumulate && this.ring.length > 0) {
            const last = this.ring.pop()!;
            this.ring.push(opts.prepend ? text + last : last + text);
        } else {
            this.ring.push(text);
        }
    }

    peek(): string | undefined;  // 查看最近条目
    rotate(): void;              // 旋转环（yank-pop 用）
}
```

**连续 Kill 累积**：如果连续执行多次 kill 操作（如 Ctrl+K Ctrl+K Ctrl+K），文本会累积到同一个 ring 条目中。方向敏感——向后删除（Ctrl+U）prepend，向前删除（Ctrl+K）append。

**Yank Pop 循环**：
```
Ring: ["hello", "world", "foo"]
                                  ↑ peek = "foo"
Ctrl+Y → 粘贴 "foo"
Alt+Y  → 删除 "foo"，rotate，粘贴 "hello"
Alt+Y  → 删除 "hello"，rotate，粘贴 "world"
Alt+Y  → 删除 "world"，rotate，粘贴 "foo"
```

#### Undo Stack (`undo-stack.ts`)

```typescript
export class UndoStack<S> {
    push(state: S): void {
        this.stack.push(structuredClone(state));  // 深拷贝！
    }
    pop(): S | undefined {
        return this.stack.pop();  // 已是独立快照，不需要再拷贝
    }
}
```

**Undo 合并**：连续输入字符时不会每个字符都创建快照。策略是：
- 输入非空白字符：如果上一操作是 `type-word`，不创建新快照（合并）
- 输入空白字符：创建新快照（空格切割 undo 单元）
- 其他操作（删除、粘贴等）：总是创建新快照

### 7.6 Grapheme-aware 操作

所有光标移动和删除操作都使用 `Intl.Segmenter` 而非简单的 `string[i]`：

```typescript
// 向左移动一个 grapheme
const beforeCursor = this.value.slice(0, this.cursor);
const graphemes = [...segmenter.segment(beforeCursor)];
const lastGrapheme = graphemes[graphemes.length - 1];
this.cursor -= lastGrapheme ? lastGrapheme.segment.length : 1;
```

这正确处理了：
- **Emoji**：😀 = 2 个 code unit，1 个 grapheme
- **组合 Emoji**：👨‍👩‍👧 = 5 个 code point，1 个 grapheme
- **CJK 字符**：占 2 列宽度
- **组合字符**：é = e + ́（2 个 code point，1 个 grapheme）

### 7.7 Word 移动算法

```typescript
private moveWordBackwards(): void {
    // 1. 跳过尾部空白
    while (isWhitespaceChar(lastGrapheme)) { ... }

    // 2. 判断第一个非空白字符
    if (isPunctuationChar(lastGrapheme)) {
        // 跳过标点符号连续序列
        while (isPunctuationChar(...)) { ... }
    } else {
        // 跳过单词字符连续序列
        while (!isWhitespace && !isPunctuation) { ... }
    }
}
```

将字符分为三类：空白、标点、单词字符。每次移动跳过一类连续序列。

### 7.8 自动补全集成

编辑器内建自动补全支持，触发条件：
- 输入 `/` 在行首 → 触发斜杠命令补全
- 输入 `@` → 触发文件引用（模糊搜索）补全
- Tab 键 → 强制触发文件路径补全

```
Editor
  │
  ├─ insertCharacter('/')
  │    │
  │    └─ 检测到行首 / → tryTriggerAutocomplete()
  │         │
  │         └─ autocompleteProvider.getSuggestions(lines, cursorLine, cursorCol)
  │              │
  │              └─ 返回 { items, prefix }
  │                   │
  │                   └─ 创建 SelectList，显示在编辑器下方
  │
  ├─ 用户继续输入字母
  │    │
  │    └─ updateAutocomplete() → 更新过滤
  │
  ├─ Tab / Enter → 应用选中项
  │    │
  │    └─ autocompleteProvider.applyCompletion(lines, cursor, item, prefix)
  │         │
  │         └─ 返回新的 lines 和光标位置
  │
  └─ Escape → cancelAutocomplete()
```

### 7.9 历史记录

```typescript
private history: string[] = [];    // 最近在前
private historyIndex: number = -1; // -1 = 当前输入

// Up 键：在编辑器为空或第一视觉行时触发
if (this.isEditorEmpty()) {
    this.navigateHistory(-1);  // 浏览更早的历史
}
```

---

## 八、单行输入框 (`src/components/input.ts`)

Input 是 Editor 的简化版——单行、水平滚动：

```typescript
export class Input implements Component, Focusable {
    private value: string = "";
    private cursor: number = 0;

    render(width: number): string[] {
        const prompt = "> ";
        const availableWidth = width - prompt.length;

        // 水平滚动：当内容超宽时
        if (totalWidth >= availableWidth) {
            // 计算可视窗口（光标居中）
            const halfWidth = Math.floor(scrollWidth / 2);
            const startCol = cursorCol < halfWidth ? 0
                : cursorCol > totalWidth - halfWidth ? totalWidth - scrollWidth
                : cursorCol - halfWidth;

            visibleText = sliceByColumn(this.value, startCol, scrollWidth, true);
        }

        // 用反色表示光标
        const cursorChar = `\x1b[7m${atCursor}\x1b[0m`;
        return [prompt + beforeCursor + marker + cursorChar + afterCursor + padding];
    }
}
```

输入框完整支持：Emacs 按键绑定、Kill Ring、Undo、Bracketed Paste、Kitty CSI-u、Grapheme-aware 操作。

---

## 九、Markdown 渲染器 (`src/components/markdown.ts`)

### 9.1 架构

```
Markdown 文本
    │
    ├─ marked.lexer(text)  → Token 数组
    │
    ├─ renderToken(token, width)  → string[]
    │    ├─ heading → 加粗/下划线 + 主题色
    │    ├─ paragraph → renderInlineTokens()
    │    ├─ code → 代码块边框 + highlightCode
    │    ├─ list → 递归嵌套 + 缩进
    │    ├─ table → 自适应列宽 + Unicode 边框
    │    ├─ blockquote → "│ " 前缀 + 斜体
    │    ├─ hr → "─" 重复
    │    └─ html → 原文输出
    │
    ├─ wrapTextWithAnsi(line, contentWidth)
    │
    └─ 添加 padding 和背景色
```

### 9.2 主题系统

```typescript
export interface MarkdownTheme {
    heading: (text: string) => string;
    link: (text: string) => string;
    code: (text: string) => string;
    codeBlock: (text: string) => string;
    codeBlockBorder: (text: string) => string;
    quote: (text: string) => string;
    quoteBorder: (text: string) => string;
    bold: (text: string) => string;
    italic: (text: string) => string;
    highlightCode?: (code: string, lang?: string) => string[];
}
```

主题函数接收纯文本，返回带 ANSI 转义码的文本。具体的颜色由调用方（pi 应用层）提供。

### 9.3 表格渲染

表格渲染是 Markdown 组件中最复杂的部分：

1. **计算自然列宽**：遍历所有单元格，取每列的最大宽度
2. **最小列宽**：每列最长单词的宽度（或 `maxUnbrokenWordWidth=30`）
3. **列宽分配**：
   - 总宽度够 → 使用自然列宽
   - 总宽度不够 → 从最小列宽开始，按自然列宽比例分配额外空间
   - 极端情况 → 回退到 raw markdown 输出
4. **单元格换行**：使用 `wrapTextWithAnsi()` 将长内容换行，行数不一致的单元格用空行填充
5. **Unicode 边框**：使用 `┌┬┐├┼┤└┴┘─│` 绘制完整的表格框架

### 9.4 InlineStyleContext

内联样式上下文解决了 ANSI 转义码嵌套的问题：

```typescript
interface InlineStyleContext {
    applyText: (text: string) => string;  // 当前上下文的样式函数
    stylePrefix: string;                  // 用于 reset 后恢复的 ANSI 前缀
}
```

问题：当代码块（`\x1b[36m`蓝色）出现在标题（`\x1b[1m`粗体）中时，代码块的 `\x1b[0m`（全局 reset）会清除标题的粗体。

解决方案：内联元素渲染后追加 `stylePrefix`：
```typescript
case "codespan":
    result += this.theme.code(token.text) + stylePrefix;  // 恢复父级样式
    break;
```

### 9.5 缓存机制

```typescript
private cachedText?: string;
private cachedWidth?: number;
private cachedLines?: string[];

render(width: number): string[] {
    if (this.cachedLines && this.cachedText === this.text && this.cachedWidth === width) {
        return this.cachedLines;  // 文本和宽度未变，直接返回缓存
    }
    // ... 重新渲染
}
```

---

## 十、选择列表 (`src/components/select-list.ts`)

```typescript
export class SelectList implements Component {
    private items: SelectItem[];
    private filteredItems: SelectItem[];
    private selectedIndex: number = 0;
    private maxVisible: number = 5;

    render(width: number): string[] {
        // 1. 计算可视范围（滚动窗口居中于 selectedIndex）
        const startIndex = Math.max(0,
            Math.min(selectedIndex - Math.floor(maxVisible / 2),
                     filteredItems.length - maxVisible));

        // 2. 渲染可见项（选中项加 "→ " 前缀和高亮）
        for (let i = startIndex; i < endIndex; i++) {
            const isSelected = i === this.selectedIndex;
            lines.push(this.renderItem(item, isSelected, width, ...));
        }

        // 3. 滚动指示器 "(3/15)"
        if (startIndex > 0 || endIndex < filteredItems.length) {
            lines.push(theme.scrollInfo(`  (${selectedIndex + 1}/${filteredItems.length})`));
        }
    }
}
```

**两列布局**：左列为项名称，右列为描述。列宽自适应——扫描所有项的名称宽度，计算出 `primaryColumnWidth`。

**循环选择**：Up 在第一项时跳到最后一项，Down 在最后一项时跳到第一项。

---

## 十一、自动补全系统 (`src/autocomplete.ts`)

### 11.1 Provider 接口

```typescript
export interface AutocompleteProvider {
    getSuggestions(lines, cursorLine, cursorCol): {
        items: AutocompleteItem[];
        prefix: string;
    } | null;

    applyCompletion(lines, cursorLine, cursorCol, item, prefix): {
        lines: string[];
        cursorLine: number;
        cursorCol: number;
    };
}
```

### 11.2 CombinedAutocompleteProvider

统一处理三种补全：

1. **斜杠命令** (`/help`, `/model`, `/clear`)
   - 触发条件：行首 `/`
   - 匹配方式：`fuzzyFilter` 模糊匹配命令名
   - 补全后自动追加空格

2. **文件引用** (`@src/index.ts`)
   - 触发条件：`@` 后跟字符
   - 匹配方式：调用 `fd` 命令进行模糊文件搜索
   - 支持引号包裹（含空格的路径）：`@"path with spaces/file.ts"`

3. **文件路径** (Tab 触发)
   - 触发条件：Tab 键 或 路径字符（`/`, `./`, `~/`）
   - 匹配方式：`readdirSync` 列出目录内容
   - 自动展开 `~` 为 home 目录

### 11.3 fd 集成

```typescript
function walkDirectoryWithFd(baseDir, fdPath, query, maxResults) {
    const args = [
        "--base-directory", baseDir,
        "--max-results", String(maxResults),
        "--type", "f",  // 文件
        "--type", "d",  // 目录
        "--full-path",
        "--hidden",
        "--exclude", ".git",
    ];

    if (query) args.push(buildFdPathQuery(query));

    const result = spawnSync(fdPath, args, {
        encoding: "utf-8",
        maxBuffer: 10 * 1024 * 1024,
    });
}
```

`fd` 是一个快速的文件查找工具（尊重 `.gitignore`），比 Node.js 的 `fs.readdir` 递归遍历快得多。当 `fd` 不可用时，回退到 `readdirSync` 单层目录列表。

---

## 十二、模糊匹配 (`src/fuzzy.ts`)

### 12.1 评分算法

```typescript
export function fuzzyMatch(query: string, text: string): FuzzyMatch {
    // 遍历 text，依次匹配 query 中的每个字符
    for (let i = 0; i < textLower.length && queryIndex < normalizedQuery.length; i++) {
        if (textLower[i] === normalizedQuery[queryIndex]) {
            // 连续匹配奖励：-5 × 连续数
            if (lastMatchIndex === i - 1) {
                consecutiveMatches++;
                score -= consecutiveMatches * 5;
            } else {
                // 间隔惩罚：+2 × 间隔距离
                score += (i - lastMatchIndex - 1) * 2;
            }

            // 词边界奖励：-10
            if (i === 0 || /[\s\-_./:]/.test(textLower[i - 1])) {
                score -= 10;
            }

            // 位置惩罚：+0.1 × 位置
            score += i * 0.1;
        }
    }
}
```

**分数越低越好**。设计权衡：
- 连续匹配 > 分散匹配（`-5` 每次连续）
- 词边界匹配 > 中间匹配（`-10`）
- 靠前匹配 > 靠后匹配（`+0.1 × i`）
- 间隔小 > 间隔大（`+2 × gap`）

### 12.2 数字字母交换

```typescript
// 如果 "abc123" 不匹配，尝试 "123abc"（反向组合）
const alphaNumericMatch = queryLower.match(/^(?<letters>[a-z]+)(?<digits>[0-9]+)$/);
const swappedQuery = `${digits}${letters}`;
```

这处理了用户输入 `g3` 但实际应匹配 `3g` 的情况（或反过来）。交换匹配的分数加 5 的惩罚。

### 12.3 多 Token 搜索

```typescript
export function fuzzyFilter<T>(items: T[], query: string, getText: (item: T) => string): T[] {
    const tokens = query.trim().split(/\s+/);

    for (const item of items) {
        let allMatch = true;
        for (const token of tokens) {
            const match = fuzzyMatch(token, getText(item));
            if (!match.matches) { allMatch = false; break; }
            totalScore += match.score;
        }
        if (allMatch) results.push({ item, totalScore });
    }

    results.sort((a, b) => a.totalScore - b.totalScore);  // 分数越低越好
}
```

空格分隔的多个 token **全部**必须匹配（AND 语义）。

---

## 十三、容器盒子 (`src/components/box.ts`)

```typescript
export class Box implements Component {
    children: Component[] = [];
    private paddingX: number;
    private paddingY: number;
    private bgFn?: (text: string) => string;

    render(width: number): string[] {
        const contentWidth = width - paddingX * 2;

        // 渲染所有子组件
        for (const child of this.children) {
            childLines.push(leftPad + ...child.render(contentWidth));
        }

        // 顶部 padding → 内容 → 底部 padding
        // 每行应用背景色（扩展到全宽）
    }
}
```

Box 带有渲染缓存——如果子组件输出、宽度、背景色都没变，直接返回缓存结果。

---

## 十四、工具函数 (`src/utils.ts`)

### 14.1 visibleWidth

```typescript
import stringWidth from "string-width";

export function visibleWidth(text: string): number {
    // 剥离 ANSI 转义码后计算可见宽度
    // CJK 字符占 2 列，Emoji 占 2 列，普通字符占 1 列
    return stringWidth(stripAnsi(text));
}
```

### 14.2 wrapTextWithAnsi

```typescript
export function wrapTextWithAnsi(text: string, maxWidth: number): string[] {
    // 1. 解析 ANSI 转义码和可见字符
    // 2. 在单词边界处换行（优先）
    // 3. 无词边界则强制在列边界断行
    // 4. 跨行传播 ANSI 状态（确保下一行继承前一行的颜色/样式）
}
```

这是 Markdown 渲染器和 Box 组件的底层依赖——正确处理包含 ANSI 转义码的文本换行。

### 14.3 applyBackgroundToLine

```typescript
export function applyBackgroundToLine(
    line: string, width: number, bgFn: (text: string) => string
): string {
    // 将背景色应用到整行（包括空白填充区域）
    // 处理已有 ANSI 转义码的文本——在每个 \x1b[0m (reset) 后重新应用背景色
}
```

### 14.4 sliceByColumn

```typescript
export function sliceByColumn(text: string, startCol: number, maxCols: number, pad?: boolean): string {
    // 按可见列（而非字符偏移）切片
    // 正确处理 CJK 双宽字符：如果切片位置落在双宽字符中间，用空格替代
}
```

### 14.5 Intl.Segmenter

```typescript
export function getSegmenter(): Intl.Segmenter {
    return new Intl.Segmenter(undefined, { granularity: "grapheme" });
}
```

所有文本操作的基础——将字符串按 grapheme cluster（书写系统中的最小视觉单元）分割。

---

## 十五、EditorComponent 接口 (`src/editor-component.ts`)

```typescript
export interface EditorComponent extends Component {
    // 必需
    getText(): string;
    setText(text: string): void;
    handleInput(data: string): void;
    onSubmit?: (text: string) => void;
    onChange?: (text: string) => void;

    // 可选
    addToHistory?(text: string): void;
    insertTextAtCursor?(text: string): void;
    getExpandedText?(): string;
    setAutocompleteProvider?(provider: AutocompleteProvider): void;
    borderColor?: (str: string) => string;
    setPaddingX?(padding: number): void;
    setAutocompleteMaxVisible?(maxVisible: number): void;
}
```

这是一个扩展点——允许外部提供自定义编辑器实现（如 Vim 模式）。内置的 `Editor` 类满足此接口的所有方法。

---

## 十六、终端图片支持 (`src/terminal-image.ts`)

### 16.1 协议检测

```
KITTY_WINDOW_ID 或 TERM_PROGRAM=kitty  → Kitty Graphics Protocol
GHOSTTY_RESOURCES_DIR                    → Kitty Graphics Protocol
TERM_PROGRAM=WezTerm                     → iTerm2 Inline Images
TERM_PROGRAM=iTerm.app                   → iTerm2 Inline Images
TERM_PROGRAM=vscode                      → 无图片支持
其他                                     → 无图片支持
```

### 16.2 像素级布局

```typescript
export interface CellDimensions {
    widthPx: number;   // 单个字符单元的像素宽度
    heightPx: number;  // 单个字符单元的像素高度
}
```

TUI 通过 CSI 14t 查询终端像素尺寸，结合 `rows × cols` 计算每个字符单元的像素大小，用于精确控制图片渲染尺寸。

---

## 十七、关键设计模式总结

### 17.1 行级渲染

```
Component.render(width) → string[]
```

所有组件只需返回行数组，不需要理解终端光标、差分、同步输出等底层细节。这是极其简单的抽象。

### 17.2 差分更新

```
Frame N:    ["line1", "line2", "line3"]
Frame N+1:  ["line1", "LINE2", "line3"]
                       ^^^^^^
                    只重绘这一行
```

### 17.3 栈式焦点

```
pushFocus(Editor)    → [Editor*]
pushFocus(Dialog)    → [Editor, Dialog*]    (*=focused)
popFocus(Dialog)     → [Editor*]            (自动恢复)
```

### 17.4 逻辑坐标 + 视觉映射

```
逻辑行: ["Hello World, this is a very long line"]
         cursorCol = 25

视觉行: ["Hello World, this is a ", "very long line"]
         cursorPos 映射到第 2 行的列 2
```

编辑器始终用逻辑坐标操作文本，只在渲染时通过 `layoutText()` + `wordWrapLine()` 转换为视觉坐标。

### 17.5 Grapheme-aware 一切

不用 `string[i]`，不用 `string.length`（做光标偏移），而是：
```typescript
const graphemes = [...segmenter.segment(text)];
const lastGrapheme = graphemes[graphemes.length - 1];
cursor -= lastGrapheme.segment.length;
```

---

## 十八、核心数据流图

```
用户按键
  │
  ├─ stdin
  │    │
  │    └─ StdinBuffer.process(data)
  │         │
  │         ├─ isCompleteSequence() → 序列完整性判断
  │         │
  │         ├─ extractCompleteSequences() → 切分完整序列
  │         │
  │         └─ emit("data", sequence)
  │              │
  │              └─ TUI.handleInput(sequence)
  │                   │
  │                   └─ focusStack[top].handleInput(sequence)
  │                        │
  │                        ├─ Editor.handleInput(data)
  │                        │    │
  │                        │    ├─ getKeybindings().matches(data, "tui.editor.cursorUp")
  │                        │    │    └─ matchesKey(data, "up")
  │                        │    │         └─ parseKeySequence(data) → 解析修饰符+键名
  │                        │    │
  │                        │    ├─ 状态修改（lines, cursorLine, cursorCol）
  │                        │    │
  │                        │    └─ tui.requestRender()
  │                        │
  │                        └─ queueMicrotask → TUI.render()
  │                              │
  │                              ├─ 收集组件输出
  │                              │    ├─ Editor.render(width) → layoutText + wordWrap
  │                              │    ├─ Markdown.render(width) → marked.lexer + renderToken
  │                              │    └─ Box.render(width) → children.render + padding + bg
  │                              │
  │                              ├─ 差分对比 previousLines
  │                              │
  │                              ├─ CSI 2026 包裹
  │                              │
  │                              ├─ 只写入变化行
  │                              │
  │                              └─ CURSOR_MARKER → 定位硬件光标
```

---

## 十九、源码阅读建议

### 推荐阅读顺序

1. **`tui.ts`**：理解 Component/Focusable 接口和渲染管线（整体架构）
2. **`terminal.ts`**：理解终端原始模式和 ANSI 转义码
3. **`keys.ts`**：理解按键编码格式（CSI/SS3/Kitty/Meta）
4. **`keybindings.ts`**：理解可配置快捷键系统
5. **`stdin-buffer.ts`**：理解输入缓冲和序列完整性判断
6. **`utils.ts`**：理解 ANSI 文本处理工具函数
7. **`components/input.ts`**：先读简单的单行输入框（Editor 的简化版）
8. **`kill-ring.ts` + `undo-stack.ts`**：理解 Emacs 操作模型
9. **`components/editor.ts`**：精读多行编辑器（最大最复杂的组件）
10. **`components/markdown.ts`**：理解 Token → ANSI 的转换
11. **`autocomplete.ts` + `fuzzy.ts`**：理解自动补全系统

### 调试技巧

```bash
# 1. 使用 tmux 进行隔离调试
tmux new-session -d -s test -x 80 -y 24
tmux send-keys -t test "cd /path/to/pi-mono && node --inspect ..." Enter

# 2. 查看原始输入序列
# 在 StdinBuffer.process() 开头加日志：
#   console.log("raw:", JSON.stringify(data));

# 3. 查看渲染输出
# 在 TUI.render() 结束前加日志：
#   console.log("lines:", lines.length, "changed:", changedLines.length);
```

### 关键断点位置

- `tui.ts` — `render()` 方法：渲染管线入口
- `tui.ts` — `handleInput()` 方法：输入处理入口
- `stdin-buffer.ts` — `process()` 方法：输入缓冲
- `keys.ts` — `matchesKey()` 函数：按键匹配
- `editor.ts` — `handleInput()` 方法：编辑器按键分发
- `editor.ts` — `layoutText()` 方法：文本布局（逻辑→视觉映射）
- `editor.ts` — `wordWrapLine()` 函数：自动换行算法
