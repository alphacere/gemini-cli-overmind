# Core 模块架构深度分析

对 Gemini CLI Core 模块的代码组织、层级结构、主要模块和关键流程的详细分析。

## 一、项目整体结构

### 1.1 顶级目录布局

```
packages/core/src/
├── core/                    # 聊天引擎核心（4 个关键文件）
│   ├── client.ts           # GeminiClient - 对话管理和流程控制
│   ├── geminiChat.ts       # GeminiChat - 单个聊天会话管理
│   ├── turn.ts             # Turn - 单次对话轮次处理
│   └── contentGenerator.ts # 内容生成辅助类
│
├── tools/                   # 工具系统（130+ 文件）
│   ├── tool-registry.ts    # 工具注册表（主入口）
│   ├── tools.ts            # 所有工具的导出
│   ├── tool-names.ts       # 工具名称常量
│   ├── tool-error.ts       # 工具错误类
│   ├── core-tools/         # 核心工具实现
│   │   ├── read-file.ts
│   │   ├── write-file.ts
│   │   ├── edit.ts
│   │   ├── shell.ts
│   │   ├── web-fetch.ts
│   │   ├── ripGrep.ts
│   │   └── ... (40+ 工具)
│   ├── mcp-tool.ts         # MCP 工具包装器
│   └── tool-scheduler/     # 工具调度（内部）
│
├── config/                  # 配置系统（250+ 文件）
│   ├── config.ts           # Config 类（39K 行）
│   ├── settings.ts         # 用户设置管理
│   ├── models.ts           # 模型配置
│   ├── auth.ts             # 认证配置
│   ├── policy.ts           # 策略配置
│   └── ... (更多配置相关文件)
│
├── services/               # 业务服务层（10+ 服务）
│   ├── fileDiscoveryService.ts   # 文件发现和过滤
│   ├── gitService.ts             # Git 操作
│   ├── shellExecutionService.ts  # Shell 执行
│   ├── chatRecordingService.ts   # 对话记录
│   ├── chatCompressionService.ts # 对话压缩
│   ├── loopDetectionService.ts   # 循环检测
│   └── ... (更多服务)
│
├── mcp/                    # MCP 和 OAuth（15+ 文件）
│   ├── mcp-client-manager.ts # MCP 客户端管理
│   ├── oauth-provider.ts      # OAuth 提供者
│   ├── oauth-token-storage.ts # 令牌存储
│   └── ... (MCP 相关)
│
├── telemetry/              # 遥测系统（10+ 文件）
│   ├── index.ts            # 遥测导出
│   ├── events.ts           # 事件定义
│   ├── metrics.ts          # 指标记录
│   ├── activity.ts         # 活动监测
│   └── ... (遥测相关)
│
├── confirmation-bus/       # 消息总线（3-5 文件）
│   ├── message-bus.ts
│   └── types.ts
│
├── policy/                 # 策略引擎（5+ 文件）
│   ├── policy-engine.ts
│   └── types.ts
│
├── ide/                    # IDE 集成（5+ 文件）
│   ├── ide-client.ts
│   └── ideContext.ts
│
├── prompts/                # 提示词管理（5+ 文件）
│   ├── prompt-registry.ts
│   └── prompts/
│
├── agents/                 # 智能体系统（10+ 文件）
│   ├── agent-registry.ts
│   └── agents/
│
├── code_assist/            # 代码协助（5+ 文件）
│   └── ...
│
├── utils/                  # 工具函数（20+ 文件）
│   ├── logger.ts
│   ├── errors.ts
│   └── ... (各种工具函数)
│
├── output/                 # 输出处理（5+ 文件）
│   └── types.ts
│
└── index.ts               # 主导出文件（180+ 导出）
```

**关键统计**：
- **总代码行数**：~107,000 行（不含测试）
- **主要文件数**：~200+ 文件
- **核心类数**：50+ 主要类
- **工具数量**：48+ 内置工具 + MCP 动态工具

### 1.2 包依赖关系

