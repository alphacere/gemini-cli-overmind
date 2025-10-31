# Web UI 包结构设计：单包 vs 双包分析

## 现有项目结构参考

Gemini CLI 项目中的其他集成方案：

```
packages/
├── cli/                    # @google/gemini-cli
│   └── 依赖: core
│
├── a2a-server/            # @google/gemini-cli-a2a-server
│   └── 依赖: core + Express
│
├── vscode-ide-companion/  # gemini-cli-vscode-ide-companion
│   └── 依赖: core + VS Code API
│
└── core/                  # @google/gemini-cli-core
    └── 核心业务逻辑（独立）
```

**观察**：
- ✅ CLI 是一个整体包（UI + 服务混合）
- ✅ a2a-server 是一个整体包（HTTP API + 服务器）
- ✅ VS Code 伴侣是一个整体包（IDE 插件 + 通信）

---

## 方案对比

### 方案 A：单包结构（⭐⭐⭐⭐⭐ 推荐）

**结构**：
```
packages/web/
├── src/
│   ├── server/            # Express 服务器代码
│   │   ├── index.ts
│   │   ├── websocket.ts   # WebSocket 处理
│   │   ├── routes/        # API 路由
│   │   │   ├── chat.ts
│   │   │   ├── config.ts
│   │   │   └── mcp.ts
│   │   └── middleware/    # 中间件
│   │
│   ├── client/            # React 前端代码
│   │   ├── index.tsx
│   │   ├── components/    # React 组件
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── store/         # Zustand 状态管理
│   │   └── api/           # API 客户端
│   │
│   └── shared/            # 前后端共享代码
│       ├── types.ts       # 类型定义
│       └── constants.ts
│
├── public/                # 静态资源
├── package.json
├── tsconfig.json
├── vite.config.ts         # Vite 构建配置
├── webpack.config.js      # 服务器构建（可选）
└── README.md
```

**package.json 配置**：
```json
{
  "name": "@google/gemini-cli-web",
  "version": "0.1.0",
  "description": "Web UI for Gemini CLI",
  "scripts": {
    "dev": "npm-run-all --parallel dev:server dev:client",
    "dev:server": "tsx watch src/server/index.ts",
    "dev:client": "vite",
    "build": "npm run build:server && npm run build:client",
    "build:server": "tsc src/server --outDir dist/server",
    "build:client": "vite build",
    "start": "node dist/server/index.js",
    "test": "vitest"
  },
  "dependencies": {
    "@google/gemini-cli-core": "workspace:*",
    "express": "^5.x",
    "ws": "^8.x",
    "react": "^19.x",
    "zustand": "^4.x"
  }
}
```

**优点**：
✅ **结构清晰** - 前后端代码在一起，好理解
✅ **易于部署** - 一个包，一个构建输出
✅ **依赖管理简单** - 共享依赖版本
✅ **符合项目风格** - 与 CLI、a2a-server 的模式一致
✅ **开发体验好** - 可以同时开发前后端
✅ **易于测试** - 前后端集成测试方便

**缺点**：
❌ 打包体积大（前后端都打进去）
❌ 依赖冲突可能性增加

---

### 方案 B：双包结构（⭐⭐⭐ 一般）

**结构**：
```
packages/
├── web-server/            # 后端 HTTP 服务
│   ├── src/
│   │   ├── index.ts
│   │   ├── websocket.ts
│   │   ├── routes/
│   │   │   ├── chat.ts
│   │   │   └── config.ts
│   │   ├── middleware/
│   │   └── types.ts
│   ├── package.json
│   └── tsconfig.json
│
└── web-ui/                # 前端 React 应用
    ├── src/
    │   ├── index.tsx
    │   ├── components/
    │   ├── pages/
    │   ├── hooks/
    │   ├── store/
    │   ├── api/
    │   └── types.ts
    ├── public/
    ├── package.json
    ├── vite.config.ts
    └── tsconfig.json
```

**优点**：
✅ 职责分离 - 前后端完全分开
✅ 独立部署 - 前端可以用 CDN，后端用 Node
✅ 独立开发 - 前端工程师只需关注 web-ui
✅ 包体积小 - 可分别优化
✅ 技术栈灵活 - 后端改 Bun 不影响前端

**缺点**：
❌ **仓库复杂** - 需要跨包协调
❌ **两套 tsconfig、package.json** - 维护麻烦
❌ **发布复杂** - 需要协调两个包的版本
❌ **不符合项目风格** - 项目目前没有前后端分离包
❌ **初期开发慢** - 需要同时启动两个 npm install
❌ **共享类型困难** - 前后端类型定义难以同步

