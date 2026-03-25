# Pi-web-ui (@mariozechner/pi-web-ui) 源码详细解读

## 1. 包概述

`@mariozechner/pi-web-ui` 是 Pi Monorepo 中的 **Web 前端包**，提供了一套可复用的 AI Chat 界面组件库。它基于 **Lit** + **mini-lit**（自定义 Web Components 框架）构建，可嵌入任意 Web 应用（包括浏览器扩展）。

### 1.1 核心职责

- 提供 **ChatPanel** —— 完整的 AI 对话面板（含消息列表、输入框、Artifact 面板）
- 实现 **Artifacts 系统** —— LLM 生成的文件实时预览（HTML、SVG、PDF、Excel、Markdown 等）
- 提供 **Sandbox 沙箱** —— 安全执行 LLM 生成的 HTML/JS 代码
- 实现 **IndexedDB 存储层** —— 会话持久化、API Key 管理、设置存储
- 支持 **CORS Proxy** —— 浏览器端通过代理服务器中转 LLM API 请求
- 支持 **本地模型发现** —— Ollama、LM Studio 等本地模型自动发现
- 提供 **国际化（i18n）** —— 内置英文/德文，可扩展其他语言

### 1.2 依赖关系

```
@mariozechner/pi-web-ui
    ├── @mariozechner/pi-ai           (LLM 抽象层)
    ├── @mariozechner/pi-tui          (复用部分类型)
    ├── @mariozechner/mini-lit        (Web Components UI 框架，peer dependency)
    ├── lit                           (Web Components 标准框架，peer dependency)
    ├── @sinclair/typebox             (JSON Schema 工具参数)
    ├── @lmstudio/sdk                 (LM Studio 模型发现)
    ├── ollama                        (Ollama 模型发现)
    ├── pdfjs-dist                    (PDF 渲染)
    ├── docx-preview                  (DOCX 预览)
    ├── xlsx                          (Excel 预览)
    ├── jszip                         (ZIP 处理)
    └── lucide                        (图标库)
```

### 1.3 文件结构

