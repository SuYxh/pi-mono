# Pi-mom (@mariozechner/pi-mom) 源码详细解读

## 1. 包概述

`@mariozechner/pi-mom` 是 Pi Monorepo 中的**自治 Slack Bot**，名为 "mom"。它将 `pi-coding-agent` 的核心能力（Agent Session、工具系统、会话管理）封装为一个 Slack Bot，能在 Docker 沙箱中自主完成编程任务、管理定时事件、维护持久记忆。

### 1.1 核心职责

- **Slack Bot**：通过 Socket Mode 实时接收 @mention 和 DM，将消息转发给 AI Agent
- **Docker 沙箱执行**：工具命令在隔离的 Docker 容器中执行
- **会话上下文持久化**：双文件模型（context.jsonl + log.jsonl）
- **事件系统**：immediate / one-shot / periodic 三种定时事件
- **记忆系统**：全局 MEMORY.md + 频道级 MEMORY.md
- **Skills 系统**：可创建可复用的 CLI 工具
- **附件处理**：自动下载 Slack 文件附件并注入工具上下文

### 1.2 依赖关系

```
@mariozechner/pi-mom
    ├── @mariozechner/pi-ai              (LLM 抽象)
    ├── @mariozechner/pi-agent-core      (Agent 运行时)
    ├── @mariozechner/pi-coding-agent    (AgentSession、SessionManager、Skills)
    ├── @slack/socket-mode               (Slack Socket Mode 客户端)
    ├── @slack/web-api                   (Slack Web API 客户端)
    ├── croner                           (Cron 调度)
    ├── chalk                            (CLI 彩色输出)
    └── diff                             (文本差异)
```

### 1.3 文件结构

```
packages/mom/
├── src/
│   ├── main.ts          # CLI 入口 (~367 行)
│   ├── slack.ts         # SlackBot 类 + 消息队列 (~623 行)
│   ├── agent.ts         # AgentRunner + 事件处理 (~883 行)
│   ├── sandbox.ts       # 沙箱执行器（Host / Docker） (~219 行)
│   ├── events.ts        # 事件监视器（immediate / one-shot / periodic） (~383 行)
│   ├── store.ts         # 频道存储 + 附件下载 (~234 行)
│   ├── context.ts       # log.jsonl ↔ context.jsonl 同步 (~180 行)
│   ├── download.ts      # 频道历史下载工具
│   ├── log.ts           # 结构化日志输出
│   └── tools/
│       ├── index.ts     # 工具注册（5 个工具）
│       ├── bash.ts      # Bash 执行工具
│       ├── read.ts      # 文件读取工具
│       ├── write.ts     # 文件写入工具
│       ├── edit.ts      # 文件编辑工具
│       ├── attach.ts    # Slack 文件上传工具
│       └── truncate.ts  # 输出截断工具
├── docs/                # 设计文档
├── dev.sh               # 开发启动脚本
├── docker.sh            # Docker 容器管理脚本
└── package.json
```

约 **3000 行** 核心代码。

---

## 2. 架构总览

```
Slack Events (Socket Mode)
        │
        ▼
    SlackBot (slack.ts)
    ├── ChannelQueue (per-channel sequential)
    ├── backfill (历史补填)
    └── logging (log.jsonl)
        │
        ▼
    main.ts (handler)
    ├── ChannelState { running, runner, store }
    └── createSlackContext() → adapter
        │
        ▼
    AgentRunner (agent.ts)
    ├── AgentSession (from pi-coding-agent)
    │   ├── Agent (from pi-agent-core)
    │   │   └── Model: claude-sonnet-4-5
    │   ├── SessionManager (context.jsonl)
    │   └── SettingsManager
    ├── Tools (5 个)
    │   ├── read / write / edit / bash → Executor
    │   └── attach → uploadFn
    └── Event handling → Slack responses
        │
        ▼
    Executor (sandbox.ts)
    ├── HostExecutor (本地执行)
    └── DockerExecutor (docker exec)

    EventsWatcher (events.ts)
    ├── fs.watch() → events/ 目录
    ├── setTimeout (one-shot)
    ├── Cron (periodic)
    └── synthetic SlackEvent → slack.enqueueEvent()
```

---

## 3. SlackBot（slack.ts）

### 3.1 消息接收

SlackBot 通过 Slack Socket Mode 建立 WebSocket 长连接：

```typescript
this.socketClient = new SocketModeClient({ appToken: config.appToken });
this.webClient = new WebClient(config.botToken);
```

