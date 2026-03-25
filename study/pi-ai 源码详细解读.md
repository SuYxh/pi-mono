# @mariozechner/pi-ai 源码详细解读

> 本文档是对 `packages/ai` 包的源码级深度解读，涵盖架构设计、类型系统、流式事件协议、Provider 实现、消息转换、OAuth 认证、工具调用验证等核心模块。

---

## 一、包概览

| 属性 | 值 |
|---|---|
| 包名 | `@mariozechner/pi-ai` |
| 定位 | 统一的 LLM API 抽象层，支持 20+ 模型提供商 |
| 入口 | `src/index.ts` → `dist/index.js` |
| CLI | `pi-ai` (OAuth 登录工具) |
| 运行环境 | Node.js >= 20 + 浏览器（Vite 构建的 web-ui） |
| 依赖 | Anthropic SDK、OpenAI SDK、Google GenAI SDK、Mistral SDK、AWS Bedrock SDK、TypeBox、AJV |

### 1.1 设计哲学

pi-ai 的核心设计目标是**一套类型、一套事件协议、N 个 Provider**：

```
调用方 → stream(model, context, options)
                ↓
         api-registry 路由
                ↓
    Provider（Anthropic / OpenAI / Google / ...）
                ↓
    AssistantMessageEventStream（统一事件流）
```

不管底层是 Anthropic Messages API、OpenAI Chat Completions、OpenAI Responses、Google Generative AI 还是 AWS Bedrock，上层消费者拿到的都是完全相同的 `AssistantMessageEvent` 事件序列。

### 1.2 目录结构

```
packages/ai/
├── src/
│   ├── index.ts                    # 公共导出（re-export 所有模块）
│   ├── types.ts                    # 核心类型定义（Message, Model, Event, Tool 等）
│   ├── stream.ts                   # 顶层 API：stream(), complete(), streamSimple(), completeSimple()
│   ├── api-registry.ts             # Provider 注册中心
│   ├── models.ts                   # 模型注册表 + 费用计算
│   ├── models.generated.ts         # 自动生成的模型数据（20+ 提供商, 数百个模型）
│   ├── env-api-keys.ts             # 环境变量 API Key 检测
│   ├── cli.ts                      # CLI 工具（OAuth 登录）
│   ├── oauth.ts                    # re-export utils/oauth/index.ts
│   ├── bedrock-provider.ts         # Bedrock 特殊入口（浏览器兼容）
│   ├── providers/
│   │   ├── register-builtins.ts    # 懒加载注册所有内置 Provider
│   │   ├── anthropic.ts            # Anthropic Messages API 实现
│   │   ├── openai-completions.ts   # OpenAI Chat Completions API 实现
│   │   ├── openai-responses.ts     # OpenAI Responses API 实现
│   │   ├── openai-responses-shared.ts  # OpenAI Responses 共享逻辑
│   │   ├── openai-codex-responses.ts   # OpenAI Codex (ChatGPT) 实现
│   │   ├── azure-openai-responses.ts   # Azure OpenAI 实现
│   │   ├── google.ts               # Google Generative AI 实现
│   │   ├── google-shared.ts        # Google 系列共享逻辑
│   │   ├── google-gemini-cli.ts    # Google Gemini CLI (Cloud Code Assist) 实现
│   │   ├── google-vertex.ts        # Google Vertex AI 实现
│   │   ├── mistral.ts              # Mistral 实现
│   │   ├── amazon-bedrock.ts       # AWS Bedrock 实现
│   │   ├── github-copilot-headers.ts   # GitHub Copilot 请求头构建
│   │   ├── simple-options.ts       # SimpleStreamOptions → ProviderOptions 映射
│   │   └── transform-messages.ts   # 跨 Provider 消息转换
│   └── utils/
│       ├── event-stream.ts         # EventStream / AssistantMessageEventStream
│       ├── json-parse.ts           # 流式 JSON 增量解析
│       ├── overflow.ts             # 上下文溢出检测
│       ├── validation.ts           # 工具参数 AJV 校验
│       ├── sanitize-unicode.ts     # Unicode 代理对清理
│       ├── typebox-helpers.ts      # TypeBox StringEnum 辅助
│       ├── hash.ts                 # 快速确定性哈希
│       └── oauth/                  # OAuth 认证子系统
│           ├── types.ts            # OAuth 类型定义
│           ├── index.ts            # OAuth Provider 注册中心
│           ├── anthropic.ts        # Anthropic OAuth 实现
│           ├── github-copilot.ts   # GitHub Copilot OAuth 实现
│           ├── google-gemini-cli.ts    # Google Gemini CLI OAuth 实现
│           ├── google-antigravity.ts   # Google Antigravity OAuth 实现
│           ├── openai-codex.ts     # OpenAI Codex OAuth 实现
│           ├── pkce.ts             # PKCE 工具函数
│           └── oauth-page.ts       # OAuth 回调页面 HTML
├── scripts/
│   └── generate-models.ts          # 模型数据生成脚本
├── test/                           # Vitest 测试用例（40+ 测试文件）
├── package.json
└── vitest.config.ts
```

---

## 二、核心类型系统 (`src/types.ts`)

`types.ts` 是整个包的类型基石，定义了从消息到模型到事件的完整类型体系。

### 2.1 API 与 Provider 标识

```typescript
// 10 种已知 API 协议
export type KnownApi =
    | "openai-completions"       // OpenAI Chat Completions API
    | "openai-responses"         // OpenAI Responses API
    | "azure-openai-responses"   // Azure OpenAI Responses
    | "openai-codex-responses"   // OpenAI Codex (ChatGPT OAuth)
    | "anthropic-messages"       // Anthropic Messages API
    | "bedrock-converse-stream"  // AWS Bedrock ConverseStream
    | "google-generative-ai"     // Google Generative AI
    | "google-gemini-cli"        // Google Cloud Code Assist
    | "google-vertex"            // Google Vertex AI
    | "mistral-conversations";   // Mistral Conversations

// 开放式联合类型：允许自定义 API
export type Api = KnownApi | (string & {});
```

**设计要点**：`Api` 类型使用了 TypeScript 的"品牌联合"技巧 `(string & {})`，既有 IDE 自动补全（KnownApi 成员），又允许传入任意字符串（自定义 Provider）。

Provider 同理：

