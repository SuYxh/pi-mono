# Pi-coding-agent (@mariozechner/pi-coding-agent) 源码详细解读

## 1. 包概述

`@mariozechner/pi-coding-agent` 是 Pi Monorepo 中最庞大、最复杂的包，是整个项目的**应用层**。它将底层的 `pi-ai`（LLM 抽象）和 `pi-agent-core`（Agent 运行时）组合为一个功能完整的 **AI 编程助手**。

### 1.1 核心职责

- 提供 **AgentSession** —— 完整的会话管理（持久化、分支、压缩、恢复）
- 内置 **7 个编程工具**（read, write, edit, bash, grep, find, ls）
- 实现 **Extension 扩展系统**（插件可注册工具、命令、快捷键、UI 组件）
- 提供 **3 种运行模式**（Interactive TUI / Print 单次 / RPC 无头）
- 管理 **Skills 技能系统**（基于 Markdown 的可复用提示词模板）
- 实现 **Model Registry**（模型发现、OAuth 认证、API Key 管理）
- 支持 **Context Compaction**（长会话上下文压缩）和 **Session Tree**（会话分支与导航）

### 1.2 依赖关系

```
@mariozechner/pi-coding-agent
    ├── @mariozechner/pi-ai          (LLM 抽象层)
    ├── @mariozechner/pi-agent-core  (Agent 运行时)
    ├── @mariozechner/pi-tui         (TUI 框架，interactive 模式)
    ├── @sinclair/typebox             (JSON Schema / 工具参数校验)
    ├── chalk                         (CLI 彩色输出)
    ├── glob                          (文件匹配)
    ├── ignore                        (gitignore 规则)
    └── image-size / sharp            (图片处理)
```

### 1.3 文件结构（核心部分）

```
packages/coding-agent/
├── src/
│   ├── index.ts                    # 公共 API 入口
│   ├── config.ts                   # 全局路径配置（~/.pi/agent 等）
│   ├── cli/
│   │   ├── args.ts                 # CLI 参数解析
│   │   ├── file-processor.ts       # @file 引用展开
│   │   ├── initial-message.ts      # 初始消息构建
│   │   ├── session-picker.ts       # 会话选择
│   │   └── list-models.ts          # --list-models
│   ├── core/
│   │   ├── sdk.ts                  # createAgentSession() 工厂函数
│   │   ├── agent-session.ts        # AgentSession 核心类 (~3200 行)
│   │   ├── messages.ts             # 自定义消息类型定义
│   │   ├── system-prompt.ts        # System Prompt 构建器
│   │   ├── model-resolver.ts       # 模型发现与选择
│   │   ├── model-registry.ts       # 模型注册表 + OAuth
│   │   ├── auth-storage.ts         # 认证凭证存储
│   │   ├── session-manager.ts      # 会话持久化（JSONL 格式）
│   │   ├── settings-manager.ts     # 用户设置管理
│   │   ├── resource-loader.ts      # 资源加载器（skills, prompts, themes, context files）
│   │   ├── skills.ts               # Skills 发现与加载
│   │   ├── prompt-templates.ts     # Prompt 模板系统
│   │   ├── slash-commands.ts       # 斜杠命令系统
│   │   ├── keybindings.ts          # 快捷键管理
│   │   ├── compaction/             # 上下文压缩子系统
│   │   │   ├── compaction.ts       # 压缩核心逻辑
│   │   │   ├── branch-summarization.ts  # 分支摘要
│   │   │   └── utils.ts            # 工具函数（文件追踪等）
│   │   ├── extensions/             # 扩展系统
│   │   │   ├── types.ts            # 扩展类型定义 (~1450 行)
│   │   │   ├── loader.ts           # 扩展加载器
│   │   │   ├── runner.ts           # 扩展运行器
│   │   │   └── wrapper.ts          # 工具包装器
│   │   ├── tools/                  # 内置工具
│   │   │   ├── index.ts            # 工具导出与注册
│   │   │   ├── read.ts             # 文件读取
│   │   │   ├── write.ts            # 文件写入
│   │   │   ├── edit.ts             # 文件编辑（search/replace）
│   │   │   ├── bash.ts             # Shell 命令执行
│   │   │   ├── grep.ts             # 正则搜索（ripgrep）
│   │   │   ├── find.ts             # 文件查找（fd）
│   │   │   ├── ls.ts               # 目录列表
│   │   │   ├── file-mutation-queue.ts  # 文件写入串行化
│   │   │   ├── truncate.ts         # 输出截断
│   │   │   └── path-utils.ts       # 路径工具
│   │   └── export-html/            # HTML 导出
│   └── modes/
│       ├── interactive/            # TUI 交互模式
│       │   ├── interactive-mode.ts # 交互模式主入口
│       │   ├── components/         # ~30 个 TUI 组件
│       │   └── theme/              # 主题系统
│       ├── print-mode.ts           # 单次打印模式
│       └── rpc/                    # RPC 无头模式
│           ├── rpc-types.ts        # RPC 协议定义
│           ├── rpc-mode.ts         # RPC 模式实现
│           └── rpc-client.ts       # RPC 客户端
├── test/                           # ~70+ 测试文件
└── package.json
```

