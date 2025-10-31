# Overmind 包设计方案

## 概述

Overmind 是一个 Web 端的 AI Agent，复用 core 的能力。它与现有的 CLI、a2a-server 和 VS Code IDE Companion 属于同一类别的集成方案，都是基于 `@google/gemini-cli-core` 提供的核心功能进行扩展。

```
packages/
├── core/                    # 核心 AI 引擎（独立）
│
├── cli/                     # 终端 UI 集成
│   └── 依赖: core + React(Ink) + Yargs
│
├── a2a-server/             # HTTP API 服务器集成
│   └── 依赖: core + Express + A2A SDK
│
├── vscode-ide-companion/   # VS Code IDE 集成
│   └── 依赖: core + VS Code API
│
└── overmind/ (NEW)         # Web UI 集成（建设中）
    └── 依赖: core + Express + React
```

## 推荐方案：单包结构

采用**单包结构**（类似 CLI 和 a2a-server），前后端在同一个包中，保持与现有项目的一致性。

### 设计理由

1. **项目一致性** ✅
   - CLI 是单包：React(Ink) + 逻辑 + Yargs
   - a2a-server 是单包：Express + 逻辑 + A2A SDK
   - overmind 保持同样的模式：Express + React + 逻辑

2. **开发体验** ✅
   - 前后端共享类型定义，零成本
   - WebSocket 通信协议在一个地方维护
   - 一个 npm install，一个构建命令

3. **部署简单** ✅
   ```bash
   npm run build
   npm run start
   ```

4. **易于扩展** ✅
   - 如果未来需要分离，前后端代码已经物理分离
   - 可以轻松提取为独立包

## 详细结构设计

### 目录布局

```
packages/overmind/
├── src/
│   ├── server/                    # Express 服务器
│   │   ├── index.ts              # 服务器启动入口
│   │   ├── app.ts                # Express 应用设置
│   │   ├── middleware/
│   │   │   ├── auth.ts           # 认证中间件
│   │   │   ├── cors.ts           # CORS 中间件
│   │   │   └── error-handler.ts  # 错误处理
│   │   ├── websocket/
│   │   │   ├── index.ts          # WebSocket 服务器设置
│   │   │   ├── message-handler.ts # 消息处理逻辑
│   │   │   └── connection-manager.ts # 连接管理
│   │   └── routes/
│   │       ├── index.ts
│   │       ├── chat.ts           # POST /api/chat, WebSocket /ws
│   │       ├── config.ts         # GET/POST /api/config
│   │       ├── mcp.ts            # GET/POST /api/mcp
│   │       └── health.ts         # GET /api/health
│   │
│   ├── client/                    # React 前端
│   │   ├── index.tsx             # React 入口
│   │   ├── App.tsx               # 主应用组件
│   │   ├── components/
│   │   │   ├── ChatInterface.tsx  # 聊天主界面
│   │   │   ├── MessageList.tsx    # 消息列表
│   │   │   ├── InputBox.tsx       # 输入框
│   │   │   ├── ToolRequestDialog.tsx # 工具确认对话
│   │   │   ├── ConfigPanel.tsx    # 配置面板
│   │   │   └── Sidebar.tsx        # 侧边栏
│   │   ├── pages/
│   │   │   ├── ChatPage.tsx       # 聊天页面
│   │   │   └── SettingsPage.tsx   # 设置页面
│   │   ├── hooks/
│   │   │   ├── useChat.ts         # 聊天逻辑 hook
│   │   │   ├── useWebSocket.ts    # WebSocket 连接 hook
│   │   │   ├── useConfig.ts       # 配置管理 hook
│   │   │   └── useToolConfirm.ts  # 工具确认 hook
│   │   ├── store/
│   │   │   ├── index.ts           # 状态管理导出
│   │   │   ├── chat-store.ts      # 聊天状态（Zustand）
│   │   │   ├── config-store.ts    # 配置状态
│   │   │   └── ui-store.ts        # UI 状态
│   │   ├── api/
│   │   │   ├── client.ts          # HTTP API 客户端
│   │   │   └── websocket-client.ts # WebSocket 客户端
│   │   ├── utils/
│   │   │   ├── message-formatter.ts # 消息格式化
│   │   │   ├── syntax-highlighter.ts # 代码高亮
│   │   │   └── validators.ts      # 验证工具
│   │   └── styles/
│   │       ├── globals.css
│   │       └── variables.css
│   │
│   ├── shared/                    # 前后端共享代码
│   │   ├── types.ts              # 共享类型定义
│   │   ├── constants.ts          # 常量
│   │   ├── websocket-messages.ts # WebSocket 消息类型
│   │   └── api-schema.ts         # API 数据结构
│   │
│   └── __tests__/                # 测试
│       ├── server/
│       ├── client/
│       └── shared/
│
├── public/                        # 静态资源
│   ├── index.html
│   ├── favicon.ico
│   └── assets/
│
├── package.json
├── tsconfig.json                 # 通用 TypeScript 配置
├── tsconfig.server.json          # 服务器特定配置
├── vite.config.ts                # Vite 构建配置（前端）
├── vitest.config.ts              # 测试配置
├── eslint.config.js
├── README.md
└── .env.example
```

