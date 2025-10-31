# Gemini CLI Web UI 架构方案

## 概述

本文档描述了如何为 Gemini CLI 项目添加一个 web-ui，通过分层架构设计实现前后端分离，同时复用现有的核心业务逻辑。

## 当前架构现状

项目已经有以下可利用的基础：

1. **a2a-server 包** (`packages/a2a-server/`) - 已有 Express.js HTTP 服务器框架
2. **core 包** (`packages/core/`) - 包含所有核心业务逻辑（工具、代理、配置等）
3. **vscode-ide-companion** - 展示了如何集成到其他 UI（可参考）
4. **Config 系统** - 集中式配置管理，已支持持久化存储

## 推荐架构方案

采用 **分层集成** 的方式：

```
┌─────────────────────────────────────────┐
│         Web UI (新增)                    │
│   (React + TypeScript)                  │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────▼──────────┐
        │  HTTP API Layer     │
        │  (Express.js)       │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ @google/gemini-cli  │
        │ -core 业务逻辑      │
        │ (现有)              │
        └─────────────────────┘
```

## 项目结构设计

### 创建新的 packages/web-ui 和 packages/web-server 包

```bash
packages/
├── web-ui/                      # 前端 UI
│   ├── src/
│   │   ├── components/         # React 组件
│   │   ├── pages/             # 页面
│   │   ├── hooks/             # 自定义 hooks
│   │   ├── api/               # API 调用
│   │   ├── store/             # 状态管理
│   │   └── index.tsx          # 入口
│   ├── public/                # 静态资源
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
│
├── web-server/                  # 后端 HTTP API
│   ├── src/
│   │   ├── routes/            # API 路由
│   │   │   ├── chat.ts       # 聊天相关
│   │   │   ├── config.ts     # 配置相关
│   │   │   ├── mcp.ts        # MCP 服务器
│   │   │   └── files.ts      # 文件操作
│   │   ├── handlers/          # 请求处理器
│   │   ├── middleware/        # 中间件
│   │   ├── websocket/         # WebSocket 处理
│   │   └── index.ts           # Express 服务器入口
│   └── package.json
```

## 核心 API 接口设计

### 聊天相关

```
POST   /api/chat/start       # 开始新对话
POST   /api/chat/send        # 发送消息
GET    /api/chat/history     # 获取对话历史
POST   /api/chat/checkpoint  # 保存检查点
GET    /api/chat/:id         # 获取特定对话
```

### 配置相关

```
GET    /api/config           # 获取配置
POST   /api/config           # 更新配置
GET    /api/models           # 获取可用模型
POST   /api/models/:model    # 切换模型
```

### MCP 服务器相关

```
GET    /api/mcp/servers      # 列出 MCP 服务器
POST   /api/mcp/servers      # 添加服务器
DELETE /api/mcp/servers/:id  # 删除服务器
```

### 文件操作

```
GET    /api/files/:path      # 读取文件
POST   /api/files/:path      # 写入文件
GET    /api/files            # 文件树/目录列表
DELETE /api/files/:path      # 删除文件
```

## 通信方式选择

### WebSocket（推荐）

适合场景：实时聊天、流式生成内容

优点：
- 双向通信
- 低延迟
- 支持流式数据推送
- 可传输二进制数据

实现方式：
```typescript
// 客户端连接
const ws = new WebSocket('ws://localhost:3000/api/chat/stream');

// 发送消息
ws.send(JSON.stringify({
  type: 'message',
  content: 'user message'
}));

// 接收流式响应
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // 处理响应数据
};
```

### Server-Sent Events (SSE)

适合场景：单向流式数据，简化实现

### 长轮询

适合场景：简单场景、不需要实时性

## 前端状态管理设计

使用 Zustand 进行状态管理：

```typescript
// 状态结构
interface AppState {
  chat: {
    messages: Message[];
    currentTurn: Turn | null;
    loading: boolean;
    error: string | null;
  };
  config: {
    selectedModel: string;
    apiKey: string;
    settings: Record<string, any>;
  };
  mcp: {
    servers: MCPServer[];
    connections: Record<string, any>;
  };
  files: {
    currentDirectory: string;
    tree: FileNode[];
  };
}
```

