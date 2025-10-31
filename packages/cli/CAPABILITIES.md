# CLI 模块能力详解

CLI 模块是用户与 Gemini AI 互动的入口，提供终端用户界面（使用 React + Ink）和命令行交互能力。

## 一、核心能力分类

### 1. 启动和初始化
### 2. 命令行参数处理
### 3. 用户界面层（Ink + React）
### 4. 交互模式
### 5. 非交互模式
### 6. 工具执行和确认流程
### 7. 斜杠命令系统
### 8. 设置和配置管理
### 9. 错误处理和日志
### 10. Core 能力集成

## 二、详细能力说明

### 1. 启动和初始化

#### 1.1 入口点（gemini.tsx）

**文件**：`packages/cli/src/gemini.tsx` (506 行)

**核心职责**：
- 解析命令行参数
- 加载用户设置和配置
- 处理认证和沙箱初始化
- 分发到交互或非交互模式

**启动流程**：
```
main()
  ├─ loadSettings()
  ├─ parseArguments(Yargs)
  ├─ validateDnsResolutionOrder()
  ├─ loadCustomThemes()
  ├─ checkSandboxRequirements()
  │   └─ startSandbox() (如需要)
  ├─ loadCliConfig()
  ├─ initializeApp()
  │   ├─ initializeAuthentication()
  │   ├─ initializeTheme()
  │   └─ initializeIdeClient()
  └─ startInteractiveUI() 或 runNonInteractive()
```

#### 1.2 配置加载

**函数**：`loadCliConfig(settings, sessionId, argv, cwd)`

**关键步骤**：
1. 加载扩展（ExtensionManager）
2. 加载内存文件（GEMINI.md）
3. 确定交互模式
4. 合并 MCP 服务器配置
5. 确定批准模式
6. 创建 Config 实例（来自 Core）

**主要配置选项**（150+）：
- 模型选择
- 工具允许/阻止列表
- 批准模式（DEFAULT/AUTO_EDIT/YOLO）
- 输出格式（TEXT/JSON/STREAM_JSON）
- 工作目录和文件过滤
- MCP 服务器配置

#### 1.3 认证初始化

**函数**：`initializeApp(config, settings)`

**包含**：
- 验证认证凭证是否有效
- 刷新过期的 OAuth 令牌
- 初始化认证状态
- 触发重新认证对话框（如需要）

**支持的认证方法**：
- OAuth（Login with Google）
- API Key（GEMINI_API_KEY）
- Vertex AI（Google Cloud）
- Cloud Shell

#### 1.4 沙箱检查

**检查项**：
- 是否启用了沙箱（--sandbox docker/podman/none）
- Docker 或 Podman 是否可用
- 需要时重启沙箱进程

**作用**：
- 在沙箱中安全执行 Shell 命令
- 隔离文件系统访问
- 防止恶意代码执行

---

### 2. 命令行参数处理

#### 2.1 Yargs 集成

**文件**：`packages/cli/src/config/config.ts` - `parseArguments()`

**框架**：Yargs（版本 17.7.2）

**核心命令结构**：
```bash
gemini [query..] [options]        # 主命令
gemini /command [args]             # 斜杠命令
gemini mcp [subcommand]            # MCP 管理
gemini extensions [subcommand]     # 扩展管理
```

#### 2.2 参数验证

**验证规则**：
- 不能同时使用位置参数和 `--prompt` 标志
- `--prompt-interactive` 和 `--prompt` 互斥
- 工具列表格式验证
- 输出格式有效性检查

**返回类型**：
```typescript
interface CliArgs {
  query?: string;                    // 位置参数
  model?: string;                    // -m, --model
  prompt?: string;                   // -p, --prompt（deprecated）
  promptInteractive?: string;        // -i, --prompt-interactive
  yolo?: boolean;                    // -y, --yolo
  approvalMode?: string;             // approval mode choice
  allowedTools?: string[];           // 允许的工具列表
  allowedMcpServerNames?: string[];  // MCP 服务器允许列表
  extensions?: string[];             // -e, --extensions
  outputFormat?: string;             // -o, --output-format
  sandbox?: string;                  // sandbox type
  // ... 更多选项
}
```

