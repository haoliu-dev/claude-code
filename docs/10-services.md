# 服务层

## 概述

服务层提供 API 调用、数据分析、MCP 协议支持、插件管理等核心服务。

## 目录结构

```
services/
├── api/                      # API 调用
│   ├── claude.ts            # Claude API
│   ├── errors.ts            # 错误处理
│   ├── filesApi.ts          # 文件 API
│   ├── bootstrap.ts         # 引导数据
│   ├── logging.ts           # 日志
│   ├── referral.ts          # 推荐
│   └── types.ts             # API 类型
├── mcp/                      # MCP 协议
│   ├── client.ts            # MCP 客户端
│   ├── server.ts            # MCP 服务器
│   ├── types.ts             # MCP 类型
│   ├── officialRegistry.ts  # 官方注册表
│   ├── config.ts            # MCP 配置
│   └── utils.ts             # 工具函数
├── analytics/                # 分析服务
│   ├── growthbook.ts        # A/B 测试
│   ├── config.ts            # 分析配置
│   └── sink.ts              # 数据sink
├── plugins/                  # 插件管理
│   ├── pluginCliCommands.ts # 插件命令
│   └── types.ts             # 插件类型
├── settings/                 # 设置服务
├── policyLimits/            # 策略限制
├── oauth/                   # OAuth 认证
├── remoteManagedSettings/   # 远程托管设置
├── teamMemorySync/          # 团队记忆同步
└── tools/                    # 工具服务
```

## API 服务

### Claude API 客户端

```typescript
// services/api/claude.ts
export class ClaudeAPIClient {
  private baseURL: string
  private apiKey: string

  constructor(config: APIConfig) {
    this.baseURL = config.baseURL ?? 'https://api.anthropic.com';
    this.apiKey = config.apiKey;
  }

  async createMessage(request: MessageRequest): Promise<MessageResponse> {
    const response = await fetch(`${this.baseURL}/v1/messages`, {
      method: 'POST',
      headers: {
        'x-api-key': this.apiKey,
        'anthropic-version': '2023-06-01',
        'content-type': 'application/json'
      },
      body: JSON.stringify(request)
    });

    return response.json();
  }
}
```

### API 请求/响应类型

```typescript
// services/api/types.ts
interface MessageRequest {
  model: string
  max_tokens: number
  messages: Message[]
  tools?: Tool[]
  system?: string
  stream?: boolean
}

interface MessageResponse {
  id: string
  type: 'message'
  role: 'assistant'
  content: ContentBlock[]
  model: string
  stop_reason: 'end_turn' | 'max_tokens' | 'stop_sequence'
  usage: {
    input_tokens: number
    output_tokens: number
  }
}

type ContentBlock =
  | { type: 'text'; text: string }
  | { type: 'tool_use'; id: string; name: string; input: Record<string, unknown> }
  | { type: 'tool_result'; tool_use_id: string; content: string }
```

### 错误处理

```typescript
// services/api/errors.ts
export class APIError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public type: 'rate_limit' | 'auth' | 'validation' | 'server' | 'unknown'
  ) {
    super(message);
  }
}

export async function handleAPIError(response: Response): Promise<never> {
  const error = await response.json();

  switch (response.status) {
    case 429:
      throw new APIError('Rate limited', 429, 'rate_limit');
    case 401:
      throw new APIError('Invalid API key', 401, 'auth');
    case 400:
      throw new APIError(error.error?.message ?? 'Bad request', 400, 'validation');
    default:
      throw new APIError('Server error', response.status, 'server');
  }
}
```

### 重试机制

```typescript
// 带重试的 API 调用
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries: number; backoff: number }
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < options.maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (!isRetryable(error)) throw error;

      await sleep(options.backoff * Math.pow(2, i));
    }
  }

  throw lastError!;
}
```

## MCP 服务

MCP (Model Context Protocol) 是与外部工具交互的协议。

### MCP 客户端

