# Pi-agent-core (@mariozechner/pi-agent-core) 源码详细解读

## 1. 包概述

`@mariozechner/pi-agent-core` 是 Pi Monorepo 中的 **Agent 运行时层**，构建在 `@mariozechner/pi-ai` 之上。它的核心职责是：

- 实现 **Agent Loop**（Prompt → LLM → Tool Execution → Loop）循环
- 提供 **双层消息模型**（AgentMessage vs LLM Message）
- 管理 **工具执行**（顺序 / 并行两种模式）
- 提供 **事件驱动架构**，实时通知 UI 层
- 支持 **Steering / Follow-up** 消息队列，允许运行时注入指令
- 提供 **Proxy 代理流** 用于浏览器端通过后端服务器中转 LLM 请求

### 1.1 依赖关系

```
@mariozechner/pi-agent-core
    └── @mariozechner/pi-ai (唯一运行时依赖)
```

只依赖 `pi-ai`，无其他第三方运行时依赖。开发依赖包括 `vitest`、`typescript`、`@types/node`。

### 1.2 文件结构

```
packages/agent/
├── src/
│   ├── index.ts          # 统一导出
│   ├── types.ts          # 核心类型定义（~310 行）
│   ├── agent.ts          # Agent 类（~613 行）
│   ├── agent-loop.ts     # Agent Loop 引擎（~616 行）
│   └── proxy.ts          # Proxy 流函数（~340 行）
├── test/
│   ├── utils/
│   │   ├── calculate.ts       # 测试用计算器工具
│   │   └── get-current-time.ts # 测试用时间工具
│   ├── agent-loop.test.ts     # agent-loop 底层 API 单元测试
│   ├── agent.test.ts          # Agent 类单元测试
│   ├── bedrock-models.test.ts # Bedrock 多模型兼容性测试
│   ├── bedrock-utils.ts       # Bedrock 凭证检测工具
│   └── e2e.test.ts            # 多 Provider 端到端测试
├── package.json
├── vitest.config.ts
└── CHANGELOG.md
```

整个包只有 **4 个源码文件**，代码量约 ~1900 行，但设计精巧、职责分明。

---

## 2. 核心类型系统（types.ts）

`types.ts` 是整个包的类型基石，定义了 Agent 运行时的所有关键接口。

### 2.1 StreamFn —— 流函数签名

```typescript
export type StreamFn = (
    ...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple> | Promise<ReturnType<typeof streamSimple>>;
```

**设计要点**：
- 复用 `pi-ai` 的 `streamSimple` 签名，保持 API 一致性
- 返回值可以是同步的 `EventStream` 也可以是 `Promise<EventStream>`
- **契约**：不能抛出异常，失败必须通过流事件 (`error` stop reason) 传递

这个类型是 Agent 与 LLM 通信的唯一接口，无论是直接调用还是走 Proxy，都通过这个签名。

### 2.2 双层消息模型

这是整个包最核心的设计思想：

```
AgentMessage（应用层）
    │
    │ convertToLlm()
    ▼
Message（LLM 层：user / assistant / toolResult）
```

#### AgentMessage

```typescript
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

`AgentMessage` 是 LLM 标准消息（`Message`）与自定义消息的联合类型。通过 TypeScript 的 **声明合并（Declaration Merging）** 机制扩展：

```typescript
export interface CustomAgentMessages {
    // 空接口，应用通过声明合并扩展
}