#### 2.3 全局选项

| 选项 | 别名 | 类型 | 描述 |
|------|------|------|------|
| `--debug` | | boolean | 调试模式 |
| `--model` | `-m` | string | 选择 Gemini 模型 |
| `--prompt` | `-p` | string | 一次性提示（deprecated） |
| `--prompt-interactive` | `-i` | string | 交互模式提示 |
| `--yolo` | `-y` | boolean | 自动批准所有工具 |
| `--approval-mode` | | choice | DEFAULT/AUTO_EDIT/YOLO |
| `--output-format` | `-o` | choice | TEXT/JSON/STREAM_JSON |
| `--allowed-tools` | | array | 允许的工具列表 |
| `--exclude-tools` | | array | 排除的工具列表 |
| `--extensions` | `-e` | array | 启用的扩展 |
| `--sandbox` | | choice | docker/podman/none |

#### 2.4 子命令

**MCP 命令**：
```bash
gemini mcp list                    # 列出已配置的 MCP 服务器
gemini mcp test <server-name>      # 测试 MCP 服务器连接
gemini mcp discover                # 发现新 MCP 服务器
```

**扩展命令**：
```bash
gemini extensions list              # 列出已安装的扩展
gemini extensions install <ext>     # 安装扩展
gemini extensions uninstall <ext>   # 卸载扩展
```

**交互模式判断逻辑**：
```typescript
const isInteractive =
  !!argv.promptInteractive ||        // 显式 -i
  (process.stdin.isTTY &&            // 是 TTY（终端）
   !hasQuery &&                       // 没有位置参数
   !argv.prompt);                     // 没有 -p
```

---

### 3. 用户界面层

#### 3.1 Ink + React 架构

**技术栈**：
- **React 19** - UI 框架
- **Ink 6.2.3** - Terminal React 渲染器
- **TypeScript** - 类型安全

**渲染流程**：
```typescript
// 1. 创建 React 组件树
const AppWrapper = () => (
  <SettingsContext.Provider value={settings}>
    <KeypressProvider>
      <SessionStatsProvider>
        <VimModeProvider>
          <AppContainer config={config} {...} />
        </VimModeProvider>
      </SessionStatsProvider>
    </KeypressProvider>
  </SettingsContext.Provider>
);

// 2. 使用 Ink 渲染到终端
const instance = render(<AppWrapper />, {
  exitOnCtrlC: false,
  isScreenReaderEnabled: config.getScreenReader(),
});

// 3. 注册清理（Ctrl+C 处理）
registerCleanup(() => instance.unmount());
```

**关键特性**：
- 流式渲染（实时更新）
- 键盘输入处理（Kitty Protocol）
- Vim 模式支持
- 屏幕阅读器适配

#### 3.2 AppContainer 中枢组件

**文件**：`packages/cli/src/ui/AppContainer.tsx` (1466 行)

**核心职责**：
- 整个 UI 状态管理
- 聊天流处理
- 工具调度和确认
- 对话框管理
- Context 提供

**主要 Hook 初始化**：
```typescript
// 历史记录管理
const historyManager = useHistory();

// 聊天流处理
const { streamingState, submitQuery, pendingHistoryItems } = useGeminiStream(...);

// 文本缓冲
const buffer = useTextBuffer({...});

// 工具调度
const [toolCalls, schedule, markAsSubmitted] = useReactToolScheduler(...);

// 斜杠命令
const { handleSlashCommand } = useSlashCommandProcessor(...);

// 设置对话框
const { openSettingsDialog, isSettingsDialogOpen } = useSettingsCommand();

// 认证管理
const { authState, setAuthState } = useAuthCommand(settings, config);
```

**提供的 Context**：
```typescript
<UIStateContext>        // UI 显示状态
<UIActionsContext>      // UI 交互事件
<ConfigContext>         // Core 配置
<ShellFocusContext>     // Shell 焦点状态
```