```typescript
export type KnownProvider =
    | "amazon-bedrock" | "anthropic" | "google" | "google-gemini-cli"
    | "google-antigravity" | "google-vertex" | "openai" | "azure-openai-responses"
    | "openai-codex" | "github-copilot" | "xai" | "groq" | "cerebras"
    | "openrouter" | "vercel-ai-gateway" | "zai" | "mistral"
    | "minimax" | "minimax-cn" | "huggingface" | "opencode" | "opencode-go"
    | "kimi-coding";
export type Provider = KnownProvider | string;
```

> **关键区分**：`Api` 是**协议类型**（通信层面），`Provider` 是**服务商标识**（业务层面）。同一个 Api 协议（如 `openai-completions`）可以被多个 Provider 使用（OpenAI、xAI、Groq、OpenRouter 等都走 OpenAI Chat Completions 协议）。

### 2.2 消息类型

pi-ai 定义了三种消息角色：

```typescript
// 用户消息
export interface UserMessage {
    role: "user";
    content: string | (TextContent | ImageContent)[];  // 支持纯文本或多模态
    timestamp: number;
}

// 助手消息（LLM 响应）
export interface AssistantMessage {
    role: "assistant";
    content: (TextContent | ThinkingContent | ToolCall)[];  // 文本 + 思考 + 工具调用
    api: Api;               // 产生该消息的 API 协议
    provider: Provider;     // 产生该消息的提供商
    model: string;          // 模型 ID
    responseId?: string;    // 提供商级响应标识
    usage: Usage;           // Token 用量 + 费用
    stopReason: StopReason; // "stop" | "length" | "toolUse" | "error" | "aborted"
    errorMessage?: string;
    timestamp: number;
}

// 工具结果消息
export interface ToolResultMessage<TDetails = any> {
    role: "toolResult";
    toolCallId: string;     // 对应 ToolCall.id
    toolName: string;
    content: (TextContent | ImageContent)[];  // 支持文本和图片
    details?: TDetails;     // 扩展数据（泛型）
    isError: boolean;
    timestamp: number;
}
```

**设计亮点**：

1. **`AssistantMessage` 携带来源元数据**（api、provider、model）—— 这是跨 Provider 消息转换的关键。当同一个对话从 Claude 切换到 GPT 时，`transform-messages.ts` 通过比较 `msg.provider === model.provider` 来决定是否保留 thinking 签名。

2. **content 是数组类型**：`(TextContent | ThinkingContent | ToolCall)[]`。一次 LLM 响应可能先输出思考过程，再输出文本，再调用工具——它们按顺序排列在同一个数组中。

3. **ToolCall 的 `thoughtSignature`**：Google Gemini 3 的 thought signature 协议要求每个 function call 都携带思考签名，即使是跨 Provider 重放的 tool call，也需要填入 `skip_thought_signature_validator` 哨兵值。

### 2.3 内容块类型

```typescript
export interface TextContent {
    type: "text";
    text: string;
    textSignature?: string;  // Google thought signature（非 thinking 内容也可能有）
}

export interface ThinkingContent {
    type: "thinking";
    thinking: string;
    thinkingSignature?: string;  // Anthropic/Google 的签名
    redacted?: boolean;          // Anthropic 安全过滤时加密的思考内容
}

export interface ImageContent {
    type: "image";
    data: string;       // base64
    mimeType: string;   // "image/jpeg", "image/png"
}

export interface ToolCall {
    type: "toolCall";
    id: string;
    name: string;
    arguments: Record<string, any>;
    thoughtSignature?: string;  // Google 思考签名
}
```

### 2.4 Model 接口

```typescript
export interface Model<TApi extends Api> {
    id: string;              // 如 "claude-sonnet-4-20250514"
    name: string;            // 如 "Claude Sonnet 4"
    api: TApi;               // 泛型绑定：限制该模型只能用于特定 API
    provider: Provider;
    baseUrl: string;         // API 端点
    reasoning: boolean;      // 是否支持思考推理
    input: ("text" | "image")[];
    cost: { input, output, cacheRead, cacheWrite };  // 每百万 token 美元价格
    contextWindow: number;
    maxTokens: number;
    headers?: Record<string, string>;
    compat?: ...;            // OpenAI 兼容性覆盖（条件类型）
}
```

**泛型设计**：`Model<TApi>` 将模型与 API 协议绑定。`Model<"anthropic-messages">` 只能传给接受 `"anthropic-messages"` 的 stream 函数。`compat` 字段使用条件类型——只有 `openai-completions` 和 `openai-responses` 的模型才有 compat 选项。

### 2.5 事件协议

```typescript
export type AssistantMessageEvent =
    | { type: "start"; partial: AssistantMessage }
    | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
    | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
    | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
    | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
    | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
    | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
    | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

**协议约定**：
- 流以 `start` 开头
- 中间是 `*_start` → `*_delta`（0~N 次）→ `*_end` 的子序列
- 以 `done` 或 `error` 结束
- 每个事件都携带 `partial` (当前累积的 AssistantMessage 快照)
- `contentIndex` 指向 `AssistantMessage.content` 数组的下标

**终结事件类型约束**：
- `done.reason` 只能是 `"stop" | "length" | "toolUse"`（成功路径）
- `error.reason` 只能是 `"aborted" | "error"`（失败路径）

这通过 `Extract<StopReason, ...>` 实现编译时约束。

### 2.6 StreamOptions 层级

```
StreamOptions                    // 所有 Provider 共享的基础选项
├── temperature, maxTokens, signal, apiKey, transport, cacheRetention,
│   sessionId, onPayload, headers, maxRetryDelayMs, metadata
│
├── SimpleStreamOptions          // 加了 reasoning + thinkingBudgets
│   └── reasoning?: ThinkingLevel
│
└── 各 Provider 专属 Options
    ├── AnthropicOptions         // thinkingEnabled, effort, interleavedThinking, toolChoice, client
    ├── OpenAICompletionsOptions // toolChoice, reasoningEffort
    ├── OpenAIResponsesOptions   // reasoningEffort, reasoningSummary, serviceTier
    ├── GoogleOptions            // toolChoice, thinking: { enabled, budgetTokens, level }
    └── ...
```

**双层 API 设计**：
- `stream()` / `complete()`：直接传入 Provider 专属 Options（精细控制）
- `streamSimple()` / `completeSimple()`：传入 `SimpleStreamOptions`（统一抽象，由 `streamSimple*` 函数内部映射到 Provider 专属选项）

---

## 三、事件流引擎 (`src/utils/event-stream.ts`)

### 3.1 泛型 EventStream

```typescript
export class EventStream<T, R = T> implements AsyncIterable<T> {
    private queue: T[] = [];
    private waiting: ((value: IteratorResult<T>) => void)[] = [];
    private done = false;
    private finalResultPromise: Promise<R>;
    private resolveFinalResult!: (result: R) => void;