两种触发方式：
- **`app_mention`**：频道中 @mom → 提取文本（去掉 @mention）→ 排队处理
- **`message`** (DM)：直接消息 → 排队处理

### 3.2 ChannelQueue —— 按频道串行化

```typescript
class ChannelQueue {
    private queue: QueuedWork[] = [];
    private processing = false;

    enqueue(work: QueuedWork): void {
        this.queue.push(work);
        this.processNext();
    }

    private async processNext(): Promise<void> {
        if (this.processing || this.queue.length === 0) return;
        this.processing = true;
        const work = this.queue.shift()!;
        try { await work(); }
        catch { /* ... */ }
        this.processing = false;
        this.processNext();  // 递归处理下一个
    }
}
```

每个频道一个 `ChannelQueue`，确保同一频道的消息**串行处理**（不同频道可并行）。

### 3.3 Stop 命令的特殊处理

```typescript
// Check for stop command - execute immediately, don't queue!
if (slackEvent.text.toLowerCase().trim() === "stop") {
    if (this.handler.isRunning(e.channel)) {
        this.handler.handleStop(e.channel, this); // Don't await, don't queue
    }
}
```

`stop` 命令**不排队**，立即调用 `runner.abort()`。这是因为 stop 需要中断当前正在执行的任务，如果排队就需要等当前任务完成才执行，失去了 stop 的意义。

### 3.4 startupTs —— 防止重放旧消息

```typescript
this.startupTs = (Date.now() / 1000).toFixed(6);

// 收到消息时
if (this.startupTs && e.ts < this.startupTs) {
    log.logInfo(`Logged old message (pre-startup), not triggering`);
    ack();
    return;
}
```

Bot 启动时记录时间戳。Slack Socket Mode 连接后可能重放历史消息，通过 `startupTs` 过滤，只记录日志不触发处理。

### 3.5 Backfill —— 离线期间消息补填

```typescript
private async backfillChannel(channelId: string): Promise<number> {
    // 找到 log.jsonl 中最新的 ts
    const latestTs = /* ... */;

    // 从 Slack API 获取 latestTs 之后的消息
    const messages = await webClient.conversations.history({
        channel: channelId,
        oldest: latestTs,
        inclusive: false,
        limit: 1000,
    });

    // 过滤并写入 log.jsonl
}
```

启动时自动补填所有已交互过的频道的历史消息（最多 3 页）。确保 log.jsonl 的完整性。

### 3.6 日志系统（log.jsonl）

所有消息（用户消息、Bot 响应）都记录到 `{channelDir}/log.jsonl`：

```jsonl
{"date":"2025-01-15T10:00:00Z","ts":"1736933400.000001","user":"U123","userName":"mario","text":"check emails","isBot":false}
{"date":"2025-01-15T10:00:05Z","ts":"1736933405.000002","user":"bot","text":"Checked inbox, 3 new emails...","isBot":true}
```

注意：**不包含 tool call 结果**。log.jsonl 是人类可读的对话历史，LLM 可以用 `grep` 搜索。

---

## 4. main.ts —— SlackContext 适配器

### 4.1 per-channel 状态

```typescript
interface ChannelState {
    running: boolean;
    runner: AgentRunner;
    store: ChannelStore;
    stopRequested: boolean;
    stopMessageTs?: string;
}

const channelStates = new Map<string, ChannelState>();
```

每个频道独立的状态和 AgentRunner。

### 4.2 createSlackContext()

`createSlackContext()` 创建一个适配器对象，将 Agent 事件转为 Slack API 操作：

```typescript
return {
    respond: async (text) => {
        // 第一次调用 → postMessage（创建新消息）
        // 后续调用 → updateMessage（追加到同一条消息）
        // 文本累加到 accumulatedText
        // 正在工作时自动追加 " ..." 指示器
    },
    respondInThread: async (text) => {
        // 在主消息下发送线程回复
    },
    replaceMessage: async (text) => {
        // 完全替换消息内容（用于最终结果）
    },
    setTyping: async (isTyping) => {
        // 发送 "_Thinking_" 占位消息
    },
    setWorking: async (working) => {
        // 更新 " ..." 工作指示器
    },
    uploadFile: async (filePath, title) => {
        // 上传文件到 Slack
    },
    deleteMessage: async () => {
        // 删除主消息和所有线程回复（用于 [SILENT] 响应）
    },
};
```