#### 3.3 Context 和状态管理

**UIState 对象**：
```typescript
interface UIState {
  history: HistoryItem[];           // 消息历史
  streamingState: StreamingState;   // IDLE / RESPONDING / WAITING_FOR_CONFIRMATION
  pendingHistoryItems: HistoryItem[]; // 未完成的消息
  thought?: string;                  // 当前思考过程
  isInputActive: boolean;            // 输入框是否活跃
  slashCommands: SlashCommand[];     // 可用的斜杠命令
  buffer: TextBuffer;                // 当前输入文本
  toolCalls: TrackedToolCall[];      // 当前工具状态
  // ... 更多状态字段
}
```

**状态流转**：
```
User Input
    ↓
IDLE
    ↓
submitQuery()
    ↓
RESPONDING
    ├─ 接收文本事件
    ├─ 接收工具请求
    └─ 收集工具响应
    ↓
WAITING_FOR_CONFIRMATION (如有工具)
    ├─ 显示确认对话框
    ├─ 等待用户决策
    └─ 执行工具
    ↓
RESPONDING (继续等待 Gemini)
    ↓
IDLE (完成)
```

#### 3.4 键盘和输入处理

**输入处理**：
- **Ctrl+C** - 取消操作或退出
- **Tab** - 命令自动完成
- **Up/Down Arrow** - 历史导航
- **Enter** - 提交消息
- **:q** (Vim) - 退出

**KeypressProvider**：
```typescript
<KeypressProvider kittyProtocolEnabled={kittyProtocolStatus}>
  // 提供 useKeypress hook
  // 支持 Kitty Keyboard Protocol 高级键盘事件
</KeypressProvider>
```

**输入框特性**：
- 多行输入支持
- 代码块识别
- 自动缩进
- 语法提示

---

### 4. 交互模式

#### 4.1 UI 启动流程

**函数**：`startInteractiveUI(config, settings, warnings, cwd, initResult)`

**步骤**：
1. 禁用行换行（提升 UI 效果）
2. 创建 Context Provider 树
3. 使用 Ink 渲染到终端
4. 注册清理处理器

**特殊处理**：
```typescript
// 为屏幕阅读器禁用某些 ANSI 序列
if (!config.getScreenReader()) {
  process.stdout.write('\x1b[?7l');  // 禁用行换行
}
```

#### 4.2 聊天窗口组件

**核心组件**：
- `App` - 主应用组件
- `ChatWindow` - 聊天消息显示
- `MessageList` - 消息列表（可虚拟滚动）
- `InputBox` - 用户输入框
- `StatusBar` - 状态栏（模型、Token、模式）

**组件通信**：
```
AppContainer (状态中心)
    ├─ ChatWindow (消息显示)
    ├─ InputBox (用户输入)
    ├─ ToolConfirmationDialog (工具确认)
    ├─ SettingsDialog (设置)
    ├─ AuthDialog (认证)
    └─ ThemeDialog (主题选择)
```

#### 4.3 消息显示和渲染

**消息类型**：
```typescript
enum MessageType {
  UserMessage = 'user_message',
  GeminiResponse = 'gemini_response',
  ToolCall = 'tool_call',
  ToolResult = 'tool_result',
  Error = 'error',
  Info = 'info',
  Thought = 'thought'
}
```

**渲染特性**：
- 代码块高亮（highlight.js）
- Markdown 格式化
- ANSI 颜色支持
- 工具输出流式显示
- 差异显示（Diff 视图）

#### 4.4 输入处理

**useGeminiStream Hook**：
```typescript
const {
  streamingState,          // 流状态
  submitQuery,             // 提交查询函数
  pendingHistoryItems,     // 未完成的历史项
  thought,                 // 当前思考
  error,                   // 错误信息
  cancelOngoingRequest     // 取消函数
} = useGeminiStream(config, onToolsRequested, onComplete);
```