### package.json 配置

```json
{
  "name": "@google/gemini-cli-overmind",
  "version": "0.1.0",
  "description": "Overmind - Web UI Agent for Gemini CLI",
  "type": "module",
  "main": "dist/server/index.js",
  "scripts": {
    "dev": "npm-run-all --parallel dev:server dev:client",
    "dev:server": "tsx watch src/server/index.ts",
    "dev:client": "vite",
    "build": "npm run build:server && npm run build:client",
    "build:server": "tsc -p tsconfig.server.json",
    "build:client": "vite build",
    "start": "node dist/server/index.js",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write src",
    "typecheck": "tsc --noEmit"
  },
  "files": [
    "dist",
    "public"
  ],
  "dependencies": {
    "@google/gemini-cli-core": "file:../core",
    "express": "^5.1.0",
    "ws": "^8.14.0",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "zustand": "^4.4.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/express": "^5.0.3",
    "@types/node": "^20.11.24",
    "@types/react": "^19.1.8",
    "@types/react-dom": "^19.1.6",
    "@types/ws": "^8.5.0",
    "typescript": "^5.3.3",
    "vite": "^6.0.0",
    "vitest": "^3.2.4",
    "tsx": "^4.20.3",
    "eslint": "^9.24.0",
    "prettier": "^3.5.3"
  },
  "engines": {
    "node": ">=20"
  }
}
```

## 核心特性设计

### 1. 前后端通信协议

**WebSocket 消息类型** （src/shared/websocket-messages.ts）

```typescript
// 客户端 -> 服务器
export type ClientMessage =
  | {
      type: 'send_message';
      content: string;
      sessionId: string;
    }
  | {
      type: 'tool_response';
      toolCallId: string;
      result: unknown;
    }
  | {
      type: 'cancel_request';
      requestId: string;
    };

// 服务器 -> 客户端
export type ServerMessage =
  | {
      type: 'text_chunk';
      content: string;
      requestId: string;
    }
  | {
      type: 'tool_request';
      toolName: string;
      args: unknown;
      toolCallId: string;
    }
  | {
      type: 'message_complete';
      requestId: string;
    }
  | {
      type: 'error';
      message: string;
      code: string;
    };
```

### 2. API 端点设计

```
POST /api/config
  获取/更新配置

GET /api/config/models
  获取可用模型列表

POST /api/sessions
  创建新对话

GET /api/sessions/:id
  获取对话历史

DELETE /api/sessions/:id
  删除对话

POST /api/mcp/discover
  发现 MCP 服务器

GET /api/health
  健康检查

WebSocket /ws
  实时聊天连接
```

### 3. 认证和授权

```typescript
// 支持多种认证方式
enum AuthMethod {
  GOOGLE_OAUTH = 'google_oauth',
  API_KEY = 'api_key',
  VERTEX_AI = 'vertex_ai',
  NONE = 'none' // 开发模式
}

// 中间件验证
// middleware/auth.ts
```

### 4. 核心业务逻辑集成

```typescript
// server/handlers/chat-handler.ts
class ChatHandler {
  constructor(
    private geminiClient: GeminiClient,
    private config: CliConfig
  ) {}

  async handleMessage(
    message: string,
    sessionId: string
  ): Promise<AsyncGenerator<ServerMessage>> {
    // 使用 GeminiClient 发送消息
    // 将响应流式传输给客户端
  }

  async handleToolRequest(
    toolName: string,
    args: unknown
  ): Promise<unknown> {
    // 通过 core 的工具系统执行工具
  }
}
```

## 开发工作流

### 本地开发

```bash
# 安装依赖（在根目录）
npm ci

# 启动开发服务器（同时启动前后端）
npm run dev --workspace @google/gemini-cli-overmind

# 或分别启动
# Terminal 1: 后端 (http://localhost:3000)
npm run dev:server --workspace @google/gemini-cli-overmind

# Terminal 2: 前端 (http://localhost:5173)
npm run dev:client --workspace @google/gemini-cli-overmind
```

### 构建和部署

```bash
# 构建
npm run build:overmind

# 生成的文件结构
dist/
├── server/
│   ├── index.js
│   ├── app.js
│   ├── middleware/
│   ├── websocket/
│   ├── routes/
│   └── ...
└── client/
    ├── index.html
    ├── assets/
    └── ...

# 启动生产服务器
NODE_ENV=production node dist/server/index.js

# 访问 http://localhost:3000
```

### 生产部署选项

#### 方案 1: 前后端一起部署（简单）

```bash
npm run build:overmind
npm start --workspace @google/gemini-cli-overmind
# 静态文件由 Express 提供
# http://localhost:3000
```

#### 方案 2: 分离部署（高级）

```bash
# 后端
NODE_ENV=production node dist/server/index.js --port 8080

# 前端用 Nginx/CDN 托管 dist/client/
# 配置反向代理到 /api 和 /ws
```

## 根 package.json 脚本集成

在根目录 package.json 中添加：