---

## 2. SDK 入口：createAgentSession()

`createAgentSession()` 是整个包的核心工厂函数，在 `core/sdk.ts` 中定义。

### 2.1 创建流程

```
createAgentSession(options?)
│
├── 1. 解析路径
│   ├── cwd (默认 process.cwd())
│   └── agentDir (默认 ~/.pi/agent)
│
├── 2. 创建基础设施
│   ├── AuthStorage.create()        → 凭证存储
│   ├── new ModelRegistry()         → 模型注册表
│   ├── SettingsManager.create()    → 用户设置
│   └── SessionManager.create()     → 会话管理
│
├── 3. 加载资源
│   ├── new DefaultResourceLoader() → 资源加载器
│   └── resourceLoader.reload()     → 加载 skills/prompts/themes/extensions/context files
│
├── 4. 模型解析
│   ├── 尝试从 session 恢复模型
│   ├── 回退到 findInitialModel()  → 检查设置默认值 → 检查 Provider 默认值
│   └── 解析 thinkingLevel（考虑模型能力 clamp）
│
├── 5. 创建 Agent（pi-agent-core）
│   ├── new Agent({
│   │   convertToLlm: convertToLlmWithBlockImages,
│   │   transformContext: extensionRunner.emitContext,
│   │   getApiKey: modelRegistry.getApiKeyForProvider,
│   │   onPayload: extensionRunner.emitBeforeProviderRequest,
│   │   ...
│   │ })
│   └── 恢复已有会话消息
│
├── 6. 创建 AgentSession
│   └── new AgentSession({ agent, sessionManager, settingsManager, ... })
│
└── 返回 { session, extensionsResult, modelFallbackMessage }
```

### 2.2 默认工具集

```typescript
const defaultActiveToolNames: ToolName[] = ["read", "bash", "edit", "write"];
```

默认启用 4 个工具：read, bash, edit, write。额外的 grep, find, ls 需要手动启用或通过 `--tools all` 参数。

### 2.3 convertToLlm 图片过滤

```typescript
const convertToLlmWithBlockImages = (messages: AgentMessage[]): Message[] => {
    const converted = convertToLlm(messages);
    if (!settingsManager.getBlockImages()) return converted;
    // 将 ImageContent 替换为 "Image reading is disabled." 文本
};
```

当 `blockImages` 设置启用时，所有图片在发送给 LLM 前被替换为文本占位符。这是纵深防御策略。

---

## 3. AgentSession —— 核心会话管理

`AgentSession` 是整个包最核心的类，约 3200 行，连接了 Agent 运行时与所有应用层功能。

### 3.1 核心职责

| 职责 | 说明 |
|------|------|
| 事件处理 | 订阅 Agent 事件，驱动持久化、扩展、UI |
| 会话持久化 | 每个 message_end 自动保存到 JSONL 文件 |
| 扩展桥接 | 将 Agent 事件转发给 Extension Runner |
| 自动压缩 | 检测上下文溢出并自动触发压缩 |
| 自动重试 | 检测可重试错误（overloaded、rate limit）并自动重试 |
| 工具注册 | 管理工具定义注册表（内置 + 扩展） |
| System Prompt | 动态构建 system prompt（含工具、skills、context files） |
| 模型管理 | 模型切换、thinking level 管理 |
| 会话分支 | 支持 fork、tree navigation、branch summarization |

### 3.2 事件处理管道

```
Agent.emit(event)
    │
    ▼
AgentSession._handleAgentEvent(event)    [同步入口]
    │
    ├── _createRetryPromiseForAgentEnd()  [预创建重试 Promise]
    │
    └── _agentEventQueue.then(() => _processAgentEvent(event))  [异步串行化]
         │
         ├── 1. 移除 steering/followUp 队列项（若 message_start + user）
         ├── 2. _emitExtensionEvent()     → 转发给 Extension Runner
         ├── 3. _emit()                   → 通知外部监听器
         ├── 4. 持久化处理（message_end）
         │   ├── custom message → appendCustomMessageEntry()
         │   └── LLM message   → appendMessage()
         ├── 5. 跟踪 assistant message（用于压缩和重试检查）
         └── 6. agent_end 时
             ├── 检查可重试错误 → _handleRetryableError()
             └── 检查压缩 → _checkCompaction()
```