**提交流程**：
```
1. 用户按 Enter
2. InputBox 调用 submitQuery(text)
3. useGeminiStream 处理：
   ├─ 验证输入
   ├─ 处理 @ 命令（文件包含）
   ├─ 调用 GeminiClient.sendMessageStream()
   ├─ 监听流事件
   ├─ 更新 AppContainer 状态
   └─ 触发工具调度（如有）
```

---

### 5. 非交互模式

#### 5.1 runNonInteractive 流程

**文件**：`packages/cli/src/commands/nonInteractive.ts` (350 行)

**执行流程**：
```
runNonInteractive(config, settings, input, prompt_id)
    ├─ 处理输入
    │  ├─ 检查是否为斜杠命令
    │  ├─ 处理 @ 命令（文件包含）
    │  └─ 普通提示
    ├─ 调用 GeminiClient.sendMessageStream()
    ├─ 收集响应和工具请求
    ├─ 执行工具（无需用户确认）
    ├─ 收集工具响应
    ├─ 重复直到完成
    ├─ 格式化输出（TEXT/JSON/STREAM_JSON）
    └─ 返回结果
```

**关键特点**：
- 无 UI，纯文本交互
- 工具自动执行（批准模式=YOLO）
- 支持管道和脚本中使用
- 适合 CI/CD 和自动化

#### 5.2 自动工具执行

```typescript
// 工具请求自动处理
for (const toolRequest of toolCallRequests) {
  const completedToolCall = await executeToolCall(
    config,
    toolRequest,
    abortSignal
  );

  // 工具响应收集
  toolResponseParts.push(...completedToolCall.response.responseParts);
}

// 继续发送给模型
const nextMessages = [{
  role: 'user',
  parts: toolResponseParts
}];
```

#### 5.3 输出格式化

**TEXT 格式**（默认）：
```
[Gemini response text]
```

**JSON 格式**：
```json
{
  "status": "success",
  "result": "response text",
  "tokens": {
    "input": 100,
    "output": 50
  },
  "toolCalls": [...]
}
```

**STREAM_JSON 格式**（换行分隔的 JSON）：
```json
{"type": "content", "data": "text chunk"}
{"type": "tool_request", "data": {...}}
{"type": "tool_result", "data": {...}}
{"type": "complete", "status": "success"}
```

**选择输出格式**：
```bash
gemini -p "query" --output-format json
gemini -p "query" -o stream-json
```

---

### 6. 工具执行和确认流程

#### 6.1 工具调度（useReactToolScheduler）

**Hook 签名**：
```typescript
const [
  toolCalls,              // TrackedToolCall[] 当前工具状态
  schedule,               // (request, signal) => void
  markAsSubmitted,        // (callIds) => void
  setToolCallsForDisplay, // (calls) => void
  cancelAll               // (signal) => void
] = useReactToolScheduler(onComplete, config, getPreferredEditor, onEditorClose);
```

**工具调度流程**：
```
GeminiClient 生成 ToolCallRequest
    ↓
useGeminiStream 调用 schedule()
    ↓
useReactToolScheduler 处理
    ├─ 验证参数 (Validating)
    ├─ 根据批准模式判断
    │  ├─ YOLO: 直接执行
    │  ├─ AUTO_EDIT: 编辑后执行
    │  └─ DEFAULT: 等待用户确认
    ├─ 显示确认对话框
    └─ 用户决策 (ProceedOnce/ProceedAlways/Cancel)
    ↓
执行工具 (Executing)
    ├─ 实时流式输出
    ├─ 更新 UI
    └─ 收集响应
    ↓
工具完成 (Success/Error)
    ↓
调用 onComplete(completedToolCalls)
    ↓
继续等待 Gemini 响应
```

#### 6.2 用户确认流程

**确认决策类型**：
```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',           // 执行一次
  ProceedAlways = 'proceed_always',       // 自动批准此工具
  ModifyWithEditor = 'modify_with_editor', // 编辑命令后执行
  Cancel = 'cancel'                        // 拒绝
}
```

**确认条件**：
- **DEFAULT 模式** - 所有工具都需要确认
- **AUTO_EDIT 模式** - 编辑类工具可编辑，执行类工具需确认
- **YOLO 模式** - 所有工具自动执行，无需确认