关键设计：**所有 Slack API 调用通过 Promise 链（`updatePromise`）串行化**，防止 Slack API 竞态条件（更新顺序错乱）。

---

## 5. AgentRunner（agent.ts）

### 5.1 getOrCreateRunner —— 缓存复用

```typescript
const channelRunners = new Map<string, AgentRunner>();

function getOrCreateRunner(sandboxConfig, channelId, channelDir): AgentRunner {
    const existing = channelRunners.get(channelId);
    if (existing) return existing;

    const runner = createRunner(sandboxConfig, channelId, channelDir);
    channelRunners.set(channelId, runner);
    return runner;
}
```

每个频道只创建一次 Runner，跨消息复用。Agent Session 和会话历史跨请求持久。

### 5.2 createRunner —— 核心初始化

```
createRunner(sandboxConfig, channelId, channelDir)
├── createExecutor(sandboxConfig)         → Host/Docker 执行器
├── createMomTools(executor)              → 5 个工具（read, bash, edit, write, attach）
├── getMemory(channelDir)                 → 读取 MEMORY.md
├── loadMomSkills(channelDir, workspace)  → 加载 Skills
├── buildSystemPrompt(...)                → 构建 System Prompt
├── SessionManager.open(contextFile)      → 打开 context.jsonl
├── AuthStorage.create(~/.pi/mom/auth.json) → 认证存储
├── new Agent({model: claude-sonnet-4-5}) → 创建 Agent
├── sessionManager.buildSessionContext()  → 恢复历史消息
├── new AgentSession({agent, ...})        → 创建 AgentSession
└── session.subscribe(eventHandler)       → 订阅事件（一次性）
```

### 5.3 事件处理 → Slack 输出

Runner 在创建时**一次性**订阅 AgentSession 事件：

```
tool_execution_start → ctx.respond("_→ label_")       (主消息追加工具标签)
tool_execution_end   → ctx.respondInThread(结果)        (线程发送详细结果)
message_end          → ctx.respond(text) / ctx.respondInThread(text)
auto_compaction_start → ctx.respond("_Compacting..._")
auto_retry_start     → ctx.respond("_Retrying..._")
```

Agent 的每个 turn 在 Slack 中的呈现：

```
Slack 主消息（不断追加更新）：
┌─────────────────────────────────────┐
│ _→ Reading config.json_             │ ← tool_execution_start
│ _→ Running npm install_             │ ← tool_execution_start
│ All dependencies installed.         │ ← message_end (final text)
└─────────────────────────────────────┘

Slack 线程（每个工具一条）：
├── ✓ read: Reading config.json (0.3s)
│   ```config.json:1-25```
│   *Result:* ```{"name": "..."}```
├── ✓ bash: Running npm install (12.5s)
│   ```npm install```
│   *Result:* ```added 150 packages...```
└── Usage: 15K in, 1.2K out, $0.05
```

### 5.4 run() 方法 —— 每次请求的执行流程

```
run(ctx, store)
├── 1. syncLogToSessionManager()     → log.jsonl → context.jsonl 同步
├── 2. sessionManager.buildSessionContext() → 重建上下文
├── 3. agent.replaceMessages()       → 刷新 Agent 消息列表
├── 4. 重建 System Prompt（刷新 memory/channels/users/skills）
├── 5. setUploadFunction()           → 配置 attach 工具的上传函数
├── 6. 构建用户消息
│   └── "[2025-01-15 10:00:00+01:00] [mario]: check emails"
│       + 图片附件（base64 ImageContent）
│       + 非图片附件（<slack_attachments> 路径列表）
├── 7. session.prompt(userMessage)   → 进入 Agent Loop
├── 8. 等待所有 Slack 消息队列完成
├── 9. 最终消息处理
│   ├── error → replaceMessage("_Sorry, something went wrong_")
│   ├── [SILENT] → deleteMessage()（静默完成）
│   └── normal → replaceMessage(finalText)
└── 10. 发送使用统计（线程）
```

### 5.5 [SILENT] 静默响应

```typescript
if (finalText.trim() === "[SILENT]" || finalText.trim().startsWith("[SILENT]")) {
    await ctx.deleteMessage();
}
```

当 Agent 对定期事件检查后没有可报告的内容时，返回 `[SILENT]`，Bot 会删除所有中间消息，不在频道中留下任何痕迹。

### 5.6 Context 同步：双文件模型

```
log.jsonl     (所有对话历史，人类可读)
    │
    │ syncLogToSessionManager()
    ▼
context.jsonl (LLM 上下文，结构化 JSONL)
```

