# Core 模块能力详解

Core 模块是 Gemini CLI 的心脏，提供了完整的 AI 对话引擎、工具系统、配置管理和业务服务。

## 一、核心能力分类

### 1. 聊天引擎能力（GeminiClient + GeminiChat）

#### GeminiClient - 对话管理和流程控制

**职责**：
- 创建和管理聊天会话
- 路由消息到 Gemini API
- 管理对话历史和上下文
- 处理多轮对话流程
- 检测和防止无限循环
- 动态模型选择和降级
- 上下文窗口管理和压缩

**主要导出**：
```typescript
export class GeminiClient {
  // 初始化聊天
  async initialize(): Promise<void>

  // 创建新聊天会话
  async startChat(extraHistory?: Content[]): Promise<GeminiChat>

  // 发送消息并流式获取响应
  async *sendMessageStream(
    request: PartListUnion,
    signal: AbortSignal,
    prompt_id: string,
    turns?: number,
    isInvalidStreamRetry?: boolean
  ): AsyncGenerator<ServerGeminiStreamEvent, Turn>
}
```

**核心能力**：
1. **流式消息处理**
   - 异步生成器模式，实时流式返回事件
   - 支持取消操作（AbortSignal）
   - 处理重试和异常恢复

2. **智能循环检测**
   - 检测模型在重复执行相同工具
   - 防止无限对话循环
   - 可配置的循环检测阈值

3. **上下文窗口管理**
   - 追踪 Token 使用情况
   - 窗口溢出预警
   - 自动触发对话压缩

4. **IDE 上下文集成**
   - 接收编辑器状态（打开文件、光标位置、选中文本）
   - 增量式 IDE 上下文发送
   - 减少重复数据传输

5. **模型路由**
   - 支持多个 Gemini 模型版本
   - 模型粘性（会话内模型一致）
   - 自动降级到 Flash（处理配额限制）

#### GeminiChat - 单个会话管理

**职责**：
- 与 Gemini API 通信（具体的 HTTP 请求）
- 维护会话的消息历史
- 处理内容验证和重试
- 记录工具调用和思维过程

**主要导出**：
```typescript
export class GeminiChat {
  // 发送消息流
  async sendMessageStream(
    model: string,
    params: SendMessageParameters,
    prompt_id: string
  ): Promise<AsyncGenerator<StreamEvent>>

  // 历史管理
  getHistory(curated: boolean = false): Content[]
  setHistory(history: Content[]): void
  stripThoughtsFromHistory(): void
  setTools(tools: Tool[]): void
}
```

**核心能力**：
1. **内容验证**
   - 验证 API 响应内容有效性
   - 防止不完整响应被添加到历史
   - 自动重试无效响应（最多 2 次）

2. **历史管理**
   - 维护"全面历史"（包括失败）和"策展历史"（仅成功）
   - 支持历史修改
   - 思维过程剥离（可选）

3. **工具记录**
   - 记录每个工具调用和结果
   - 与 ChatRecordingService 集成
   - 支持分析和重放

4. **Token 计数**
   - 实时追踪使用的 Token
   - 为压缩和窗口管理提供数据
   - 计费和分析

---

### 2. 工具系统能力

#### 工具注册表 (ToolRegistry)

**职责**：
- 注册和管理所有可用的工具
- 包括核心工具和 MCP 工具
- 提供工具发现和查询
- 生成工具声明给 API

**主要导出**：
```typescript
export class ToolRegistry {
  // 注册工具
  registerTool(tool: AnyDeclarativeTool): void

  // 发现所有可用工具（包括 MCP）
  async discoverAllTools(): Promise<void>

  // 获取工具声明（发送给 API）
  getFunctionDeclarations(): FunctionDeclaration[]

  // 查询工具
  getAllTools(): AnyDeclarativeTool[]
  getTool(name: string): AnyDeclarativeTool | undefined
}
```

#### 内置工具列表（48+ 工具）

**读取类工具**：
- `ls` - 列出目录内容
- `read-file` - 读取单个或多个文件
- `read-many-files` - 批量读取文件
- `glob` - 文件模式匹配（*.ts, src/**/*.tsx 等）
- `ripGrep` / `grep` - 全文搜索