推荐库组合：
- **Zustand** - 轻量级状态管理
- **TanStack Query** - API 缓存和同步
- **React Router** - 路由管理

## 技术栈建议

### 后端（web-server）

```json
{
  "express": "^5.x",
  "@google/gemini-cli-core": "workspace:*",
  "ws": "^8.x",
  "zod": "^3.x",
  "typescript": "^5.x"
}
```

### 前端（web-ui）

```json
{
  "react": "^19.x",
  "react-router-dom": "^6.x",
  "zustand": "^4.x",
  "@tanstack/react-query": "^5.x",
  "tailwindcss": "^3.x",
  "typescript": "^5.x",
  "vite": "^6.x"
}
```

## 实现路径

### 第一阶段：最小可行产品 (MVP)

完成基础聊天功能

```
1. 创建 web-server 包
   - Express 基础设置
   - /api/chat/* 路由
   - WebSocket 连接处理

2. 创建 web-ui 包
   - 基础 React 应用
   - 聊天页面
   - API 客户端

3. 集成 core 包业务逻辑
   - GeminiChat 封装
   - 工具执行处理
   - 流式响应推送
```

### 第二阶段：核心功能

- 对话历史管理和存储
- 文件浏览和编辑
- MCP 服务器配置管理
- 工具执行可视化
- 模型切换

### 第三阶段：高级功能

- 检查点和会话恢复
- 令牌缓存管理
- 策略编辑器 UI
- 沙箱配置
- 用户认证和权限管理

## 关键实现注意事项

### 安全性

- 需要继承 core 包的认证机制（API 密钥、OAuth）
- WebSocket 连接需要认证和授权
- 敏感操作需要用户确认（重用 confirmation-bus）
- 文件操作需要遵守 policy 规则

### 状态同步

- web-ui 需要与 CLI 共享同一个 Config 实例
- 需要考虑多客户端场景（web-ui + CLI 同时运行）
- 文件修改需要实时同步，避免冲突

### 工具执行

- 所有工具执行依然走 core 包的 ToolRegistry
- 需要在 web-ui 展示工具确认对话
- 流式输出需要实时推送给前端
- 错误处理和超时管理

### 性能优化

- 使用 TanStack Query 缓存 API 响应
- 大文件分块上传/下载
- WebSocket 心跳检测
- 消息分页加载

## 启动脚本配置

在根目录 `package.json` 添加：

```json
{
  "scripts": {
    "start:web-server": "npm start --workspace @google/gemini-cli-web-server",
    "start:web-ui": "npm start --workspace @google/gemini-cli-web-ui",
    "dev:web": "npm-run-all --parallel start:web-server start:web-ui",
    "build:web": "npm run build --workspaces web-server web-ui"
  }
}
```

## 与现有架构的集成

### 复用的组件

- **GeminiChat** - 聊天编排核心
- **ToolRegistry** - 工具执行系统
- **ConfirmationBus** - 用户确认机制
- **Config** - 配置管理系统
- **Policy Engine** - 执行策略控制
- **MCP Client** - MCP 服务器集成

### 新增的适配层

```typescript
// packages/web-server/src/adapters/gemini-chat-adapter.ts
// 将 core 的 GeminiChat 适配为 HTTP API
class GeminiChatAPI {
  private chat: GeminiChat;

  async startChat(options: ChatOptions) {
    // 创建新的 chat 会话
  }

  async sendMessage(chatId: string, message: string) {
    // 发送消息，返回流式响应
  }

  // 其他方法...
}
```

## 部署考虑

### 开发环境

```bash
# 终端 1：启动后端
npm start:web-server

# 终端 2：启动前端
npm start:web-ui
```

### 生产环境

- web-server 作为独立 Node.js 进程
- web-ui 构建为静态文件，由 web-server 托管（或 CDN）
- Docker 容器化部署
- 反向代理（Nginx）负载均衡

## 下一步建议

1. **详细设计 web-server 的 API 接口规范**（OpenAPI/Swagger）
2. **创建项目初始结构和配置文件**
3. **实现第一个功能模块**（比如聊天接口）
4. **处理认证、WebSocket 通信和流式传输**
5. **前端 UI 组件库搭建**