**核心依赖** (`package.json`)：
```json
{
  "dependencies": {
    "@google/genai": "1.16.0",                    // Gemini API 官方 SDK
    "@modelcontextprotocol/sdk": "^1.15.1",     // MCP 协议 SDK
    "zod": "^3.23.8",                            // 数据验证和 schema
    "simple-git": "^3.28.0",                     // Git 操作库
    "tree-sitter": "^0.25.10",                   // 代码解析
    "@google-cloud/storage": "^7.16.0",          // Google Cloud 存储
    "undici": "^7.10.0",                         // HTTP 客户端
    "tar": "^7.5.1",                             // TAR 文件处理
    "glob": "^10.4.5",                           // 文件模式匹配
    "@opentelemetry/*": "^0.x.x"                 // 遥测（多个包）
  },
  "devDependencies": {
    "typescript": "^5.3.3",
    "vitest": "^3.2.4",
    "@types/node": "^20.11.24"
  }
}
```

**内部依赖关系**：
```
GeminiClient
  ├─ GeminiChat（发送消息）
  ├─ ToolRegistry（工具操作）
  ├─ Config（配置和初始化）
  ├─ Services（文件、Git 等）
  ├─ LoopDetectionService（循环检测）
  └─ ChatCompressionService（压缩）

Config
  ├─ GeminiClient（初始化）
  ├─ ToolRegistry（工具发现）
  ├─ FileDiscoveryService（文件）
  ├─ GitService（Git 操作）
  ├─ MCP 服务器配置
  └─ 各种服务

ToolRegistry
  ├─ 核心工具（tool-registry 自身定义）
  ├─ MCP 工具（通过 McpClientManager）
  └─ 工具调度器
```

### 1.3 导出和公共 API

**主导出文件** `index.ts`：
```typescript
// 配置和初始化
export * from './config/config.js';
export * from './output/types.js';
export * from './policy/types.js';
export * from './confirmation-bus/types.js';

// 核心对话引擎
export * from './core/client.js';           // GeminiClient
export * from './core/geminiChat.js';       // GeminiChat
export * from './core/turn.js';             // Turn
export * from './core/contentGenerator.js'; // ContentGenerator

// 工具系统
export * from './tools/tool-registry.js';
export * from './tools/tools.js';
export * from './tools/tool-names.js';
export * from './tools/tool-error.js';

// 服务层
export * from './services/fileDiscoveryService.js';
export * from './services/gitService.js';
export * from './services/shellExecutionService.js';
export * from './services/chatRecordingService.js';
export * from './services/chatCompressionService.js';

// MCP 和 OAuth
export * from './mcp/oauth-provider.js';
export * from './mcp/oauth-token-storage.js';

// 遥测
export * from './telemetry/index.js';
export * from './utils/session.js';

// IDE
export * from './ide/ide-client.js';
export * from './ide/ideContext.js';
```

**总导出数**：180+ 接口、类、函数、常量

---

## 二、核心聊天引擎（/core 目录）

### 2.1 GeminiClient 深度剖析

**文件**：`packages/core/src/core/client.ts` (~2000+ 行)

**核心职责**：
- 管理聊天生命周期（创建、初始化、发送消息）
- 处理多轮对话流程
- 实现循环检测和防止无限对话
- 管理上下文窗口和自动压缩
- 动态模型选择和降级

**关键属性**：
```typescript
export class GeminiClient {
  private chat?: GeminiChat;                    // 当前聊天会话
  private readonly config: Config;               // 配置对象
  private readonly loopDetector: LoopDetectionService;
  private readonly compressionService: ChatCompressionService;
  private currentSequenceModel: string | null = null; // 模型粘性

  // 初始化和聊天
  async initialize(): Promise<void>
  async startChat(extraHistory?: Content[]): Promise<GeminiChat>
  async *sendMessageStream(
    request: PartListUnion,
    signal: AbortSignal,
    prompt_id: string,
    turns?: number,
    isInvalidStreamRetry?: boolean
  ): AsyncGenerator<ServerGeminiStreamEvent, Turn>
}
```

**关键方法详解**：

1. **initialize()** - 初始化阶段
   ```
   1. 验证认证状态
   2. 初始化底层 LLM 客户端
   3. 设置 API 参数（温度、top_k 等）
   4. 验证模型可用性
   ```

2. **startChat()** - 创建新聊天会话
   ```
   1. 创建 GeminiChat 实例
   2. 设置工具声明（来自 ToolRegistry）
   3. 加载额外历史（如提供）
   4. 初始化 Token 计数
   ```

3. **sendMessageStream()** - 核心消息发送
   ```
   1. 准备请求（验证、添加 IDE 上下文）
   2. 调用 GeminiChat.sendMessageStream()
   3. 监听流事件，yield 给调用者
   4. 处理特殊情况：
      ├─ 循环检测 → LoopDetectionService
      ├─ 窗口溢出 → ChatCompressionService
      ├─ 无效响应 → 重试
      └─ 工具请求 → 返回给调用者
   5. 返回完成的 Turn 对象
   ```