    constructor(
        private isComplete: (event: T) => boolean,
        private extractResult: (event: T) => R,
    ) { ... }
}
```

这是一个**生产者-消费者模式**的异步可迭代对象：

- **生产端**：`push(event)` 推入事件，`end()` 结束流
- **消费端**：`for await (const event of stream)` 迭代消费，`stream.result()` 获取最终结果

**背压机制**：
- 如果有等待的消费者（`waiting` 数组非空），事件直接送达
- 否则事件排队等待（`queue` 数组缓冲）
- 消费者在没有事件时通过 `Promise` 挂起等待

```typescript
async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
        if (this.queue.length > 0) {
            yield this.queue.shift()!;        // 有缓冲事件，立即消费
        } else if (this.done) {
            return;                            // 流结束
        } else {
            // 没有事件也没结束，挂起等待
            const result = await new Promise<IteratorResult<T>>((resolve) =>
                this.waiting.push(resolve)
            );
            if (result.done) return;
            yield result.value;
        }
    }
}
```

### 3.2 AssistantMessageEventStream

```typescript
export class AssistantMessageEventStream
    extends EventStream<AssistantMessageEvent, AssistantMessage> {
    constructor() {
        super(
            (event) => event.type === "done" || event.type === "error",
            (event) => {
                if (event.type === "done") return event.message;
                if (event.type === "error") return event.error;
                throw new Error("Unexpected event type for final result");
            },
        );
    }
}
```

`isComplete` 判断：收到 `done` 或 `error` 事件时流完成。`extractResult` 提取最终的 `AssistantMessage`。

**使用方式**：

```typescript
// 流式消费
const eventStream = stream(model, context, options);
for await (const event of eventStream) {
    if (event.type === "text_delta") {
        process.stdout.write(event.delta);
    }
}

// 直接获取结果
const message = await eventStream.result();

// 或使用 complete() 快捷方式
const message = await complete(model, context, options);
```

---

## 四、API 注册中心 (`src/api-registry.ts`)

### 4.1 注册机制

```typescript
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

export interface ApiProvider<TApi extends Api, TOptions extends StreamOptions> {
    api: TApi;
    stream: StreamFunction<TApi, TOptions>;
    streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}

export function registerApiProvider<TApi extends Api, TOptions extends StreamOptions>(
    provider: ApiProvider<TApi, TOptions>,
    sourceId?: string,
): void {
    apiProviderRegistry.set(provider.api, {
        provider: {
            api: provider.api,
            stream: wrapStream(provider.api, provider.stream),
            streamSimple: wrapStreamSimple(provider.api, provider.streamSimple),
        },
        sourceId,
    });
}
```

`wrapStream` 添加了 API 类型校验——防止把 `Model<"anthropic-messages">` 传给 `"openai-completions"` 的 stream 函数。

### 4.2 sourceId 机制

`registerApiProvider` 接受可选的 `sourceId`，用于批量注销：

```typescript
export function unregisterApiProviders(sourceId: string): void {
    for (const [api, entry] of apiProviderRegistry.entries()) {
        if (entry.sourceId === sourceId) {
            apiProviderRegistry.delete(api);
        }
    }
}
```

这为外部扩展（如 coding-agent 的自定义 Provider）提供了生命周期管理。

---

## 五、顶层 API (`src/stream.ts`)

```typescript
import "./providers/register-builtins.js";  // 副作用导入：注册所有内置 Provider

function resolveApiProvider(api: Api) {
    const provider = getApiProvider(api);
    if (!provider) throw new Error(`No API provider registered for api: ${api}`);
    return provider;
}

export function stream<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ProviderStreamOptions,
): AssistantMessageEventStream {
    const provider = resolveApiProvider(model.api);
    return provider.stream(model, context, options as StreamOptions);
}

export async function complete<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ProviderStreamOptions,
): Promise<AssistantMessage> {
    const s = stream(model, context, options);
    return s.result();  // 等待流完成并返回最终消息
}
```

**`stream.ts` 的角色**：纯粹的路由层。通过 `model.api` 查找注册的 Provider，委托调用。注意第一行的副作用导入——`import "./providers/register-builtins.js"` 确保内置 Provider 在模块加载时自动注册。

`ProviderStreamOptions` 类型是 `StreamOptions & Record<string, unknown>`，允许传入 Provider 专属选项而不需要类型断言。

---

## 六、懒加载注册系统 (`src/providers/register-builtins.ts`)

这是 pi-ai 中最精巧的设计之一——**懒加载 Provider 模块**。

### 6.1 问题

pi-ai 支持 10 种 API 协议，每个协议的实现都依赖各自的 SDK：
- `anthropic.ts` → `@anthropic-ai/sdk`
- `openai-completions.ts` → `openai`
- `google.ts` → `@google/genai`
- `amazon-bedrock.ts` → `@aws-sdk/client-bedrock-runtime`
- `mistral.ts` → `@mistralai/mistralai`

如果在 `import` 时就加载所有 SDK，会：
1. **启动缓慢**：加载 5 个大型 SDK 的代码
2. **环境不兼容**：AWS Bedrock SDK 在浏览器中不可用

### 6.2 解决方案：Module-level Promise 缓存

```typescript
// 每个 Provider 有一个 Promise 缓存变量
let anthropicProviderModulePromise:
    | Promise<LazyProviderModule<"anthropic-messages", AnthropicOptions, SimpleStreamOptions>>
    | undefined;