#### 6.3 确认对话框

**对话框类型**：
- `ShellConfirmationDialog` - Shell 命令确认
- `EditConfirmationDialog` - 文件编辑确认
- `GeneralToolConfirmationDialog` - 通用工具确认

**显示信息**：
```typescript
{
  type: 'exec',        // 或 'edit', 'mcp', 'info'
  title: 'Run Shell Command',
  description: 'The AI wants to execute a shell command',
  details: [
    'Command: rm -rf /',
    'Directory: /project'
  ],
  onConfirm: async (outcome) => { ... }
}
```

#### 6.4 工具输出显示

**输出特性**：
- 实时流式显示（不等待完成）
- ANSI 颜色支持
- 错误和成功区分
- 输出截断（16MB 限制）
- 二进制数据检测

**消息结构**：
```typescript
interface ToolMessage {
  type: 'tool_call';
  toolName: string;
  args: Record<string, unknown>;
  timestamp: number;
}

interface ToolResultMessage {
  type: 'tool_result';
  toolName: string;
  output: string;
  error?: string;
  success: boolean;
}
```

---

### 7. 斜杠命令系统

#### 7.1 内置斜杠命令

**内置命令列表**：
- `/help` - 显示帮助信息
- `/clear` - 清空聊天历史
- `/model` - 切换 AI 模型
- `/settings` - 打开设置对话框
- `/theme` - 切换主题
- `/auth` - 重新认证
- `/version` - 显示版本信息
- `/quit` / `/exit` - 退出应用
- `/checkpoint` - 创建代码检查点
- `/restore` - 从检查点恢复
- `/memory` - 查看用户记忆
- `/mcp` - MCP 服务器管理

#### 7.2 命令处理流程

**Hook**：`useSlashCommandProcessor`

```typescript
const {
  handleSlashCommand,      // (rawQuery) => Promise<Result | false>
  slashCommands,           // SlashCommand[]
  pendingHistoryItems,     // 待显示的历史项
  commandContext,          // 命令执行上下文
  shellConfirmationRequest // Shell 确认请求
} = useSlashCommandProcessor(config, settings, addItem, clearItems, ...);
```

**处理流程**：
```
1. InputBox 提交消息
2. 检查是否以 / 或 ? 开头
3. 调用 handleSlashCommand()
4. 解析命令和参数
5. 查找命令定义
6. 执行命令 action()
7. 返回结果：
   ├─ 'handled' - 命令已处理
   ├─ 'message' - 显示消息
   ├─ 'dialog' - 打开对话框
   ├─ 'schedule_tool' - 安排工具
   ├─ 'submit_prompt' - 提交新提示
   ├─ 'confirm_shell_commands' - Shell 确认
   └─ 'quit' - 退出
```

#### 7.3 MCP 提示集成

**MCP 提示加载**：
```typescript
const commandService = new CommandService([
  new McpPromptLoader(config),      // MCP 服务器的提示
  new BuiltinCommandLoader(config), // 内置命令
  new FileCommandLoader(config),    // 项目目录的自定义命令
]);
```

**MCP 提示使用**：
```bash
/my-mcp-tool arg1 arg2    # 执行 MCP 提示
```

#### 7.4 自定义命令加载

**自定义命令文件位置**：
```
$PROJECT_ROOT/.gemini/commands/
├── my-command.ts         # 自定义命令
└── helper.ts             # 辅助函数
```

**自定义命令定义**：
```typescript
export const myCommand: SlashCommand = {
  name: 'my-command',
  displayName: 'My Custom Command',
  description: 'Does something custom',
  action: async (context, args) => {
    // context 包含 config, settings, git, logger, ui 等
    return {
      type: 'message',
      content: 'Command executed'
    };
  }
};
```

---

### 8. 设置和配置管理

#### 8.1 设置文件位置