**IDE 上下文管理**：
```typescript
private ideContext?: IDEContextData;

// 追踪打开的文件
updateIDEContext(ideContextDelta: IDEContextDelta): void

// 决定是否发送 IDE 上下文
shouldSendIDEContext(): boolean

// 增量更新（不重复发送）
getIDEContextIncrement(): IDEContextIncrement | null
```

**模型粘性机制**：
- 会话内选择一个模型并坚持使用
- 避免中途切换导致的不一致
- 支持手动模型切换

### 2.2 GeminiChat 实现细节

**文件**：`packages/core/src/core/geminiChat.ts` (~1500+ 行)

**核心职责**：
- 与 Gemini API 的直接通信
- 维护聊天会话的消息历史
- 内容验证和重试逻辑
- 记录工具调用和思维过程

**关键属性**：
```typescript
export class GeminiChat {
  private readonly config: Config;
  private readonly generationConfig: GenerateContentConfig;
  private history: Content[] = [];              // 消息历史
  private chatRecordingService: ChatRecordingService;
  private lastPromptTokenCount: number = 0;

  async sendMessageStream(
    model: string,
    params: SendMessageParameters,
    prompt_id: string
  ): Promise<AsyncGenerator<StreamEvent>>

  getHistory(curated: boolean = false): Content[]
  setHistory(history: Content[]): void
  setTools(tools: Tool[]): void
}
```

**历史管理**：
- **全面历史** - 包括所有轮次，包括失败的响应
- **策展历史** - 仅包括有效轮次，用于展示

```typescript
// 获取历史
const fullHistory = geminiChat.getHistory(curated: false);     // 所有
const displayHistory = geminiChat.getHistory(curated: true);   // 清理后

// 修改历史（用于重试或编辑）
geminiChat.setHistory(newHistory);

// 去除思维（privacy mode）
geminiChat.stripThoughtsFromHistory();
```

**内容验证和重试**：
```typescript
private function isValidContent(content: Content): boolean {
  // 检查是否有有效的 parts
  if (!content.parts || content.parts.length === 0) return false;

  // 检查是否有有效的文本（不只是思维）
  const hasValidText = content.parts.some(part =>
    part.text && !part.text.includes('[thought]')
  );

  return hasValidText;
}

// 重试逻辑（最多 2 次）
for (let attempt = 0; attempt < maxAttempts; attempt++) {
  try {
    const response = await api.generateContent(...);
    if (isValidContent(response)) {
      break;  // 成功
    } else {
      // 无效，增加温度重试
      generationConfig.temperature = (attempt + 1) * 0.5;
    }
  } catch (error) {
    // 异常，也增加温度重试
  }
}
```

**工具记录集成**：
```typescript
// 记录工具调用
private recordToolCalls(model: string, toolCalls: ToolCallRecord[]): void {
  this.chatRecordingService.recordToolCalls(model, toolCalls);
}

// 记录思维过程
private recordThought(thought: string): void {
  this.chatRecordingService.recordThought({
    subject: 'thinking',
    description: thought
  });
}
```

### 2.3 Turn 和事件系统

**文件**：`packages/core/src/core/turn.ts` (~1000+ 行)

**Turn 的含义**：
- 一次完整的请求-响应周期
- 可能包含多个工具调用和响应
- 是对话流程中的基本单位

**Turn 结构**：
```typescript
export interface Turn {
  userContent: Content;              // 用户请求
  userContentTokenCount: number;

  assistantContent: Content;         // AI 响应
  assistantTokenCount: number;

  toolCalls: ToolCallRecord[];       // 执行的工具
  toolResponses: ToolResponseRecord[];

  totalTokenCount: number;
  timestamp: number;

  // 元数据
  model: string;
  thought?: string;
  isRetry: boolean;
}
```

**事件系统**：
```typescript
export enum GeminiEventType {
  // 响应内容
  Content = 'content',
  Thought = 'thought',

  // 工具相关
  ToolCallRequest = 'tool_call_request',
  ToolCallResponse = 'tool_call_response',

  // 状态
  Error = 'error',
  ChatCompressed = 'chat_compressed',
  ContextWindowWillOverflow = 'context_window_will_overflow',
  LoopDetected = 'loop_detected',
  MaxSessionTurns = 'max_session_turns',
  Finished = 'finished'
}

// 事件对象
export type ServerGeminiStreamEvent =
  | { type: GeminiEventType.Content; value: ContentPart }
  | { type: GeminiEventType.ToolCallRequest; value: ToolCallRequestInfo }
  | { type: GeminiEventType.Error; value: ErrorInfo }
  // ... 更多事件类型
```