function loadAnthropicProviderModule(): Promise<...> {
    // ||= 运算符：首次调用时创建 Promise，后续复用
    anthropicProviderModulePromise ||= import("./anthropic.js").then((module) => {
        const provider = module as AnthropicProviderModule;
        return {
            stream: provider.streamAnthropic,
            streamSimple: provider.streamSimpleAnthropic,
        };
    });
    return anthropicProviderModulePromise;
}
```

### 6.3 懒加载包装

```typescript
function createLazyStream<TApi, TOptions, TSimpleOptions>(
    loadModule: () => Promise<LazyProviderModule<TApi, TOptions, TSimpleOptions>>,
): StreamFunction<TApi, TOptions> {
    return (model, context, options) => {
        const outer = new AssistantMessageEventStream();

        loadModule()
            .then((module) => {
                const inner = module.stream(model, context, options);
                forwardStream(outer, inner);      // 桥接内部流到外部流
            })
            .catch((error) => {
                // 模块加载失败也通过事件流协议报告
                const message = createLazyLoadErrorMessage(model, error);
                outer.push({ type: "error", reason: "error", error: message });
                outer.end(message);
            });

        return outer;  // 立即返回空流，模块加载完成后数据开始流入
    };
}
```

**执行流程**：

1. 调用 `stream(model, context, options)` 时，`register-builtins.ts` 的懒加载包装立即返回一个空的 `AssistantMessageEventStream`
2. 异步触发 `import("./anthropic.js")`（如果是首次调用）
3. 模块加载完成后，调用真实的 `streamAnthropic()`，将内部事件流桥接到外部流
4. `forwardStream()` 逐事件转发，最后调用 `outer.end()`

**错误处理**：即使模块加载失败（如 Node.js 原生模块在浏览器中不可用），也通过 `AssistantMessageEventStream` 的 error 事件优雅报告，不会抛异常。

### 6.4 注册时机

```typescript
// 文件末尾——模块加载时立即执行
registerBuiltInApiProviders();
```

这行代码在 `register-builtins.ts` 被导入时执行，将 10 个 API 协议全部注册为懒加载代理。真正的 Provider 模块只在第一次 `stream()` 调用时才加载。

---

## 七、Provider 实现深度剖析

### 7.1 统一的 Provider 实现模式

每个 Provider 都遵循相同的代码结构：

```typescript
// 1. 导出 StreamFunction 签名的两个函数
export const streamXxx: StreamFunction<"xxx-api", XxxOptions> = (model, context, options?) => {
    const stream = new AssistantMessageEventStream();

    (async () => {
        // 2. 初始化 output（AssistantMessage 累积器）
        const output: AssistantMessage = { role: "assistant", content: [], ... };

        try {
            // 3. 创建 SDK 客户端
            const client = createClient(model, apiKey, ...);

            // 4. 构建请求参数
            let params = buildParams(model, context, options);

            // 5. onPayload 钩子（允许调用方修改请求）
            const nextParams = await options?.onPayload?.(params, model);
            if (nextParams !== undefined) params = nextParams;

            // 6. 发起流式请求
            const sdkStream = client.xxx.stream(params);
            stream.push({ type: "start", partial: output });

            // 7. 逐事件处理 SDK 响应
            for await (const event of sdkStream) {
                // 映射到 AssistantMessageEvent
                // 累积到 output
            }

            // 8. 成功结束
            stream.push({ type: "done", reason: output.stopReason, message: output });
            stream.end();
        } catch (error) {
            // 9. 错误处理（统一通过事件流报告）
            output.stopReason = options?.signal?.aborted ? "aborted" : "error";
            output.errorMessage = error instanceof Error ? error.message : JSON.stringify(error);
            stream.push({ type: "error", reason: output.stopReason, error: output });
            stream.end();
        }
    })();

    return stream;  // 立即返回，流式处理异步进行
};

