# 系统架构

## 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│                     Presentation Layer                       │
│  (components/, ink/, screens/)                              │
├─────────────────────────────────────────────────────────────┤
│                     Application Layer                       │
│  (commands/, hooks/, state/, context.ts)                    │
├─────────────────────────────────────────────────────────────┤
│                      Service Layer                          │
│  (services/api/, services/mcp/, services/analytics/)       │
├─────────────────────────────────────────────────────────────┤
│                       Tool Layer                            │
│  (tools/BashTool/, tools/FileEditTool/, ...)               │
├─────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer                     │
│  (bridge/, utils/, cli/)                                   │
└─────────────────────────────────────────────────────────────┘
```

## 模块职责

### Presentation Layer - 展示层

负责 CLI 界面的渲染和用户交互。

| 模块 | 职责 |
|------|------|
| `ink/` | Ink 渲染引擎，处理 terminal 输出和输入事件 |
| `components/` | 业务 UI 组件（消息、输入框、对话框等） |
| `screens/` | 全屏界面（设置向导、帮助等） |
| `design-system/` | 设计系统（主题、颜色、通用组件） |

### Application Layer - 应用层

协调用户输入、命令执行和状态管理。

| 模块 | 职责 |
|------|------|
| `commands/` | CLI 命令定义和 slash command 处理 |
| `hooks/` | React hooks（状态订阅、事件处理） |
| `state/` | 全局状态管理（AppState） |
| `context.ts` | 全局上下文（设置、主题、权限） |
| `query.ts` | 查询引擎，用户输入处理 |

### Service Layer - 服务层

提供外部交互和业务逻辑服务。

| 模块 | 职责 |
|------|------|
| `services/api/` | Claude API 调用、错误处理、日志 |
| `services/mcp/` | Model Context Protocol 客户端 |
| `services/analytics/` | 分析、GrowthBook A/B 测试 |
| `services/plugins/` | 插件管理 |
| `services/settings/` | 设置管理 |

### Tool Layer - 工具层

实现 Agent 可调用的各种工具。

| 模块 | 职责 |
|------|------|
| `tools/BashTool/` | 执行 bash 命令 |
| `tools/FileEditTool/` | 编辑文件 |
| `tools/AgentTool/` | 创建子 Agent |
| `tools/MCPTool/` | 调用 MCP 工具 |
| `tools/Task*/` | 任务管理工具 |

### Infrastructure Layer - 基础设施层

底层支撑模块。

| 模块 | 职责 |
|------|------|
| `bridge/` | 与 IDE（VSCode、JetBrains）通信 |
| `utils/` | 通用工具函数 |
| `cli/transports/` | CLI 网络传输（SSE、WebSocket） |
| `memdir/` | 内存目录（会话记忆） |

## 数据流

### 用户输入到响应的完整流程

```
┌──────────────────────────────────────────────────────────────┐
│ 1. User Input (Terminal)                                     │
│    └─► ink/termio/termio.ts 读取按键事件                      │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. Input Processing (hooks/)                                 │
│    └─► useInput.ts, useCommandKeybindings.tsx                │
│    └─► 解析快捷键、命令补全                                    │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. Command Dispatch (commands.ts, commands/)                │
│    └─► slash command识别 (/commit, /branch)                   │
│    └─► 内置命令执行                                            │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. Query Processing (query.ts, QueryEngine.ts)               │
│    └─► 用户消息转为 API 请求                                   │
│    └─► 工具调用规划                                            │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. Tool Execution (tools/*/)                                 │
│    └─► BashTool: 执行命令                                     │
│    └─► FileEditTool: 编辑文件                                 │
│    └─► AgentTool: 创建子 Agent                               │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 6. API Call (services/api/claude.ts)                        │
│    └─► 调用 Claude API                                        │
│    └─► 处理流式响应                                            │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│ 7. Rendering (components/, ink/)                            │
│    └─► MessageRow.tsx 渲染消息                                │
│    └─► ink 渲染到终端                                          │
└──────────────────────────────────────────────────────────────┘
```

### 状态更新流程

```
User Action
     │
     ▼
┌─────────────┐
│  Component  │ ◄────────── React Context
│  (calls     │                      │
│   setState) │                      │
└──────┬──────┘                      │
       │                            │
       ▼                            │
┌─────────────┐                     │
│ AppState    │─────────────────────┤
│ Store       │                     │
└──────┬──────┘                     │
       │                            │
       ▼                            │
┌─────────────┐                     │
│  Subscribed │─────────────────────┘
│  Components │
└─────────────┘
```

## 关键设计模式

### 1. React Context + useSyncExternalStore

项目使用自定义的 AppStateStore，结合 React Context 和 useSyncExternalStore：

```typescript
// state/AppStateStore.ts
export function createStore(initialState: AppState, onChange?: Callback) {
  // 返回符合 React 状态的 store
}

export const AppStoreContext = React.createContext<AppStateStore | null>(null);
```

### 2. 工具模式 (Tool Pattern)

所有工具实现统一的 Tool 接口：

```typescript
// Tool.ts
interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  execute(input: unknown, context: ToolContext): Promise<ToolResult>
}
```

### 3. 命令模式 (Command Pattern)

每个命令独立目录/文件，通过统一接口注册：

```
commands/
├── commit.ts           # commit 命令
├── commit/
│   ├── index.ts        # 命令入口
│   └── commit.ts       # 具体实现
```

### 4. 桥接模式 (Bridge Pattern)

IDE 桥接使用独立的通信协议：

```
bridge/
├── bridgeMain.ts       # 主逻辑
├── bridgeMessaging.ts  # 消息格式
├── bridgeApi.ts        # API 定义
└── types.ts            # 类型定义
```

## 模块间依赖

```
注意: 存在循环依赖警告

main.tsx
    │
    ├──► state/AppState.tsx
    │         │
    │         └─► (使用 context.ts 的类型)
    │
    ├──► context.ts
    │         │
    │         └─► (被多个模块引用)
    │
    └──► query.ts
              │
              └─► 使用 Tool.ts 的类型
```

---

下一章：[核心概念](./03-core-concepts.md)