**事件流示例**：
```
Content("正在分析...")
  ↓
ToolCallRequest(name: "read-file", args: {file: "app.ts"})
  ↓
ToolCallResponse(result: "...")
  ↓
Content("我发现了...")
  ↓
Finished(Turn {...})
```

### 2.4 模型路由和动态选择

**模型管理类**：`ModelRouterService`

**支持的模型**：
- `gemini-2.5-pro` - 高级模型（推荐）
- `gemini-2.5-flash` - 快速模型
- `gemini-1.5-pro` - 前代高级模型
- `gemini-1.5-flash` - 前代快速模型

**模型选择策略**：
```typescript
// 1. 用户显式选择
if (argv.model) {
  return argv.model;
}

// 2. 会话粘性（不切换模型）
if (this.currentSequenceModel) {
  return this.currentSequenceModel;
}

// 3. 配置默认值
return config.getModel();  // 默认 gemini-2.5-pro
```

**自动降级（配额处理）**：
```typescript
// 当遇到 quota_exceeded 错误时
if (error.code === 'RESOURCE_EXHAUSTED') {
  // 降级到 Flash 模型
  const flashModel = getFlashModel(currentModel);

  // 重试
  return sendMessageStream(
    userQuery,
    signal,
    promptId,
    turns,
    isInvalidStreamRetry: true
  );
}
```

**Token 计数和窗口管理**：
```typescript
private contextWindowSize = 1_000_000;  // Gemini 2.5 的 1M token 窗口

// 计算使用率
const usagePercent = totalTokenCount / contextWindowSize;

// 触发压缩（> 90%）
if (usagePercent > 0.9) {
  await this.compressionService.compress(
    this.chat,
    prompt_id,
    force: false
  );
}

// 警告（> 95%）
if (usagePercent > 0.95) {
  yield {
    type: GeminiEventType.ContextWindowWillOverflow,
    value: { usagePercent }
  };
}
```

---

## 三、工具系统架构（/tools 目录 - 130+ 文件）

### 3.1 工具系统概览

**关键文件**：
- `tool-registry.ts` - 工具注册表（主类）
- `tools.ts` - 所有工具的统一导出
- `core-tools/` - 48+ 核心工具实现
- `mcp-tool.ts` - MCP 工具包装器
- `tool-scheduler/` - 工具调度逻辑（内部）

**工具分类**：
- **读取工具** (5) - read-file, ls, glob, grep, ripGrep
- **编辑工具** (3) - write-file, edit, smartEdit
- **执行工具** (1) - shell
- **Web 工具** (2) - web-fetch, web-search
- **其他工具** (35+) - memory, todos, mcp-tool 等

### 3.2 工具定义和基类

**基类层级**：
```
BaseDeclarativeTool (抽象)
  ├─ ReadFileTool
  ├─ EditTool
  ├─ ShellTool
  └─ ... (所有工具)

ToolInvocation (验证 + 准备)
  ├─ ReadFileInvocation
  ├─ ShellInvocation
  └─ ... (对应工具调用)

ToolResult (执行结果)
  ├─ { success: true, output: string }
  └─ { success: false, error: string }
```

### 3.3 核心工具实现

**关键工具示例**：

1. **shell.ts** (~500+ 行)
   - 支持 PTY（伪终端）和子进程
   - 实时流式输出
   - 超时和信号处理

2. **edit.ts** (~400+ 行)
   - Diff 格式编辑
   - 智能行匹配
   - 前后检查

3. **ripGrep.ts** (~300+ 行)
   - 高性能正则表达式搜索
   - 支持分页
   - 颜色输出

### 3.4 MCP 工具集成

**McpClientManager**：
- 发现和连接 MCP 服务器
- 动态工具发现
- 工具执行包装

**工具包装**：
```typescript
class DiscoveredMCPTool extends BaseDeclarativeTool {
  // 包装 MCP 工具为标准工具接口
  build(params: any): ToolInvocation {
    return {
      execute: async () => {
        // 调用 MCP 服务器
      }
    };
  }
}
```

### 3.5 工具调度和执行流程