**编辑类工具**：
- `edit` - 基于 Diff 的文件编辑
- `smartEdit` - 智能编辑（更好的上下文感知）
- `write-file` - 创建/覆写文件

**执行类工具**：
- `shell` - 执行 Shell 命令（支持 PTY 和 非 PTY）

**Web 和搜索**：
- `web-fetch` - HTTP 请求
- `web-search` - Google 搜索（支持分页）

**其他工具**：
- `memory` - 用户记忆存储和检索
- `write-todos` - 待做事项管理
- `mcp-tool` - MCP 工具的包装

#### 工具执行流程

**状态机**：
```
Validating (参数验证)
    ↓
Scheduled (准备执行)
    ├─→ Executing (执行中)
    │     ↓
    │   Success/Error/Cancelled
    │
└─→ AwaitingApproval (等待用户确认)
        ↓
    Proceed/ProceedAlways/Cancel
        ↓
    [Scheduled or Cancelled]
```

**工具执行生命周期**：
```typescript
// 1. 构建工具调用
const toolInvocation = tool.build(params);

// 2. 验证参数
const validationError = toolInvocation.validateParams();

// 3. 判断是否需要确认
const confirmationDetails = await toolInvocation.shouldConfirmExecute(signal);

// 4. 等待用户确认（如需要）
const userDecision = await waitForUserConfirmation(confirmationDetails);

// 5. 执行工具
const result = await toolInvocation.execute(signal, updateCallback);

// 6. 返回结果给模型
return result;
```

**工具确认参数**：
```typescript
interface ToolCallConfirmationDetails {
  type: 'edit' | 'exec' | 'mcp' | 'info';
  title: string;
  description?: string;
  details?: string[];  // 具体操作详情
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}

enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',         // 执行一次
  ProceedAlways = 'proceed_always',     // 此会话自动批准
  ModifyWithEditor = 'modify_with_editor', // 编辑后执行
  Cancel = 'cancel'                      // 取消
}
```

---

### 3. 配置系统能力 (Config)

**职责**：
- 统一的配置管理和初始化
- 提供对所有核心服务的访问
- 管理运行时状态
- 控制特性开关和限制

**主要导出**：
```typescript
export class Config {
  // 初始化
  async initialize(): Promise<void>

  // 模型和 API
  getModel(): string
  setModel(newModel: string): void
  getContentGenerator(): ContentGenerator
  getBaseLlmClient(): BaseLlmClient

  // 工具和服务
  getToolRegistry(): ToolRegistry
  getPromptRegistry(): PromptRegistry
  getAgentRegistry(): AgentRegistry

  // 文件系统
  getFileService(): FileDiscoveryService
  async getGitService(): Promise<GitService>
  getWorkspaceContext(): WorkspaceContext

  // 执行配置
  getApprovalMode(): ApprovalMode
  getAllowedTools(): string[] | undefined
  getExcludeTools(): string[] | undefined

  // MCP
  getMcpServers(): Record<string, MCPServerConfig>
  setMcpServers(servers: Record<string, MCPServerConfig>): void

  // 特性开关
  getDebugMode(): boolean
  getIdeMode(): boolean
  getScreenReader(): boolean
  getUseSmartEdit(): boolean
  getUseRipgrep(): boolean
}
```

**核心能力**：
1. **模型管理**
   - 选择 Gemini 模型版本
   - 动态模型切换
   - 配额和配置验证

2. **工具管理**
   - 允许列表/阻止列表
   - 工具发现和注册
   - 工具权限控制

3. **认证配置**
   - 多种认证方法支持
   - OAuth 集成
   - API 密钥管理

4. **工作空间配置**
   - 目标目录设置
   - 文件包含/排除
   - .gitignore 和 .geminiignore 支持

5. **运行时控制**
   - 批准模式（DEFAULT/AUTO_EDIT/YOLO）
   - 输出格式（TEXT/JSON/STREAM_JSON）
   - 交互 vs 非交互模式

---

### 4. 服务层能力

#### FileDiscoveryService - 文件发现和过滤

```typescript
export class FileDiscoveryService {
  // 过滤文件列表
  filterFiles(
    filePaths: string[],
    options: FilterFilesOptions
  ): string[]

  // 检查是否应忽略文件
  shouldIgnoreFile(
    filePath: string,
    options: FilterFilesOptions
  ): boolean

  // 获取过滤报告
  filterFilesWithReport(
    filePaths: string[],
    opts: FilterFilesOptions
  ): FilterReport
}
```

