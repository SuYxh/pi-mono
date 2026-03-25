# Pi Monorepo 源码学习路线与规划

> 本文档为 Pi Monorepo 源码的系统性学习指南，从底层基础到上层应用，由浅入深，每个阶段都标注了具体的文件路径和学习目标。

---

## 目录

- [学习原则](#学习原则)
- [前置准备](#前置准备)
- [整体架构鸟瞰](#整体架构鸟瞰)
- [Phase 1：基础设施层 — pi-ai（统一 LLM API）](#phase-1基础设施层--pi-ai统一-llm-api)
- [Phase 2：基础设施层 — pi-tui（终端 UI 框架）](#phase-2基础设施层--pi-tui终端-ui-框架)
- [Phase 3：运行时层 — pi-agent-core（Agent 运行时）](#phase-3运行时层--pi-agent-coreagent-运行时)
- [Phase 4：应用层 — pi-coding-agent（交互式编码 Agent）](#phase-4应用层--pi-coding-agent交互式编码-agent)
- [Phase 5：应用层 — pi-mom（Slack Bot）](#phase-5应用层--pi-momslack-bot)
- [Phase 6：运行时层 — pi-web-ui（Web 组件库）](#phase-6运行时层--pi-web-uiweb-组件库)
- [Phase 7：部署层 — pi-pods（GPU 部署管理）](#phase-7部署层--pi-podsgpu-部署管理)
- [Phase 8：综合实践](#phase-8综合实践)
- [附录：核心设计模式速查](#附录核心设计模式速查)

---

## 学习原则

1. **自底向上**：先学基础包（ai, tui），再学依赖它们的上层包（agent, coding-agent, web-ui）
2. **类型先行**：每个包先读 `types.ts`，理解核心数据结构，再读实现
3. **测试驱动理解**：每个阶段都配合读对应的测试文件，测试是最好的"可执行文档"
4. **小步验证**：每学完一个模块，尝试写一段最小可运行代码验证理解
5. **画图辅助**：遇到复杂流程（如 Agent Loop、差分渲染）时画流程图

---

## 前置准备

### 环境搭建

```bash
# 克隆并安装
cd /Users/bytedance/Desktop/ai-study/pi-mono
npm install
npm run build

# 验证构建
npm run check
```

### 需要的前置知识

| 领域 | 具体内容 | 重要程度 |
|------|---------|---------|
| TypeScript | 泛型、条件类型、declaration merging、ESM 模块 | 必须 |
| Node.js | Streams、EventEmitter、child_process、AbortController | 必须 |
| LLM 基础 | 什么是 token、context window、tool calling (function calling)、streaming | 必须 |
| 终端知识 | ANSI 转义码、CSI 序列、终端输入/输出模型 | Phase 2 需要 |
| Web Components | Custom Elements、Shadow DOM | Phase 6 需要 |
| TypeBox | JSON Schema 的类型安全构建库 | 建议提前了解 |

### 推荐的阅读顺序文档

在开始源码之前，建议先阅读以下文档：

1. `README.md` — 项目总览
2. `CONTRIBUTING.md` — 贡献指南和哲学
3. `AGENTS.md` — 开发规范（对人和 AI Agent 同时生效）
4. `packages/ai/README.md` — LLM API 使用指南
5. `packages/agent/README.md` — Agent 运行时使用指南
6. `packages/coding-agent/README.md` — 编码 Agent 使用指南

---

## 整体架构鸟瞰

```
┌─────────────────────────────────────────────────────────────────────┐
│                          应用层 (Applications)                       │
│  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐       │
│  │  coding-agent  │   │      mom       │   │     pods       │       │
│  │  (终端编码助手) │   │  (Slack Bot)   │   │ (GPU 部署管理)  │       │
│  └───────┬────────┘   └───────┬────────┘   └────────────────┘       │
│          │                    │                                      │
├──────────┼────────────────────┼──────────────────────────────────────┤
│          │      运行时层 (Runtime)                                    │
│  ┌───────┴────────┐   ┌──────┴─────────┐                            │
│  │   agent-core   │   │    web-ui      │                            │
│  │  (Agent 循环)   │   │  (Web 组件库)  │                            │
│  └───────┬────────┘   └──────┬─────────┘                            │
│          │                   │                                       │
├──────────┼───────────────────┼───────────────────────────────────────┤
│          │      基础设施层 (Foundation)                                │
│  ┌───────┴────────┐   ┌─────┴──────────┐                            │
│  │       ai       │   │      tui       │                            │
│  │  (LLM 统一 API) │   │ (终端 UI 框架)  │                            │
│  └────────────────┘   └────────────────┘                            │
└─────────────────────────────────────────────────────────────────────┘

依赖关系（→ 表示 "依赖"）：
  coding-agent → agent-core → ai
  coding-agent → tui
  mom          → agent-core → ai
  web-ui       → agent-core → ai
  pods         → (独立，仅使用 OpenAI 兼容 API)
```

**学习路线总图**：

```
Phase 1: ai (types → models → api-registry → stream → providers)
    ↓
Phase 2: tui (utils → terminal → keys → tui → components)
    ↓
Phase 3: agent-core (types → agent-loop → agent)
    ↓
Phase 4: coding-agent (core → tools → modes → extensions)
    ↓
Phase 5: mom (store → context → agent → slack → tools)
    ↓
Phase 6: web-ui (storage → components → tools → artifacts)
    ↓
Phase 7: pods (独立学习)
    ↓
Phase 8: 综合实践
```

---

## Phase 1：基础设施层 — pi-ai（统一 LLM API）

> **目标**：理解如何将 20+ 个 LLM Provider 抽象为统一的流式 API
>
> **预计时间**：3-5 天
>
> **包路径**：`packages/ai/`

### Step 1.1：核心类型系统

**文件**：`packages/ai/src/types.ts`

**学习目标**：
- 理解 `KnownApi` 联合类型 — 每种 API 协议的标识符（如 `"openai-completions"`, `"anthropic-messages"`, `"google-generative-ai"`）
- 理解 `KnownProvider` 联合类型 — 每个 Provider 的标识符（如 `"openai"`, `"anthropic"`, `"google"`）
- 区分 `Api` vs `Provider`：一个 Provider 可能使用不同的 Api（如 OpenAI 有 completions 和 responses 两种 API）
- 理解 `StreamOptions` 接口 — 所有 Provider 共享的流式请求选项
- 理解 `SimpleStreamOptions` — 简化版选项（供 agent 层使用）
- 理解 `Model<TApi>` 泛型 — 模型元数据，包含 provider、api、定价、context window 等
- 理解 `Context` 类型 — 对话上下文（systemPrompt + messages + tools）
- 理解 `Message` 联合类型 — user / assistant / toolResult 三种消息角色
- 理解 `AssistantMessage` — 助手回复的完整结构（content 数组 + usage + stopReason）
- 理解 `ThinkingLevel` — 思考等级（minimal/low/medium/high/xhigh）

**关键思考题**：
1. 为什么需要同时有 `Api` 和 `Provider` 两个维度？给出一个 Provider 对应多个 Api 的例子。
2. `StreamOptions` 和 `SimpleStreamOptions` 的区别是什么？为什么需要两层抽象？
3. `AssistantMessage.content` 为什么是一个数组？里面可以包含哪些类型的 content block？

### Step 1.2：模型注册表

**文件**：
- `packages/ai/src/models.ts` — 模型注册表核心逻辑
- `packages/ai/src/models.generated.ts` — 自动生成的模型数据
- `packages/ai/scripts/generate-models.ts` — 模型数据生成脚本

**学习目标**：
- 理解 `modelRegistry: Map<string, Map<string, Model>>` 的二层 Map 结构（provider → modelId → Model）
- 理解 `getModel(provider, modelId)` 的类型推断机制
- 理解 `getProviders()` 和 `getModels(provider)` 的查询方式
- 理解 `calculateCost()` 的计费计算逻辑
- 了解 `models.generated.ts` 是如何由脚本自动生成的

**关键思考题**：
1. `getModel('openai', 'gpt-4o')` 返回的类型是什么？为什么可以实现 Provider 和 Model 的自动补全？
2. 为什么模型数据要自动生成而不是手写？

### Step 1.3：API Provider 注册表

**文件**：`packages/ai/src/api-registry.ts`

**学习目标**：
- 理解 `ApiProvider<TApi, TOptions>` 接口 — 每个 Provider 的实现契约
- 理解 `registerApiProvider()` / `getApiProvider()` — Provider 的注册与查找
- 理解 `unregisterApiProviders(sourceId)` — 按来源批量注销（用于扩展卸载）
- 理解 `wrapStream()` / `wrapStreamSimple()` 辅助函数 — 类型安全的 Provider 包装

**关键思考题**：
1. `ApiProvider` 接口要求 Provider 实现哪些方法？
2. 为什么需要 `sourceId` 机制？什么场景下需要批量注销 Provider？

### Step 1.4：统一流式调用入口

**文件**：`packages/ai/src/stream.ts`

**学习目标**：
- 理解 `stream()` 函数的完整调用链：`stream() → resolveApiProvider(model.api) → provider.stream()`
- 理解 `complete()` 如何封装 `stream()` 返回完整结果
- 理解 `streamSimple()` / `completeSimple()` 的简化调用路径
- 理解第一行的 `import "./providers/register-builtins.js"` 的副作用导入机制

**关键思考题**：
1. 当你调用 `stream(model, context, options)` 时，内部发生了什么？画一个调用链图。
2. 为什么 `register-builtins.js` 是通过副作用导入的？如果不导入会怎样？

### Step 1.5：Provider 实现（选读 2-3 个）

**文件**（建议按此顺序阅读）：
1. `packages/ai/src/providers/register-builtins.ts` — **必读**，理解懒加载注册机制
2. `packages/ai/src/providers/openai-completions.ts` — OpenAI 实现（最经典）
3. `packages/ai/src/providers/anthropic.ts` — Anthropic 实现（对比学习）
4. `packages/ai/src/providers/transform-messages.ts` — 消息格式转换（跨 Provider 核心）
5. `packages/ai/src/providers/simple-options.ts` — SimpleStreamOptions 到具体 Options 的映射

**学习目标**：
- 理解 Provider 的懒加载注册模式：`registerApiProvider({ api: "xxx", stream: async () => (await import("./xxx.js")).streamXxx, ... })`
- 理解一个完整 Provider 的实现结构：消息转换 → HTTP 请求 → SSE 流解析 → 事件发射
- 理解 `transform-messages.ts` 如何将统一的内部消息格式转换为各 Provider 的特定格式
- 对比 OpenAI 和 Anthropic 的 API 差异，理解统一抽象的价值

**关键思考题**：
1. 懒加载注册有什么好处？如果所有 Provider 都在启动时加载会有什么问题？
2. OpenAI 和 Anthropic 的流式 API 格式有哪些关键差异？Pi 是如何抹平这些差异的？

### Step 1.6：工具函数与事件流

**文件**：
- `packages/ai/src/utils/event-stream.ts` — 事件流实现
- `packages/ai/src/utils/json-parse.ts` — 流式 JSON 解析
- `packages/ai/src/utils/overflow.ts` — 上下文溢出处理
- `packages/ai/src/utils/validation.ts` — Tool 参数校验

**学习目标**：
- 理解 `AssistantMessageEventStream` 的实现 — 这是整个库最核心的类型
- 理解流式 JSON 解析如何处理 tool_call 的增量参数
- 理解上下文溢出检测逻辑

### Step 1.7：测试文件（选读）

**文件**：
- `packages/ai/test/stream.test.ts` — 核心流式测试
- `packages/ai/test/tokens.test.ts` — Token 计算测试
- `packages/ai/test/cross-provider-handoff.test.ts` — 跨 Provider 交接测试
- `packages/ai/test/tool-call-without-result.test.ts` — 工具调用边界情况

**学习目标**：
- 通过测试理解 API 的实际使用方式
- 理解跨 Provider 交接的工作原理

### Step 1.8：阶段验证

尝试写一段代码，使用 `pi-ai` 完成一次带工具调用的对话：

```typescript
import { Type, getModel, stream, Context, Tool } from '@mariozechner/pi-ai';

// 1. 选择模型
// 2. 定义一个简单的 Tool
// 3. 构建 Context
// 4. 调用 stream() 并消费事件流
// 5. 处理 tool_call 事件
```

---

## Phase 2：基础设施层 — pi-tui（终端 UI 框架）

> **目标**：理解如何从零构建一个高性能的终端 UI 渲染引擎
>
> **预计时间**：2-3 天
>
> **包路径**：`packages/tui/`

### Step 2.1：工具函数

**文件**：`packages/tui/src/utils.ts`

**学习目标**：
- 理解 `visibleWidth()` — 如何计算含 ANSI 转义码的字符串的可见宽度（需处理东亚宽字符、零宽字符等）
- 理解 `truncateToWidth()` — 如何安全截断含 ANSI 码的字符串而不破坏格式
- 理解 `wrapTextWithAnsi()` — 如何在换行时保持 ANSI 样式的连续性

**关键思考题**：
1. 为什么不能简单用 `string.length` 来判断终端中字符串的显示宽度？
2. 截断一个含 ANSI 转义码的字符串时，最需要注意什么？

### Step 2.2：终端抽象

**文件**：`packages/tui/src/terminal.ts`

**学习目标**：
- 理解 `Terminal` 接口的设计 — `start()`, `stop()`, `write()`, `columns`, `rows`, `moveBy()`, `hideCursor()`, `clearScreen()` 等
- 理解 `ProcessTerminal` 实现 — 如何基于 `process.stdin` / `process.stdout` 实现终端操作
- 理解终端 raw mode 的概念

### Step 2.3：键盘输入处理

**文件**：
- `packages/tui/src/keys.ts` — 键盘输入解析
- `packages/tui/src/keybindings.ts` — 快捷键绑定系统
- `packages/tui/src/stdin-buffer.ts` — 输入缓冲

**学习目标**：
- 理解终端输入的底层机制：raw mode 下收到的是原始字节序列（ANSI 转义序列）
- 理解 `Key` 对象和 `matchesKey()` 的匹配机制
- 理解 Kitty Keyboard Protocol 的支持
- 理解 `StdinBuffer` 如何批量处理输入事件

### Step 2.4：TUI 核心渲染引擎

**文件**：`packages/tui/src/tui.ts`

**学习目标**：
- **这是 tui 包最核心的文件**
- 理解 `Component` 接口 — `render(width: number): string[]`
- 理解 `Focusable` 接口 — IME 支持的关键
- 理解 `CURSOR_MARKER` 机制 — 如何在组件中标记硬件光标位置
- 理解 `TUI` 类的三策略差分渲染：
  1. 首次渲染：直接输出所有行
  2. 宽度变化或视口上方有变化：清屏全量重绘
  3. 正常更新：定位到第一个变化行，只更新变化部分
- 理解 `Container` 类 — 组件的组合模式
- 理解 Overlay 系统 — 浮层的定位、聚焦、层叠管理
- 理解同步输出 `CSI 2026` 的工作原理

**关键思考题**：
1. 差分渲染相比全量重绘有什么优势？为什么需要三种策略？
2. 同步输出 (`\x1b[?2026h` ... `\x1b[?2026l`) 解决了什么问题？
3. Overlay 的焦点管理是如何实现的？

### Step 2.5：内置组件（选读 3-4 个）

**文件**（建议顺序）：
1. `packages/tui/src/components/text.ts` — 最简单的组件，理解基本模式
2. `packages/tui/src/components/input.ts` — 单行输入，理解焦点和光标
3. `packages/tui/src/components/select-list.ts` — 列表选择，理解键盘交互
4. `packages/tui/src/components/editor.ts` — 多行编辑器，理解复杂组件
5. `packages/tui/src/components/markdown.ts` — Markdown 渲染，理解文本处理

**学习目标**：
- 理解组件的一般实现模式：状态 + render() + handleInput()
- 理解编辑器的 undo/redo（`undo-stack.ts`）和 kill ring（`kill-ring.ts`）
- 理解自动补全系统（`autocomplete.ts`）

### Step 2.6：终端图片支持

**文件**：
- `packages/tui/src/terminal-image.ts`
- `packages/tui/src/components/image.ts`

**学习目标**：
- 了解 Kitty Graphics Protocol 和 iTerm2 Inline Images Protocol
- 了解如何在终端中渲染图片

### Step 2.7：阶段验证

尝试用 `pi-tui` 创建一个简单的终端应用：

```typescript
import { TUI, Text, Editor, ProcessTerminal } from "@mariozechner/pi-tui";

// 1. 创建 ProcessTerminal
// 2. 创建 TUI
// 3. 添加 Text 和 Editor 组件
// 4. 处理 onSubmit
// 5. 启动 TUI
```

也可以运行自带的 demo：`npx tsx packages/tui/test/chat-simple.ts`

---

## Phase 3：运行时层 — pi-agent-core（Agent 运行时）

> **目标**：理解 Agent Loop 的核心机制——如何将 LLM 调用与 Tool 执行编排为自动化循环
>
> **预计时间**：2-3 天
>
> **包路径**：`packages/agent/`

### Step 3.1：核心类型

**文件**：`packages/agent/src/types.ts`

**学习目标**：
- 理解 `AgentMessage` 联合类型 — 比 LLM `Message` 更丰富，支持自定义角色
- 理解 `AgentContext` — 包含 systemPrompt、messages、tools
- 理解 `AgentTool` — 工具定义（name + description + parameters schema + execute 函数）
- 理解 `AgentEvent` 联合类型 — Agent 循环中的所有事件类型
- 理解 `AgentLoopConfig` — 循环配置
- 理解 `StreamFn` — 流函数签名（Agent 调用 LLM 的方式）
- 理解 `ToolExecutionMode` — "sequential" vs "parallel"
- 理解 `BeforeToolCallContext` / `AfterToolCallContext` — 工具调用的前后钩子

**关键思考题**：
1. `AgentMessage` 和 `Message`（来自 pi-ai）的关系是什么？为什么需要两层消息类型？
2. `convertToLlm` 函数的职责是什么？为什么它是必要的？
3. `beforeToolCall` 钩子可以用来做什么？给出 2-3 个使用场景。

### Step 3.2：Agent Loop 核心循环

**文件**：`packages/agent/src/agent-loop.ts`

**学习目标**：
- **这是整个 Agent 系统的核心**
- 理解 `agentLoop()` 的完整流程：

```
agentLoop(prompts, context, config)
│
├─ 发射 agent_start
├─ 将 prompts 添加到 context.messages
│
├─ 开始循环 ─────────────────────────────────────────┐
│   ├─ 发射 turn_start                               │
│   ├─ 发射 message_start (user message)             │
│   ├─ 发射 message_end (user message)               │
│   │                                                 │
│   ├─ transformContext() — 可选的上下文变换           │
│   ├─ convertToLlm() — AgentMessage[] → Message[]   │
│   │                                                 │
│   ├─ streamFn(model, context, options) — 调用 LLM   │
│   ├─ 发射 message_start (assistant)                 │
│   ├─ 发射 message_update (streaming chunks)         │
│   ├─ 发射 message_end (assistant)                   │
│   │                                                 │
│   ├─ 如果 assistant 有 tool_call：                   │
│   │   ├─ beforeToolCall() — 预检                    │
│   │   ├─ 发射 tool_execution_start                  │
│   │   ├─ tool.execute() — 执行工具                  │
│   │   ├─ afterToolCall() — 后处理                   │
│   │   ├─ 发射 tool_execution_end                    │
│   │   ├─ 发射 message_start/end (toolResult)        │
│   │   ├─ 发射 turn_end                              │
│   │   └─ 继续循环 ↑                                │
│   │                                                 │
│   └─ 如果无 tool_call：                             │
│       ├─ 发射 turn_end                              │
│       └─ 退出循环                                   │
│                                                     │
├─ 发射 agent_end                                     │
└─────────────────────────────────────────────────────┘
```

- 理解 `agentLoopContinue()` — 从现有上下文继续（用于重试）
- 理解 `EventStream` 包装 — 将异步循环变为可消费的事件流
- 理解并行 vs 顺序工具执行的实现

**关键思考题**：
1. Agent Loop 什么时候会退出循环？有哪些退出条件？
2. 并行工具执行时，如果某个工具失败了，其他工具会怎样？
3. `transformContext` 在循环中的哪个位置被调用？为什么是这个位置？

### Step 3.3：Agent 类

**文件**：`packages/agent/src/agent.ts`

**学习目标**：
- 理解 `Agent` 类如何封装 `agentLoop`
- 理解 `AgentState` — Agent 的完整状态（model, systemPrompt, messages, tools, isStreaming, streamMessage, pendingToolCalls, error）
- 理解事件发布/订阅模式（`subscribe()` / `listeners`）
- 理解 `prompt()` / `continue()` 的执行语义
- 理解 Steering（转向）和 Follow-up（跟进）消息机制
- 理解 `AbortController` 的取消机制
- 理解 `defaultConvertToLlm` — 默认的消息过滤函数

**关键思考题**：
1. `Agent` 类和 `agentLoop` 函数的关系是什么？各自适合什么场景？
2. Steering 消息和 Follow-up 消息的区别是什么？它们分别在什么时机被处理？
3. `Agent.state.streamMessage` 在流式传输时有什么作用？

### Step 3.4：Proxy 支持

**文件**：`packages/agent/src/proxy.ts`

**学习目标**：
- 理解 `streamProxy()` — 如何通过代理后端使用 Agent（浏览器环境需要）
- 理解代理模式的使用场景

### Step 3.5：测试文件

**文件**：
- `packages/agent/test/agent.test.ts` — Agent 类测试
- `packages/agent/test/agent-loop.test.ts` — Agent Loop 测试

**学习目标**：
- 通过测试理解 Agent 的事件序列
- 理解 mock 工具的编写方式
- 理解边界情况的处理（abort、error、empty response 等）

### Step 3.6：阶段验证

写一段代码创建一个带工具的 Agent：

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, Type } from "@mariozechner/pi-ai";

// 1. 定义一个 AgentTool（如计算器）
// 2. 创建 Agent，配置 model、systemPrompt、tools
// 3. subscribe 事件，打印事件流
// 4. 调用 agent.prompt("计算 123 * 456")
// 5. 观察 Agent Loop 的完整事件序列
```

---

## Phase 4：应用层 — pi-coding-agent（交互式编码 Agent）

> **目标**：理解一个完整的 AI 编码助手是如何组装的
>
> **预计时间**：5-7 天（这是最大的包）
>
> **包路径**：`packages/coding-agent/`

### Step 4.1：入口与启动流程

**文件**：
- `packages/coding-agent/src/cli.ts` — 可执行文件入口
- `packages/coding-agent/src/main.ts` — 主函数
- `packages/coding-agent/src/config.ts` — 全局配置

**学习目标**：
- 理解从 `cli.ts` → `main.ts` 的启动链
- 理解 CLI 参数解析（`cli/args.ts`）
- 理解配置文件路径（`~/.pi/agent/`）
- 理解 `main()` 函数中的初始化序列：
  1. 解析 CLI 参数
  2. 创建 AuthStorage, ModelRegistry, SessionManager, SettingsManager
  3. 调用 `createAgentSession()` 创建核心会话
  4. 根据模式启动对应的运行模式

### Step 4.2：SDK 核心工厂

**文件**：`packages/coding-agent/src/core/sdk.ts`

**学习目标**：
- 理解 `CreateAgentSessionOptions` — 这是整个 coding-agent 的配置中心
- 理解 `createAgentSession()` 的完整流程：
  1. 初始化设置管理器
  2. 解析模型（model-resolver）
  3. 加载扩展（extensions/loader）
  4. 创建工具集
  5. 构建系统提示词
  6. 创建 AgentSession
  7. 返回 session + 扩展加载结果

**关键思考题**：
1. `createAgentSession()` 是 SDK 的核心入口。如果你要在自己的应用中嵌入 pi，你需要理解哪些选项？
2. 工具集是在哪里确定的？默认包含哪些工具？

### Step 4.3：AgentSession

**文件**：`packages/coding-agent/src/core/agent-session.ts`

**学习目标**：
- 理解 `AgentSession` 如何包装 `Agent`（来自 pi-agent-core）
- 理解会话生命周期管理
- 理解消息队列机制
- 理解模型切换
- 理解 compaction（上下文压缩）的触发机制

### Step 4.4：系统提示词构建

**文件**：`packages/coding-agent/src/core/system-prompt.ts`

**学习目标**：
- 理解 `BuildSystemPromptOptions` 接口
- 理解系统提示词的组成部分：
  1. 核心指令
  2. 工具描述（toolSnippets）
  3. 指导方针（promptGuidelines）
  4. 上下文文件（contextFiles，如 AGENTS.md）
  5. 技能描述（skills）
  6. 自定义追加（appendSystemPrompt）
- 理解当前日期和工作目录的注入

### Step 4.5：内置工具系统

**文件**（按推荐阅读顺序）：
1. `packages/coding-agent/src/core/tools/index.ts` — 工具注册和预设集合
2. `packages/coding-agent/src/core/tools/read.ts` — 文件读取工具
3. `packages/coding-agent/src/core/tools/write.ts` — 文件写入工具
4. `packages/coding-agent/src/core/tools/edit.ts` — 文件编辑工具（search-and-replace）
5. `packages/coding-agent/src/core/tools/bash.ts` — Shell 执行工具
6. `packages/coding-agent/src/core/tools/grep.ts` — 代码搜索工具
7. `packages/coding-agent/src/core/tools/find.ts` — 文件查找工具
8. `packages/coding-agent/src/core/tools/ls.ts` — 目录列表工具
9. `packages/coding-agent/src/core/tools/file-mutation-queue.ts` — 文件修改队列
10. `packages/coding-agent/src/core/tools/truncate.ts` — 输出截断
11. `packages/coding-agent/src/core/tools/path-utils.ts` — 路径工具

**学习目标**：
- 理解每个工具的 TypeBox schema 定义
- 理解工具的 `execute()` 实现
- 重点学习 `bash.ts` — 理解 shell 命令执行的安全性考量
- 重点学习 `edit.ts` — 理解 search-and-replace 编辑模式
- 理解 `file-mutation-queue.ts` — 为什么需要串行化文件写入
- 理解 `truncate.ts` — 如何处理超长工具输出

**关键思考题**：
1. `edit` 工具为什么采用 search-and-replace 模式而不是行号替换？
2. `bash` 工具的 execute 实现中有哪些安全措施？
3. `file-mutation-queue` 解决了什么并发问题？

### Step 4.6：上下文压缩（Compaction）

**文件**：
- `packages/coding-agent/src/core/compaction/compaction.ts` — 主逻辑
- `packages/coding-agent/src/core/compaction/branch-summarization.ts` — 分支摘要
- `packages/coding-agent/src/core/compaction/utils.ts` — 工具函数

**学习目标**：
- 理解为什么需要 compaction — LLM 的 context window 有限
- 理解 compaction 的触发时机
- 理解 compaction 的策略：保留近期消息 + 摘要旧消息
- 理解 branch-summarization 在会话分支中的作用

### Step 4.7：扩展系统

**文件**：
- `packages/coding-agent/src/core/extensions/types.ts` — 扩展类型定义
- `packages/coding-agent/src/core/extensions/loader.ts` — 扩展加载
- `packages/coding-agent/src/core/extensions/runner.ts` — 扩展执行
- `packages/coding-agent/src/core/extensions/wrapper.ts` — 扩展包装

**学习目标**：
- 理解 `Extension` 接口 — 扩展可以 hook 进哪些生命周期
- 理解扩展的加载流程：文件系统 → 编译 → 实例化
- 理解 `ExtensionAPI` — 扩展可以访问哪些能力

**辅助文件**：
- `packages/coding-agent/examples/extensions/` — 大量扩展示例
- `packages/coding-agent/docs/extensions.md` — 扩展文档

建议阅读 2-3 个简单的示例扩展：
- `examples/extensions/hello.ts` — 最简单的扩展
- `examples/extensions/tools.ts` — 自定义工具的扩展
- `examples/extensions/commands.ts` — 自定义斜杠命令

### Step 4.8：三种运行模式

**文件**：
- `packages/coding-agent/src/modes/index.ts` — 模式导出
- `packages/coding-agent/src/modes/print-mode.ts` — 打印模式（非交互）
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts` — 交互式 TUI 模式
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts` — RPC 模式

**学习目标**：
- 理解三种模式的适用场景
- 重点学习 `interactive-mode.ts` — 它是如何使用 `pi-tui` 组件构建完整 TUI 的
- 了解 RPC 模式的 JSON-L 协议设计

### Step 4.9：交互式模式的 TUI 组件（选读）

**文件**：`packages/coding-agent/src/modes/interactive/components/`

这个目录包含 30+ 个组件，建议选读：
- `footer.ts` — 底栏（展示模型、token、快捷键等信息）
- `assistant-message.ts` — 助手消息渲染
- `user-message.ts` — 用户消息渲染
- `tool-execution.ts` — 工具执行状态展示
- `bash-execution.ts` — Bash 执行的特殊渲染
- `diff.ts` — Diff 展示
- `model-selector.ts` — 模型选择器
- `session-selector.ts` — 会话选择器

### Step 4.10：其他核心模块

**文件**（快速浏览即可）：
- `packages/coding-agent/src/core/model-resolver.ts` — 模型解析逻辑
- `packages/coding-agent/src/core/session-manager.ts` — 会话持久化
- `packages/coding-agent/src/core/settings-manager.ts` — 设置管理
- `packages/coding-agent/src/core/skills.ts` — 技能系统
- `packages/coding-agent/src/core/slash-commands.ts` — 斜杠命令
- `packages/coding-agent/src/core/prompt-templates.ts` — 提示词模板
- `packages/coding-agent/src/core/event-bus.ts` — 事件总线
- `packages/coding-agent/src/core/keybindings.ts` — 快捷键配置

### Step 4.11：阶段验证

1. 运行 `./pi-test.sh`（从 repo 根目录），体验完整的交互式编码 Agent
2. 阅读 SDK 示例：`packages/coding-agent/examples/sdk/01-minimal.ts` 到 `12-full-control.ts`
3. 尝试编写一个简单的扩展

---

## Phase 5：应用层 — pi-mom（Slack Bot）

> **目标**：理解如何将 Agent 能力集成到 Slack Bot 中
>
> **预计时间**：1-2 天
>
> **包路径**：`packages/mom/`

### Step 5.1：入口与架构总览

**文件**：`packages/mom/src/main.ts`

**学习目标**：
- 理解 mom 的启动流程：CLI 参数解析 → Slack 连接 → 事件监听
- 理解 `SandboxConfig` — Docker vs Host 模式

### Step 5.2：Slack 集成

**文件**：`packages/mom/src/slack.ts`

**学习目标**：
- 理解 Slack Socket Mode 连接
- 理解 `SlackBot` 类 — 消息接收和发送
- 理解 `MomHandler` — 消息处理器

### Step 5.3：上下文与存储

**文件**：
- `packages/mom/src/store.ts` — 频道数据持久化（ChannelStore）
- `packages/mom/src/context.ts` — 上下文管理（context.jsonl）

**学习目标**：
- 理解双文件上下文模型：`log.jsonl`（不可变历史）+ `context.jsonl`（LLM 上下文）
- 理解 log → context 的同步机制
- 理解上下文 compaction

### Step 5.4：Agent 运行器

**文件**：`packages/mom/src/agent.ts`

**学习目标**：
- 理解 `AgentRunner` — 如何为每个 Slack 频道创建和管理 Agent 实例
- 理解与 pi-agent-core 的集成

### Step 5.5：工具实现

**文件**：`packages/mom/src/tools/`

**学习目标**：
- 对比 mom 的工具和 coding-agent 的工具，理解差异
- 重点关注 `attach.ts` — Slack 文件附件工具
- 理解 `bash.ts` 中的沙箱执行

### Step 5.6：事件系统

**文件**：`packages/mom/src/events.ts`

**学习目标**：
- 理解 `createEventsWatcher` — 文件系统监听
- 理解三种事件类型：immediate、one-shot、periodic

---

## Phase 6：运行时层 — pi-web-ui（Web 组件库）

> **目标**：理解如何在浏览器中构建 AI 聊天界面
>
> **预计时间**：2-3 天
>
> **包路径**：`packages/web-ui/`

### Step 6.1：存储层

**文件**：
- `packages/web-ui/src/storage/types.ts` — 存储抽象类型
- `packages/web-ui/src/storage/store.ts` — 通用 Store 抽象
- `packages/web-ui/src/storage/backends/indexeddb-storage-backend.ts` — IndexedDB 实现
- `packages/web-ui/src/storage/stores/` — 四个具体 Store（settings, provider-keys, sessions, custom-providers）
- `packages/web-ui/src/storage/app-storage.ts` — AppStorage 聚合

**学习目标**：
- 理解 `StorageBackend` 抽象
- 理解 IndexedDB 的使用方式
- 理解 Store 的 CRUD 模式

### Step 6.2：核心组件

**文件**：
- `packages/web-ui/src/ChatPanel.ts` — 顶层聊天面板
- `packages/web-ui/src/components/AgentInterface.ts` — Agent 界面
- `packages/web-ui/src/components/MessageList.ts` — 消息列表
- `packages/web-ui/src/components/Messages.ts` — 消息类型组件
- `packages/web-ui/src/components/Input.ts` — 输入组件
- `packages/web-ui/src/components/StreamingMessageContainer.ts` — 流式消息容器

**学习目标**：
- 理解 Web Components 的使用（基于 mini-lit）
- 理解 Agent 事件如何驱动 UI 更新
- 理解流式消息的渲染机制
- 理解 `defaultConvertToLlm` 在 web-ui 中的实现

### Step 6.3：Artifacts 系统

**文件**：`packages/web-ui/src/tools/artifacts/`

**学习目标**：
- 理解 `ArtifactsPanel` — 如何渲染和管理多种格式的 Artifact
- 理解 iframe 沙箱执行（`SandboxedIframe.ts`）
- 理解各种 Artifact 类型的实现（HTML, SVG, Markdown, PDF 等）

### Step 6.4：沙箱运行时

**文件**：`packages/web-ui/src/components/sandbox/`

**学习目标**：
- 理解 `RuntimeMessageBridge` — 主页面与 iframe 之间的消息桥
- 理解 `RuntimeMessageRouter` — 消息路由
- 理解各种 RuntimeProvider（Artifacts, Attachments, Console, FileDownload）

### Step 6.5：工具渲染器

**文件**：
- `packages/web-ui/src/tools/renderer-registry.ts` — 渲染器注册
- `packages/web-ui/src/tools/renderers/` — 内置渲染器

**学习目标**：
- 理解 `ToolRenderer` 接口
- 理解如何为自定义工具注册渲染器

### Step 6.6：阶段验证

运行 web-ui 的示例应用：`packages/web-ui/example/`

---

## Phase 7：部署层 — pi-pods（GPU 部署管理）

> **目标**：了解如何在远程 GPU Pod 上部署和管理 LLM
>
> **预计时间**：1 天（选学）
>
> **包路径**：`packages/pods/`

### Step 7.1：快速浏览

**文件**（建议浏览顺序）：
- `packages/pods/README.md` — 完整使用文档
- `packages/pods/src/` — 源码结构
- 重点关注 Pod 管理、模型配置、vLLM 部署的实现

**学习目标**：
- 了解 vLLM 的基本概念
- 了解 GPU Pod 的 SSH 管理方式
- 了解自动模型配置的逻辑

---

## Phase 8：综合实践

> **目标**：通过实践项目巩固所学知识
>
> **预计时间**：持续

### 练习 1：写一个自定义 Provider

参考 `AGENTS.md` 中 "Adding a New LLM Provider" 部分，为一个 OpenAI 兼容的本地 LLM（如 Ollama）创建一个自定义 Provider。

**涉及知识**：Phase 1 (types, api-registry, stream, providers)

### 练习 2：写一个 coding-agent 扩展

参考 `packages/coding-agent/examples/extensions/` 中的示例，编写一个扩展：
- 在 Agent 启动时注入额外的系统提示词
- 添加一个自定义工具
- 添加一个自定义斜杠命令

**涉及知识**：Phase 3 (Agent), Phase 4 (extensions, tools, slash-commands)

### 练习 3：用 SDK 嵌入 Agent

参考 `packages/coding-agent/examples/sdk/`，编写一个程序：
- 使用 `createAgentSession()` 创建会话
- 在 print-mode 下运行
- 处理 Agent 事件并自定义输出格式

**涉及知识**：Phase 3 (Agent), Phase 4 (sdk, agent-session)

### 练习 4：构建一个 Web 聊天界面

参考 `packages/web-ui/example/`，构建一个简单的 Web 聊天应用：
- 配置 Agent 和存储
- 使用 ChatPanel 组件
- 添加自定义 Artifact 类型

**涉及知识**：Phase 3 (Agent), Phase 6 (web-ui 全部)

### 练习 5：为 Pi 贡献代码

阅读 GitHub Issues，找一个感兴趣的 bug 或 feature request，尝试提交 PR。

**涉及知识**：全部

---

## 附录：核心设计模式速查

### 1. Provider 插件注册模式

```
types.ts 定义接口
    ↓
api-registry.ts 提供注册/查找
    ↓
register-builtins.ts 懒加载注册所有内置 Provider
    ↓
stream.ts 根据 model.api 路由到对应 Provider
```

**关键文件**：`api-registry.ts`, `register-builtins.ts`, `stream.ts`

### 2. 事件驱动的 Agent Loop

```
prompt → agentLoop() → [turn_start → LLM 调用 → tool 执行 → turn_end] × N → agent_end
```

**关键文件**：`agent-loop.ts`, `agent.ts`

### 3. 消息双层模型

```
AgentMessage（应用层，支持自定义角色）
    ↓ convertToLlm()
Message（LLM 层，只有 user/assistant/toolResult）
```

**关键文件**：`agent/src/types.ts`, `agent/src/agent.ts`

### 4. 差分渲染引擎

```
Component.render(width) → string[]
    ↓
TUI 比较新旧行数组
    ↓
只更新变化的行（CSI 2026 同步输出）
```

**关键文件**：`tui/src/tui.ts`

### 5. 扩展系统（Hook 模式）

```
Extension 定义 hook 函数
    ↓
Loader 加载/编译扩展
    ↓
Runner 在生命周期的各个点调用 hook
```

**关键文件**：`extensions/types.ts`, `extensions/loader.ts`, `extensions/runner.ts`

### 6. 上下文压缩

```
context.messages 超过 token 限制
    ↓
保留最近 N 条消息
    ↓
将旧消息发给 LLM 生成摘要
    ↓
用摘要替换旧消息
```

**关键文件**：`compaction/compaction.ts`

---

## 学习检查清单

完成每个 Phase 后，检查你是否能回答以下问题：

### Phase 1 完成检查
- [ ] 我能解释 `Api`, `Provider`, `Model` 三者的关系
- [ ] 我能画出 `stream()` 调用链的完整路径
- [ ] 我能解释懒加载注册机制的工作原理
- [ ] 我能说出至少 3 个 Provider 的 API 差异点

### Phase 2 完成检查
- [ ] 我能解释 TUI 的三策略差分渲染
- [ ] 我能实现一个简单的 `Component`
- [ ] 我能解释 `CURSOR_MARKER` 和 IME 支持的关系

### Phase 3 完成检查
- [ ] 我能画出 Agent Loop 的完整事件序列图
- [ ] 我能解释 `AgentMessage` 到 `Message` 的转换流程
- [ ] 我能解释 Steering 和 Follow-up 的区别

### Phase 4 完成检查
- [ ] 我能解释 `createAgentSession()` 的完整初始化流程
- [ ] 我能解释每个内置工具的作用和实现方式
- [ ] 我能解释扩展系统的 hook 机制
- [ ] 我能解释上下文压缩的策略

### Phase 5 完成检查
- [ ] 我能解释 mom 的双文件上下文模型
- [ ] 我能解释 Skills 系统的工作原理

### Phase 6 完成检查
- [ ] 我能解释 web-ui 的存储架构
- [ ] 我能解释 iframe 沙箱运行时的消息桥接机制
- [ ] 我能解释 Artifacts 系统的渲染流程

---

> **最后的建议**：不要试图一次读完所有代码。按 Phase 推进，每个 Phase 花足够的时间理解核心概念，确保"能回答检查清单中的所有问题"后再进入下一个 Phase。遇到不理解的地方，多看测试文件和示例代码——它们往往比源码注释更有说明力。
