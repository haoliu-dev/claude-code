# IDE 桥接系统

## 概述

桥接系统允许 Claude Code 与 IDE（VSCode、JetBrains）通信，实现功能如跳转到定义、代码编辑、终端集成等。

## 目录结构

```
bridge/
├── bridgeMain.ts           # 桥接主逻辑
├── bridgeApi.ts           # API 定义
├── bridgeMessaging.ts     # 消息协议
├── bridgeConfig.ts        # 配置管理
├── bridgeUI.ts            # UI 通信
├── bridgeStatusUtil.ts    # 状态工具
├── bridgePointer.ts       # 指针操作
├── bridgePermissionCallbacks.ts  # 权限回调
├── bridgeDebug.ts         # 调试工具
├── bridgeEnabled.ts       # 启用检查
├── capacityWake.ts       # 容量唤醒
├── createSession.ts       # 会话创建
├── initReplBridge.ts      # REPL 桥接初始化
├── jwtUtils.ts            # JWT 工具
├── pollConfig.ts          # 配置轮询
├── remoteBridgeCore.ts    # 远程桥接核心
├── replBridge.ts          # REPL 桥接
├── replBridgeHandle.ts    # 桥接句柄
├── replBridgeTransport.ts # 传输层
├── sessionRunner.ts       # 会话运行
├── trustedDevice.ts       # 信任设备
├── types.ts               # 类型定义
├── workSecret.ts          # 工作密钥
├── envLessBridgeConfig.ts # 环境无关配置
├── flushGate.ts           # 刷新门控
├── inboundAttachments.ts  # 入站附件
├── inboundMessages.ts     # 入站消息
├── debugUtils.ts          # 调试工具
└── sessionIdCompat.ts     # 会话 ID 兼容
```

## 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code (CLI)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐         ┌──────────────┐               │
│  │  bridgeMain  │◄────────│   replBridge │               │
│  └──────┬───────┘         └──────┬───────┘               │
│         │                         │                        │
│         ▼                         ▼                        │
│  ┌──────────────┐         ┌──────────────┐               │
│  │  bridgeApi   │         │    bridge     │               │
│  │  (接口定义)  │         │  Messaging    │               │
│  └──────────────┘         │  (消息协议)   │               │
│                           └──────┬───────┘               │
│                                  │                        │
└──────────────────────────────────┼────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │      Bridge Protocol        │
                    │   (WebSocket / SSE / IPC)   │
                    └──────────────┬──────────────┘
                                   │
┌──────────────────────────────────┼────────────────────────┐
│                     IDE Extension                           │
│                                                             │
│  ┌──────────────┐         ┌──────────────┐               │
│  │  IDE Bridge  │         │   Language   │               │
│  │   Client     │         │   Server     │               │
│  └──────────────┘         └──────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 消息协议

### 消息类型

```typescript
// bridge/types.ts
type BridgeMessage =
  | { type: 'request'; id: string; method: string; params: unknown }
  | { type: 'response'; id: string; result: unknown; error?: string }
  | { type: 'notification'; method: string; params: unknown }
  | { type: 'event'; event: string; data: unknown }
```

### 消息流程

```
Claude Code                          IDE
     │                                 │
     │──── request: gotoLocation ────►│
     │                                 │ 执行跳转
     │◄─── response: success ─────────│
     │                                 │
     │──── notification:cursorMove ───►│
     │                                 │
```

## 核心 API

### 跳转到定义

```typescript
// bridge/bridgeApi.ts
interface BridgeAPI {
  // 跳转到文件/行号
  gotoLocation(uri: string, line: number, column?: number): Promise<void>

  // 打开文件
  openFile(uri: string): Promise<void>

  // 获取当前文件
  getCurrentFile(): Promise<string>

  // 获取选中内容
  getSelection(): Promise<{ start: Position; end: Position }>

  // 执行 IDE 命令
  executeCommand(command: string, args?: unknown[]): Promise<void>

  // 显示通知
  showNotification(message: string, severity: 'info' | 'warning' | 'error'): void
}
```

### 文件操作

```typescript
interface FileBridgeAPI {
  // 读取文件
  readFile(uri: string): Promise<string>

  // 写入文件
  writeFile(uri: string, content: string): Promise<void>

  // 编辑文件
  editFile(uri: string, edits: TextEdit[]): Promise<void>

  // 获取文件 URI
  getFileUri(relativePath: string): string
}
```

### 终端操作

```typescript
interface TerminalBridgeAPI {
  // 创建终端
  createTerminal(name: string): Promise<string>

  // 发送命令
  sendToTerminal(terminalId: string, command: string): Promise<void>

  // 关闭终端
  closeTerminal(terminalId: string): Promise<void>
}
```

## 会话管理

### 会话创建

```typescript
// bridge/createSession.ts
export async function createBridgeSession(): Promise<BridgeSession> {
  // 1. 生成会话 ID
  const sessionId = generateUUID();

  // 2. 获取认证令牌
  const token = await getSessionToken();

  // 3. 建立连接
  const transport = await createTransport({
    type: 'websocket',
    url: getBridgeUrl(),
    token
  });

  // 4. 返回会话
  return {
    id: sessionId,
    transport,
    api: createBridgeAPI(transport)
  };
}
```

### 传输类型

```typescript
// 支持多种传输方式
type TransportType = 'websocket' | 'sse' | 'ipc';

interface Transport {
  send(message: BridgeMessage): Promise<void>
  onMessage(handler: (msg: BridgeMessage) => void): void
  close(): void
}
```

## 权限回调

IDE 桥接需要处理权限请求：

```typescript
// bridge/bridgePermissionCallbacks.ts
export const permissionCallbacks = {
  // 文件读取
  async onReadFile(path: string): Promise<boolean> {
    return confirm(`Allow reading ${path}?`);
  },

  // 文件写入
  async onWriteFile(path: string): Promise<boolean> {
    return confirm(`Allow writing to ${path}?`);
  },

  // 执行命令
  async onExecuteCommand(cmd: string): Promise<boolean> {
    return confirm(`Allow executing ${cmd}?`);
  },

  // 跳转到位置
  async onGotoLocation(uri: string): Promise<boolean> {
    return confirm(`Jump to ${uri}?`);
  }
};
```

## 远程桥接

对于远程连接，使用安全的远程桥接：

```typescript
// bridge/remoteBridgeCore.ts
export async function connectRemoteBridge(
  remoteUrl: string,
  sessionToken: string
): Promise<BridgeSession> {
  // 1. 验证远程服务器
  const serverCert = await verifyServerCertificate(remoteUrl);

  // 2. 建立 WebSocket 连接
  const ws = new WebSocket(remoteUrl, {
    headers: { Authorization: `Bearer ${sessionToken}` }
  });

  // 3. 协商协议版本
  const version = await negotiateProtocol(ws);

  // 4. 返回会话
  return createSession(ws, version);
}
```

## 调试功能

```typescript
// bridge/bridgeDebug.ts
export const bridgeDebug = {
  // 启用调试模式
  enable() {
    localStorage.set('bridgeDebug', 'true');
  },

  // 记录消息
  logMessage(msg: BridgeMessage) {
    if (!isDebugEnabled()) return;
    console.log('[Bridge]', msg);
  },

  // 获取连接状态
  getStatus(): BridgeStatus {
    return {
      connected: !!currentSession,
      latency: measureLatency(),
      messageCount: messageLog.length
    };
  }
};
```

---

下一章：[服务层](./10-services.md)