// 应用端扩展示例
declare module "@mariozechner/pi-agent-core" {
    interface CustomAgentMessages {
        notification: { role: "notification"; text: string; timestamp: number };
        artifact: { role: "artifact"; content: string; language: string; timestamp: number };
    }
}
```

**为什么需要双层消息？**

LLM 只理解 `user`、`assistant`、`toolResult` 三种角色。但实际应用中，消息列表中可能包含通知、状态指示、Artifact 等 UI 专用消息。双层模型让应用可以在消息列表中混入任意自定义消息，由 `convertToLlm` 函数负责在发送给 LLM 前过滤/转换。

### 2.3 AgentState —— 完整的 Agent 状态快照

```typescript
export interface AgentState {
    systemPrompt: string;
    model: Model<any>;
    thinkingLevel: ThinkingLevel;
    tools: AgentTool<any>[];
    messages: AgentMessage[];
    isStreaming: boolean;
    streamMessage: AgentMessage | null;
    pendingToolCalls: Set<string>;
    error?: string;
}
```

| 字段 | 说明 |
|------|------|
| `systemPrompt` | 系统提示词 |
| `model` | 当前使用的 LLM 模型 |
| `thinkingLevel` | 推理深度级别：off / minimal / low / medium / high / xhigh |
| `tools` | 注册的工具列表 |
| `messages` | 完整的对话历史（AgentMessage 层） |
| `isStreaming` | 是否正在流式处理中 |
| `streamMessage` | 流式过程中的部分消息（partial message） |
| `pendingToolCalls` | 正在执行的工具调用 ID 集合 |
| `error` | 最后一次错误信息 |

### 2.4 AgentTool —— 工具定义

```typescript
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
    label: string;
    execute: (
        toolCallId: string,
        params: Static<TParameters>,
        signal?: AbortSignal,
        onUpdate?: AgentToolUpdateCallback<TDetails>,
    ) => Promise<AgentToolResult<TDetails>>;
}
```

`AgentTool` 继承自 `pi-ai` 的 `Tool`（定义了 `name`、`description`、`parameters`），新增：

- **`label`**：人类可读标签，用于 UI 展示
- **`execute()`**：实际执行函数，返回 `AgentToolResult`

执行函数的四个参数：
1. `toolCallId` —— LLM 分配的工具调用 ID
2. `params` —— 经过 TypeBox 校验的参数（类型安全）
3. `signal` —— AbortSignal，支持取消
4. `onUpdate` —— 流式进度回调，工具可以在执行过程中推送中间结果

### 2.5 AgentToolResult

```typescript
export interface AgentToolResult<T> {
    content: (TextContent | ImageContent)[];
    details: T;
}
```

工具执行结果分为两部分：
- **`content`**：发送给 LLM 的结果内容（文本或图片）
- **`details`**：发送给 UI 的详细数据（泛型 T，可以是任意结构化数据）

这个分离让 LLM 看到的是精简的文本摘要，而 UI 可以展示丰富的结构化信息（如文件树、diff 视图等）。

### 2.6 AgentLoopConfig —— 循环配置

```typescript
export interface AgentLoopConfig extends SimpleStreamOptions {
    model: Model<any>;
    convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
    transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
    getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;
    getSteeringMessages?: () => Promise<AgentMessage[]>;
    getFollowUpMessages?: () => Promise<AgentMessage[]>;
    toolExecution?: ToolExecutionMode;
    beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>;
    afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>;
}
```

关键配置项：

| 配置 | 说明 |
|------|------|
| `convertToLlm` | **必需**。将 AgentMessage[] 转换为 LLM 可理解的 Message[] |
| `transformContext` | 可选。在 convertToLlm 前对上下文进行变换（裁剪、注入） |
| `getApiKey` | 可选。动态解析 API Key（用于会过期的 OAuth Token） |
| `getSteeringMessages` | 可选。获取运行时注入的 Steering 消息 |
| `getFollowUpMessages` | 可选。获取 Agent 停止后的后续消息 |
| `toolExecution` | 工具执行模式："sequential" 或 "parallel"（默认） |
| `beforeToolCall` | 工具执行前钩子，可阻止执行 |
| `afterToolCall` | 工具执行后钩子，可修改结果 |

### 2.7 AgentEvent —— 事件类型

```typescript
export type AgentEvent =
    | { type: "agent_start" }
    | { type: "agent_end"; messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
    | { type: "message_end"; message: AgentMessage }
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

事件体系分为三个层级：

1. **Agent 生命周期**：`agent_start` → `agent_end`
2. **Turn 生命周期**：`turn_start` → `turn_end`（一个 Turn = 一次 LLM 调用 + 对应的工具执行）
3. **消息/工具生命周期**：`message_start/update/end`、`tool_execution_start/update/end`

### 2.8 Tool Call 钩子

```typescript
export interface BeforeToolCallResult {
    block?: boolean;
    reason?: string;
}

export interface AfterToolCallResult {
    content?: (TextContent | ImageContent)[];
    details?: unknown;
    isError?: boolean;
}
```

- **`beforeToolCall`**：在工具执行前调用，可返回 `{ block: true }` 阻止执行（如权限控制）
- **`afterToolCall`**：在工具执行后调用，可覆盖 `content`、`details`、`isError`（如审计、结果过滤）

---

## 3. Agent Loop 引擎（agent-loop.ts）

这是整个包的核心——Agent 循环引擎。它实现了 "Prompt → LLM → Tool Execution → Loop" 的完整循环。

### 3.1 公开 API

提供两对 API，分别对应 EventStream 接口和 Async 接口：

#### EventStream 接口（observational）

```typescript
function agentLoop(prompts, context, config, signal?, streamFn?): EventStream<AgentEvent, AgentMessage[]>
function agentLoopContinue(context, config, signal?, streamFn?): EventStream<AgentEvent, AgentMessage[]>
```

返回 `EventStream`，可以用 `for await...of` 消费。注意 README 中的警告：这些流是 **observational** 的——事件处理不构成 barrier，不能保证你的异步 handler 在下一阶段开始前完成。

#### Async 接口（barrier 语义）

```typescript
async function runAgentLoop(prompts, context, config, emit, signal?, streamFn?): Promise<AgentMessage[]>
async function runAgentLoopContinue(context, config, emit, signal?, streamFn?): Promise<AgentMessage[]>
```

接收 `emit` 回调作为事件接收器。`Agent` 类内部使用的就是这对 API，因为它需要 barrier 语义——在 `message_end` 处理完成后才开始工具执行。

### 3.2 核心循环：runLoop()

```
runLoop()
├── 外层 while(true)：处理 Follow-up 消息
│   ├── 内层 while：处理 Tool Calls + Steering 消息
│   │   ├── 注入 pending messages
│   │   ├── streamAssistantResponse()  → 获取 LLM 响应
│   │   ├── 检查 error / aborted → 提前退出
│   │   ├── executeToolCalls()  → 执行工具
│   │   ├── emit turn_end
│   │   └── 检查 Steering 消息 → 作为 pendingMessages 进入下一轮
│   └── 检查 Follow-up 消息 → 作为 pendingMessages 继续外层循环
└── emit agent_end
```

关键代码逻辑：

```typescript
async function runLoop(currentContext, newMessages, config, signal, emit, streamFn?) {
    let firstTurn = true;
    let pendingMessages = (await config.getSteeringMessages?.()) || [];

    // 外层循环：Follow-up
    while (true) {
        let hasMoreToolCalls = true;

        // 内层循环：Tool Calls + Steering
        while (hasMoreToolCalls || pendingMessages.length > 0) {
            // 1. 注入 pending messages
            if (pendingMessages.length > 0) {
                for (const message of pendingMessages) {
                    emit({ type: "message_start", message });
                    emit({ type: "message_end", message });
                    currentContext.messages.push(message);
                    newMessages.push(message);
                }
                pendingMessages = [];
            }

            // 2. 获取 LLM 响应
            const message = await streamAssistantResponse(...);

            // 3. 错误/中止 → 退出
            if (message.stopReason === "error" || message.stopReason === "aborted") {
                emit({ type: "turn_end", message, toolResults: [] });
                emit({ type: "agent_end", messages: newMessages });
                return;
            }

            // 4. 执行工具
            const toolCalls = message.content.filter(c => c.type === "toolCall");
            hasMoreToolCalls = toolCalls.length > 0;
            if (hasMoreToolCalls) {
                const toolResults = await executeToolCalls(...);
                // 工具结果加入上下文
            }

            emit({ type: "turn_end", message, toolResults });

            // 5. 检查 Steering 消息
            pendingMessages = (await config.getSteeringMessages?.()) || [];
        }

        // 6. 检查 Follow-up 消息
        const followUpMessages = (await config.getFollowUpMessages?.()) || [];
        if (followUpMessages.length > 0) {
            pendingMessages = followUpMessages;
            continue;
        }
        break;
    }

    emit({ type: "agent_end", messages: newMessages });
}
```

### 3.3 LLM 调用：streamAssistantResponse()

```
streamAssistantResponse()
├── transformContext()    → AgentMessage[] → AgentMessage[]（可选裁剪）
├── convertToLlm()       → AgentMessage[] → Message[]（过滤自定义消息）
├── 构建 LLM Context     → { systemPrompt, messages, tools }
├── 解析 API Key         → getApiKey() 动态获取
├── streamFn()           → 调用 LLM
└── 处理流事件
    ├── start            → push partial message, emit message_start
    ├── text_delta/...   → 更新 partial, emit message_update
    └── done/error       → 替换为 final message, emit message_end
```

关键细节：

1. **Partial Message 管理**：在 `start` 事件时，将 partial message push 到 `context.messages` 数组。后续每个 delta 事件更新这个位置。`done` 时替换为最终消息。这确保上下文始终反映最新状态。

2. **消息转换流水线**：`transformContext` → `convertToLlm` 是两步管道。`transformContext` 在 AgentMessage 层面操作（如裁剪旧消息），`convertToLlm` 将结果转为 LLM 格式。

3. **动态 API Key**：`getApiKey` 在每次 LLM 调用前执行，支持短命 OAuth Token（如 GitHub Copilot）。

### 3.4 工具执行：executeToolCalls()

工具执行分为两种模式：

#### 顺序模式（sequential）

```
executeToolCallsSequential()
├── for each toolCall:
│   ├── emit tool_execution_start
│   ├── prepareToolCall()     → 查找工具、校验参数、执行 beforeToolCall
│   ├── executePreparedToolCall()  → 实际执行
│   ├── finalizeExecutedToolCall() → 执行 afterToolCall
│   └── emit tool_execution_end
```

#### 并行模式（parallel，默认）

```
executeToolCallsParallel()
├── Phase 1: 顺序预检
│   for each toolCall:
│   ├── emit tool_execution_start
│   ├── prepareToolCall()     → 查找工具、校验参数、执行 beforeToolCall
│   └── 若立即结果（错误/阻止）→ 直接 emit end
│       否则 → 加入 runnableCalls
│
├── Phase 2: 并行执行
│   const running = runnableCalls.map(p => executePreparedToolCall(p))
│
├── Phase 3: 按源顺序收集结果
│   for each running:
│   ├── await execution
│   ├── finalizeExecutedToolCall()
│   └── emit tool_execution_end
```

**并行模式的设计精妙之处**：

1. **预检顺序化**：`beforeToolCall` 按顺序执行，确保权限检查不会互相干扰
2. **执行并行化**：通过 `map` 创建所有 Promise，然后按顺序 `await`
3. **结果保序**：虽然执行是并行的，但最终的 `tool_execution_end` 和 `toolResult` 消息严格按照 assistant 消息中 toolCall 的出现顺序发射

### 3.5 工具执行管道细节

工具执行经过三个阶段：

#### 阶段 1：prepareToolCall()

```typescript
async function prepareToolCall(context, assistantMessage, toolCall, config, signal)
    : Promise<PreparedToolCall | ImmediateToolCallOutcome>
```

1. 查找工具（`context.tools?.find(t => t.name === toolCall.name)`）
2. 若工具不存在 → 返回 `ImmediateToolCallOutcome`（错误）
3. 使用 `validateToolArguments(tool, toolCall)` 校验参数（TypeBox 校验）
4. 执行 `beforeToolCall` 钩子 → 若返回 `{ block: true }` → 返回 `ImmediateToolCallOutcome`
5. 否则返回 `PreparedToolCall`（准备就绪）

#### 阶段 2：executePreparedToolCall()

```typescript
async function executePreparedToolCall(prepared, signal, emit)
    : Promise<ExecutedToolCallOutcome>
```

调用 `tool.execute()`，传入 `onUpdate` 回调用于流式进度。异常被捕获并转为错误结果。

#### 阶段 3：finalizeExecutedToolCall()

```typescript
async function finalizeExecutedToolCall(context, assistantMessage, prepared, executed, config, signal, emit)
    : Promise<ToolResultMessage>
```

执行 `afterToolCall` 钩子，允许修改结果的 `content`、`details`、`isError`。然后发射 `tool_execution_end` 事件和 `toolResult` 消息事件。

### 3.6 EventStream 封装

```typescript
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
    return new EventStream<AgentEvent, AgentMessage[]>(
        (event) => event.type === "agent_end",
        (event) => (event.type === "agent_end" ? event.messages : []),
    );
}
```

使用 `pi-ai` 的 `EventStream` 类，配置终止条件为 `agent_end` 事件，结果提取为 `agent_end.messages`。

---

## 4. Agent 类（agent.ts）

`Agent` 是面向应用层的高级封装，在 `agent-loop` 之上提供状态管理、事件订阅、消息队列等能力。

### 4.1 构造与初始化

```typescript
export class Agent {
    private _state: AgentState = {
        systemPrompt: "",
        model: getModel("google", "gemini-2.5-flash-lite-preview-06-17"),
        thinkingLevel: "off",
        tools: [],
        messages: [],
        isStreaming: false,
        streamMessage: null,
        pendingToolCalls: new Set<string>(),
        error: undefined,
    };

    constructor(opts: AgentOptions = {}) {
        this._state = { ...this._state, ...opts.initialState };
        this.convertToLlm = opts.convertToLlm || defaultConvertToLlm;
        this.streamFn = opts.streamFn || streamSimple;
        // ...
    }
}
```

默认使用 Google Gemini Flash Lite 模型，`convertToLlm` 默认只保留标准 LLM 消息（user/assistant/toolResult）。

### 4.2 事件系统

```typescript
private listeners = new Set<(e: AgentEvent) => void>();

subscribe(fn: (e: AgentEvent) => void): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);
}

private emit(e: AgentEvent) {
    for (const listener of this.listeners) {
        listener(e);
    }
}
```

简洁的发布-订阅模式。`subscribe` 返回取消订阅函数。

### 4.3 Prompt 方法

```typescript
async prompt(message: AgentMessage | AgentMessage[]): Promise<void>;
async prompt(input: string, images?: ImageContent[]): Promise<void>;
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]) {
    if (this._state.isStreaming) {
        throw new Error("Agent is already processing a prompt...");
    }
    // ... 构建消息
    await this._runLoop(msgs);
}
```

支持三种调用方式：
1. `prompt("text")` —— 纯文本
2. `prompt("text", [images])` —— 文本 + 图片
3. `prompt(agentMessage)` —— 直接传入 AgentMessage

**互斥锁**：如果 `isStreaming` 为 true，直接抛出异常。用户应该使用 `steer()` / `followUp()` 队列消息。

### 4.4 Continue 方法

```typescript
async continue() {
    if (this._state.isStreaming) throw new Error("...");

    const messages = this._state.messages;
    if (messages.length === 0) throw new Error("No messages to continue from");

    if (messages[messages.length - 1].role === "assistant") {
        // 特殊处理：assistant 结尾时，尝试消费队列消息
        const queuedSteering = this.dequeueSteeringMessages();
        if (queuedSteering.length > 0) {
            await this._runLoop(queuedSteering, { skipInitialSteeringPoll: true });
            return;
        }
        const queuedFollowUp = this.dequeueFollowUpMessages();
        if (queuedFollowUp.length > 0) {
            await this._runLoop(queuedFollowUp);
            return;
        }
        throw new Error("Cannot continue from message role: assistant");
    }

    await this._runLoop(undefined);  // 无消息 = continue 语义
}
```

`continue()` 的语义：
- 最后消息是 `user` 或 `toolResult` → 直接 continue
- 最后消息是 `assistant` → 先消费 steering 队列，再消费 follow-up 队列
- 无队列消息时从 assistant 消息 continue → 抛出错误

### 4.5 Steering / Follow-up 消息队列

```typescript
private steeringQueue: AgentMessage[] = [];
private followUpQueue: AgentMessage[] = [];
private steeringMode: "all" | "one-at-a-time";
private followUpMode: "all" | "one-at-a-time";

steer(m: AgentMessage) { this.steeringQueue.push(m); }
followUp(m: AgentMessage) { this.followUpQueue.push(m); }
```

两种消息队列的区别：

| 特性 | Steering | Follow-up |
|------|----------|-----------|
| 投递时机 | 当前 turn 的工具执行完成后 | Agent 完全停止后 |
| 用途 | 运行时中断/改变方向 | 追加后续任务 |
| 检查时机 | 每个 turn_end 后 | 所有 steering + tool calls 处理完后 |

出队模式：
- **one-at-a-time**（默认）：每次只出队一条消息
- **all**：一次性出队所有消息

```typescript
private dequeueSteeringMessages(): AgentMessage[] {
    if (this.steeringMode === "one-at-a-time") {
        if (this.steeringQueue.length > 0) {
            const first = this.steeringQueue[0];
            this.steeringQueue = this.steeringQueue.slice(1);
            return [first];
        }
        return [];
    }
    const steering = this.steeringQueue.slice();
    this.steeringQueue = [];
    return steering;
}
```

### 4.6 _runLoop 方法

```typescript
private async _runLoop(messages?: AgentMessage[], options?) {
    this.runningPrompt = new Promise<void>(resolve => {
        this.resolveRunningPrompt = resolve;
    });

    this.abortController = new AbortController();
    this._state.isStreaming = true;

    const config: AgentLoopConfig = {
        model,
        convertToLlm: this.convertToLlm,
        transformContext: this.transformContext,
        getApiKey: this.getApiKey,
        getSteeringMessages: async () => this.dequeueSteeringMessages(),
        getFollowUpMessages: async () => this.dequeueFollowUpMessages(),
        // ...
    };

    try {
        if (messages) {
            await runAgentLoop(messages, context, config, async (event) => this._processLoopEvent(event), ...);
        } else {
            await runAgentLoopContinue(context, config, async (event) => this._processLoopEvent(event), ...);
        }
    } catch (err) {
        // 构建错误消息，追加到 messages
    } finally {
        this._state.isStreaming = false;
        this.resolveRunningPrompt?.();
    }
}
```

关键设计：
1. **runningPrompt**：一个 Promise，外部可通过 `waitForIdle()` 等待
2. **AbortController**：每次 loop 创建新的，`abort()` 可取消
3. **事件处理作为 barrier**：`async (event) => this._processLoopEvent(event)` 是 async 回调，`runAgentLoop` 会 `await emit(event)`，确保 `_processLoopEvent` 完成后才继续

### 4.7 _processLoopEvent

```typescript
private _processLoopEvent(event: AgentEvent): void {
    switch (event.type) {
        case "message_start":
            this._state.streamMessage = event.message;
            break;
        case "message_update":
            this._state.streamMessage = event.message;
            break;
        case "message_end":
            this._state.streamMessage = null;
            this.appendMessage(event.message);
            break;
        case "tool_execution_start":
            // 添加到 pendingToolCalls
            break;
        case "tool_execution_end":
            // 从 pendingToolCalls 移除
            break;
        case "turn_end":
            // 检查错误
            break;
        case "agent_end":
            this._state.isStreaming = false;
            break;
    }
    this.emit(event);  // 转发给外部监听器
}
```

Agent 类在内部处理事件来维护状态，然后将同一事件转发给外部监听器。这是 "barrier + relay" 模式。

---

## 5. Proxy 流函数（proxy.ts）

`streamProxy` 为浏览器端应用提供了通过后端服务器中转 LLM 请求的能力。

### 5.1 设计背景

浏览器不能直接调用 LLM API（CORS 限制 + API Key 暴露风险）。解决方案是通过后端服务器代理：

```
Browser → streamProxy() → POST /api/stream → Server → LLM Provider
                          ← SSE ←                ← SSE ←
```

### 5.2 ProxyAssistantMessageEvent

```typescript
export type ProxyAssistantMessageEvent =
    | { type: "start" }
    | { type: "text_delta"; contentIndex: number; delta: string }
    | { type: "done"; reason: "stop" | "length" | "toolUse"; usage: ... }
    | { type: "error"; reason: "aborted" | "error"; errorMessage?: string; usage: ... }
    // ...
```

与标准 `AssistantMessageEvent` 的关键区别：**没有 `partial` 字段**。服务器端裁剪掉 partial message 以减少带宽。客户端自行从 delta 事件重建 partial。

### 5.3 streamProxy 实现

```typescript
export function streamProxy(model, context, options: ProxyStreamOptions): ProxyMessageEventStream {
    const stream = new ProxyMessageEventStream();

    (async () => {
        const partial: AssistantMessage = { /* 初始化空消息 */ };

        const response = await fetch(`${options.proxyUrl}/api/stream`, {
            method: "POST",
            headers: {
                Authorization: `Bearer ${options.authToken}`,
                "Content-Type": "application/json",
            },
            body: JSON.stringify({ model, context, options: { temperature, maxTokens, reasoning } }),
            signal: options.signal,
        });

        const reader = response.body!.getReader();
        let buffer = "";

        while (true) {
            const { done, value } = await reader.read();
            if (done) break;
            buffer += decoder.decode(value, { stream: true });

            // 解析 SSE data: 行
            const lines = buffer.split("\n");
            buffer = lines.pop() || "";
            for (const line of lines) {
                if (line.startsWith("data: ")) {
                    const proxyEvent = JSON.parse(data);
                    const event = processProxyEvent(proxyEvent, partial);
                    if (event) stream.push(event);
                }
            }
        }

        stream.end();
    })();

    return stream;
}
```

### 5.4 processProxyEvent —— 客户端 Partial 重建

```typescript
function processProxyEvent(proxyEvent, partial): AssistantMessageEvent | undefined {
    switch (proxyEvent.type) {
        case "text_delta": {
            const content = partial.content[proxyEvent.contentIndex];
            if (content?.type === "text") {
                content.text += proxyEvent.delta;  // 累加文本
                return { type: "text_delta", contentIndex, delta, partial };
            }
        }
        case "toolcall_delta": {
            const content = partial.content[proxyEvent.contentIndex];
            if (content?.type === "toolCall") {
                (content as any).partialJson += proxyEvent.delta;
                content.arguments = parseStreamingJson((content as any).partialJson) || {};
                return { type: "toolcall_delta", contentIndex, delta, partial };
            }
        }
        // ...
    }
}
```

关键技术：
- **累加式文本重建**：每个 `text_delta` 将 delta 追加到 partial message 的对应 content block
- **流式 JSON 解析**：`toolcall_delta` 使用 `parseStreamingJson`（来自 pi-ai）解析不完整的 JSON

### 5.5 Exhaustive Check

```typescript
default: {
    const _exhaustiveCheck: never = proxyEvent;
    console.warn(`Unhandled proxy event type: ${(proxyEvent as any).type}`);
    return undefined;
}
```

使用 TypeScript 的 `never` 类型实现穷举检查，确保所有事件类型都被处理。

---

## 6. 完整的事件流时序

### 6.1 基本 Prompt（无工具）

```
prompt("Hello")
│
├── agent_start
├── turn_start
│
├── message_start   { role: "user", content: "Hello" }
├── message_end     { role: "user" }
│
│   ← LLM 流式响应 →
├── message_start   { role: "assistant", partial }
├── message_update  { text_delta: "Hi" }
├── message_update  { text_delta: " there" }
├── message_update  { text_delta: "!" }
├── message_end     { role: "assistant", final }
│
├── turn_end        { message: assistant, toolResults: [] }
└── agent_end       { messages: [user, assistant] }
```

### 6.2 带工具调用

```
prompt("Calculate 2+3")
│
├── agent_start
├── turn_start
│
├── message_start/end   { role: "user" }
│
│   ← LLM 返回 toolCall →
├── message_start       { role: "assistant", toolCall: calculate(2+3) }
├── message_update      { toolcall_start, toolcall_delta, toolcall_end }
├── message_end         { role: "assistant", stopReason: "toolUse" }
│
├── tool_execution_start  { toolCallId: "tc1", toolName: "calculate" }
├── tool_execution_end    { toolCallId: "tc1", result: "5", isError: false }
│
├── message_start/end   { role: "toolResult", toolCallId: "tc1" }
├── turn_end            { message: assistant, toolResults: [toolResult] }
│
├── turn_start          ← 新的 Turn
│
│   ← LLM 根据工具结果响应 →
├── message_start       { role: "assistant" }
├── message_update      { text_delta: "The result is 5" }
├── message_end         { role: "assistant" }
│
├── turn_end
└── agent_end
```

### 6.3 带 Steering 消息

```
prompt("Do task A, B, C")
│
├── agent_start
├── turn_start
├── message_start/end   { role: "user" }
│
├── message_start/end   { role: "assistant", toolCall: doTaskA }
├── tool_execution_start/end   { doTaskA }
├── message_start/end   { role: "toolResult" }
├── turn_end
│
│   ← getSteeringMessages() 返回 "Stop! Do X instead" →
│
├── turn_start
├── message_start/end   { role: "user", "Stop! Do X instead" }  ← Steering 注入
│
├── message_start/end   { role: "assistant", "OK, doing X..." }
├── turn_end
└── agent_end
```

---

## 7. 设计模式与架构洞察

### 7.1 消息转换管道

```
AgentMessage[] ──transformContext()──> AgentMessage[] ──convertToLlm()──> Message[] ──> LLM
```

这是一个经典的**管道模式（Pipeline Pattern）**。两个阶段职责分离：
- `transformContext`：操作应用层消息（裁剪、注入），不关心 LLM 格式
- `convertToLlm`：格式转换，过滤自定义消息

### 7.2 Observational vs Barrier 事件

`agentLoop()` 返回 EventStream（observational）：事件只是通知，不阻塞循环。
`Agent._processLoopEvent()` 通过 async emit（barrier）：确保状态更新完成后才继续。

README 中明确指出这个区别：

> These low-level streams are observational. If you need message processing to act as a barrier before tool preflight, use the Agent class.

### 7.3 类型安全的工具系统

```typescript
const tool: AgentTool<typeof schema, DetailType> = {
    parameters: schema,  // TypeBox Schema → 编译时类型安全
    execute: async (_id, params /* 自动推导为 Static<typeof schema> */) => {
        return { content: [...], details: { /* DetailType */ } };
    },
};
```

通过 TypeBox + TypeScript 泛型，工具参数和返回值都有完整的类型推导。

### 7.4 错误处理哲学

整个包遵循一个核心原则：**Agent Loop 内部不抛异常**。

- `StreamFn` 契约："Must not throw, failures encoded via stream events"
- `convertToLlm` 契约："Must not throw, return safe fallback"
- `transformContext` 契约："Must not throw, return original messages"
- `getApiKey` 契约："Must not throw, return undefined"
- `getSteeringMessages` 契约："Must not throw, return []"

只有 `Agent.prompt()` 和 `Agent.continue()` 在入口处会抛出（如 isStreaming 检查），内部循环完全通过事件传递错误。

---

## 8. 测试策略

### 8.1 单元测试（agent-loop.test.ts）

使用 `MockAssistantStream` 模拟 LLM 响应，测试 agent-loop 的核心逻辑：

- 基本事件流（agent_start → turn_start → message → turn_end → agent_end）
- 自定义消息类型通过 convertToLlm 过滤
- transformContext 在 convertToLlm 之前执行
- 工具调用和结果处理
- **并行执行验证**：通过 Promise 延迟证明两个工具确实并行执行
- **结果保序验证**：并行执行后 toolResult 按源顺序发射
- **Steering 消息注入**：验证所有 tool calls 完成后才注入 steering 消息

### 8.2 Agent 类测试（agent.test.ts）

- 默认状态初始化
- 自定义初始状态
- 事件订阅/取消订阅
- 状态修改方法
- Steering / Follow-up 队列
- 并发 prompt 互斥
- `continue()` 处理队列消息
- sessionId 传递

### 8.3 E2E 测试（e2e.test.ts）

对多个 Provider 跑相同的测试套件：

| 测试 | 说明 |
|------|------|
| basicPrompt | 基本文本问答 |
| toolExecution | 工具调用 + 结果验证 |
| abortExecution | 取消执行 |
| stateUpdates | 事件流完整性 |
| multiTurnConversation | 多轮对话上下文保持 |

覆盖的 Provider：Google、OpenAI、Anthropic、xAI、Groq、zAI、Amazon Bedrock。每个 Provider 通过环境变量控制跳过。

### 8.4 Bedrock 模型兼容性测试（bedrock-models.test.ts）

专门针对 Amazon Bedrock 的大规模模型兼容性测试，包含：
- 已知问题分类（需要推理配置、无效模型 ID、maxTokens 超限、不支持推理重放等）
- 自动跳过已知不兼容的模型
- 三个测试维度：基本文本、多轮对话（含 thinking）、合成签名重放

### 8.5 测试工具（test/utils/）

提供两个示例工具实现：

- **calculateTool**：使用 `new Function()` 执行数学表达式
- **getCurrentTimeTool**：获取当前时间（支持时区参数）

这些工具同时作为 `AgentTool` 接口的参考实现。

---

## 9. 与其他包的关系

```
                    @mariozechner/pi-ai
                         │
                         ▼
              @mariozechner/pi-agent-core  ← 本包
                     │          │
                     ▼          ▼
     @mariozechner/pi-coding-agent    @mariozechner/pi-web-ui
                     │
                     ▼
              @mariozechner/pi-mom
```

- **被 pi-ai 服务**：使用 pi-ai 的 StreamFn、EventStream、Message 类型、validateToolArguments
- **服务 pi-coding-agent**：提供 Agent 类和 AgentTool 接口，pi-coding-agent 定义具体的编码工具
- **服务 pi-web-ui**：通过 `streamProxy` 为浏览器端提供 LLM 通信能力

---

## 10. 源码阅读建议

### 推荐阅读顺序

1. **types.ts** → 理解全部类型定义，这是其他文件的基础
2. **agent-loop.ts** 的 `runLoop()` → 理解核心循环逻辑
3. **agent-loop.ts** 的 `streamAssistantResponse()` → 理解 LLM 调用流程
4. **agent-loop.ts** 的 `executeToolCalls*()` → 理解工具执行管道
5. **agent.ts** 的 `Agent` 类 → 理解状态管理和高级封装
6. **proxy.ts** → 理解浏览器端代理方案

### 关键断点位置（调试用）

| 文件 | 函数/行 | 说明 |
|------|---------|------|
| agent-loop.ts | `runLoop()` 的 `while (true)` | Agent 循环入口 |
| agent-loop.ts | `streamAssistantResponse()` | LLM 调用前后 |
| agent-loop.ts | `prepareToolCall()` | 工具预检 |
| agent-loop.ts | `executePreparedToolCall()` | 工具执行 |
| agent-loop.ts | `finalizeExecutedToolCall()` | 工具结果后处理 |
| agent.ts | `_processLoopEvent()` | Agent 状态更新 |
| agent.ts | `_runLoop()` | Agent 启动循环 |
| proxy.ts | `processProxyEvent()` | Proxy 事件处理 |

### 核心设计思想总结

1. **双层消息模型**：AgentMessage（应用层）vs Message（LLM 层），通过 convertToLlm 桥接
2. **事件驱动**：三层事件体系（Agent → Turn → Message/Tool）
3. **管道式上下文处理**：transformContext → convertToLlm
4. **不抛异常原则**：循环内部所有错误通过事件传递
5. **可插拔流函数**：streamFn 参数化，支持直连/代理/自定义
6. **并行工具执行**：预检顺序化 + 执行并行化 + 结果保序
7. **Steering/Follow-up 双队列**：运行时中断 vs 完成后追加