**CoreToolScheduler**：
- 管理多个工具的并发执行
- 处理工具确认和权限
- 状态管理（Validating → Scheduled → Executing → Success）

---

## 四、配置系统（/config 目录 - 250+ 文件）

### 4.1 Config 类的完整结构

**Config 类** (`config.ts` - 39K 行)：
- 超大配置类（150+ 属性 + 100+ 方法）
- 懒加载各种服务
- 单一职责但集中管理

**关键属性分组**：
- **模型和 API** - getModel(), getGeminiClient()
- **工具和注册表** - getToolRegistry(), getAllTools()
- **文件系统** - getFileService(), getGitService()
- **执行配置** - getApprovalMode(), getShellExecutionConfig()
- **MCP** - getMcpServers(), setMcpServers()
- **特性开关** - getDebugMode(), getScreenReader()

### 4.2 初始化流程

**完整初始化序列**：
```
new Config(options)
  ↓
config.initialize()
  ├─ getFileService()         // FileDiscoveryService
  ├─ getGitService()          // GitService（可选）
  ├─ createPromptRegistry()   // PromptRegistry
  ├─ createAgentRegistry()    // AgentRegistry
  ├─ createToolRegistry()     // ToolRegistry + discoverAllTools()
  └─ geminiClient.initialize() // GeminiClient
```

### 4.3 模型和 API 配置

**模型配置** (`models.ts`)：
- 支持多个 Gemini 版本
- 温度、top_k、top_p 等参数
- 上下文窗口大小

**API 配置** (`auth.ts`)：
- OAuth token 管理
- API 密钥存储
- 认证刷新逻辑

### 4.4 特性开关和运行时选项

**150+ 配置选项**：
- `debugMode` - 调试输出
- `approvalMode` - 工具批准模式
- `useRipgrep` - 搜索引擎选择
- `enableCheckpointing` - Git 检查点
- `outputFormat` - 输出格式
- 等等...

---

## 五、业务服务层（/services 目录）

### 5.1 FileDiscoveryService

**功能**：
- .gitignore 和 .geminiignore 过滤
- 文件发现和遍历
- 模糊匹配禁用

### 5.2 GitService 和检查点系统

**功能**：
- 创建隐藏 Git 仓库
- 快照和恢复
- 变更跟踪

### 5.3 ShellExecutionService

**功能**：
- PTY 和子进程模式
- 实时输出流
- 超时和信号处理

### 5.4 ChatRecordingService

**功能**：
- 对话历史记录
- 工具调用日志
- 思维过程记录

### 5.5 ChatCompressionService

**功能**：
- 对话摘要和压缩
- Token 计数验证
- 失败检测

### 5.6 LoopDetectionService

**功能**：
- 重复工具调用检测
- 循环模式识别
- 提前中止

---

## 六、MCP 和 OAuth（/mcp 目录）

### 6.1 MCP 客户端管理

**McpClientManager**：
- 多种传输方式（stdio, SSE, HTTP, WebSocket）
- 动态工具发现
- 连接生命周期管理

### 6.2 OAuth 提供者

**MCPOAuthProvider**：
- PKCE 授权码流程
- 令牌刷新管理
- 令牌存储和加密

### 6.3 工具发现和包装

**DiscoveredMCPTool**：
- 将 MCP 工具转换为标准接口
- 参数验证
- 错误处理

---

## 七、遥测系统（/telemetry 目录）

### 7.1 遥测初始化和配置

**TelemetrySettings**：
```typescript
{
  enabled?: boolean;
  target?: 'GCP' | 'LOCAL';
  otlpEndpoint?: string;
  logPrompts?: boolean;
  outfile?: string;
}
```

### 7.2 事件和指标记录

**事件类型**：
- CliConfigurationEvent
- UserPromptEvent
- ToolCallEvent
- ApiRequestEvent
- ApiErrorEvent
- ChatCompressionEvent

**指标**：
- 工具执行时间
- Token 使用情况
- API 响应延迟
- 内存和 CPU 使用

### 7.3 活动监测

**ActivityType**：
- USER_TYPING
- TOOL_EXECUTION
- API_REQUEST

### 7.4 性能指标

**Metrics**：
- 启动时间
- 内存快照
- CPU 使用
- 性能评分

---

## 八、支撑系统

### 8.1 消息总线（/confirmation-bus）

**MessageBus**：
- 解耦工具执行和 UI
- 发布-订阅模式
- 政策决策流程

### 8.2 策略引擎（/policy）