```
packages/web-ui/
├── src/
│   ├── index.ts                       # 公共 API 入口（~120 个导出）
│   ├── ChatPanel.ts                   # 顶层聊天面板组件
│   ├── app.css                        # Tailwind CSS 样式
│   ├── components/
│   │   ├── AgentInterface.ts          # Agent 交互主界面
│   │   ├── MessageList.ts             # 消息列表组件
│   │   ├── Messages.ts               # 消息类型组件（User/Assistant/Tool）
│   │   ├── MessageEditor.ts           # 消息输入编辑器
│   │   ├── Input.ts                   # 输入框组件
│   │   ├── StreamingMessageContainer.ts # 流式消息容器
│   │   ├── SandboxedIframe.ts         # 沙箱 iframe 组件
│   │   ├── ThinkingBlock.ts           # 推理过程展示
│   │   ├── ConsoleBlock.ts            # 控制台输出展示
│   │   ├── AttachmentTile.ts          # 附件预览磁贴
│   │   ├── ExpandableSection.ts       # 可折叠区域
│   │   ├── ProviderKeyInput.ts        # API Key 输入组件
│   │   ├── CustomProviderCard.ts      # 自定义 Provider 卡片
│   │   ├── message-renderer-registry.ts # 消息渲染器注册表
│   │   └── sandbox/                   # 沙箱运行时通信子系统
│   │       ├── SandboxRuntimeProvider.ts  # Provider 接口
│   │       ├── RuntimeMessageRouter.ts    # 全局消息路由器（单例）
│   │       ├── RuntimeMessageBridge.ts    # 消息桥（代码注入）
│   │       ├── ArtifactsRuntimeProvider.ts # Artifact 数据注入
│   │       ├── AttachmentsRuntimeProvider.ts # 附件数据注入
│   │       ├── ConsoleRuntimeProvider.ts  # 控制台捕获
│   │       └── FileDownloadRuntimeProvider.ts # 文件下载能力
│   ├── dialogs/
│   │   ├── ModelSelector.ts           # 模型选择对话框
│   │   ├── SettingsDialog.ts          # 设置面板（API Keys / Proxy / Settings）
│   │   ├── SessionListDialog.ts       # 会话列表
│   │   ├── ApiKeyPromptDialog.ts      # API Key 提示
│   │   ├── AttachmentOverlay.ts       # 附件预览覆盖层
│   │   ├── CustomProviderDialog.ts    # 自定义 Provider 对话框
│   │   ├── PersistentStorageDialog.ts # 持久化存储提示
│   │   └── ProvidersModelsTab.ts      # Provider 模型管理
│   ├── storage/
│   │   ├── app-storage.ts             # AppStorage 全局单例
│   │   ├── store.ts                   # Store 抽象基类
│   │   ├── types.ts                   # 存储类型定义
│   │   ├── backends/
│   │   │   └── indexeddb-storage-backend.ts  # IndexedDB 后端实现
│   │   └── stores/
│   │       ├── settings-store.ts      # 设置存储
│   │       ├── provider-keys-store.ts # API Key 存储
│   │       ├── sessions-store.ts      # 会话存储
│   │       └── custom-providers-store.ts # 自定义 Provider 存储
│   ├── tools/
│   │   ├── index.ts                   # 工具渲染器注册
│   │   ├── types.ts                   # ToolRenderer 类型
│   │   ├── renderer-registry.ts       # 渲染器注册表
│   │   ├── javascript-repl.ts         # JavaScript REPL 工具
│   │   ├── extract-document.ts        # 文档提取工具
│   │   ├── artifacts/                 # Artifacts 子系统
│   │   │   ├── artifacts.ts           # ArtifactsPanel + Artifact 工具定义
│   │   │   ├── ArtifactElement.ts     # Artifact 元素基类
│   │   │   ├── HtmlArtifact.ts        # HTML Artifact（沙箱渲染）
│   │   │   ├── SvgArtifact.ts         # SVG Artifact
│   │   │   ├── MarkdownArtifact.ts    # Markdown Artifact
│   │   │   ├── ImageArtifact.ts       # 图片 Artifact
│   │   │   ├── PdfArtifact.ts         # PDF Artifact（pdfjs-dist）
│   │   │   ├── ExcelArtifact.ts       # Excel Artifact（xlsx）
│   │   │   ├── DocxArtifact.ts        # DOCX Artifact（docx-preview）
│   │   │   ├── TextArtifact.ts        # 纯文本/代码 Artifact
│   │   │   ├── GenericArtifact.ts     # 通用 Artifact（下载链接）
│   │   │   └── ArtifactPill.ts        # Artifact 内联胶囊标签
│   │   └── renderers/                 # 内置工具渲染器
│   │       ├── DefaultRenderer.ts
│   │       ├── BashRenderer.ts
│   │       ├── CalculateRenderer.ts
│   │       └── GetCurrentTimeRenderer.ts
│   ├── prompts/
│   │   └── prompts.ts                 # LLM Prompt 模板（Artifact/REPL 工具描述）
│   └── utils/
│       ├── proxy-utils.ts             # CORS Proxy 决策逻辑
│       ├── model-discovery.ts         # Ollama / LM Studio 模型发现
│       ├── attachment-utils.ts        # 附件处理
│       ├── auth-token.ts              # OAuth Token 管理
│       ├── format.ts                  # 数字/Token/Cost 格式化
│       └── i18n.ts                    # 国际化（英文/德文）
├── example/                           # 完整示例应用
│   ├── src/main.ts                    # 示例入口
│   └── vite.config.ts                 # Vite 开发服务器
└── package.json
```

---

## 2. 组件架构

### 2.1 组件层次

```
ChatPanel (pi-chat-panel)
├── AgentInterface (agent-interface)
│   ├── MessageList (message-list)
│   │   ├── UserMessage (user-message)
│   │   ├── AssistantMessage (assistant-message)
│   │   │   ├── ThinkingBlock (thinking-block)
│   │   │   └── ToolMessage (tool-message) → ToolRenderer
│   │   └── Custom Message Renderers
│   ├── StreamingMessageContainer (streaming-message-container)
│   ├── MessageEditor (message-editor)
│   │   └── AttachmentTile (attachment-tile)
│   └── Footer（Token 统计、模型选择、Thinking 选择器）
├── ArtifactsPanel (artifacts-panel)
│   ├── HtmlArtifact → SandboxIframe
│   ├── SvgArtifact
│   ├── MarkdownArtifact
│   ├── ImageArtifact
│   ├── PdfArtifact
│   ├── ExcelArtifact
│   ├── DocxArtifact
│   ├── TextArtifact
│   └── GenericArtifact
└── ArtifactPill（浮动胶囊，折叠时显示）
```