**优先级（低到高）**：
1. **默认设置** - 代码内置
2. **系统级** - `$GEMINI_HOME/settings.json`（可选）
3. **用户级** - `~/.gemini/settings.json`
4. **工作空间级** - `.gemini/settings.json`（当前项目）

#### 8.2 设置加载和合并

**函数**：`loadSettings()`

**返回**：
```typescript
interface LoadedSettings {
  merged: Settings;      // 合并后的所有设置
  defaults: Settings;    // 默认值
  user: Settings;        // 用户级设置
  workspace: Settings;   // 工作空间级设置
  system?: Settings;     // 系统级设置
  setValue: (scope, key, value) => void;  // 保存设置
  getValue: (scope, key) => unknown;      // 读取设置
}
```

**主要设置字段**：
```typescript
{
  // 认证
  auth: {
    selectedType: AuthType;
    googleOAuthTokens?: OAuthToken;
    apiKey?: string;
    // ...
  },

  // UI
  ui: {
    theme: string;
    fontSize: number;
    customThemes?: Theme[];
    // ...
  },

  // MCP 服务器
  mcpServers?: Record<string, MCPServerConfig>;

  // 工具设置
  tools: {
    allowedTools?: string[];
    excludeTools?: string[];
    // ...
  },

  // 特性开关
  features: {
    enableCheckpointing?: boolean;
    enableRipgrep?: boolean;
    enableSmartEdit?: boolean;
    // ...
  }
}
```

#### 8.3 设置持久化

```typescript
// 保存设置
settings.setValue(SettingScope.User, 'auth.selectedType', AuthType.GOOGLE_OAUTH);
settings.setValue(SettingScope.Workspace, 'tools.allowedTools', ['read-file', 'write-file']);

// 读取设置
const authType = settings.getValue(SettingScope.User, 'auth.selectedType');
```

---

### 9. 错误处理和日志

#### 9.1 错误分类和处理

**错误类型**：
```typescript
// 认证错误
- InvalidCredentialsError
- ExpiredTokenError
- MissingAuthenticationError

// 工具错误
- ToolValidationError      // 参数验证失败
- ToolExecutionError       // 工具执行失败
- FatalToolError           // 致命工具错误

// 配置错误
- ConfigurationError
- SandboxError

// 取消和限制
- CancellationError        // 用户取消
- MaxTurnsExceededError    // 对话轮次超限
- ContextWindowExceededError // 上下文窗口溢出
```

**处理函数**：
```typescript
export function handleError(error: unknown, config: Config): never
export function handleToolError(toolName, error, config, errorType?)
export function handleCancellationError(config: Config)
export function handleMaxTurnsExceededError(config: Config)
```

#### 9.2 错误输出格式

**TEXT 格式**：
```
Error: [error message]
```

**JSON 格式**：
```json
{
  "status": "error",
  "error": {
    "type": "ToolExecutionError",
    "message": "Command failed",
    "code": "ENOENT"
  }
}
```

**STREAM_JSON 格式**：
```json
{"type": "error", "error": {"type": "...", "message": "..."}}
```

#### 9.3 日志收集

**日志来源**：
- CLI 输出（console.log, console.error）
- Core 日志（debugLogger）
- 工具执行日志
- 网络请求日志

**日志拦截**：
```typescript
const consolePatcher = new ConsolePatcher({
  stderr: true,
  debugMode: isDebugMode,
  onNewMessage: (msg) => {
    // 实时收集日志
    addConsoleMessage(msg);
  }
});
consolePatcher.patch();
```

**调试模式**：
```bash
gemini --debug    # 启用调试日志
```

---

### 10. Core 能力集成

#### 10.1 配置创建和初始化

```typescript
// 1. 创建 Config 实例（由 loadCliConfig 完成）
const config = new Config({
  sessionId: generateId(),
  targetDir: process.cwd(),
  model: 'gemini-2.5-pro',
  approvalMode: ApprovalMode.DEFAULT,
  // ... 150+ 其他配置
});

// 2. 初始化 Config
await config.initialize();
  // - 初始化文件发现服务
  // - 创建 Git 服务
  // - 创建工具注册表
  // - 初始化 GeminiClient
```