```json
{
  "scripts": {
    "dev:overmind": "npm run dev --workspace @google/gemini-cli-overmind",
    "build:overmind": "npm run build --workspace @google/gemini-cli-overmind",
    "start:overmind": "npm start --workspace @google/gemini-cli-overmind",
    "test:overmind": "npm run test --workspace @google/gemini-cli-overmind"
  }
}
```

## 与 Core 的集成

### 依赖关系

```typescript
// 获取配置
import { loadCliConfig } from '@google/gemini-cli-core';

const config = await loadCliConfig({
  projectRoot: process.cwd(),
  // ... other options
});

// 创建客户端
import { GeminiClient } from '@google/gemini-cli-core';

const client = new GeminiClient(config);

// 发送消息
const response = client.sendMessageStream('用户输入');

// 执行工具
const result = await config.executeToolCall({
  name: 'tool_name',
  args: { /* ... */ }
});
```

### 共享功能

- **GeminiClient** - 聊天管理和 API 通信
- **Tool Registry** - 工具注册和执行
- **Config System** - 配置管理和认证
- **Services** - 文件操作、Git、Shell 等

## 前端技术选择

### 核心库
- **React 19** - 与 CLI 保持一致
- **Zustand** - 轻量级状态管理
- **Vite** - 快速构建和开发体验
- **TypeScript** - 类型安全

### 可选库（后期添加）
- **@tanstack/react-query** - 数据获取
- **TailwindCSS** - 样式框架
- **Radix UI** - 无样式组件库
- **Framer Motion** - 动画库

## 后端技术选择

### 核心库
- **Express 5.x** - 与 a2a-server 保持一致
- **ws** - WebSocket 支持
- **Zod** - 数据验证

### 可选库
- **@google/genai** - 已在 core 中使用
- **uuid** - ID 生成
- **dotenv** - 环境变量管理

## 测试策略

### 单元测试

```typescript
// src/__tests__/client/hooks/useChat.test.ts
import { renderHook, act } from '@testing-library/react';
import { useChat } from '../../../client/hooks/useChat';

describe('useChat', () => {
  it('should send message and receive response', async () => {
    // ...
  });
});
```

### 集成测试

```typescript
// src/__tests__/server/routes/chat.test.ts
import request from 'supertest';
import { app } from '../../../server/app';

describe('POST /api/chat', () => {
  it('should handle chat request', async () => {
    // ...
  });
});
```

### E2E 测试

```typescript
// 使用 Playwright 或 Cypress
// 测试完整的用户交互流程
```

## 类型定义共享

### 后端可以导出类型

```typescript
// src/server/routes/chat.ts
export interface ChatRequest {
  content: string;
  sessionId: string;
}

export interface ChatResponse {
  requestId: string;
  content: string;
}
```

### 前端导入使用

```typescript
// src/client/api/client.ts
import type { ChatRequest, ChatResponse } from '../../server/routes/chat';

export async function sendChat(req: ChatRequest): Promise<ChatResponse> {
  // ...
}
```

## 未来的扩展和迁移

### 如果需要分离前后端（第 2 阶段）

1. 创建 `packages/overmind-ui` - 复制 `src/client/`
2. 创建 `packages/overmind-server` - 复制 `src/server/`
3. 创建 `packages/overmind-shared` - 复制 `src/shared/`
4. 调整依赖和导入路径

由于前后端代码已经物理分离，迁移会很顺畅。

## 环境变量

`.env.example`

```env
# 服务器配置
NODE_ENV=development
PORT=3000
LOG_LEVEL=info

# Gemini API 配置（传递给 core）
GEMINI_API_KEY=
GOOGLE_CLOUD_PROJECT=

# CORS
CORS_ORIGIN=http://localhost:5173

# WebSocket
WS_HEARTBEAT_INTERVAL=30000
```

## 监控和日志

```typescript
// 使用 OpenTelemetry（与 core 保持一致）
// 或简单的 console 日志配合结构化日志库
```

## 性能考虑

1. **WebSocket 连接池** - 管理多个并发连接
2. **消息流式传输** - 避免缓冲整个响应
3. **前端虚拟化** - 大量消息时使用虚拟滚动
4. **错误恢复** - 自动重新连接和消息重试

## 安全考虑

1. **认证** - 支持多种认证方法
2. **授权** - 检查用户有权访问的工具和资源
3. **速率限制** - 防止滥用
4. **输入验证** - Zod 验证所有输入
5. **CORS** - 配置允许的来源
6. **XSS 防护** - React 自动转义，额外验证用户输入
7. **CSRF** - Token-based CSRF 保护

## 总结

**单包结构的优势**：
- ✅ 与现有项目风格一致（CLI、a2a-server）
- ✅ 开发体验最优（类型共享、一键启动）
- ✅ 部署简单（一个构建输出）
- ✅ 易于维护（集中管理依赖版本）
- ✅ 未来扩展性强（可轻松分离）

**下一步**：
1. 创建基础项目结构
2. 实现后端框架（Express + WebSocket）
3. 实现前端框架（React 应用骨架）
4. 集成 core 的聊天功能
5. 实现工具执行和确认流程
6. 添加测试和文档