### 2.2 技术选型

- **Lit**：Google 的 Web Components 框架，提供 `html` tagged template、`@customElement` 装饰器、reactive properties
- **mini-lit**：项目自定义的 UI 组件库（Button、Badge、Input、ThemeToggle、MarkdownBlock 等）
- **Tailwind CSS**：原子化 CSS 框架
- **Light DOM**：所有组件使用 `createRenderRoot() { return this; }` 禁用 Shadow DOM，共享全局样式

### 2.3 ChatPanel —— 顶层容器

```typescript
@customElement("pi-chat-panel")
export class ChatPanel extends LitElement {
    @state() public agent?: Agent;
    @state() public agentInterface?: AgentInterface;
    @state() public artifactsPanel?: ArtifactsPanel;
    @state() private hasArtifacts = false;
    @state() private showArtifactsPanel = false;
    @state() private windowWidth = 0;
}
```

ChatPanel 是最外层组件，负责：
1. **布局管理**：根据 `windowWidth` 在 side-by-side（>800px）和 overlay（≤800px）模式间切换
2. **Agent 绑定**：`setAgent()` 方法初始化 AgentInterface + ArtifactsPanel + 工具注册
3. **Artifact 面板控制**：显示/隐藏、浮动胶囊、响应式布局

### 2.4 AgentInterface —— 核心交互界面

```typescript
@customElement("agent-interface")
export class AgentInterface extends LitElement {
    @property({ attribute: false }) session?: Agent;
    @property({ type: Boolean }) enableAttachments = true;
    @property({ type: Boolean }) enableModelSelector = true;
    @property({ type: Boolean }) enableThinkingSelector = true;
}
```

AgentInterface 是对话的核心 UI，负责：

1. **Agent 事件订阅**：`setupSessionSubscription()` 订阅 Agent 事件驱动 UI 更新
2. **Proxy 自动配置**：如果检测到 Agent 使用默认 `streamSimple`，自动注入 proxy-aware 的 `createStreamFn()`
3. **API Key 自动获取**：从 IndexedDB 的 `ProviderKeysStore` 读取 API Key
4. **自动滚动**：`ResizeObserver` 监听内容变化，自动滚动到底部
5. **流式消息**：`StreamingMessageContainer` 接收 `message_update` 事件的 partial message 实时渲染

事件处理：

```typescript
this._unsubscribeSession = this.session.subscribe(async (ev: AgentEvent) => {
    switch (ev.type) {
        case "message_start":
        case "turn_start":
        case "turn_end":
        case "agent_start":
            this.requestUpdate();
            break;
        case "message_end":
            this._streamingContainer.setMessage(null, true);  // 清除流式消息
            this.requestUpdate();
            break;
        case "message_update":
            this._streamingContainer.setMessage(ev.message, !isStreaming);  // 更新流式消息
            this.requestUpdate();
            break;
    }
});
```

---

## 3. 自定义消息类型

### 3.1 声明合并扩展

`Messages.ts` 使用 TypeScript 声明合并为 `pi-agent-core` 的 `CustomAgentMessages` 添加 Web 专用消息类型：

```typescript
declare module "@mariozechner/pi-agent-core" {
    interface CustomAgentMessages {
        "user-with-attachments": UserMessageWithAttachments;
        artifact: ArtifactMessage;
    }
}
```

- **`user-with-attachments`**：带附件的用户消息（包含 `Attachment[]` 数组）
- **`artifact`**：Artifact 操作消息（create/update/delete），用于会话持久化

### 3.2 消息渲染器注册表

