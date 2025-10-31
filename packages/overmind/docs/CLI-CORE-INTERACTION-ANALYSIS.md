# Gemini CLI - CLI 与 Core 层交互分析

## 概述

本文档分析了 Gemini CLI 中 CLI 层与 Core 层的交互方式，为设计 Web UI 提供架构参考。

## 架构分层

```
┌─────────────────────────────────────────┐
│         UI 层（可替换）                   │
│    CLI (React/Ink) / Web / IDE / API    │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────▼──────────┐
        │   Integration 适配   │
        │  useGeminiStream    │
        │  AppContainer       │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │   Core API 层        │
        │  GeminiClient       │
        │  Config             │
        │  ToolRegistry       │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │   业务逻辑层         │
        │  GeminiChat         │
        │  Turn 管理          │
        │  工具执行           │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │   内容生成层         │
        │ ContentGenerator   │
        │ Gemini API 调用     │
        └─────────────────────┘
```

## 1. CLI 主入口与初始化

**位置**: `packages/cli/src/gemini.tsx`

```typescript
async function main() {
  // 1️⃣ 加载配置
  const settings = loadSettings();
  const config = await loadCliConfig(
    settings.merged,
    sessionId,
    argv
  );

  // 2️⃣ 初始化 Core
  await config.initialize();

  // 3️⃣ 路由两种模式
  if (config.isInteractive()) {
    // 交互模式：React + Ink TUI
    await startInteractiveUI(config, settings, ...);
  } else {
    // 非交互模式：直接处理请求
    await runNonInteractive({ config, settings, input, ... });
  }
}
```

## 2. Core 的核心类

### 2.1 GeminiClient（推荐使用）

**位置**: `packages/core/src/core/client.ts`

这是 Web UI 应该直接使用的主要接口：

```typescript
export class GeminiClient {
  // ✅ 最重要的方法 - 发送消息流
  async *sendMessageStream(
    request: PartListUnion,      // 用户输入
    signal: AbortSignal,         // 取消信号
    prompt_id: string,           // 用于追踪
    turns: number = MAX_TURNS,   // 最大回合数
  ): AsyncGenerator<ServerGeminiStreamEvent, Turn>

  // 初始化客户端
  async initialize(): Promise<void>

  // 获取聊天实例
  getChat(): GeminiChat

  // 历史管理
  setHistory(history: Content[]): void
  getHistory(): Content[]

  // 工具管理
  async setTools(): Promise<void>
}
```

**特性**：
- ✅ 返回异步生成器，支持流式处理
- ✅ 自动处理重试、循环检测、内容验证
- ✅ IDE 上下文自动注入
- ✅ 聊天压缩优化

### 2.2 GeminiChat（底层会话管理）

**位置**: `packages/core/src/core/gemini-chat.ts`

```typescript
export class GeminiChat {
  // 发送消息流（低级 API）
  async sendMessageStream(
    model: string,
    params: SendMessageParameters,
    prompt_id: string,
  ): Promise<AsyncGenerator<StreamEvent>>

  // 历史管理
  getHistory(curated: boolean = false): Content[]
  setHistory(history: Content[]): void
  addHistory(content: Content): void

  // 工具调用记录
  recordCompletedToolCalls(
    model: string,
    toolCalls: CompletedToolCall[]
  ): void
}
```

## 3. 流式事件系统

### 3.1 事件类型定义

**位置**: `packages/core/src/types.ts`

```typescript
export enum ServerGeminiEventType {
  Content = 'content',                   // 文本内容
  ToolCallRequest = 'tool_call_request', // 工具请求
  ToolCallResponse = 'tool_call_response',
  Thought = 'thought',                   // 思考过程
  Error = 'error',
  ChatCompressed = 'chat_compressed',    // 聊天被压缩
  Finished = 'finished',                 // 完成
  LoopDetected = 'loop_detected',        // 循环检测
  Citation = 'citation',                 // 引用来源
  // ... 更多事件
}

export type ServerGeminiStreamEvent =
  | ContentEvent
  | ToolCallRequestEvent
  | ThoughtEvent
  | ErrorEvent
  | CitationEvent
  | FinishedEvent
  // ... 等等
```