关键设计：`_agentEventQueue` 使用 Promise 链实现事件串行化。即使事件处理是异步的（如扩展事件发射），也保证严格的顺序执行。

### 3.3 Prompt 方法

```typescript
async prompt(text: string, options?: PromptOptions): Promise<void> {
    // 1. 扩展命令检查
    if (text.startsWith("/")) {
        const handled = await this._tryExecuteExtensionCommand(text);
        if (handled) return;
    }

    // 2. 扩展 input 事件拦截
    const inputResult = await extensionRunner.emitInput(text, images, source);
    if (inputResult.action === "handled") return;
    if (inputResult.action === "transform") { text = inputResult.text; }

    // 3. Skill 命令 + Prompt 模板展开
    expandedText = this._expandSkillCommand(expandedText);
    expandedText = expandPromptTemplate(expandedText, templates);

    // 4. 流式中 → 队列
    if (this.isStreaming) {
        if (streamingBehavior === "followUp") await this._queueFollowUp();
        else await this._queueSteer();
        return;
    }

    // 5. 验证模型和 API Key
    // 6. before_agent_start 扩展事件
    // 7. 刷新 system prompt + 工具集
    // 8. agent.prompt() → 进入 Agent Loop
    // 9. waitForRetry() → 等待自动重试完成
}
```

### 3.4 自动压缩

```typescript
private async _checkCompaction(lastAssistant: AssistantMessage): Promise<void> {
    // 1. 检查是否为上下文溢出
    if (isContextOverflow(lastAssistant)) {
        if (!this._overflowRecoveryAttempted) {
            this._overflowRecoveryAttempted = true;
            this._emit({ type: "auto_compaction_start", reason: "overflow" });
            // 压缩并自动重试
            const result = await this._performAutoCompaction("overflow");
            if (result) {
                await this.agent.continue();
            }
        }
        return;
    }

    // 2. 检查是否达到阈值（默认 60% context window）
    if (shouldCompact(estimatedTokens, model.contextWindow, settings.threshold)) {
        this._emit({ type: "auto_compaction_start", reason: "threshold" });
        await this._performAutoCompaction("threshold");
    }
}
```

两种触发条件：
- **上下文溢出**：LLM 返回 context overflow 错误，立即压缩并自动 continue
- **阈值触发**：预估 token 数超过 context window 的 60%（默认），压缩但不自动 continue

### 3.5 自动重试

```typescript
private async _handleRetryableError(msg: AssistantMessage): Promise<boolean> {
    const settings = this.settingsManager.getRetrySettings();
    if (!settings.enabled) return false;

    const maxAttempts = settings.maxAttempts ?? 3;
    if (this._retryAttempt >= maxAttempts) {
        this._emit({ type: "auto_retry_end", success: false, attempt: this._retryAttempt, ... });
        this._retryAttempt = 0;
        return false;
    }

    this._retryAttempt++;
    const delayMs = this._calculateRetryDelay(this._retryAttempt, settings);
    this._emit({ type: "auto_retry_start", attempt: this._retryAttempt, maxAttempts, delayMs, ... });

    await sleep(delayMs);
    await this.agent.continue();
    return true;
}
```

支持可配置的指数退避重试策略，针对 overloaded、rate limit、5xx 服务器错误。

### 3.6 _buildRuntime —— 工具注册体系

```
_buildRuntime()
├── 创建内置工具 ToolDefinition → AgentTool 包装
├── 加载扩展工具 (extensions/*.ts)
├── 加载 SDK 自定义工具
├── 合并工具注册表 (_toolRegistry, _toolDefinitions)
├── 构建 system prompt
│   ├── 基础 prompt (cwd, 日期, 工具列表, 使用指南)
│   ├── Skills XML 列表
│   ├── Context files (AGENTS.md / CLAUDE.md)
│   ├── 自定义 system prompt
│   └── 工具的 promptSnippet 和 promptGuidelines
└── 设置 agent.tools 和 agent.systemPrompt
```

---

## 4. 内置工具系统

### 4.1 工具架构

每个工具都遵循 `ToolDefinition` 接口：