```typescript
// message-renderer-registry.ts
const messageRenderers = new Map<string, MessageRenderer>();

export function registerMessageRenderer(role: string, renderer: MessageRenderer): void;
export function getMessageRenderer(role: string): MessageRenderer | undefined;
export function renderMessage(message: AgentMessage): TemplateResult | null;
```

应用可以注册自定义消息角色的渲染器，例如 example app 中注册了 `system-notification` 渲染器。

### 3.3 convertToLlm

```typescript
export function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
    return messages.filter(
        (m) => m.role === "user" || m.role === "assistant" || m.role === "toolResult"
    );
}
```

默认的 `convertToLlm` 过滤掉所有自定义消息（artifact、user-with-attachments 等），只保留 LLM 标准消息。`convertAttachments()` 将 `user-with-attachments` 转换为标准 `user` 消息，将 Attachment 内容内联。

---

## 4. 存储层架构

### 4.1 分层设计

```
AppStorage (全局单例)
├── SettingsStore        → "settings" object store
├── ProviderKeysStore    → "provider-keys" object store
├── SessionsStore        → "sessions" + "sessions-metadata" object stores
├── CustomProvidersStore → "custom-providers" object store
└── StorageBackend (接口)
    └── IndexedDBStorageBackend (实现)
```

### 4.2 StorageBackend 接口

```typescript
interface StorageBackend {
    get<T>(storeName: string, key: string): Promise<T | null>;
    set<T>(storeName: string, key: string, value: T): Promise<void>;
    delete(storeName: string, key: string): Promise<void>;
    keys(storeName: string, prefix?: string): Promise<string[]>;
    getAllFromIndex<T>(storeName: string, indexName: string, direction?): Promise<T[]>;
    clear(storeName: string): Promise<void>;
    has(storeName: string, key: string): Promise<boolean>;
    transaction<T>(storeNames, mode, operation): Promise<T>;
    getQuotaInfo(): Promise<{ usage, quota, percent }>;
    requestPersistence(): Promise<boolean>;
}
```

关键设计：
- **多 store 事务**：`transaction()` 支持跨 object store 的原子操作
- **配额管理**：`getQuotaInfo()` 监控 IndexedDB 使用量，`requestPersistence()` 请求持久化（防止浏览器自动清理）
- **索引查询**：`getAllFromIndex()` 支持按索引排序查询（如按 `lastModified` 降序列出会话）

### 4.3 Store 抽象基类

```typescript
abstract class Store {
    abstract getConfig(): StoreConfig;
    setBackend(backend: StorageBackend): void;
    protected getBackend(): StorageBackend;
}
```

所有 Store 继承此基类。`getConfig()` 返回 IndexedDB object store 配置（名称、keyPath、索引）。

### 4.4 SessionsStore —— 双 Store 设计

```
sessions           → 完整会话数据（SessionData: messages[] 等）
sessions-metadata  → 轻量元数据（SessionMetadata: title, messageCount, usage 等）
```

分离设计的优势：
- **列表页只读 metadata**：会话列表/搜索只需加载轻量的 metadata，无需加载完整消息
- **原子保存**：`save()` 在单个事务中同时更新 data 和 metadata
- **索引支持**：metadata store 有 `lastModified` 索引，支持按时间排序

### 4.5 IndexedDB 初始化

```typescript
const backend = new IndexedDBStorageBackend({
    dbName: "pi-web-ui-example",
    version: 2,
    stores: [
        settings.getConfig(),
        SessionsStore.getMetadataConfig(),
        providerKeys.getConfig(),
        customProviders.getConfig(),
        sessions.getConfig(),
    ],
});

// 连线
settings.setBackend(backend);
providerKeys.setBackend(backend);
sessions.setBackend(backend);
customProviders.setBackend(backend);

const storage = new AppStorage(settings, providerKeys, sessions, customProviders, backend);
setAppStorage(storage);  // 设置全局单例
```

---

## 5. Artifacts 系统

### 5.1 概述

Artifacts 是 LLM 生成的文件的实时预览系统。LLM 通过 `artifacts` 工具创建/更新文件，ArtifactsPanel 实时渲染预览。

### 5.2 Artifact 工具定义

