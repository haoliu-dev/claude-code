# 状态管理

## 概述

项目使用 React Context + useSyncExternalStore 实现状态管理，主要状态存储在 `state/` 目录。

## 目录结构

```
state/
├── AppState.tsx           # AppState 主类型定义
├── AppStateStore.ts       # 状态存储实现
├── store.ts               # Store 创建
├── onChangeAppState.ts    # 状态变更处理
└── selectors.ts           # 选择器函数
```

## AppState 结构

`AppState` 是应用的核心状态：

```typescript
// state/AppState.tsx
interface AppState {
  // 会话状态
  session: {
    id: string
    messages: Message[]
    model: string
    temperature: number
  }

  // UI 状态
  ui: {
    theme: ThemeName
    expandedSections: Set<string>
    activeDialog: string | null
    isLoading: boolean
  }

  // 权限状态
  permissions: {
    mode: PermissionMode
    toolPermissions: Record<string, PermissionResult>
    deniedTools: string[]
  }

  // 设置
  settings: {
    model: string
    maxTokens: number
    temperature: number
    // ...
  }

  // 工具状态
  tools: {
    available: Tool[]
    running: Map<string, ToolExecution>
    results: Record<string, ToolResult>
  }

  // MCP 状态
  mcp: {
    servers: MCPServerConnection[]
    tools: Record<string, MCPTool>
  }

  // Agent 状态
  agents: {
    active: Agent[]
    definitions: AgentDefinition[]
  }
}
```

## AppStateStore

状态存储使用自定义实现：

```typescript
// state/AppStateStore.ts
export interface AppStateStore {
  getState(): AppState
  setState(partial: Partial<AppState>): void
  subscribe(callback: () => void): () => void
  getSnapshot(): AppState
}

export function createStore(
  initialState: AppState,
  onChange?: (newState: AppState, oldState: AppState) => void
): AppStateStore {
  // 返回符合 useSyncExternalStore 契约的 store
}
```

## Provider 模式

```typescript
// state/AppState.tsx
export function AppStateProvider({
  children,
  initialState,
  onChangeAppState
}: Props) {
  const store = useMemo(
    () => createStore(initialState, onChangeAppState),
    [initialState]
  );

  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  );
}

// 使用
const store = useContext(AppStoreContext);
const state = useSyncExternalStore(
  store.subscribe,
  store.getSnapshot,
  store.getSnapshot // 服务端 snapshot
);
```

## 状态更新模式

### 直接更新

```typescript
// 设置单个字段
store.setState({ ui: { ...state.ui, isLoading: true } });
```

### 函数式更新

```typescript
// 基于当前状态更新
store.setState(prev => ({
  ui: { ...prev.ui, isLoading: false }
}));
```

### 批量更新

```typescript
// 多次 setState 会被合并
store.setState({ a: 1 });
store.setState({ b: 2 });
// 实际只触发一次重新渲染
```

## Context 体系

项目使用多个 Context：

| Context | 用途 |
|---------|------|
| AppStoreContext | 全局状态 |
| ThemeContext | 主题信息 |
| PermissionContext | 权限状态 |
| ToolContext | 工具执行上下文 |
| MailboxContext | 消息邮箱 |
| VoiceContext | 语音模式 |

### MailboxContext

用于组件间通信：

```typescript
// context/mailbox.ts
interface MailboxMessage {
  type: string
  payload: unknown
  sender: string
}

interface MailboxContextValue {
  postMessage(msg: MailboxMessage): void
  subscribe(handler: (msg: MailboxMessage) => void): () => void
}
```

## 选择器模式

避免不必要的重渲染：

```typescript
// state/selectors.ts

// 基础选择器
const selectTheme = (state: AppState) => state.ui.theme;

// 派生选择器
const selectIsLoading = (state: AppState) => state.ui.isLoading;

// 使用
function MyComponent() {
  // 只在 theme 变化时重渲染
  const theme = useSyncExternalStore(
    store.subscribe,
    () => selectTheme(store.getState())
  );

  // 只在 isLoading 变化时重渲染
  const isLoading = useSyncExternalStore(
    store.subscribe,
    () => selectIsLoading(store.getState())
  );
}
```

## 状态变更处理

```typescript
// state/onChangeAppState.ts
export function onChangeAppState(
  newState: AppState,
  oldState: AppState
): void {
  // 主题变更
  if (newState.ui.theme !== oldState.ui.theme) {
    applyTheme(newState.ui.theme);
  }

  // 权限变更
  if (newState.permissions.mode !== oldState.permissions.mode) {
    notifyPermissionChange(newState.permissions.mode);
  }

  // 设置变更
  if (newState.settings !== oldState.settings) {
    saveSettingsToStorage(newState.settings);
  }
}
```

## 持久化

状态可以持久化到本地存储：

```typescript
// 会话恢复
export async function loadSession(sessionId: string): Promise<AppState> {
  const stored = await storage.get(`session:${sessionId}`);
  return stored ? JSON.parse(stored) : getDefaultAppState();
}

// 会话保存
export async function saveSession(state: AppState): Promise<void> {
  await storage.set(`session:${state.session.id}`, JSON.stringify(state));
}
```

## 状态初始化流程

```
应用启动
    │
    ▼
┌─────────────────┐
│ getDefaultAppState() │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 加载持久化状态  │
│ (如果有)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 合并配置        │
│ (全局配置覆盖)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ AppStateProvider │
│ 注入初始状态    │
└─────────────────┘
```

---

下一章：[IDE 桥接系统](./09-bridge-system.md)