```typescript
interface ToolDefinition<TParams, TDetails, TState> {
    name: string;
    label: string;
    description: string;
    promptSnippet?: string;          // system prompt 中的一行摘要
    promptGuidelines?: string[];     // system prompt 中的使用指南
    parameters: TParams;             // TypeBox Schema
    execute(id, params, signal, onUpdate, ctx): Promise<AgentToolResult<TDetails>>;
    renderCall?(args, theme, ctx): Component;      // TUI 渲染工具调用
    renderResult?(result, opts, theme, ctx): Component;  // TUI 渲染结果
}
```

关键区别于 `pi-agent-core` 的 `AgentTool`：增加了 `renderCall/renderResult` 用于 TUI 展示，以及 `promptSnippet/promptGuidelines` 用于 system prompt。

### 4.2 read 工具

```
readSchema:
  - path: string (文件路径)
  - offset?: number (起始行号)
  - limit?: number (读取行数)

特性：
  - 自动解析路径（相对路径 → 基于 cwd 的绝对路径）
  - 输出行号标注（LINE_NUMBER→LINE_CONTENT 格式）
  - 大文件自动截断（DEFAULT_MAX_BYTES）
  - 支持读取图片文件（返回 base64 ImageContent）
  - 返回 ReadToolDetails { truncation? }
```

### 4.3 write 工具

```
writeSchema:
  - path: string
  - content: string
  - createDirectories?: boolean

特性：
  - 使用 withFileMutationQueue 串行化同文件写入
  - 自动创建目录（可选）
  - 扩展钩子 beforeToolCall / afterToolCall
```

### 4.4 edit 工具

```
editSchema:
  - path: string
  - old_string: string (搜索文本)
  - new_string: string (替换文本)

特性：
  - Search/Replace 模式（非全文替换）
  - 只替换第一个匹配
  - 如果没有匹配 → 返回错误
  - 如果多个匹配 → 返回错误 + 上下文提示
  - 使用 withFileMutationQueue 串行化
  - 返回 EditToolDetails { diff, hunks }
  - TUI 中展示 unified diff（彩色）
```

### 4.5 bash 工具

```
bashSchema:
  - command: string
  - timeout?: number (默认 120s)

特性：
  - 通过 child_process.spawn 执行
  - 支持 BashOperations 接口（可被扩展覆盖，如 SSH 远程执行）
  - 实时流式输出（onUpdate 回调）
  - 大输出自动截断（只保留头尾）
  - 返回 BashToolDetails { exitCode, stdout, stderr, truncation }
  - TUI 中实时渲染终端输出
```

### 4.6 grep 工具

```
grepSchema:
  - pattern: string (正则或字面量)
  - path?: string
  - glob?: string (文件过滤)
  - ignoreCase?: boolean
  - literal?: boolean
  - context?: number (上下文行数)
  - limit?: number (最大匹配数)

底层工具：ripgrep (rg)
  - 通过 ensureTool("rg") 确保可用
  - 使用 spawn 调用 rg，逐行解析输出
```

### 4.7 find 工具

```
findSchema:
  - pattern: string (glob 模式)
  - path?: string
  - limit?: number

底层工具：fd（优先）或 globSync（回退）
  - 通过 ensureTool("fd") 确保可用
  - 尊重 .gitignore 规则
```

### 4.8 ls 工具

```
lsSchema:
  - path?: string
  - limit?: number

纯 Node.js 实现：
  - readdirSync + statSync
  - 目录标记 "/" 后缀
  - 大目录自动截断
```

### 4.9 文件写入串行化

```typescript
// file-mutation-queue.ts
const fileMutationQueues = new Map<string, Promise<void>>();

async function withFileMutationQueue<T>(filePath: string, fn: () => Promise<T>): Promise<T> {
    const key = await getMutationQueueKey(filePath);  // realpath 去符号链接
    const currentQueue = fileMutationQueues.get(key) ?? Promise.resolve();

    // 链式 Promise 实现队列
    const nextQueue = new Promise<void>(...);
    fileMutationQueues.set(key, currentQueue.then(() => nextQueue));

    await currentQueue;  // 等待前序操作
    try {
        return await fn();
    } finally {
        releaseNext();
    }
}
```

当 LLM 在同一个 turn 中并行调用多个 edit/write 针对同一文件时，这个队列确保它们按到达顺序串行执行，防止数据竞争。不同文件的操作仍然并行。

---

## 5. Extension 扩展系统

扩展系统是 pi-coding-agent 最复杂的子系统之一，约 1450 行类型定义 + 独立的 loader/runner/wrapper。

### 5.1 扩展发现与加载