**为什么需要两个文件？**

- `log.jsonl`：记录所有频道消息（包括 Bot 不在线时的消息），LLM 可以用 grep 搜索历史
- `context.jsonl`：结构化的 Agent 消息历史（含 tool results），用于 LLM 上下文

`syncLogToSessionManager()` 在每次 run() 开始时执行：
1. 读取 `log.jsonl` 中的用户消息
2. 比对 `context.jsonl` 中已有的消息（通过文本去重）
3. 将缺失的消息追加到 context.jsonl

这确保了即使 Bot 离线期间的频道消息也会进入 LLM 上下文。

---

## 6. 沙箱系统（sandbox.ts）

### 6.1 Executor 接口

```typescript
interface Executor {
    exec(command: string, options?: ExecOptions): Promise<ExecResult>;
    getWorkspacePath(hostPath: string): string;
}
```

### 6.2 HostExecutor —— 本地执行

```typescript
class HostExecutor implements Executor {
    async exec(command, options): Promise<ExecResult> {
        const child = spawn("sh", ["-c", command], { detached: true });
        // 超时处理、AbortSignal、输出收集（限制 10MB）
        // 使用 process group kill 确保子进程树被终止
    }
    getWorkspacePath(hostPath): string { return hostPath; }
}
```

关键特性：
- **Process Group Kill**：`spawn` 使用 `detached: true`，通过 `process.kill(-pid, "SIGKILL")` 杀死整个进程组
- **输出限制**：stdout/stderr 各限制 10MB，防止内存溢出
- **超时处理**：可配置的超时，超时后 kill 进程树

### 6.3 DockerExecutor —— Docker 容器执行

```typescript
class DockerExecutor implements Executor {
    async exec(command, options): Promise<ExecResult> {
        const dockerCmd = `docker exec ${this.container} sh -c ${shellEscape(command)}`;
        return new HostExecutor().exec(dockerCmd, options);
    }
    getWorkspacePath(): string { return "/workspace"; }
}
```

DockerExecutor 通过 `docker exec` 将命令发送到预创建的容器。路径映射：`hostPath` → `/workspace`。

### 6.4 启动验证

```typescript
async function validateSandbox(config: SandboxConfig): Promise<void> {
    // 检查 Docker 是否安装
    await execSimple("docker", ["--version"]);
    // 检查容器是否运行
    const result = await execSimple("docker", ["inspect", "-f", "{{.State.Running}}", config.container]);
    if (result.trim() !== "true") { /* 错误退出 */ }
}
```

---

## 7. 事件系统（events.ts）

### 7.1 三种事件类型

| 类型 | 触发方式 | 使用场景 | 触发后 |
|------|----------|----------|--------|
| `immediate` | 文件创建即触发 | 外部 webhook/脚本 | 自动删除 |
| `one-shot` | 指定时间触发 | 提醒、一次性任务 | 自动删除 |
| `periodic` | Cron 表达式循环 | 定期检查、报告 | 保留文件 |

### 7.2 EventsWatcher

```typescript
class EventsWatcher {
    private timers: Map<string, NodeJS.Timeout>;   // one-shot 定时器
    private crons: Map<string, Cron>;              // periodic Cron 任务
    private watcher: FSWatcher;                     // 文件监视器

    start() {
        this.scanExisting();      // 扫描已有事件文件
        this.watcher = watch(eventsDir, (_, filename) => {
            this.debounce(filename, () => this.handleFileChange(filename));
        });
    }
}
```

**运作流程**：

1. `fs.watch()` 监视 `{workspace}/events/` 目录
2. 新文件创建 → `debounce(100ms)` → 解析 JSON → 按类型处理
3. `immediate` → 立即执行（检查是否比启动时间新）
4. `one-shot` → `setTimeout(delay)` 定时执行
5. `periodic` → `new Cron(schedule, timezone, callback)` 循环执行

### 7.3 事件执行

```typescript
private execute(filename, event, deleteAfter = true): void {
    // 构造合成 SlackEvent
    const message = `[EVENT:${filename}:${event.type}:${scheduleInfo}] ${event.text}`;
    const syntheticEvent: SlackEvent = {
        type: "mention",
        channel: event.channelId,
        user: "EVENT",
        text: message,
        ts: Date.now().toString(),
    };

    // 通过 SlackBot 排队
    this.slack.enqueueEvent(syntheticEvent);

    // 立即/一次性事件自动删除文件
    if (deleteAfter) this.deleteFile(filename);
}
```