### 3.2 事件流处理示例

```typescript
// 这就是 CLI 的处理方式，Web 也应该类似
for await (const event of geminiClient.sendMessageStream(
  userInput,
  abortSignal,
  promptId
)) {
  switch (event.type) {
    case ServerGeminiEventType.Content:
      // 增量更新文本内容
      appendText(event.value);
      break;

    case ServerGeminiEventType.ToolCallRequest:
      // 收集工具调用请求
      const { toolName, input } = event.value;
      // Web UI 应该显示确认对话
      const confirmed = await showConfirmDialog(
        `Execute ${toolName}?`
      );
      if (confirmed) {
        const result = await executeToolCall(
          config,
          event.value,
          abortSignal
        );
        // 记录完成的工具调用
        geminiClient.getChat().recordCompletedToolCalls(
          config.getModel(),
          [result]
        );
      }
      break;

    case ServerGeminiEventType.Thought:
      // 显示思考过程（可选）
      showThought(event.value);
      break;

    case ServerGeminiEventType.Error:
      // 错误处理
      showError(event.value);
      break;

    case ServerGeminiEventType.Citation:
      // 显示引用
      showCitation(event.value);
      break;

    case ServerGeminiEventType.Finished:
      // 对话完成
      markAsFinished();
      break;
  }
}
```

## 4. CLI 的交互集成方案

### 4.1 React Hook - useGeminiStream

**位置**: `packages/cli/src/ui/hooks/useGeminiStream.ts`

这是 CLI 连接 UI 和 Core 的关键 Hook：

```typescript
export const useGeminiStream = (
  geminiClient: GeminiClient,           // Core 客户端
  history: HistoryItem[],
  addItem: UseHistoryManagerReturn['addItem'],
  config: Config,                        // 配置
  settings: LoadedSettings,
  confirmationBus: ConfirmationBus,      // 用户确认
  ...
) => {
  const submitQuery = useCallback(async (
    query: string,
    options?: SubmitQueryOptions,
    prompt_id?: string
  ) => {
    // 1. 准备查询（处理 @命令、/命令）
    const { queryToSend } = await prepareQueryForGemini(
      query,
      config,
      ...
    );

    // 2. 启动 Core 流
    const stream = geminiClient.sendMessageStream(
      queryToSend,
      abortSignal,
      prompt_id || generatePromptId(),
    );

    // 3. 处理事件流
    const processingStatus = await processGeminiStreamEvents(
      stream,
      addItem,
      geminiMessageBuffer,
      confirmedToolCall,
      ...
    );

    // 4. 记录完成的工具调用
    if (completedToolCalls.length > 0) {
      await handleCompletedTools(completedToolCalls);
    }

    return processingStatus;
  }, [dependencies]);

  return {
    streamingState,      // 当前流状态
    submitQuery,         // 提交查询函数
    initError,          // 初始化错误
    pendingHistoryItems, // 待处理的历史项
    thought,            // 当前思考过程
    cancelOngoingRequest, // 取消请求
  };
};
```

### 4.2 事件处理函数

```typescript
const processGeminiStreamEvents = async (
  stream: AsyncIterable<GeminiEvent>,
  addItem: Function,
  messageBuffer: string,
  confirmedToolCall: boolean,
  ...
): Promise<StreamProcessingStatus> => {
  const toolCallRequests: ToolCallRequestInfo[] = [];
  let receivedContent = false;
  let lastContentTime = Date.now();

  for await (const event of stream) {
    switch (event.type) {
      case ServerGeminiEventType.Content:
        receivedContent = true;
        lastContentTime = Date.now();
        // 累积内容
        messageBuffer += event.value;
        // 实时显示（增量更新）
        updateDisplayBuffer(event.value);
        break;

      case ServerGeminiEventType.ToolCallRequest:
        toolCallRequests.push(event.value);
        // Web 应该显示工具调用确认 UI
        break;

      case ServerGeminiEventType.Thought:
        // 显示思考过程
        setThought(event.value);
        break;

      case ServerGeminiEventType.Error:
        // 错误处理和显示
        throw new StreamError(event.value);
        break;

      case ServerGeminiEventType.LoopDetected:
        // 检测到循环，可能需要用户干预
        showLoopWarning();
        break;

      case ServerGeminiEventType.ChatCompressed:
        // 聊天已被压缩（为了节省令牌）
        updateCompressionStatus(event.value);
        break;

      case ServerGeminiEventType.Finished:
        // 完成
        return {
          success: true,
          toolCallRequests,
          content: messageBuffer,
        };
    }
  }
};
```