**核心能力**：
- 基于 `.gitignore` 过滤文件
- 基于 `.geminiignore` 的额外过滤
- 递归目录遍历
- 性能优化（缓存、并行处理）

#### GitService - Git 操作

```typescript
export class GitService {
  async initialize(): Promise<void>
  async createFileSnapshot(message: string): Promise<string>
  async restoreProjectFromSnapshot(commitHash: string): Promise<void>
  async getCurrentCommitHash(): Promise<string>
}
```

**核心能力**：
- **检查点（Checkpointing）** - 保存项目快照
- **变更跟踪** - 隐藏 Git 仓库记录修改
- **快照恢复** - 从快照恢复项目状态
- **与 .gitignore 集成** - 遵守 Git 忽略规则

#### ShellExecutionService - Shell 命令执行

```typescript
export class ShellExecutionService {
  static async execute(
    commandToExecute: string,
    cwd: string,
    onOutputEvent: (event: ShellOutputEvent) => void,
    abortSignal: AbortSignal,
    shouldUseNodePty: boolean,
    shellExecutionConfig: ShellExecutionConfig
  ): Promise<ShellExecutionHandle>

  static writeToPty(pid: number, input: string): void
  static resizePty(pid: number, cols: number, rows: number): void
}
```

**核心能力**：
1. **两种执行模式**
   - **PTY 模式** - 伪终端，支持交互、颜色输出
   - **子进程模式** - 简单模式，适合无交互命令

2. **输出处理**
   - 实时流式输出事件
   - ANSI 颜色代码支持
   - 二进制数据检测
   - 输出缓冲和截断（16MB 限制）

3. **进程管理**
   - 超时处理
   - 信号传递（SIGINT, SIGTERM）
   - 进程树杀死（子进程清理）

4. **故障处理**
   - 自动降级（PTY → 子进程）
   - 超时重试
   - 异常恢复

#### ChatRecordingService - 对话记录

```typescript
export class ChatRecordingService {
  recordMessage(message: {
    type: 'user' | 'gemini';
    content: string;
    model: string;
  }): void

  recordToolCalls(model: string, toolCalls: ToolCallRecord[]): void
  recordThought(thought: { subject: string; description: string }): void
}
```

**核心能力**：
- 完整的聊天历史记录
- 工具调用记录
- 思维过程记录
- 导出和分析支持

#### ChatCompressionService - 对话压缩

```typescript
export class ChatCompressionService {
  async compress(
    chat: GeminiChat,
    prompt_id: string,
    force: boolean,
    model: string,
    config: Config,
    hasFailedCompressionAttempt: boolean
  ): Promise<{ newHistory: Content[] | null; info: ChatCompressionInfo }>
}
```

**核心能力**：
- **摘要长对话** - 将早期轮次压缩为摘要
- **Token 计数验证** - 确保压缩有效
- **失败检测** - 不压缩包含失败的对话
- **恢复机制** - 压缩失败时回滚

#### LoopDetectionService - 循环检测

**核心能力**：
- 检测重复的工具调用序列
- 识别无限循环模式
- 提前中止对话
- 可配置的阈值

---

### 5. MCP 支持能力

#### MCP 服务器配置和发现

```typescript
export class MCPServerConfig {
  // 传输方式（选其一）
  readonly command?: string;      // 命令行启动
  readonly url?: string;          // SSE 传输
  readonly httpUrl?: string;      // HTTP 传输
  readonly tcp?: string;          // WebSocket 传输

  // 工具过滤
  readonly includeTools?: string[];
  readonly excludeTools?: string[];

  // OAuth 配置
  readonly oauth?: MCPOAuthConfig;
  readonly authProviderType?: AuthProviderType;

  // 通用配置
  readonly timeout?: number;
  readonly trust?: boolean;
  readonly description?: string;
}
```

**核心能力**：
1. **多种传输方式**
   - 标准输入/输出（stdio）
   - SSE（Server-Sent Events）
   - HTTP/REST
   - WebSocket/TCP