```typescript
const artifactsParamsSchema = Type.Object({
    command: StringEnum(["create", "update", "rewrite", "get", "delete", "logs"]),
    filename: Type.String(),
    content: Type.Optional(Type.String()),
    old_str: Type.Optional(Type.String()),    // update 命令：搜索文本
    new_str: Type.Optional(Type.String()),    // update 命令：替换文本
});
```

6 个操作：

| 命令 | 说明 |
|------|------|
| `create` | 创建新文件 |
| `update` | Search/Replace 更新（类似 edit 工具） |
| `rewrite` | 完全重写文件内容 |
| `get` | 获取文件当前内容 |
| `delete` | 删除文件 |
| `logs` | 获取 HTML Artifact 的控制台日志 |

### 5.3 ArtifactsPanel

```typescript
@customElement("artifacts-panel")
export class ArtifactsPanel extends LitElement {
    private _artifacts = new Map<string, Artifact>();
    private artifactElements = new Map<string, ArtifactElement>();
    private _activeFilename: string | null = null;
}
```

核心功能：
- **文件类型路由**：根据扩展名创建对应的 ArtifactElement（HTML → HtmlArtifact, SVG → SvgArtifact, etc.）
- **Tab 式导航**：多文件通过标签页切换
- **实时更新**：`update` 命令执行 search/replace 后立即刷新预览
- **持久化**：通过 `ArtifactMessage` 记录操作历史，`reconstructFromMessages()` 从消息历史重建

### 5.4 文件类型 → Artifact 组件映射

```
.html               → HtmlArtifact（沙箱 iframe 渲染）
.svg                → SvgArtifact（内联 SVG）
.md / .markdown     → MarkdownArtifact（Markdown 渲染）
.png/.jpg/.gif/...  → ImageArtifact（图片展示）
.pdf                → PdfArtifact（pdfjs-dist 渲染）
.xlsx/.xls          → ExcelArtifact（xlsx 库解析 + 表格展示）
.docx               → DocxArtifact（docx-preview 渲染）
.txt/.json/.js/...  → TextArtifact（代码高亮）
其他                 → GenericArtifact（下载链接）
```

### 5.5 HtmlArtifact —— 沙箱渲染

HtmlArtifact 是最复杂的 Artifact 类型，通过 `SandboxedIframe` 在隔离环境中渲染 HTML：

```
HtmlArtifact
├── SandboxedIframe.loadContent(sandboxId, html, runtimeProviders)
│   ├── RuntimeMessageRouter.registerSandbox()
│   ├── prepareHtmlDocument()   → 注入 RuntimeBridge + Provider 数据
│   ├── iframe.srcdoc = html    → 加载到沙箱
│   └── 消息通信 ← → RuntimeMessageRouter
└── Runtime Providers
    ├── ConsoleRuntimeProvider  → 捕获 console.log 输出
    ├── ArtifactsRuntimeProvider → 注入其他 Artifact 文件内容
    └── AttachmentsRuntimeProvider → 注入用户上传的附件
```

---

## 6. Sandbox 沙箱系统

### 6.1 安全模型

```html
<iframe sandbox="allow-scripts allow-modals" srcdoc="...">
```

- **`sandbox` 属性**：限制 iframe 的能力
- **`allow-scripts`**：允许执行 JavaScript
- **`allow-modals`**：允许 alert/confirm
- **不允许**：同源访问、表单提交、导航、popups
- **跨域隔离**：srcdoc iframe 与主页面完全隔离

### 6.2 通信架构

```
主页面（Host）                        沙箱（Sandbox iframe）
┌──────────────────┐                ┌──────────────────┐
│  RuntimeMessage   │                │  window.send     │
│  Router          │◄─postMessage──│  RuntimeMessage() │
│  (全局单例)       │──postMessage─►│                   │
│  ┌─────────────┐ │                │  注入的代码：      │
│  │ Providers[] │ │                │  - getData()      │
│  │ Consumers[] │ │                │  - getRuntime()   │
│  └─────────────┘ │                │  - Bridge code    │
└──────────────────┘                └──────────────────┘
```

### 6.3 RuntimeMessageRouter（全局单例）