## 5. 非交互 CLI 的实现

**位置**: `packages/cli/src/nonInteractiveCli.ts`

非交互模式展示了如何使用相同的 Core API 实现不同的 UI：

```typescript
export async function runNonInteractive({
  config,
  settings,
  input,
  prompt_id,
}: RunNonInteractiveParams): Promise<void> {
  // 1. 获取 Core 客户端
  const geminiClient = config.getGeminiClient();

  // 2. 初始化
  await geminiClient.initialize();

  // 3. 发送查询
  const stream = geminiClient.sendMessageStream(
    input,
    abortController.signal,
    prompt_id
  );

  // 4. 处理流
  for await (const event of stream) {
    if (event.type === GeminiEventType.Content) {
      // 纯文本输出
      process.stdout.write(event.value);
    } else if (event.type === GeminiEventType.ToolCallRequest) {
      // 自动执行工具（无需用户确认）
      const result = await executeToolCall(
        config,
        event.value,
        abortSignal
      );
      geminiClient.getChat().recordCompletedToolCalls(
        config.getModel(),
        [result]
      );
    }
  }
}
```

## 6. 工具执行接口

**位置**: `packages/core/src/tools/execute.ts`

```typescript
export async function executeToolCall(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  signal: AbortSignal
): Promise<CompletedToolCall> {
  const { toolName, input } = toolCallRequest;

  // 获取工具执行器
  const toolExecutor = config.getToolRegistry().getToolExecutor(toolName);

  // 执行工具
  const result = await toolExecutor.execute(
    input,
    signal
  );

  // 返回结构化结果
  return {
    id: toolCallRequest.id,
    toolName,
    input,
    result,
    timestamp: Date.now(),
  };
}
```

## 7. 配置系统（无 UI 依赖）

**位置**: `packages/core/src/config/config.ts`

```typescript
export class Config {
  // ✅ Core 客户端
  getGeminiClient(): GeminiClient
  getContentGenerator(): ContentGenerator

  // ✅ 初始化
  async initialize(): Promise<void>

  // ✅ 注册表
  getToolRegistry(): ToolRegistry
  getPromptRegistry(): PromptRegistry

  // ✅ 认证
  async refreshAuth(authMethod: AuthType): Promise<void>
  getAuthToken(): string

  // ✅ 模型和设置
  getModel(): string
  setModel(model: string): void
  getSettings(): Settings

  // ✅ 会话管理
  getSessionId(): string
  getHistory(): Content[]
  setHistory(history: Content[]): void

  // ✅ 事件系统
  on(event: CoreEvent, callback: Callback): void
  off(event: CoreEvent, callback: Callback): void
}
```

**关键特性**：
- ✅ 完全无 UI 依赖的配置管理
- ✅ 所有 getter/setter 都是异步安全的
- ✅ 支持事件订阅和通知
- ✅ 支持多会话管理

## 8. a2a-server 的参考实现

**位置**: `packages/a2a-server/src/agent/executor.ts`

a2a-server 展示了如何在完全不同的上下文中使用 Core：

```typescript
export class CoderAgentExecutor implements AgentExecutor {
  async execute(
    requestContext: RequestContext,
    eventBus: ExecutionEventBus,
  ): Promise<void> {
    // 1. 创建 Core Config（与 CLI 相同）
    const config = await this.getConfig(settings);

    // 2. 初始化
    await config.initialize();
    const geminiClient = config.getGeminiClient();

    // 3. 核心循环
    let agentEvents = geminiClient.sendMessageStream(
      userMessage,
      abortSignal,
      promptId
    );

    while (true) {
      const toolCallRequests = [];

      // 4. 处理事件
      for await (const event of agentEvents) {
        if (event.type === GeminiEventType.ToolCallRequest) {
          toolCallRequests.push(event.value);
        } else {
          // 转发事件到 A2A EventBus
          await eventBus.publish({
            type: event.type,
            data: event.value,
          });
        }
      }

      // 5. 执行工具
      if (toolCallRequests.length > 0) {
        for (const req of toolCallRequests) {
          const result = await executeToolCall(
            config,
            req,
            abortSignal
          );
          completedTools.push(result);
        }

        // 6. 将结果发送回 Core
        agentEvents = geminiClient.sendMessageStream(
          {
            // 工具结果作为新的消息
            toolResults: completedTools,
          },
          abortSignal,
          promptId
        );
      } else {
        break; // 循环完成
      }
    }
  }
}
```