2. **工具集成**
   - 动态工具发现
   - 工具级别的过滤（包含/排除）
   - 工具故障隔离

3. **OAuth 和认证**
   - 自动 OAuth 流程
   - 令牌刷新管理
   - 令牌存储和加密

4. **错误处理**
   - 服务器连接失败降级
   - 单工具故障不影响其他工具
   - 自动重连

---

### 6. 遥测系统能力

**核心能力**：

1. **事件日志**
   - CLI 配置事件
   - 用户提示事件
   - 工具调用事件
   - API 请求/响应事件
   - 错误事件
   - 对话压缩事件

2. **指标记录**
   - 工具执行时间和成功率
   - Token 使用情况
   - API 响应延迟
   - 内存和 CPU 使用
   - 启动性能

3. **活动监测**
   - 用户活动追踪
   - 工具执行活动
   - API 请求活动

4. **上报方式**
   - OpenTelemetry（gRPC 或 HTTP）
   - 本地日志文件
   - Google Cloud Logging

5. **隐私控制**
   - 提示词内容日志开关（默认关闭）
   - 遥测完全禁用开关
   - 数据最小化

---

## 二、核心初始化流程

```typescript
// 1. 创建 Config 实例
const config = new Config({
  sessionId: generateId(),
  targetDir: process.cwd(),
  model: 'gemini-2.5-pro',
  approvalMode: ApprovalMode.DEFAULT,
  // ... 150+ 其他配置选项
});

// 2. 初始化 Config
await config.initialize();
  // 2.1 初始化文件发现服务
  // 2.2 创建 Git 服务（如启用检查点）
  // 2.3 创建提示词注册表
  // 2.4 创建智能体注册表
  // 2.5 创建工具注册表并发现工具
  // 2.6 初始化 GeminiClient

// 3. 初始化 GeminiClient
await config.getGeminiClient().initialize();
  // 3.1 创建初始聊天会话
  // 3.2 设置工具声明
  // 3.3 初始化 Token 计数

// 4. 获取工具注册表进行后续使用
const toolRegistry = config.getToolRegistry();

// 5. 使用 GeminiClient 进行聊天
const stream = config.getGeminiClient().sendMessageStream(
  userQuery,    // 字符串或 Part[] 数组
  abortSignal,  // 支持取消
  promptId      // 追踪 ID
);

for await (const event of stream) {
  // 处理流事件
  switch (event.type) {
    case GeminiEventType.Content:
      // 处理文本内容
      break;
    case GeminiEventType.ToolCallRequest:
      // 调度工具执行
      scheduleToolExecution(event.value);
      break;
    case GeminiEventType.Error:
      // 处理错误
      break;
  }
}
```

---

## 三、主要导出和使用示例

### 创建和初始化

```typescript
import { Config, GeminiClient } from '@google/gemini-cli-core';

// 创建配置
const config = new Config({
  sessionId: 'session-123',
  targetDir: '/path/to/project',
  model: 'gemini-2.5-pro',
});

// 初始化
await config.initialize();

// 获取客户端
const client = config.getGeminiClient();
```

### 发送消息

```typescript
// 简单字符串提示
const stream1 = client.sendMessageStream(
  'How many lines of TypeScript code are in this project?',
  abortSignal,
  promptId
);

// Part 数组（多模态）
const stream2 = client.sendMessageStream(
  [
    { text: 'Describe this image:' },
    { inlineData: { mimeType: 'image/png', data: imageBuffer } }
  ],
  abortSignal,
  promptId
);
```

### 访问服务

```typescript
// 文件服务
const fileService = config.getFileService();
const filtered = fileService.filterFiles(allFiles, options);

// Git 服务
const gitService = await config.getGitService();
const commit = await gitService.createFileSnapshot('my changes');

// 工具注册表
const toolRegistry = config.getToolRegistry();
const tools = toolRegistry.getFunctionDeclarations();
const tool = toolRegistry.getTool('edit');

// 工具执行
const toolInvocation = tool.build(params);
const result = await toolInvocation.execute(abortSignal);
```

### 处理事件