扩展来源：
1. **用户全局**：`~/.pi/agent/extensions/*.ts`
2. **项目本地**：`.pi/extensions/*.ts`
3. **CLI 参数**：`--extensions path1,path2`
4. **npm 包**：在 package.json 中声明

加载流程：

```
loadExtensions(paths)
├── 对每个扩展路径：
│   ├── import(path)                   → 获取模块
│   ├── module.default 或 module       → 获取 ExtensionFactory
│   ├── factory(pi: ExtensionAPI)      → 执行工厂函数
│   └── 收集注册的 handlers / tools / commands / shortcuts / flags
└── 返回 LoadExtensionsResult { extensions, errors, runtime }
```

### 5.2 ExtensionAPI —— 扩展 API 面

扩展通过 `ExtensionAPI` 接口（传入工厂函数的 `pi` 参数）注册各种能力：

| API | 说明 |
|-----|------|
| `pi.on(event, handler)` | 注册事件处理器（30+ 事件类型） |
| `pi.registerTool(def)` | 注册 LLM 可调用的工具 |
| `pi.registerCommand(name, opts)` | 注册斜杠命令 |
| `pi.registerShortcut(key, opts)` | 注册快捷键 |
| `pi.registerFlag(name, opts)` | 注册 CLI 参数 |
| `pi.registerMessageRenderer(type, fn)` | 注册自定义消息渲染器 |
| `pi.registerProvider(name, config)` | 注册/覆盖 LLM Provider |
| `pi.sendMessage(msg, opts)` | 发送自定义消息 |
| `pi.sendUserMessage(content, opts)` | 发送用户消息 |
| `pi.setModel(model)` | 切换模型 |
| `pi.getActiveTools()` / `pi.setActiveTools()` | 管理活跃工具 |
| `pi.events` | 共享事件总线（扩展间通信） |

### 5.3 事件体系

扩展系统定义了 30+ 种事件，分为六大类：

**资源事件**：
- `resources_discover` — 启动/重载时发现额外资源路径

**会话事件**：
- `session_directory` / `session_start` / `session_shutdown`
- `session_before_switch` / `session_switch`
- `session_before_fork` / `session_fork`
- `session_before_compact` / `session_compact`
- `session_before_tree` / `session_tree`

**Agent 事件**：
- `before_agent_start` / `agent_start` / `agent_end`
- `turn_start` / `turn_end`
- `message_start` / `message_update` / `message_end`
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end`
- `context` / `before_provider_request`

**工具事件**：
- `tool_call` — 工具执行前（可阻止）
- `tool_result` — 工具执行后（可修改结果）

**用户事件**：
- `input` — 用户输入（可拦截/转换）
- `user_bash` — 用户直接执行 bash 命令
- `model_select` — 模型切换

### 5.4 ExtensionRunner

```
ExtensionRunner
├── bindCore(actions, context)       → 绑定核心动作（初始化后）
├── bindUI(uiContext)                → 绑定 UI 上下文
├── emit(event)                      → 派发事件给所有扩展 handlers
├── emitContext(messages)            → context 事件（可修改消息）
├── emitToolCall(event)              → tool_call 事件（可阻止）
├── emitToolResult(event)            → tool_result 事件（可修改结果）
├── emitInput(text, images, source)  → input 事件（可拦截/转换）
├── emitBeforeProviderRequest(payload) → 修改 LLM 请求
└── resolveCommand(text)             → 解析斜杠命令
```

Runner 在事件处理中使用 **First Handler Wins** 策略：多个扩展注册相同事件时，第一个返回非空结果的 handler 胜出。

### 5.5 ToolDefinition vs AgentTool 的关系

```
ToolDefinition (coding-agent 层)
    │
    │ wrapToolDefinition()
    ▼