---

### 方案 C：单文件应用（⭐⭐ 不推荐）

直接把前端打包进后端，单个可执行文件：

```
packages/web/
└── src/
    ├── server.ts          # Express 服务器
    ├── index.tsx          # React 入口
    └── ...
```

构建输出：`dist/web.js` 包含前后端所有代码

**缺点**：
❌ 打包体积巨大
❌ 开发体验差
❌ 难以维护

---

## 推荐方案：单包结构（方案 A）

### 理由

1. **符合项目现状**
   - CLI 包是单包（React + Ink + 服务）
   - a2a-server 包是单包（Express + 逻辑）
   - 保持一致的包设计理念

2. **团队规模**
   - 你目前是一个人开发
   - 单包更容易管理
   - 如果后期需要分离，很容易重构

3. **部署简单**
   ```bash
   # 单包：
   npm install
   npm run build
   npm start

   # 双包：
   npm install --workspaces
   npm run build --workspaces
   npm start --workspace web-server
   # （还要另起一个终端启动前端）
   ```

4. **开发体验**
   - 类型共享无缝
   - WebSocket 通信类型定义在一个地方
   - 前后端接口同步容易

5. **代码共享**
   - 可以在 `src/shared/` 中共享类型、常量、工具函数
   - 不需要重复定义 API 响应类型

---

## 具体实施方案（单包）

### 项目布局

```
packages/web/
├── src/
│   ├── server/
│   │   ├── index.ts              # 启动入口
│   │   ├── app.ts                # Express 应用创建
│   │   ├── websocket-handler.ts  # WebSocket 连接处理
│   │   ├── routes/
│   │   │   ├── index.ts
│   │   │   ├── chat.ts           # POST /api/chat/*
│   │   │   ├── config.ts         # GET /api/config
│   │   │   ├── mcp.ts            # GET/POST /api/mcp
│   │   │   └── files.ts          # GET/POST /api/files
│   │   ├── handlers/
│   │   │   ├── chat-handler.ts   # 聊天业务逻辑
│   │   │   ├── tool-handler.ts   # 工具执行
│   │   │   └── config-handler.ts
│   │   ├── middleware/
│   │   │   ├── auth.ts           # 认证中间件
│   │   │   ├── error-handler.ts
│   │   │   └── cors.ts
│   │   └── utils/
│   │       ├── logger.ts
│   │       └── validators.ts
│   │
│   ├── client/
│   │   ├── index.tsx             # React 入口
│   │   ├── App.tsx               # 主组件
│   │   ├── components/
│   │   │   ├── ChatWindow.tsx    # 聊天窗口
│   │   │   ├── MessageList.tsx   # 消息列表
│   │   │   ├── InputBox.tsx      # 输入框
│   │   │   └── ToolConfirm.tsx   # 工具确认对话
│   │   ├── pages/
│   │   │   ├── ChatPage.tsx
│   │   │   └── SettingsPage.tsx
│   │   ├── hooks/
│   │   │   ├── useGeminiChat.ts  # 聊天 hook
│   │   │   ├── useWebSocket.ts   # WebSocket hook
│   │   │   └── useConfig.ts      # 配置 hook
│   │   ├── store/
│   │   │   ├── index.ts
│   │   │   ├── chat-store.ts     # Zustand store
│   │   │   ├── config-store.ts
│   │   │   └── types.ts
│   │   ├── api/
│   │   │   ├── client.ts         # API 客户端
│   │   │   └── websocket-client.ts
│   │   └── utils/
│   │       ├── formatting.ts
│   │       └── validators.ts
│   │
│   └── shared/
│       ├── types.ts              # 前后端共享类型
│       ├── constants.ts          # 常量
│       ├── api-schema.ts         # API 规范（Zod）
│       └── websocket-messages.ts # WebSocket 消息类型
│
├── public/
│   ├── index.html
│   ├── favicon.ico
│   └── styles/
│
├── package.json
├── tsconfig.json
├── tsconfig.server.json          # 服务器特定配置
├── vite.config.ts
├── vitest.config.ts
├── README.md
└── .env.example
```

### package.json 示例