```typescript
class RuntimeMessageRouter {
    private sandboxes = new Map<string, SandboxContext>();

    registerSandbox(id, providers, consumers): void;
    setSandboxIframe(id, iframe): void;
    unregisterSandbox(id): void;
}

export const RUNTIME_MESSAGE_ROUTER = new RuntimeMessageRouter();
```

单一全局 `window.addEventListener("message", ...)` 监听器：
1. 接收 iframe postMessage
2. 根据 `message.sandboxId` 路由到对应的 sandbox context
3. 将消息分发给 providers（可返回响应）和 consumers（只接收）

### 6.4 SandboxRuntimeProvider 接口

```typescript
interface SandboxRuntimeProvider {
    getData(): Record<string, any>;       // 注入到 window 的数据
    getRuntime(): (sandboxId: string) => void;  // 注入到 sandbox 的运行时代码
    handleMessage?(message, respond): Promise<void>;  // 消息处理
    getDescription(): string;             // LLM 工具描述
    onExecutionStart?(sandboxId, signal): void;  // 生命周期：开始
    onExecutionEnd?(sandboxId): void;     // 生命周期：结束
}
```

四个内置 Provider：

| Provider | 说明 |
|----------|------|
| ConsoleRuntimeProvider | 劫持 `console.log/warn/error`，通过 postMessage 传回主页面 |
| ArtifactsRuntimeProvider | 注入 `window.artifacts`，提供 `readFile/writeFile` API |
| AttachmentsRuntimeProvider | 注入 `window.attachments`，提供用户上传的附件数据 |
| FileDownloadRuntimeProvider | 注入 `window.downloadFile()`，支持沙箱内文件下载 |

### 6.5 RuntimeMessageBridge —— 代码注入

```typescript
class RuntimeMessageBridge {
    static generateBridgeCode(options: { context, sandboxId }): string;
}
```

生成的 bridge 代码（字符串形式）被注入到沙箱的 `<script>` 标签中：

```javascript
// 注入到 sandbox 的代码（简化版）
window.sendRuntimeMessage = async (message) => {
    return new Promise((resolve, reject) => {
        const handler = (e) => {
            if (e.data.messageId === messageId) {
                resolve(e.data);
            }
        };
        window.addEventListener('message', handler);
        window.parent.postMessage({
            ...message,
            sandboxId: "sandbox-123",
            messageId: messageId
        }, '*');
    });
};
```

这实现了 iframe ↔ 主页面的**双向 RPC 通信**：
- **Sandbox → Host**：`window.parent.postMessage()` → Router → Provider.handleMessage() → 响应
- **Host → Sandbox**：`iframe.contentWindow.postMessage()` → sandbox 内 message listener

### 6.6 两种沙箱加载模式

1. **srcdoc 模式**（Web 应用）：直接将 HTML 设置为 `iframe.srcdoc`
2. **sandbox URL 模式**（浏览器扩展）：iframe 指向 `chrome.runtime.getURL("sandbox.html")`，通过 postMessage 传递内容

---

## 7. JavaScript REPL 工具

```typescript
export function createJavaScriptReplTool(
    runtimeProviders: () => SandboxRuntimeProvider[],
    sandboxUrlProvider?: () => string,
): AgentTool<typeof replSchema>
```

REPL 工具让 LLM 可以在沙箱中执行 JavaScript 代码。

执行流程：
1. LLM 调用 `javascript_repl` 工具，传入 `code` 参数
2. 创建临时 `SandboxIframe`（隐藏）
3. `sandbox.execute(sandboxId, code, runtimeProviders)` 执行代码
4. 收集 console 输出 + return value + 生成的文件
5. 返回给 LLM 作为工具结果

Runtime Providers 赋予 REPL 代码额外能力：
- 读取/写入 Artifacts
- 访问用户上传的附件
- 下载文件

---

## 8. CORS Proxy 系统

### 8.1 问题背景

浏览器直接调用 LLM API 面临两个问题：
1. **CORS 限制**：Anthropic 等 API 不允许浏览器直接请求
2. **API Key 安全**：OAuth Token 类型的 Key 不能暴露在前端

### 8.2 代理决策逻辑