**PolicyEngine**：
- 自动批准/拒绝规则
- 权限管理
- 工具级别的策略

### 8.3 IDE 集成（/ide）

**IdeClient**：
- 编辑器状态接收
- 上下文增量管理
- VS Code 集成

### 8.4 提示词管理（/prompts）

**PromptRegistry**：
- 系统提示词
- 可自定义提示词
- 提示词模板

### 8.5 智能体系统（/agents）

**AgentRegistry**：
- 预定义的智能体
- 自定义智能体支持

### 8.6 工具函数（/utils）

**主要工具**：
- `logger.ts` - 调试日志
- `errors.ts` - 错误处理
- `validators.ts` - 数据验证
- `session.ts` - 会话管理

---

## 九、关键数据结构和类型

### 9.1 事件类型系统

**GeminiEventType enum**：
- Content, Thought
- ToolCallRequest, ToolCallResponse
- Error, ChatCompressed
- LoopDetected, ContextWindowWillOverflow
- (12+ 总计)

### 9.2 工具调用类型

**ToolCallState**：
- Validating (参数验证)
- AwaitingApproval (等待确认)
- Scheduled (准备执行)
- Executing (执行中)
- Success / Error / Cancelled (终端状态)

### 9.3 消息和内容结构

**Content**：
```typescript
interface Content {
  role: 'user' | 'model';
  parts: (TextPart | ToolUseBlock | ToolResultBlock)[];
}
```

### 9.4 错误类型

**主要错误**：
- `ToolExecutionError` - 工具执行失败
- `InvalidCredentialsError` - 认证错误
- `ContextWindowExceededError` - 上下文溢出
- `LoopDetectedError` - 循环检测
- `CancellationError` - 用户取消

---

## 十、完整初始化和请求流程

### 10.1 启动阶段

```
创建 Config
  ↓
await config.initialize()
  ├─ 文件发现服务
  ├─ Git 服务
  ├─ 工具注册表
  └─ GeminiClient 初始化
```

### 10.2 聊天流程

```
GeminiClient.sendMessageStream(query)
  ├─ 调用 GeminiChat.sendMessageStream()
  ├─ 接收流事件
  ├─ 处理工具请求
  └─ 返回 Turn 对象
```

### 10.3 工具执行流程

```
工具请求
  ├─ 参数验证
  ├─ 权限检查
  ├─ 执行工具
  └─ 返回结果
```

### 10.4 错误处理和恢复

```
错误发生
  ├─ 验证错误 → 返回
  ├─ 执行错误 → 重试（可选）
  ├─ API 错误 → 降级
  └─ 取消 → 中止
```

---

## 十一、代码组织最佳实践

### 11.1 模块划分原则

- **单一职责** - 每个模块一个主要类
- **清晰接口** - 导出的公共 API 明确
- **内部隐藏** - 细节实现隐藏在模块内
- **依赖向下** - 上层不依赖下层

### 11.2 接口设计

- 使用 TypeScript 接口定义契约
- 异步操作使用 Promise 和 AsyncGenerator
- 事件系统使用类型化的事件对象

### 11.3 异步处理模式

- **AsyncGenerator** - 流式处理
- **Promise** - 一次性异步操作
- **回调** - 事件监听

### 11.4 错误处理策略

- 类型化的错误
- 用户友好的错误消息
- 自动恢复机制

---

## 十二、性能优化和扩展性

### 12.1 流式处理和内存管理

- 使用 AsyncGenerator 避免缓冲
- 工具输出增量发送
- 自动压缩管理 Token

### 12.2 缓存策略

- 工具声明缓存
- 文件过滤缓存
- IDE 上下文增量缓存

### 12.3 可扩展性考虑

- MCP 支持无限扩展
- 自定义命令和工具
- 提示词和智能体系统
- 政策引擎实现

---

## 总结

Core 模块是一个**精心设计的分层架构**：

- **聊天引擎层** - GeminiClient + GeminiChat + Turn
- **工具层** - ToolRegistry + 48+ 工具 + MCP 支持
- **配置层** - Config 单一入口
- **服务层** - 6+ 核心服务
- **支撑层** - 消息总线、策略、遥测等

**核心特点**：
- ✅ 高度解耦的模块化设计
- ✅ 异步生成器流式处理
- ✅ 灵活的扩展机制（MCP）
- ✅ 完整的错误处理和恢复
- ✅ 性能优化（压缩、缓存、流式）
- ✅ 隐私和安全考虑（政策、沙箱）

---