// 10. streamSimple 函数：将 SimpleStreamOptions 映射为 Provider 专属选项
export const streamSimpleXxx: StreamFunction<"xxx-api", SimpleStreamOptions> = (model, context, options?) => {
    const base = buildBaseOptions(model, options, apiKey);
    // 映射 reasoning level → Provider 专属参数
    return streamXxx(model, context, { ...base, ... });
};
```

### 7.2 Anthropic Provider (`anthropic.ts`)

**核心特性**：

1. **三种认证模式**：
   - API Key：标准认证
   - OAuth Token：`sk-ant-oat` 前缀的 OAuth Token，需要模拟 Claude Code 身份
   - GitHub Copilot：Bearer Token + Copilot 专用 Header

2. **Claude Code 隐身模式**：
   ```typescript
   const claudeCodeVersion = "2.1.75";
   const claudeCodeTools = ["Read", "Write", "Edit", "Bash", "Grep", ...];
   ```
   当使用 OAuth Token 时，pi-ai 会模拟 Claude Code CLI 的身份：
   - 设置 `user-agent: claude-cli/2.1.75`
   - 将工具名映射为 Claude Code 的规范大小写（如 `grep` → `Grep`）
   - 注入系统提示：`"You are Claude Code, Anthropic's official CLI for Claude."`

3. **Thinking 模式**：
   - **自适应思考**（Opus 4.6 / Sonnet 4.6）：`params.thinking = { type: "adaptive" }` + `params.output_config = { effort }`
   - **预算思考**（旧模型）：`params.thinking = { type: "enabled", budget_tokens: N }`
   - **禁用思考**：`params.thinking = { type: "disabled" }`

4. **缓存控制**：
   ```typescript
   function getCacheControl(baseUrl, cacheRetention?) {
       // "short" → cache_control: { type: "ephemeral" }
       // "long" → cache_control: { type: "ephemeral", ttl: "1h" }（仅 api.anthropic.com）
       // "none" → 不添加 cache_control
   }
   ```
   缓存标记添加在系统提示和最后一条用户消息上。

5. **SDK 事件映射**（简化）：
   ```
   Anthropic SDK Event          →  pi-ai Event
   ─────────────────────────────────────────────
   message_start                →  output.responseId, output.usage
   content_block_start (text)   →  text_start
   content_block_start (thinking) → thinking_start
   content_block_start (redacted_thinking) → thinking_start (redacted=true)
   content_block_start (tool_use) → toolcall_start
   content_block_delta (text)   →  text_delta
   content_block_delta (thinking) → thinking_delta
   content_block_delta (input_json) → toolcall_delta
   content_block_delta (signature) → thinkingSignature 累积
   content_block_stop           →  text_end / thinking_end / toolcall_end
   message_delta                →  stopReason + usage 更新
   ```

6. **流式 JSON 解析**：
   工具调用的参数是增量 JSON（`input_json_delta`），通过 `parseStreamingJson()` 实时解析部分 JSON，使上层可以在工具参数完成前就看到中间状态。

### 7.3 OpenAI Completions Provider (`openai-completions.ts`)

这是最复杂的 Provider，因为它要兼容 20+ 个使用 OpenAI 协议的不同提供商。

**兼容性层 (`OpenAICompletionsCompat`)**：

```typescript
function detectCompat(model): Required<OpenAICompletionsCompat> {
    // 根据 provider 和 baseUrl 自动检测兼容性设置
    const isZai = provider === "zai" || baseUrl.includes("api.z.ai");
    const isNonStandard = provider === "cerebras" || ... || isZai;

    return {
        supportsStore: !isNonStandard,         // 是否支持 store 字段
        supportsDeveloperRole: !isNonStandard,  // 是否支持 developer 角色（vs system）
        supportsReasoningEffort: !isGrok && !isZai,
        thinkingFormat: isZai ? "zai" : isOpenRouter ? "openrouter" : "openai",
        maxTokensField: useMaxTokens ? "max_tokens" : "max_completion_tokens",
        requiresToolResultName: false,
        requiresAssistantAfterToolResult: false,
        requiresThinkingAsText: false,
        // ...
    };
}
```

**5 种思考格式**：
| 格式 | Provider | 参数形式 |
|---|---|---|
| `"openai"` | OpenAI, Groq, Cerebras | `reasoning_effort: "low"` |
| `"openrouter"` | OpenRouter | `reasoning: { effort: "low" }` |
| `"zai"` | z.ai | `enable_thinking: true` |
| `"qwen"` | Qwen | `enable_thinking: true` |
| `"qwen-chat-template"` | Qwen (chat template) | `chat_template_kwargs: { enable_thinking: true }` |

**Reasoning 字段检测**：
```typescript
// 不同提供商返回推理内容的字段名不同
const reasoningFields = ["reasoning_content", "reasoning", "reasoning_text"];
// 取第一个非空字段，避免重复（如 chutes.ai 同时返回 reasoning_content 和 reasoning）
```

**OpenRouter 缓存**：
```typescript
function maybeAddOpenRouterAnthropicCacheControl(model, messages) {
    // 当通过 OpenRouter 使用 Anthropic 模型时，在最后一条消息上添加 cache_control
    if (model.provider !== "openrouter" || !model.id.startsWith("anthropic/")) return;
    // 添加 { cache_control: { type: "ephemeral" } }
}
```

### 7.4 Google Provider (`google.ts`)

**特有设计**：

1. **Gemini 3 vs Gemini 2 思考模式**：
   - Gemini 3 Pro/Flash：使用 `thinkingLevel`（"MINIMAL" / "LOW" / "MEDIUM" / "HIGH"）
   - Gemini 2.x：使用 `thinkingBudget`（数字，0 表示禁用，-1 表示动态）

2. **Tool Call ID 生成**：
   Google API 的 function call 不一定提供 ID，pi-ai 自行生成唯一 ID：
   ```typescript
   const toolCallId = needsNewId
       ? `${part.functionCall.name}_${Date.now()}_${++toolCallCounter}`
       : providedId;
   ```

3. **禁用思考的特殊处理**：
   ```typescript
   function getDisabledThinkingConfig(model) {
       if (isGemini3ProModel(model)) return { thinkingLevel: "LOW" as any };  // Pro 不能完全禁用
       if (isGemini3FlashModel(model)) return { thinkingLevel: "MINIMAL" };
       return { thinkingBudget: 0 };  // Gemini 2.x 用 0 禁用
   }
   ```

### 7.5 Google Shared (`google-shared.ts`)

Google 系列（Google、Gemini CLI、Vertex）共享大量逻辑：

1. **Thought Signature 协议**：
   ```typescript
   // thought: true 是确定标记——表示该 part 是思考内容
   // thoughtSignature 是加密签名——用于多轮对话中保持推理上下文
   // thoughtSignature 可以出现在任何 part 类型上（text, functionCall 等）
   export function isThinkingPart(part) { return part.thought === true; }
   ```

2. **Gemini 3 Function Call 签名**：
   ```typescript
   const SKIP_THOUGHT_SIGNATURE = "skip_thought_signature_validator";
   // Gemini 3 要求所有 function call 都有 thoughtSignature
   // 对于跨 Provider 重放的无签名 tool call，使用此哨兵值跳过校验
   ```

3. **多模态 Function Response**：
   Gemini 3+ 支持在 `functionResponse.parts` 中嵌套图片，旧模型需要单独的 user 消息。

---

## 八、消息转换引擎 (`src/providers/transform-messages.ts`)

这是跨 Provider 兼容性的核心。当对话从 Provider A 切换到 Provider B 时，之前的消息需要转换。

### 8.1 转换逻辑

```typescript
export function transformMessages<TApi extends Api>(
    messages: Message[],
    model: Model<TApi>,
    normalizeToolCallId?: (id, model, source) => string,
): Message[] {
    // 第一遍：内容转换
    const transformed = messages.map((msg) => {
        if (msg.role === "assistant") {
            const isSameModel = msg.provider === model.provider
                             && msg.api === model.api
                             && msg.model === model.id;

            const transformedContent = msg.content.flatMap((block) => {
                if (block.type === "thinking") {
                    if (block.redacted) return isSameModel ? block : [];  // 加密思考只保留给同模型
                    if (isSameModel && block.thinkingSignature) return block;
                    if (!block.thinking?.trim()) return [];  // 空思考直接删除
                    if (isSameModel) return block;
                    return { type: "text", text: block.thinking };  // 跨模型：思考→文本
                }
                if (block.type === "toolCall" && !isSameModel) {
                    // 跨模型：删除 thoughtSignature，规范化 toolCallId
                }
                return block;
            });
        }
    });

    // 第二遍：修复孤立 tool call
    // 如果 assistant 消息有 tool call 但没有对应的 toolResult，插入合成结果
    // 如果 assistant 消息的 stopReason 是 error/aborted，整条消息跳过
}
```

### 8.2 关键转换规则

| 场景 | 处理方式 |
|---|---|
| 同模型的 thinking block | 保留（含签名） |
| 跨模型的 thinking block | 转为 text block（不加 `<thinking>` 标签，避免模型模仿） |
| 加密的 redacted thinking | 同模型保留，跨模型删除 |
| 空 thinking block | 删除 |
| error/aborted 的 assistant 消息 | 整条跳过（不重放不完整的轮次） |
| 无 toolResult 的 tool call | 插入合成错误结果 `"No result provided"` |
| Tool Call ID 不兼容 | 通过 normalizeToolCallId 回调规范化（如 OpenAI Responses 的 450+ 字符 ID → Anthropic 的 64 字符限制） |

---

## 九、模型系统 (`src/models.ts` + `models.generated.ts`)

### 9.1 自动生成的模型数据

`scripts/generate-models.ts` 从多个数据源拉取模型信息：
- OpenRouter API (`openrouter.ai/api/v1/models`)
- models.dev JSON 文件
- Vercel AI Gateway API
- 手动硬编码（Copilot、Bedrock 等）

生成的 `models.generated.ts` 包含数百个模型定义，按 Provider 分组：

```typescript
export const MODELS = {
    "amazon-bedrock": { ... },
    "anthropic": { ... },
    "google": { ... },
    "openai": { ... },
    "openrouter": { ... },
    "github-copilot": { ... },
    // ...
};
```

### 9.2 模型注册表

```typescript
const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();