```json
{
  "name": "@google/gemini-cli-web",
  "version": "0.1.0",
  "description": "Web UI for Gemini CLI",
  "type": "module",
  "main": "dist/server/index.js",
  "exports": {
    ".": {
      "node": "./dist/server/index.js",
      "browser": "./dist/client/index.js"
    }
  },
  "scripts": {
    "dev": "npm-run-all --parallel dev:server dev:client",
    "dev:server": "tsx watch src/server/index.ts",
    "dev:client": "vite",
    "build": "npm run build:server && npm run build:client",
    "build:server": "tsc -p tsconfig.server.json",
    "build:client": "vite build",
    "start": "node dist/server/index.js",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint src",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@google/gemini-cli-core": "workspace:*",
    "express": "^5.0.0",
    "ws": "^8.14.0",
    "zod": "^3.22.0",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "zustand": "^4.4.0",
    "@tanstack/react-query": "^5.28.0",
    "tailwindcss": "^3.4.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.0",
    "@types/ws": "^8.5.0",
    "@types/react": "^18.2.0",
    "typescript": "^5.3.0",
    "vite": "^6.0.0",
    "vitest": "^3.2.0",
    "tsx": "^4.20.0",
    "eslint": "^9.24.0",
    "prettier": "^3.5.0"
  }
}
```

### 根 package.json 中的脚本

```json
{
  "scripts": {
    "start:web": "npm start --workspace @google/gemini-cli-web",
    "dev:web": "npm run dev --workspace @google/gemini-cli-web",
    "build:web": "npm run build --workspace @google/gemini-cli-web",
    "test:web": "npm run test --workspace @google/gemini-cli-web"
  }
}
```

---

## 开发工作流

### 本地开发

```bash
# Terminal 1: 启动后端 + 前端
npm run dev:web

# 或分别启动
# Terminal 1: 后端
npm run dev:server --workspace @google/gemini-cli-web

# Terminal 2: 前端
npm run dev:client --workspace @google/gemini-cli-web
```

### 构建和部署

```bash
# 构建
npm run build:web

# 生成的文件
dist/
├── server/              # 后端代码
│   └── index.js
└── client/              # 前端静态文件
    ├── index.html
    ├── assets/
    └── ...

# 部署
# 方案 1：前后端一起（简单）
npm run start --workspace @google/gemini-cli-web
# 访问 http://localhost:3000

# 方案 2：分离部署（生产优选）
# 后端
NODE_ENV=production node dist/server/index.js
# 前端用 Nginx 或 CDN 托管 dist/client/
```

---

## 前后端通信示例

### 共享类型定义（src/shared/types.ts）

```typescript
// 所有 WebSocket 消息都在这里定义
export type WebSocketMessage =
  | { type: 'send_message'; content: string; promptId: string }
  | { type: 'execute_tool'; toolRequest: ToolRequest }
  | { type: 'tool_executed'; result: CompletedToolCall }
  | { type: 'gemini_event'; event: ServerGeminiStreamEvent }
  | { type: 'error'; error: string };
```

### 后端实现（src/server/websocket-handler.ts）

```typescript
import type { WebSocketMessage } from '../shared/types.js';

ws.on('message', (rawData) => {
  const message: WebSocketMessage = JSON.parse(rawData);

  switch (message.type) {
    case 'send_message':
      // 处理...
      break;
    case 'execute_tool':
      // 处理...
      break;
  }
});
```

### 前端实现（src/client/hooks/useWebSocket.ts）

```typescript
import type { WebSocketMessage } from '../../shared/types.js';

const sendMessage = (message: WebSocketMessage) => {
  ws.send(JSON.stringify(message));
};

const handleMessage = (message: WebSocketMessage) => {
  switch (message.type) {
    case 'gemini_event':
      // 处理...
      break;
    case 'tool_executed':
      // 处理...
      break;
  }
};
```

---

## 迁移路径（如果未来需要）

如果后期你需要将前后端分离，迁移也很简单：

1. **Step 1**: 创建新的 web-ui 包，复制 `src/client/` 内容
2. **Step 2**: 创建新的 web-server 包，复制 `src/server/` 内容
3. **Step 3**: 将 `src/shared/` 提取为共享包
4. **Step 4**: 调整依赖和导入路径

因为前后端代码本身就是分离的，这个过程会很顺畅。

---

## 总结

| 方面 | 单包（推荐） | 双包 |
|------|-----------|------|
| 初期开发 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 代码共享 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 类型安全 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 部署简单 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 扩展灵活 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 团队协作 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**结论**：
- 👉 **现在**：用单包（快速开发）
- 👉 **未来**：如果需要，轻松迁移到双包（前端独立部署等）