事件通过构造**合成 SlackEvent** 注入处理管道，与用户消息走相同的 Agent 处理流程。

---

## 8. 工具系统

### 8.1 5 个内置工具

| 工具 | 说明 | 特殊之处 |
|------|------|----------|
| `read` | 文件读取 | 通过 Executor 在沙箱中执行 `cat` |
| `write` | 文件写入 | 通过 Executor 在沙箱中执行 `tee` |
| `edit` | 文件编辑 | Search/Replace，通过 Executor |
| `bash` | Shell 命令 | 通过 Executor，支持超时和取消 |
| `attach` | Slack 上传 | 直接调用 Slack API，不经过 Executor |

### 8.2 bash 工具详解

```typescript
execute: async (_id, { command, timeout }, signal?) => {
    const result = await executor.exec(command, { timeout, signal });

    // 大输出截断（保留尾部，最后 500 行或 50KB）
    const truncation = truncateTail(output);

    // 超大输出保存到临时文件
    if (totalBytes > DEFAULT_MAX_BYTES) {
        tempFilePath = getTempFilePath();  // /tmp/mom-bash-xxxxx.log
        createWriteStream(tempFilePath).write(output);
    }

    if (result.code !== 0) {
        throw new Error(`output\n\nCommand exited with code ${result.code}`);
    }

    return { content: [{ type: "text", text: outputText }], details };
}
```

### 8.3 attach 工具 —— 运行时注入

```typescript
let uploadFn: ((filePath: string, title?: string) => Promise<void>) | null = null;

export function setUploadFunction(fn): void {
    uploadFn = fn;
}
```

`attach` 工具的 `uploadFn` 在每次 `run()` 时动态注入。这是因为上传需要 Slack 频道上下文（不同频道不同），而工具在 Runner 创建时就已注册。

---

## 9. System Prompt

`buildSystemPrompt()` 构建了一个非常详细的 System Prompt，包含：

1. **身份**："You are mom, a Slack bot assistant"
2. **Slack 格式指南**：mrkdwn 格式（非 Markdown）
3. **Slack ID 映射**：所有频道和用户的 ID ↔ 名称映射
4. **环境描述**：Host 或 Docker 容器信息
5. **工作区布局**：文件目录结构说明
6. **Skills 系统**：如何创建和使用自定义 CLI 工具
7. **事件系统**：三种事件类型的创建/管理指南 + Cron 格式说明
8. **记忆系统**：何时更新 MEMORY.md
9. **系统配置日志**：SYSTEM.md 用途说明
10. **日志查询**：log.jsonl 的 grep/jq 查询示例
11. **工具列表**：5 个工具的简述
12. **当前记忆**：MEMORY.md 当前内容

---

## 10. 记忆系统

### 10.1 双层记忆

```
{workspace}/
├── MEMORY.md          # 全局记忆（所有频道共享）
└── {channelId}/
    └── MEMORY.md      # 频道级记忆
```

Agent 在每次运行时读取记忆并注入 System Prompt。Agent 可以主动更新 MEMORY.md 来持久化重要信息。

### 10.2 getMemory()

```typescript
function getMemory(channelDir: string): string {
    // 读取全局 MEMORY.md
    const workspaceMemory = readFileSync(join(channelDir, "..", "MEMORY.md"));
    // 读取频道 MEMORY.md
    const channelMemory = readFileSync(join(channelDir, "MEMORY.md"));

    return `### Global Workspace Memory\n${workspaceMemory}\n\n### Channel-Specific Memory\n${channelMemory}`;
}
```

---

## 11. Skills 系统

### 11.1 Skills 发现

```typescript
function loadMomSkills(channelDir, workspacePath): Skill[] {
    const skillMap = new Map<string, Skill>();

    // 全局 Skills：{workspace}/skills/
    for (const skill of loadSkillsFromDir({ dir: workspaceSkillsDir })) {
        skillMap.set(skill.name, skill);
    }

    // 频道 Skills：{channelDir}/skills/（优先级更高，覆盖同名全局 Skill）
    for (const skill of loadSkillsFromDir({ dir: channelSkillsDir })) {
        skillMap.set(skill.name, skill);
    }

    return Array.from(skillMap.values());
}
```

频道级 Skills 覆盖同名全局 Skills。

### 11.2 路径翻译

```typescript
const translatePath = (hostPath: string): string => {
    if (hostPath.startsWith(hostWorkspacePath)) {
        return workspacePath + hostPath.slice(hostWorkspacePath.length);
    }
    return hostPath;
};
```

Docker 模式下，Skill 的 `filePath` 和 `baseDir` 从 Host 路径翻译为容器内路径。

---

## 12. ChannelStore（store.ts）

### 12.1 附件处理

```
Slack message with files
    │
    ▼