// 初始化：从 MODELS 静态数据构建
for (const [provider, models] of Object.entries(MODELS)) {
    const providerModels = new Map<string, Model<Api>>();
    for (const [id, model] of Object.entries(models)) {
        providerModels.set(id, model);
    }
    modelRegistry.set(provider, providerModels);
}
```

双层 Map：`Provider → ModelId → Model`。

### 9.3 费用计算

```typescript
export function calculateCost<TApi extends Api>(model: Model<TApi>, usage: Usage): Usage["cost"] {
    usage.cost.input = (model.cost.input / 1_000_000) * usage.input;
    usage.cost.output = (model.cost.output / 1_000_000) * usage.output;
    usage.cost.cacheRead = (model.cost.cacheRead / 1_000_000) * usage.cacheRead;
    usage.cost.cacheWrite = (model.cost.cacheWrite / 1_000_000) * usage.cacheWrite;
    usage.cost.total = usage.cost.input + usage.cost.output + usage.cost.cacheRead + usage.cost.cacheWrite;
    return usage.cost;
}
```

`model.cost` 存的是 $/百万 token，乘以 token 数后得到实际美元费用。

---

## 十、环境变量与 API Key 检测 (`src/env-api-keys.ts`)

```typescript
export function getEnvApiKey(provider: any): string | undefined {
    // 特殊处理
    if (provider === "github-copilot") return process.env.COPILOT_GITHUB_TOKEN || ...;
    if (provider === "anthropic") return process.env.ANTHROPIC_OAUTH_TOKEN || process.env.ANTHROPIC_API_KEY;
    if (provider === "google-vertex") { /* ADC 凭据检测 */ }
    if (provider === "amazon-bedrock") { /* 6 种 AWS 凭据源检测 */ }

    // 通用映射
    const envMap: Record<string, string> = {
        openai: "OPENAI_API_KEY",
        google: "GEMINI_API_KEY",
        groq: "GROQ_API_KEY",
        // ...20+ 映射
    };
    return envVar ? process.env[envVar] : undefined;
}
```

**浏览器兼容**：文件开头用动态 `import()` 加载 `node:fs`, `node:os`, `node:path`，避免在浏览器环境中崩溃：

```typescript
// NEVER convert to top-level imports - breaks browser/Vite builds (web-ui)
let _existsSync: typeof import("node:fs").existsSync | null = null;
const NODE_FS_SPECIFIER = "node:" + "fs";  // 字符串拼接避免 Vite 静态分析
```

---

## 十一、工具参数校验 (`src/utils/validation.ts`)

```typescript
function canUseRuntimeCodegen(): boolean {
    if (isBrowserExtension) return false;  // Chrome Extension MV3 禁止 eval
    try { new Function("return true;"); return true; } catch { return false; }
}

let ajv = null;
if (canUseRuntimeCodegen()) {
    ajv = new Ajv({ allErrors: true, strict: false, coerceTypes: true });
    addFormats(ajv);
}

export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
    if (!ajv) return toolCall.arguments;  // 无法校验时直接放行

    const args = structuredClone(toolCall.arguments);  // 克隆（AJV 会就地修改）
    const validate = ajv.compile(tool.parameters);

    if (validate(args)) return args;  // 校验通过（含类型转换）

    // 格式化错误信息
    throw new Error(`Validation failed for tool "${toolCall.name}":\n${errors}\n\nReceived:\n${JSON.stringify(...)}`);
}
```

**设计考量**：
- **AJV `coerceTypes: true`**：LLM 经常把数字发成字符串，自动转换比报错更实用
- **`structuredClone`**：AJV 的类型转换会修改原对象，用深拷贝保护原始数据
- **环境自适应**：浏览器扩展禁止 `eval`/`new Function`，此时静默跳过校验

---

## 十二、辅助工具模块

### 12.1 流式 JSON 解析 (`json-parse.ts`)

```typescript
import { parse as partialParse } from "partial-json";