AgentTool (agent-core 层)
```

`wrapToolDefinition()` 将 `ToolDefinition`（含 renderCall/renderResult）转换为标准 `AgentTool`（只含 execute）。渲染信息存储在 `_toolDefinitions` Map 中供 TUI 使用。

---

## 6. 会话管理系统

### 6.1 SessionManager —— JSONL 持久化

会话使用 **JSONL（JSON Lines）** 格式存储，每行一个 JSON 对象：

```jsonl
{"type":"session","version":3,"id":"abc123","timestamp":"...","cwd":"/path"}
{"type":"message","id":"e1","parentId":null,"message":{"role":"user","content":"..."}}
{"type":"message","id":"e2","parentId":"e1","message":{"role":"assistant","content":"..."}}
{"type":"thinking_level_change","id":"e3","parentId":"e2","thinkingLevel":"medium"}
{"type":"compaction","id":"e4","parentId":"e3","summary":"...","firstKeptEntryId":"e2"}
```

**Entry 类型**：

| 类型 | 说明 |
|------|------|
| `session` | 文件头（版本、ID、cwd） |
| `message` | LLM 消息（user/assistant/toolResult） |
| `thinking_level_change` | Thinking Level 变更记录 |
| `model_change` | 模型切换记录 |
| `compaction` | 压缩摘要（记录压缩前 token 数、首个保留 entry） |
| `branch_summary` | 分支摘要（离开分支时生成） |
| `custom` | 扩展自定义持久化数据 |
| `custom_message` | 扩展自定义消息（参与 LLM 上下文） |
| `label` | 书签/标签 |
| `session_info` | 会话元数据（显示名称） |

### 6.2 会话树（Tree Model）

每个 entry 都有 `id` 和 `parentId`，形成**树状结构**（而非线性列表）：

```
e1 (user: "Hello")
└── e2 (assistant: "Hi!")
    ├── e3 (user: "Do A")        ← Branch 1
    │   └── e4 (assistant: "...")
    └── e5 (user: "Do B")        ← Branch 2 (fork)
        └── e6 (assistant: "...")
```

`getBranch()` 方法返回从根到当前叶子的路径，忽略其他分支。`navigateTree()` 可以导航到任意节点，自动生成分支摘要。

### 6.3 buildSessionContext()

```typescript
buildSessionContext(): {
    messages: AgentMessage[];
    model?: { provider: string; modelId: string };
    thinkingLevel?: string;
}
```

从 JSONL 文件重建 Agent 上下文：
1. 获取当前分支路径（getBranch()）
2. 如果有 compaction entry → 用摘要替代被压缩的消息
3. 如果有 branch_summary entry → 用摘要替代被总结的分支
4. 保留最新的 model_change 和 thinking_level_change
5. 按顺序组装消息列表

---

## 7. 上下文压缩（Compaction）

### 7.1 压缩流程

```
shouldCompact(estimatedTokens, contextWindow, threshold)
    │ true
    ▼
prepareCompaction()
├── 收集压缩范围内的消息
├── 提取文件操作记录（read/edit/write/bash）
└── 返回 CompactionPreparation { messages, fileOperations }
    │
    ▼
compact(preparation, model)
├── 构建 summarization prompt
│   ├── SUMMARIZATION_SYSTEM_PROMPT（指导 LLM 如何总结）
│   ├── serializeConversation()（将消息序列化为文本）
│   ├── formatFileOperations()（文件操作摘要）
│   └── computeFileLists()（合并 read/modified 文件列表）
├── 调用 completeSimple()（一次性 LLM 调用，非流式）
└── 返回 CompactionResult { summary, details, tokensBefore, tokensAfter }
```

### 7.2 核心设计决策

1. **使用同一模型压缩**：用当前对话模型做摘要，保持风格一致
2. **文件操作追踪**：跨压缩记录所有文件的 read/modified 操作
3. **扩展钩子**：`session_before_compact` 事件允许扩展完全替代压缩逻辑
4. **渐进式压缩**：保留 `firstKeptEntryId` 作为截断点，新压缩基于上次压缩的 details 累加

---

## 8. 运行模式

### 8.1 Interactive 模式（TUI）

基于 `pi-tui` 构建的全功能终端 UI：

```
InteractiveMode
├── Terminal（pi-tui 核心）
├── ~30 个组件
│   ├── UserMessage / AssistantMessage / ToolExecution
│   ├── ModelSelector / ThinkingSelector / SettingsSelector
│   ├── SessionSelector / SessionSelectorSearch
│   ├── ConfigSelector / TreeSelector
│   ├── Footer / KeybindingHints
│   ├── Diff / BashExecution
│   ├── LoginDialog / OAuthSelector
│   └── ExtensionEditor / ExtensionInput / ExtensionSelector
├── Theme 系统
│   ├── dark.json / light.json
│   └── 用户自定义主题
└── Keybinding 系统
    ├── 默认快捷键
    └── 用户自定义 keybindings.json