```typescript
// services/mcp/client.ts
export class MCPClient {
  private connections: Map<string, MCPServerConnection> = new Map();

  async connect(config: McpServerConfig): Promise<void> {
    const transport = await createTransport(config);

    const connection: MCPServerConnection = {
      id: config.id,
      transport,
      tools: new Map(),
      resources: new Map()
    };

    // 初始化握手
    await this.initialize(connection);

    this.connections.set(config.id, connection);
  }

  async callTool(name: string, args: Record<string, unknown>): Promise<ToolResult> {
    const connection = this.findConnection(name);

    return connection.transport.send({
      method: 'tools/call',
      params: { name, arguments: args }
    });
  }
}
```

### MCP 工具调用

```typescript
// services/mcp/client.ts
export async function getMcpToolsCommandsAndResources(): Promise<{
  tools: Tool[]
  resources: Resource[]
}> {
  const connections = mcpClient.getConnections();
  const tools: Tool[] = [];
  const resources: Resource[] = [];

  for (const conn of connections) {
    // 获取服务器提供的工具
    const serverTools = await conn.transport.request('tools/list');
    for (const tool of serverTools) {
      tools.push(wrapMcpTool(tool, conn));
    }

    // 获取资源
    const serverResources = await conn.transport.request('resources/list');
    resources.push(...serverResources);
  }

  return { tools, resources };
}
```

### MCP 配置

```typescript
// services/mcp/config.ts
interface McpServerConfig {
  id: string
  name: string
  command: string              // 启动命令
  args?: string[]              // 命令参数
  env?: Record<string, string> // 环境变量
  url?: string                 // HTTP URL (远程服务器)
}

interface MCPServerConnection {
  id: string
  transport: MCPTransport
  tools: Map<string, MCPTool>
  resources: Map<string, MCPResource>
  status: 'connecting' | 'connected' | 'disconnected'
}
```

## 分析服务

### GrowthBook A/B 测试

```typescript
// services/analytics/growthbook.ts
import { GrowthBook } from '@growthbook/growthbook-react';

export const growthbook = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: GROWTHBOOK_CLIENT_KEY,
  enableDevMode: isDevEnvironment()
});

export function useFeatureValue<T>(featureKey: string, defaultValue: T): T {
  const [value, setValue] = useState<T>(() =>
    growthbook.feature(featureKey).value ?? defaultValue
  );

  useEffect(() => {
    const handler = () => {
      setValue(growthbook.feature(featureKey).value ?? defaultValue);
    };

    growthbook.subscribe(handler);
    return () => growthbook.unsubscribe(handler);
  }, [featureKey]);

  return value;
}
```

### 事件追踪

```typescript
// services/analytics/sink.ts
export function logEvent(
  event: string,
  properties?: Record<string, unknown>
): void {
  if (isAnalyticsDisabled()) return;

  const payload = {
    event,
    properties: {
      ...properties,
      timestamp: Date.now(),
      sessionId: getSessionId()
    }
  };

  // 发送到分析后端
  fetch('/api/analytics', {
    method: 'POST',
    body: JSON.stringify(payload)
  });
}
```

## 插件服务

### 插件加载

```typescript
// services/plugins/pluginLoader.ts
export class PluginLoader {
  async loadPlugin(pluginSpec: PluginSpec): Promise<Plugin> {
    // 1. 解析插件规格
    const { type, source } = parsePluginSpec(pluginSpec);

    // 2. 获取插件代码
    const code = await this.fetchPlugin(type, source);

    // 3. 验证插件签名
    await this.verifySignature(code);

    // 4. 加载插件模块
    const module = await this.loadModule(code);

    // 5. 初始化插件
    const plugin = await module.init();

    return plugin;
  }
}
```

### 插件 API

```typescript
// services/plugins/types.ts
interface Plugin {
  id: string
  name: string
  version: string

  // 生命周期
  onLoad(): Promise<void>
  onUnload(): Promise<void>

  // 扩展点
  tools?: Tool[]
  commands?: Command[]
  hooks?: Hook[]
  components?: Component[]
}
```

---

下一章：[UI 组件](./11-components.md)