#### 10.2 GeminiClient 使用

```typescript
// 获取客户端
const geminiClient = config.getGeminiClient();

// 发送消息
const stream = geminiClient.sendMessageStream(
  userQuery,      // 字符串或 Part[] 数组
  abortSignal,    // 支持取消
  promptId        // 追踪 ID
);

// 监听流事件
for await (const event of stream) {
  // 事件处理
}
```

#### 10.3 事件处理

**事件类型**：
```typescript
import { GeminiEventType } from '@google/gemini-cli-core';

for await (const event of stream) {
  switch (event.type) {
    case GeminiEventType.Content:
      // 文本响应
      addMessageToHistory(event.value.text);
      break;

    case GeminiEventType.ToolCallRequest:
      // 工具请求
      scheduleToolExecution(event.value);
      break;

    case GeminiEventType.Error:
      // 错误
      handleError(event.value);
      break;

    case GeminiEventType.Thought:
      // 思考过程
      setThought(event.value.thought);
      break;

    case GeminiEventType.LoopDetected:
      // 循环检测
      cancelOperation();
      break;
  }
}
```

#### 10.4 历史管理

**历史对象**：
```typescript
interface HistoryItem {
  id: string;
  type: MessageType;           // 用户/Gemini/工具/错误等
  content: string;
  timestamp: number;
  metadata?: {
    model?: string;
    tokens?: { input: number; output: number };
    toolName?: string;
    thought?: string;
  }
}
```

**历史操作**：
```typescript
// 添加消息
historyManager.addItem(item);

// 清空历史
historyManager.clearItems();

// 获取历史
const history = historyManager.getItems();

// 删除特定项
historyManager.removeItem(itemId);

// 编辑历史（重新发送）
historyManager.editAndResubmit(itemId, newContent);
```

---

## 三、完整请求流程

### 从用户输入到响应的数据流

```
1. 用户在 InputBox 中输入提示
   ↓
2. InputBox 验证输入，处理特殊命令
   ├─ 如果是斜杠命令 → handleSlashCommand()
   └─ 如果是 @ 命令 → 加载指定文件
   ↓
3. submitQuery(userInput)
   ↓
4. useGeminiStream 处理
   ├─ 调用 GeminiClient.sendMessageStream()
   ├─ 开始流式接收事件
   └─ 更新 AppContainer 状态为 RESPONDING
   ↓
5. 处理流事件
   ├─ Content 事件 → 添加到 pendingHistoryItems
   ├─ ToolCallRequest 事件 → 调用 schedule()
   ├─ LoopDetected 事件 → 停止并提示
   └─ Error 事件 → 错误处理
   ↓
6. 如有工具请求
   ├─ useReactToolScheduler 调用 schedule()
   ├─ 显示确认对话框
   ├─ 等待用户决策
   ├─ 执行工具
   ├─ 流式显示输出
   └─ 收集响应
   ↓
7. 工具完成后，发送响应给 Gemini
   ├─ 状态变回 RESPONDING
   └─ 继续接收 Gemini 响应
   ↓
8. 最后消息后，状态变为 IDLE
   ↓
9. 显示最终结果和工具执行摘要
```

---

## 四、主要 Hook 和 API

| Hook | 来源 | 主要功能 |
|------|------|---------|
| `useGeminiStream` | AppContainer | 聊天流处理，消息提交 |
| `useReactToolScheduler` | useGeminiStream | 工具调度和确认 |
| `useSlashCommandProcessor` | AppContainer | 斜杠命令处理 |
| `useHistory` | AppContainer | 消息历史管理 |
| `useTextBuffer` | AppContainer | 输入框文本缓冲 |
| `useAuthCommand` | AppContainer | 认证管理 |
| `useSettingsCommand` | AppContainer | 设置对话框 |
| `useKeypress` | KeypressProvider | 键盘输入处理 |

---

## 五、与 Core 的集成点