```typescript
function shouldUseProxyForProvider(provider: string, apiKey: string): boolean {
    switch (provider) {
        case "zai":         return true;              // 始终需要代理
        case "anthropic":   return apiKey.startsWith("sk-ant-oat");  // OAuth Token 需要
        case "openai-codex": return true;             // chatgpt.com 后端无 CORS
        case "openai":
        case "google":
        case "groq":
        case "ollama":      return false;             // 不需要代理
    }
}
```

### 8.3 代理应用

```typescript
function applyProxyIfNeeded(model, apiKey, proxyUrl) {
    if (!shouldUseProxyForProvider(model.provider, apiKey)) return model;
    return {
        ...model,
        baseUrl: `${proxyUrl}/?url=${encodeURIComponent(model.baseUrl)}`,
    };
}
```

代理通过修改 model 的 `baseUrl` 实现，将原始 URL 编码为 query parameter。

### 8.4 createStreamFn

```typescript
export function createStreamFn(getProxyUrl): StreamFn {
    return async (model, context, options) => {
        const proxyUrl = await getProxyUrl();
        const apiKey = options?.apiKey || "";
        const proxiedModel = applyProxyIfNeeded(model, apiKey, proxyUrl);
        return streamSimple(proxiedModel, context, options);
    };
}
```

`createStreamFn` 是一个高阶函数，包装 `streamSimple`，在调用前自动应用代理。

---

## 9. 模型发现

### 9.1 本地模型自动发现

支持三种本地 AI 运行时：

#### Ollama

```typescript
async function discoverOllamaModels(baseUrl): Promise<Model[]> {
    const ollama = new Ollama({ host: baseUrl });
    const { models } = await ollama.list();
    // 过滤不支持 tools 的模型
    // 获取 context_length、reasoning 能力等
    return ollamaModels;
}
```

#### LM Studio

```typescript
async function discoverLMStudioModels(baseUrl): Promise<Model[]> {
    const client = new LMStudioClient({ baseUrl });
    const models = await client.llm.listLoaded();
    return models.map(m => ({
        id: m.identifier,
        api: "openai-completions",
        provider: "",
        baseUrl: `${baseUrl}/v1`,
        // ...
    }));
}
```

#### llama.cpp（OpenAI 兼容端点）

```typescript
async function discoverLlamaCppModels(baseUrl): Promise<Model[]> {
    const response = await fetch(`${baseUrl}/v1/models`);
    const { data } = await response.json();
    return data.map(/* ... */);
}
```

---

## 10. 国际化（i18n）

```typescript
// i18n.ts
declare module "@mariozechner/mini-lit" {
    interface i18nMessages extends MiniLitRequiredMessages {
        Free: string;
        "Select Model": string;
        "Type your message...": string;
        // ~100+ 个翻译键
    }
}
```

使用 mini-lit 的 i18n 系统：
- 通过 TypeScript 声明合并扩展 `i18nMessages` 接口，获得编译时类型检查
- 内置英文和德文翻译
- `setLanguage("en" | "de")` 切换语言
- 所有 UI 文本通过 `i18n("key")` 获取

---

## 11. 工具渲染器系统

### 11.1 ToolRenderer 接口

```typescript
interface ToolRenderer {
    renderCall(toolCall: ToolCall, tools: AgentTool[]): ToolRenderResult;
    renderResult(toolResult: ToolResultMessage, toolCall: ToolCall, tools: AgentTool[]): ToolRenderResult;
}

interface ToolRenderResult {
    header: TemplateResult;     // 折叠标题
    content: TemplateResult;    // 展开内容
}
```

### 11.2 注册与使用

```typescript
registerToolRenderer("artifacts", new ArtifactsToolRenderer(artifactsPanel));
registerToolRenderer("calculate", new CalculateRenderer());
registerToolRenderer("bash", new BashRenderer());

// 消息组件中使用
const rendered = renderTool(toolCall, toolResult, tools);
```

未注册工具名使用 `DefaultRenderer`，展示 JSON 格式的参数和结果。

---

## 12. Example App

`example/` 目录提供了完整的示例应用，演示如何使用 `pi-web-ui`：