export function parseStreamingJson<T>(partialJson: string | undefined): T {
    if (!partialJson?.trim()) return {} as T;
    try { return JSON.parse(partialJson); }         // 完整 JSON 快速路径
    catch {
        try { return (partialParse(partialJson) ?? {}) as T; }  // 部分 JSON
        catch { return {} as T; }                    // 兜底
    }
}
```

用于工具调用参数的增量解析——LLM 流式返回的 JSON 可能不完整（如 `{"file_path": "/Users/b`），`partial-json` 库能将其解析为 `{ file_path: "/Users/b" }`。

### 12.2 上下文溢出检测 (`overflow.ts`)

```typescript
const OVERFLOW_PATTERNS = [
    /prompt is too long/i,                   // Anthropic
    /exceeds the context window/i,           // OpenAI
    /input token count.*exceeds the maximum/i, // Google
    /maximum prompt length is \d+/i,          // xAI
    // ...15+ 正则
];

export function isContextOverflow(message: AssistantMessage, contextWindow?: number): boolean {
    // Case 1: 错误消息匹配
    if (message.stopReason === "error" && message.errorMessage) {
        if (OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!))) return true;
    }
    // Case 2: 静默溢出（z.ai 不报错但 input > contextWindow）
    if (contextWindow && message.stopReason === "stop") {
        if (message.usage.input + message.usage.cacheRead > contextWindow) return true;
    }
    return false;
}
```

覆盖 15+ Provider 的错误消息格式。注释中详细列出了每个 Provider 的错误示例——对调试极其有用。

### 12.3 Unicode 清理 (`sanitize-unicode.ts`)

```typescript
export function sanitizeSurrogates(text: string): string {
    return text.replace(
        /[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g,
        ""
    );
}
```

移除未配对的 Unicode 代理字符。正常 emoji 是成对代理（如 🙈 = D83D + DE48），不受影响。未配对的代理会导致 JSON 序列化错误。

### 12.4 TypeBox 辅助 (`typebox-helpers.ts`)

```typescript
export function StringEnum<T extends readonly string[]>(
    values: T,
    options?: { description?: string; default?: T[number] },
): TUnsafe<T[number]> {
    return Type.Unsafe<T[number]>({
        type: "string",
        enum: values as any,
        // TypeBox 标准 Union(Literal("a"), Literal("b")) 生成 anyOf/const
        // Google API 不支持 anyOf/const，所以用 enum
    });
}
```

### 12.5 确定性哈希 (`hash.ts`)

```typescript
export function shortHash(str: string): string {
    let h1 = 0xdeadbeef;
    let h2 = 0x41c6ce57;
    // 双哈希 MurmurHash 变体
    // 返回 base36 编码的短哈希（约 13 字符）
}
```

用于缩短过长的字符串（如 tool call ID）。

---

## 十三、OAuth 认证子系统 (`src/utils/oauth/`)

### 13.1 架构

```
OAuthProviderInterface
├── anthropicOAuthProvider       # Anthropic Claude Pro/Max
├── githubCopilotOAuthProvider   # GitHub Copilot
├── geminiCliOAuthProvider       # Google Gemini CLI (Cloud Code Assist)
├── antigravityOAuthProvider     # Google Antigravity
└── openaiCodexOAuthProvider     # OpenAI Codex (ChatGPT)
```

### 13.2 Provider 接口

```typescript
export interface OAuthProviderInterface {
    readonly id: OAuthProviderId;
    readonly name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    usesCallbackServer?: boolean;
    refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;
    getApiKey(credentials: OAuthCredentials): string;
    modifyModels?(models: Model<Api>[], credentials: OAuthCredentials): Model<Api>[];
}
```

`login()` 接收回调对象——因为 OAuth 流程需要用户交互（打开浏览器、输入代码等），而具体的 UI 层由调用方实现：

```typescript
export interface OAuthLoginCallbacks {
    onAuth: (info: OAuthAuthInfo) => void;          // 提示用户打开浏览器
    onPrompt: (prompt: OAuthPrompt) => Promise<string>;  // 获取用户输入
    onProgress?: (message: string) => void;         // 进度提示
    onManualCodeInput?: () => Promise<string>;      // 手动输入授权码
    signal?: AbortSignal;
}
```

### 13.3 OAuth 注册中心

```typescript
const BUILT_IN_OAUTH_PROVIDERS: OAuthProviderInterface[] = [
    anthropicOAuthProvider,
    githubCopilotOAuthProvider,
    geminiCliOAuthProvider,
    antigravityOAuthProvider,
    openaiCodexOAuthProvider,
];

const oauthProviderRegistry = new Map<string, OAuthProviderInterface>(...);

export function getOAuthProvider(id): OAuthProviderInterface | undefined;
export function registerOAuthProvider(provider): void;
export function unregisterOAuthProvider(id): void;  // 内置的卸载后会恢复默认
```

### 13.4 Token 刷新

```typescript
export async function getOAuthApiKey(
    providerId, credentials
): Promise<{ newCredentials, apiKey } | null> {
    let creds = credentials[providerId];
    if (!creds) return null;

    // 自动刷新过期 Token
    if (Date.now() >= creds.expires) {
        creds = await provider.refreshToken(creds);
    }

    return { newCredentials: creds, apiKey: provider.getApiKey(creds) };
}
```

---

## 十四、SimpleStreamOptions 映射 (`src/providers/simple-options.ts`)

这个模块将统一的 `SimpleStreamOptions` 映射为 Provider 能理解的基础选项。

```typescript
export function buildBaseOptions(model, options?, apiKey?): StreamOptions {
    return {
        temperature: options?.temperature,
        maxTokens: options?.maxTokens || Math.min(model.maxTokens, 32000),
        signal: options?.signal,
        apiKey: apiKey || options?.apiKey,
        cacheRetention: options?.cacheRetention,
        // ...所有通用字段
    };
}

// 思考预算计算
export function adjustMaxTokensForThinking(
    baseMaxTokens, modelMaxTokens, reasoningLevel, customBudgets?
): { maxTokens, thinkingBudget } {
    const defaultBudgets = { minimal: 1024, low: 2048, medium: 8192, high: 16384 };
    const budgets = { ...defaultBudgets, ...customBudgets };
    const thinkingBudget = budgets[level];
    const maxTokens = Math.min(baseMaxTokens + thinkingBudget, modelMaxTokens);
    // 保证至少 1024 tokens 给输出
    if (maxTokens <= thinkingBudget) {
        thinkingBudget = Math.max(0, maxTokens - 1024);
    }
    return { maxTokens, thinkingBudget };
}

// xhigh → high 降级（不支持 xhigh 的 Provider）
export function clampReasoning(effort) {
    return effort === "xhigh" ? "high" : effort;
}
```

---

## 十五、CLI 工具 (`src/cli.ts`)

```bash
npx @mariozechner/pi-ai login              # 交互式选择 Provider 登录
npx @mariozechner/pi-ai login anthropic    # 直接登录指定 Provider
npx @mariozechner/pi-ai list               # 列出所有 OAuth Provider
```

CLI 逻辑简单：读取 `auth.json` 文件存储凭据，调用 `provider.login()` 执行 OAuth 流程。

---

## 十六、package.json 子路径导出

```json
{
    "exports": {
        ".": { "import": "./dist/index.js" },
        "./anthropic": { "import": "./dist/providers/anthropic.js" },
        "./google": { "import": "./dist/providers/google.js" },
        "./openai-completions": { "import": "./dist/providers/openai-completions.js" },
        "./oauth": { "import": "./dist/oauth.js" },
        "./bedrock-provider": { "import": "./dist/bedrock-provider.js" },
        // ...
    }
}
```

这允许直接导入单个 Provider 模块：

```typescript
// 常规使用（通过主入口，自动懒加载所有 Provider）
import { stream, getModel } from "@mariozechner/pi-ai";

// 直接使用单个 Provider（跳过注册中心）
import { streamAnthropic } from "@mariozechner/pi-ai/anthropic";
```

**Bedrock 特殊处理**：`bedrock-provider.ts` 独立导出 `setBedrockProviderModule()`，因为 AWS SDK 在浏览器中不可用，web-ui 包需要完全绕开它。

---

## 十七、测试体系概览

测试使用 Vitest，40+ 测试文件覆盖：

| 类别 | 测试文件 | 说明 |
|---|---|---|
| 核心流式 | `stream.test.ts` | 所有 Provider 的基本流式响应 |
| Token 统计 | `tokens.test.ts`, `total-tokens.test.ts` | 费用计算正确性 |
| 中止处理 | `abort.test.ts` | AbortSignal 中断 |
| 空响应 | `empty.test.ts` | 模型返回空内容 |
| 上下文溢出 | `context-overflow.test.ts` | 溢出检测 |
| 工具调用 | `tool-call-without-result.test.ts`, `image-tool-result.test.ts` | 工具流程 |
| 跨 Provider | `cross-provider-handoff.test.ts` | A→B Provider 切换 |
| 消息转换 | `transform-messages-copilot-openai-to-anthropic.test.ts` | 消息格式转换 |
| Provider 特有 | `anthropic-thinking-disable.test.ts`, `google-thinking-signature.test.ts`, ... | 特定 Provider 行为 |
| 参数校验 | `validation.test.ts` | AJV 校验 |
| 懒加载 | `lazy-module-load.test.ts` | 模块按需加载 |
| OAuth | `anthropic-oauth.test.ts`, `github-copilot-oauth.test.ts` | OAuth 流程 |
| Unicode | `unicode-surrogate.test.ts` | 代理对清理 |

大部分测试需要真实的 API Key（通过环境变量配置），少数测试（如 `validation.test.ts`, `lazy-module-load.test.ts`）可以离线运行。

---

## 十八、关键设计模式总结

### 18.1 事件流协议（Event Stream Protocol）

所有 Provider 的输出都通过 `AssistantMessageEventStream` 统一，这是整个包的核心抽象：

```
Producer (Provider)          Consumer (Agent/UI)
      │                            │
      │ push(start)    ──────→     │ for await (const event of stream)
      │ push(text_start) ────→     │   if (event.type === "text_delta")
      │ push(text_delta) ────→     │     render(event.delta)
      │ push(text_delta) ────→     │
      │ push(text_end)   ────→     │
      │ push(done)       ────→     │ // 循环结束
      │ end()                      │
```

### 18.2 懒加载 + Promise 缓存

```
首次调用 stream(anthropic_model, ...)
    → createLazyStream 创建空 outer stream
    → import("./anthropic.js") 触发（首次）
    → Promise 缓存到 anthropicProviderModulePromise
    → 模块加载完成 → forwardStream(outer, inner)

后续调用 stream(anthropic_model, ...)
    → ||= 直接复用已缓存的 Promise
    → Promise 已 resolved → 直接拿到 module
```

### 18.3 onPayload 钩子

所有 Provider 都支持 `onPayload` 回调，允许在发送前修改请求：

```typescript
const stream = stream(model, context, {
    onPayload: (payload, model) => {
        // 可以修改、记录或替换请求体
        console.log("Request:", JSON.stringify(payload));
        return undefined;  // undefined = 不修改
    },
});
```

### 18.4 双层 API

```
应用层（Agent / Coding-Agent / Web-UI）
    │
    │ streamSimple(model, context, { reasoning: "high" })
    │
    ▼
简化层（simple-options.ts + 各 Provider 的 streamSimple*）
    │ 映射 reasoning: "high" → thinkingEnabled: true, effort: "high" (Anthropic)
    │                        → thinking: { enabled: true, budgetTokens: 32768 } (Google)
    │                        → reasoningEffort: "high" (OpenAI)
    ▼
原始层（各 Provider 的 stream* 函数）
    │ 直接传入 Provider 专属选项
    ▼
SDK → HTTP → LLM API
```

---

## 十九、核心数据流图

```
用户代码
  │
  │ stream(model, context, options)
  │
  ├─ stream.ts: resolveApiProvider(model.api)
  │    │
  │    └─ api-registry.ts: getApiProvider("anthropic-messages")
  │         │
  │         └─ 返回 { stream: lazyStream, streamSimple: lazyStreamSimple }
  │
  ├─ lazyStream(model, context, options)
  │    │
  │    ├─ new AssistantMessageEventStream()  ← 立即返回给用户
  │    │
  │    └─ import("./anthropic.js")  ← 异步加载
  │         │
  │         └─ streamAnthropic(model, context, options)
  │              │
  │              ├─ createClient()  ← 创建 Anthropic SDK 客户端
  │              │    ├─ API Key / OAuth Token / Copilot Token
  │              │    ├─ Beta headers (interleaved-thinking, fine-grained-tool-streaming)
  │              │    └─ Claude Code 伪装 (OAuth 模式)
  │              │
  │              ├─ buildParams()
  │              │    ├─ convertMessages()  ← 调用 transformMessages() 做跨 Provider 转换
  │              │    ├─ convertTools()
  │              │    ├─ thinking 配置 (adaptive / budget / disabled)
  │              │    └─ cache_control
  │              │
  │              ├─ onPayload() hook
  │              │
  │              ├─ client.messages.stream(params)  ← SDK 发起 HTTP 请求
  │              │
  │              └─ for await (event of sdkStream)
  │                   │
  │                   ├─ message_start → usage
  │                   ├─ content_block_start → text_start / thinking_start / toolcall_start
  │                   ├─ content_block_delta → text_delta / thinking_delta / toolcall_delta
  │                   ├─ content_block_stop → text_end / thinking_end / toolcall_end
  │                   └─ message_delta → stopReason + usage
  │
  └─ 用户消费 AssistantMessageEventStream
       │
       ├─ for await (const event of stream) { ... }
       │
       └─ const result = await stream.result()
```

---

## 二十、源码阅读建议

### 推荐阅读顺序

1. **`types.ts`**：理解核心类型（10 分钟搞清所有数据结构）
2. **`utils/event-stream.ts`**：理解事件流引擎（生产者-消费者模式）
3. **`api-registry.ts`**：理解注册中心（简单的 Map + 类型包装）
4. **`stream.ts`**：理解顶层 API（纯路由，3 个函数）
5. **`providers/register-builtins.ts`**：理解懒加载（核心设计）
6. **`providers/anthropic.ts`**：精读一个完整 Provider（最复杂也最典型）
7. **`providers/transform-messages.ts`**：理解跨 Provider 兼容性
8. **`providers/simple-options.ts`**：理解 SimpleStreamOptions 映射
9. **`models.ts` + `env-api-keys.ts`**：理解模型系统
10. **`utils/`**：按需阅读辅助工具

### 调试技巧

```typescript
// 1. 使用 onPayload 查看发送给 Provider 的完整请求
const s = stream(model, context, {
    onPayload: (payload) => { console.log(JSON.stringify(payload, null, 2)); },
});

// 2. 逐事件打印流
for await (const event of s) {
    console.log(event.type, event.type.includes("delta") ? (event as any).delta?.slice(0, 50) : "");
}

// 3. 查看最终消息的完整数据
const msg = await s.result();
console.log(JSON.stringify(msg, null, 2));
```

### 关键断点位置

- `stream.ts:30` — `stream()` 入口
- `register-builtins.ts:174` — `loadModule().then(...)` 首次模块加载
- `anthropic.ts:265` — `for await (const event of anthropicStream)` 事件循环
- `transform-messages.ts:17` — `transformMessages()` 消息转换入口
- `event-stream.ts:30` — `push()` 事件推送