```typescript
import {
  GeminiEventType,
  ServerGeminiStreamEvent
} from '@google/gemini-cli-core';

for await (const event of stream) {
  const e = event as ServerGeminiStreamEvent;

  if (e.type === GeminiEventType.Content) {
    console.log(e.value.text);
  } else if (e.type === GeminiEventType.ToolCallRequest) {
    const { toolName, toolArgs } = e.value;
    console.log(`Tool requested: ${toolName}`, toolArgs);
  } else if (e.type === GeminiEventType.Error) {
    console.error(e.value.message);
  }
}
```

---

## 四、关键特性

### 1. 异步生成器流式处理

Core 使用 TypeScript 的异步生成器（AsyncGenerator）实现流式处理：

```typescript
async *sendMessageStream(...): AsyncGenerator<ServerGeminiStreamEvent, Turn> {
  // 每个事件立即 yield，不缓冲
  // 最后返回完整的 Turn 对象
}

// 使用方：
for await (const event of stream) {
  // 实时处理每个事件
}
const finalTurn = (await stream.return()).value;
```

**优点**：
- 实时 UI 更新
- 内存高效
- 可取消操作

### 2. 消息总线模式

Core 使用消息总线实现策略决策：

```typescript
// 发布工具确认请求
messageBus.publish({
  type: MessageBusType.TOOL_CONFIRMATION_REQUEST,
  toolCall: { name: 'shell', args: { command: 'rm -rf /' } },
  correlationId: '123'
});

// 政策引擎可以自动回应
messageBus.subscribe(
  MessageBusType.TOOL_CONFIRMATION_RESPONSE,
  (response) => {
    // 自动批准/拒绝
  }
);

// UI 也可以等待用户响应
await waitForResponse(correlationId);
```

### 3. 分层工具系统

```typescript
// 工具定义（abstract）
class EditTool extends BaseDeclarativeTool<EditParams, EditResult> {
  build(params: EditParams): ToolInvocation {
    return new EditToolInvocation(params);
  }
}

// 工具调用（实例）
class EditToolInvocation implements ToolInvocation<EditParams, EditResult> {
  async shouldConfirmExecute(signal: AbortSignal) {
    // 决定是否需要确认
    return { type: 'edit', title: '编辑文件' };
  }

  async execute(signal: AbortSignal, updateOutput: Callback) {
    // 执行工具
    updateOutput('编辑中...');
    // ...
    return result;
  }
}
```

### 4. 内容验证和重试

```typescript
// GeminiChat 内部自动重试
for (let attempt = 0; attempt < maxAttempts; attempt++) {
  try {
    const response = await generateContent(model, request);
    if (isValidContent(response)) {
      // 有效，添加到历史
      break;
    }
    // 无效，增加温度后重试
  } catch (error) {
    // 异常，增加温度后重试
  }
}
```

### 5. 智能压缩

当对话接近 Token 限制时：
```
1. 检测 Token 使用率 > 90%
2. 调用 ChatCompressionService
3. 生成早期轮次的摘要
4. 验证压缩后的 Token 数
5. 如果失败，保持原始历史
```

---

## 五、核心用例

| 用例 | 所需能力 | 关键类 |
|------|---------|--------|
| 聊天问答 | 消息流处理、历史管理 | GeminiClient, GeminiChat |
| 文件操作 | 工具执行、用户确认 | ToolRegistry, Tool Scheduler |
| 代码分析 | 文件发现、搜索工具 | FileDiscoveryService, RipGrep |
| 自动化脚本 | Shell 执行、循环检测 | ShellExecutionService, LoopDetector |
| 项目变更跟踪 | Git 检查点、快照恢复 | GitService |
| MCP 扩展 | MCP 发现、OAuth、工具包装 | MCP 相关类 |
| 性能分析 | 遥测、指标记录 | Telemetry 类 |

---

## 总结

Core 模块提供了一个**完整、解耦、可扩展**的 AI Agent 基础设施：

- **聊天引擎**：智能的多轮对话管理
- **工具系统**：灵活的工具注册、执行、确认流程
- **服务层**：文件、Git、Shell 等实用服务
- **配置系统**：统一的配置管理接口
- **MCP 支持**：无限扩展能力
- **遥测系统**：完整的性能和使用监控
- **流式处理**：高效的内存使用和实时响应

任何想要集成 Gemini AI 的应用（Web UI、IDE、自定义工具、API 服务等）都可以基于 Core 的这些能力快速构建。