| CLI 层 | → | Core 层 |
|--------|---|---------|
| AppContainer | 初始化 | Config |
| useGeminiStream | 调用 | GeminiClient.sendMessageStream() |
| useReactToolScheduler | 使用 | ToolRegistry + CoreToolScheduler |
| useSlashCommandProcessor | 读取 | Config（文件服务、Git 服务等） |
| Tool execution | 调用 | Tool.execute() |
| Error handling | 使用 | Core 的错误类型和处理 |
| Telemetry | 调用 | logUserPrompt(), logToolCall() 等 |

---

## 总结

**CLI 模块的核心能力**：
- ✅ **多模式支持** - 交互（Ink UI）和非交互（脚本）
- ✅ **用户友好** - 流式 UI、实时确认、清晰错误
- ✅ **灵活配置** - 命令行参数、设置文件、扩展
- ✅ **强大工具支持** - 工具执行、确认、输出显示
- ✅ **可扩展** - 斜杠命令、MCP 提示、自定义命令
- ✅ **安全** - 工具批准模式、权限控制
- ✅ **健壮** - 错误处理、取消支持、日志收集

**关键特点**：
- 完全依赖 Core 提供的聊天和工具能力
- 负责 UI 呈现、用户交互和工具确认流程
- 支持两种完全不同的运行模式（交互 vs 非交互）
- 高度模块化的 Hook 和 Context 架构
- 与 React + Ink 的深度集成

---

## 六、CLI 和 Core 的关系总结

### 职责划分

**Core 提供**（业务逻辑层）：
- AI 聊天引擎（GeminiClient）
- 工具系统（注册、验证、执行）
- 业务服务（文件、Git、Shell 等）
- 配置管理（Config 类）
- MCP 集成
- 遥测系统

**CLI 提供**（用户界面层）：
- 命令行参数解析
- 终端 UI（React + Ink）
- 用户交互（输入、确认）
- 工具执行的 UI 展示
- 斜杠命令处理
- 设置和认证 UI
- 错误友好化展示

### 依赖关系

```
CLI
 ├─ 依赖: @google/gemini-cli-core
 │  ├─ Config（配置和初始化）
 │  ├─ GeminiClient（聊天引擎）
 │  ├─ ToolRegistry（工具系统）
 │  ├─ Services（文件、Git、Shell）
 │  └─ 遥测系统
 │
 └─ 自有能力
    ├─ 命令行界面（Yargs）
    ├─ 终端 UI（React + Ink）
    ├─ Context 状态管理
    └─ Hook 和组件
```

### 数据流

```
用户输入
    ↓
CLI (parseArguments, loadSettings)
    ↓
Core (Config, GeminiClient initialization)
    ↓
Gemini API
    ↓
Core (Event stream, Tool Registry)
    ↓
CLI (UI rendering, Tool confirmation)
    ↓
用户界面
```

### 关键交互点

| 阶段 | CLI 角色 | Core 角色 |
|------|---------|---------|
| **初始化** | 加载设置、解析参数 | 创建 Config、初始化服务 |
| **聊天** | 显示 UI、接收输入 | GeminiClient 与 API 通信 |
| **工具请求** | 显示确认对话框 | ToolRegistry 发现工具 |
| **工具执行** | 用户确认、显示输出 | Tool.execute() 实际执行 |
| **错误** | 格式化显示 | 生成错误对象 |
| **遥测** | 调用日志函数 | 收集和上报数据 |

### 代码重用示例

```typescript
// CLI 使用 Core 的 Config
import { Config } from '@google/gemini-cli-core';
const config = new Config(options);
await config.initialize();

// CLI 使用 Core 的 GeminiClient
const client = config.getGeminiClient();
const stream = client.sendMessageStream(query, signal, promptId);

// CLI 使用 Core 的 ToolRegistry
const registry = config.getToolRegistry();
const tools = registry.getAllTools();

// CLI 使用 Core 的 Services
const fileService = config.getFileService();
const gitService = await config.getGitService();

// CLI 使用 Core 的 事件类型
import { GeminiEventType } from '@google/gemini-cli-core';
```