```

### 8.2 Print 模式（单次执行）

```bash
pi -p "Explain this code"           # text 模式
pi --mode json "Explain this code"  # JSON event stream
```

Print 模式：
- 发送 prompt → 等待完成 → 输出结果 → 退出
- `text` 模式只输出最终 assistant 文本
- `json` 模式输出所有事件（JSONL 格式）

### 8.3 RPC 模式（无头操作）

```bash
pi --mode rpc
```

RPC 协议：
- **输入**（stdin）：JSON Lines 命令
- **输出**（stdout）：JSON Lines 响应 + 事件

支持的命令：
- `prompt` / `steer` / `follow_up` / `abort`
- `get_state` / `get_messages`
- `set_model` / `cycle_model` / `get_available_models`
- `set_thinking_level` / `cycle_thinking_level`
- `compact` / `set_auto_compaction`
- `bash` / `abort_bash`
- `new_session` / `switch_session` / `fork`
- `export_html` / `get_session_stats`

RPC 模式使 pi 可以作为后端服务被其他工具（如 pi-web-ui）驱动。

---

## 9. Skills 技能系统

### 9.1 Skill 定义

Skills 是带有 YAML frontmatter 的 Markdown 文件：

```markdown
---
name: my-skill
description: Helps with database migrations
---

When performing database migrations, follow these steps:
1. ...
```

### 9.2 发现规则

```
loadSkills()
├── ~/.pi/agent/skills/         (用户全局)
├── .pi/skills/                 (项目本地)
└── --skills path1,path2        (CLI 指定)

目录扫描规则：
  - 如果目录包含 SKILL.md → 视为 skill 根目录，不递归
  - 否则 → 扫描根目录的 .md 文件 + 递归子目录找 SKILL.md
  - 尊重 .gitignore / .ignore / .fdignore
```

### 9.3 名称验证

```
- 必须匹配父目录名
- 仅允许 a-z, 0-9, 连字符
- 不能以连字符开头/结尾
- 不能包含连续连字符
- 最大 64 字符
```

### 9.4 System Prompt 注入

```xml
<available_skills>
  <skill>
    <name>my-skill</name>
    <description>Helps with database migrations</description>
    <location>/path/to/my-skill/SKILL.md</location>
  </skill>
</available_skills>
```

LLM 根据描述判断是否需要使用某个 skill，如果需要则用 `read` 工具读取完整内容。

---

## 10. Model Registry 与认证

### 10.1 ModelRegistry

```
ModelRegistry
├── find(provider, modelId)           → 查找模型
├── getApiKey(model)                  → 获取 API Key
│   ├── 环境变量 (OPENAI_API_KEY 等)
│   ├── OAuth Token（自动刷新）
│   └── 手动设置
├── getApiKeyForProvider(provider)    → 按 Provider 获取
├── isUsingOAuth(model)               → 是否使用 OAuth
├── getAllModels()                     → 列出所有可用模型
└── registerProvider(name, config)    → 注册自定义 Provider
```

### 10.2 认证流程

```
getApiKey(model)
├── 检查环境变量
├── 检查 AuthStorage（~/.pi/agent/auth.json）
│   ├── 如果有 OAuth 凭证
│   │   ├── 检查过期 → refreshToken()
│   │   └── getApiKey(credentials) → 返回 access token
│   └── 如果有静态 API Key → 直接返回
└── 返回 undefined（无凭证）
```

### 10.3 动态 Provider 注册

扩展可以通过 `pi.registerProvider()` 注册自定义 Provider：

```typescript
pi.registerProvider("corporate-ai", {
    baseUrl: "https://ai.corp.com",
    api: "openai-responses",
    models: [...],
    oauth: {
        name: "Corporate AI (SSO)",
        login: async (callbacks) => { /* OAuth 流程 */ },
        refreshToken: async (credentials) => { /* 刷新 token */ },
        getApiKey: (credentials) => credentials.access,
    },
});
```

---

## 11. System Prompt 构建

`buildSystemPrompt()` 动态组装 system prompt：

```
buildSystemPrompt()
├── 1. 基础信息
│   ├── "You are pi, a coding assistant."
│   ├── 当前日期
│   └── 工作目录（cwd）
│
├── 2. 工具列表
│   └── 每个活跃工具的 promptSnippet
│
├── 3. 使用指南
│   ├── 通用编码指南
│   ├── 文件操作指南
│   └── 每个工具的 promptGuidelines
│
├── 4. Skills（XML 格式）
│
├── 5. Context Files（AGENTS.md / CLAUDE.md 内容）
│
├── 6. 自定义 system prompt（--system-prompt）
│
└── 7. Append system prompt（--append-system-prompt + 扩展追加）
```

---

## 12. 测试策略

### 12.1 Test Harness

`test-harness.ts` 提供了强大的测试基础设施：

```typescript
const harness = await createTestHarness({
    responses: [
        "Hello!",
        { text: "Result", toolCalls: [{ name: "read", args: { path: "foo.ts" } }] },
        { error: "Server overloaded", stopReason: "error" },
    ],
    tools: [myCustomTool],
    settings: { autoCompaction: { enabled: true, threshold: 0.6 } },
});
```

核心功能：
- **FauxStreamFn**：模拟 LLM 响应，支持声明式序列
- **内存会话管理**：`SessionManager.inMemory()` 避免文件 IO
- **事件捕获**：`harness.events` 收集所有事件用于断言
- **fauxModel**：虚拟模型（无实际 LLM 调用）

### 12.2 测试覆盖范围

约 70+ 测试文件，覆盖：

| 分类 | 测试 |
|------|------|
| AgentSession | 自动压缩、分支、并发、动态工具、模型切换、重试、tree 导航 |
| SessionManager | 上下文构建、文件操作、标签、迁移、tree 遍历、自定义 session ID |
| Extensions | 发现、runner、input 事件、动态 Provider |
| Compaction | 序列化、扩展钩子、thinking 模型、推理摘要 |
| Tools | 工具执行、文件写入队列、路径工具 |
| CLI | 参数解析、初始消息构建 |
| Settings | 设置管理器、block images |
| Skills | 加载、验证、碰撞检测 |
| RPC | 协议解析、事件流 |
| UI | footer、tree selector、tool execution 组件、主题 |

---

## 13. 与其他包的关系

```
    @mariozechner/pi-ai  ←──────────────────┐
         │                                    │
         ▼                                    │
    @mariozechner/pi-agent-core               │
         │                                    │
         ▼                                    │
    @mariozechner/pi-coding-agent ─────────→──┘
         │         │         │
         │         │         └──→ @mariozechner/pi-tui (interactive mode)
         │         │
         ▼         ▼
    @mariozechner/pi-mom    @mariozechner/pi-web-ui (via RPC)