processAttachments(channelId, files, timestamp)
├── 生成唯一文件名：{timestamp}_{sanitized_name}
├── 记录到 Attachment[] 元数据
├── 加入下载队列
└── processDownloadQueue()（后台异步下载）
    └── fetch(url, { Authorization: Bearer botToken })
        → writeFile({channelDir}/attachments/{filename})
```

下载使用 Bot Token 认证（Slack 私有文件需要认证）。

### 12.2 去重机制

```typescript
private recentlyLogged = new Map<string, number>();

async logMessage(channelId, message): Promise<boolean> {
    const dedupeKey = `${channelId}:${message.ts}`;
    if (this.recentlyLogged.has(dedupeKey)) return false;

    this.recentlyLogged.set(dedupeKey, Date.now());
    setTimeout(() => this.recentlyLogged.delete(dedupeKey), 60000);  // 60s 后自动清理
}
```

使用 `channelId:ts` 作为去重键，60 秒窗口防止重复写入。

---

## 13. 与其他包的关系

```
@mariozechner/pi-ai
    │
    ▼
@mariozechner/pi-agent-core
    │
    ▼
@mariozechner/pi-coding-agent
    │
    ▼
@mariozechner/pi-mom ←── 本包
```

Mom 是整个包层次的**最上层应用**之一（与 pi-web-ui 平行）。它复用了：
- `pi-ai`：LLM API（硬编码 claude-sonnet-4-5）
- `pi-agent-core`：Agent 类 + 事件系统
- `pi-coding-agent`：AgentSession（自动压缩、重试）、SessionManager（JSONL 持久化）、Skills 系统、convertToLlm

---

## 14. 源码阅读建议

### 推荐阅读顺序

1. **main.ts** → 理解启动流程和 handler 结构
2. **slack.ts** 的 `SlackBot.start()` 和 `setupEventHandlers()` → 理解 Slack 事件接收
3. **slack.ts** 的 `ChannelQueue` → 理解消息串行化
4. **main.ts** 的 `createSlackContext()` → 理解 Slack API 适配器
5. **agent.ts** 的 `createRunner()` → 理解 Runner 初始化
6. **agent.ts** 的 `run()` → 理解每次请求的执行流程
7. **agent.ts** 的 `session.subscribe()` → 理解事件到 Slack 消息的映射
8. **sandbox.ts** → 理解 Host/Docker 执行模型
9. **events.ts** → 理解定时事件系统
10. **context.ts** 的 `syncLogToSessionManager()` → 理解双文件同步
11. **store.ts** → 理解附件下载和日志去重

### 关键断点位置

| 文件 | 位置 | 说明 |
|------|------|------|
| main.ts | `handler.handleEvent()` | 消息处理入口 |
| main.ts | `createSlackContext()` | Slack 适配器创建 |
| agent.ts | `createRunner()` | Agent 初始化 |
| agent.ts | `run()` | 每次请求执行 |
| agent.ts | `session.subscribe()` | 事件处理逻辑 |
| slack.ts | `setupEventHandlers()` | Slack 事件路由 |
| slack.ts | `ChannelQueue.processNext()` | 消息队列处理 |
| events.ts | `EventsWatcher.execute()` | 事件触发 |
| context.ts | `syncLogToSessionManager()` | 上下文同步 |
| sandbox.ts | `HostExecutor.exec()` | 命令执行 |

### 核心设计思想总结

1. **频道隔离**：每个频道独立的 State、Runner、Queue、Store
2. **串行化**：ChannelQueue 保证频道内消息串行、Promise 链保证 Slack API 调用有序
3. **双文件上下文**：log.jsonl（人类可读历史）+ context.jsonl（LLM 结构化上下文）
4. **合成事件**：定时事件通过合成 SlackEvent 复用消息处理管道
5. **沙箱抽象**：Executor 接口隔离 Host/Docker 差异
6. **自治能力**：Memory + Events + Skills 让 Bot 可以自我管理、学习和定时执行
7. **[SILENT] 静默模式**：定期事件无事可报告时删除所有中间消息
8. **运行时注入**：uploadFn 在每次 run() 时动态绑定频道上下文