```typescript
// 1. 创建 stores
const settings = new SettingsStore();
const providerKeys = new ProviderKeysStore();
const sessions = new SessionsStore();

// 2. 创建 IndexedDB backend
const backend = new IndexedDBStorageBackend({ dbName: "pi-web-ui-example", version: 2, stores: configs });

// 3. 连线
settings.setBackend(backend);
sessions.setBackend(backend);
providerKeys.setBackend(backend);

// 4. 全局存储
const storage = new AppStorage(settings, providerKeys, sessions, customProviders, backend);
setAppStorage(storage);

// 5. 创建 Agent
const agent = new Agent({ convertToLlm: customConvertToLlm });

// 6. 创建 ChatPanel
const chatPanel = new ChatPanel();
await chatPanel.setAgent(agent, {
    toolsFactory: (agent, agentInterface, artifactsPanel, runtimeProvidersFactory) => [
        createJavaScriptReplTool(runtimeProvidersFactory),
    ],
});

// 7. 挂载
document.getElementById("chat-container")!.appendChild(chatPanel);
```

---

## 13. 与其他包的关系

```
@mariozechner/pi-ai ─────────────────────────────┐
    │                                              │
    ▼                                              │
@mariozechner/pi-agent-core                        │
    │                                              │
    ▼                                              ▼
@mariozechner/pi-web-ui ──────────────────────────→┘
    │
    ├── @mariozechner/mini-lit (UI 组件框架)
    └── lit (Web Components)
```

与 `pi-coding-agent` 的区别：
- `pi-coding-agent`：服务器端运行，文件系统操作，Terminal TUI
- `pi-web-ui`：浏览器端运行，IndexedDB 存储，Sandbox 隔离，CORS Proxy

---

## 14. 源码阅读建议

### 推荐阅读顺序

1. **example/src/main.ts** → 理解完整的初始化和使用流程
2. **ChatPanel.ts** → 理解顶层组件结构和 setAgent() 流程
3. **components/AgentInterface.ts** → 理解 Agent 事件订阅和 UI 更新
4. **components/Messages.ts** → 理解消息组件和自定义消息类型
5. **storage/types.ts** → 理解存储层抽象
6. **storage/backends/indexeddb-storage-backend.ts** → 理解 IndexedDB 实现
7. **tools/artifacts/artifacts.ts** → 理解 Artifact 工具和 Panel
8. **components/SandboxedIframe.ts** → 理解沙箱加载和通信
9. **components/sandbox/RuntimeMessageRouter.ts** → 理解全局消息路由
10. **components/sandbox/SandboxRuntimeProvider.ts** → 理解 Provider 接口
11. **utils/proxy-utils.ts** → 理解 CORS Proxy 决策
12. **utils/model-discovery.ts** → 理解本地模型发现

### 关键断点位置

| 文件 | 位置 | 说明 |
|------|------|------|
| ChatPanel.ts | `setAgent()` | 完整初始化流程 |
| AgentInterface.ts | `setupSessionSubscription()` | Agent 事件驱动 UI |
| AgentInterface.ts | `_handleSend()` | 消息发送入口 |
| SandboxedIframe.ts | `loadContent()` | 沙箱 HTML 加载 |
| SandboxedIframe.ts | `execute()` | 沙箱代码执行 |
| RuntimeMessageRouter.ts | `setupListener()` | 全局消息路由 |
| artifacts.ts | `ArtifactsPanel.execute()` | Artifact 工具执行 |
| proxy-utils.ts | `createStreamFn()` | Proxy 包装流函数 |
| sessions-store.ts | `save()` | 会话持久化 |

### 核心设计思想总结

1. **Web Components**：Lit + Light DOM，组件可嵌入任意 Web 页面
2. **沙箱安全**：iframe sandbox + postMessage RPC，LLM 生成的代码在隔离环境运行
3. **存储抽象**：StorageBackend 接口 → IndexedDB 实现，可替换为远程 API
4. **双 Store 会话**：metadata（轻量列表）+ data（完整消息），优化列表性能
5. **Provider 注入**：SandboxRuntimeProvider 向沙箱注入数据和 API
6. **全局消息路由**：单一 window message listener 路由到多个沙箱实例
7. **CORS Proxy 透明化**：自动检测是否需要代理，对上层透明
8. **渲染器注册表**：工具/消息渲染器可插拔，支持自定义扩展