```

---

## 14. 源码阅读建议

### 推荐阅读顺序

1. **core/sdk.ts** 的 `createAgentSession()` → 理解入口和组装流程
2. **core/tools/index.ts** → 理解工具注册体系
3. **core/tools/read.ts** → 最简单的工具实现参考
4. **core/tools/edit.ts** → 理解 search/replace 和 file-mutation-queue
5. **core/tools/bash.ts** → 理解 BashOperations 可插拔架构
6. **core/agent-session.ts** 的构造函数 + `_buildRuntime()` → 理解运行时组装
7. **core/agent-session.ts** 的 `prompt()` → 理解 prompt 管道
8. **core/agent-session.ts** 的 `_processAgentEvent()` → 理解事件处理
9. **core/extensions/types.ts** → 理解扩展 API 全貌
10. **core/session-manager.ts** → 理解 JSONL 持久化和树模型
11. **core/compaction/compaction.ts** → 理解上下文压缩
12. **core/skills.ts** → 理解 Skills 系统

### 关键断点位置

| 文件 | 位置 | 说明 |
|------|------|------|
| sdk.ts | `createAgentSession()` | 全局入口 |
| agent-session.ts | `prompt()` | Prompt 完整管道 |
| agent-session.ts | `_processAgentEvent()` | 事件处理中枢 |
| agent-session.ts | `_checkCompaction()` | 自动压缩触发 |
| agent-session.ts | `_handleRetryableError()` | 自动重试逻辑 |
| agent-session.ts | `_buildRuntime()` | 工具和 system prompt 组装 |
| extensions/runner.ts | `emit()` | 扩展事件派发 |
| tools/edit.ts | `execute()` | Search/Replace 核心逻辑 |
| tools/bash.ts | `execute()` | Shell 执行 |
| session-manager.ts | `buildSessionContext()` | 会话上下文重建 |
| compaction/compaction.ts | `compact()` | 压缩核心 |

### 核心设计思想总结

1. **工厂模式**：`createAgentSession()` 一次性组装所有依赖
2. **事件驱动架构**：AgentSession 通过事件连接 Agent、Extensions、UI、Persistence
3. **可插拔工具**：`ToolDefinition` + `BashOperations/GrepOperations` 支持远程执行
4. **文件写入安全**：`withFileMutationQueue` 防止并发文件损坏
5. **三模式适配**：Interactive / Print / RPC 共享 AgentSession 核心，只替换 IO 层
6. **扩展系统**：30+ 事件类型 + 工具/命令/快捷键/Provider 注册能力
7. **会话树**：JSONL + parentId 实现分支、fork、导航
8. **渐进式压缩**：跨压缩累积文件操作记录，保持上下文连贯