## 9. Web UI 的集成方案

基于以上分析，Web UI 应该这样集成 Core：

```typescript
// packages/web-server/src/websocket-handler.ts

const handleWebSocketConnection = async (ws: WebSocket) => {
  // 1. 初始化 Config 和 GeminiClient
  const config = new Config({
    sessionId: `web-${Date.now()}`,
    targetDir: process.cwd(),
    model: 'gemini-2.0-flash',
  });
  await config.initialize();

  const geminiClient = config.getGeminiClient();
  await geminiClient.initialize();

  // 2. 监听客户端消息
  ws.on('message', async (rawData) => {
    const message = JSON.parse(rawData);

    if (message.type === 'send_message') {
      // 3. 发送到 Core
      const stream = geminiClient.sendMessageStream(
        message.content,
        abortSignal,
        message.promptId
      );

      // 4. 流式推送事件到客户端
      for await (const event of stream) {
        ws.send(JSON.stringify({
          type: 'gemini_event',
          event: event.type,
          data: event.value,
        }));
      }
    } else if (message.type === 'execute_tool') {
      // 5. 执行工具
      const result = await executeToolCall(
        config,
        message.toolRequest,
        abortSignal
      );

      // 6. 记录结果
      geminiClient.getChat().recordCompletedToolCalls(
        config.getModel(),
        [result]
      );

      // 7. 发送确认
      ws.send(JSON.stringify({
        type: 'tool_executed',
        result,
      }));
    }
  });
};
```

## 10. Web UI 前端的事件处理

```typescript
// packages/web-ui/src/api/gemini-stream.ts

export async function* streamGeminiMessage(
  message: string,
  onEvent: (event: GeminiEvent) => void
) {
  const ws = new WebSocket(
    `${WS_URL}/api/chat/stream`
  );

  // 发送消息
  ws.send(JSON.stringify({
    type: 'send_message',
    content: message,
    promptId: generatePromptId(),
  }));

  // 监听事件
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    onEvent({
      type: data.event,
      value: data.data,
    });
  };

  // React 组件中使用
  const { submitMessage } = useGeminiChat();

  const handleSendMessage = async (message: string) => {
    for await (const event of streamGeminiMessage(
      message,
      (event) => {
        switch (event.type) {
          case 'content':
            setResponseText(prev => prev + event.value);
            break;
          case 'tool_call_request':
            showToolConfirmation(event.value);
            break;
          case 'thought':
            setThinkingProcess(event.value);
            break;
          case 'finished':
            setLoading(false);
            break;
        }
      }
    )) {
      // 流处理
    }
  };
}
```

## 总结

### ✅ Web UI 可以直接复用的

1. **GeminiClient** - 核心 API 接口，返回事件流
2. **ServerGeminiStreamEvent** - 事件类型定义
3. **Config** - 配置管理系统
4. **executeToolCall** - 工具执行函数
5. **ToolRegistry** - 工具注册表
6. **ConfirmationBus** - 用户确认机制

### 🎯 Web UI 需要新建的

1. **WebSocket Handler** - WebSocket 连接管理
2. **Stream Adapters** - 将 Core 事件适配到 WebSocket 消息
3. **React Hooks** - 替代 useGeminiStream 的 Web 版本
4. **UI Components** - React 组件替代 Ink

### 📊 架构优势

- ✅ Core 完全无 UI 依赖，可被任何 UI 使用
- ✅ 事件驱动设计，易于实现流式传输
- ✅ 现有的 a2a-server 证明了可行性
- ✅ 只需实现适配层，无需修改 Core

