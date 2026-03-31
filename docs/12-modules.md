# 其他重要模块

## 概述

本章节涵盖不属于前面类别的其他重要模块。

## Buddy 宠物系统

### 目录结构

```
buddy/
├── companion.ts        # 宠物实体
├── prompt.ts           # 宠物提示词
├── sprites.ts          # ASCII 艺术
├── types.ts            # 类型定义
```

### 宠物系统架构

```typescript
// buddy/types.ts
interface Buddy {
  id: string
  species: BuddySpecies
  nickname: string
  level: number
  exp: number
  stats: BuddyStats
  personality: Personality
  isShiny: boolean
}

interface BuddySpecies {
  id: string
  name: string
  rarity: 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'
  baseStats: BuddyStats
  sprite: string[]      // ASCII art
}

interface Personality {
  traits: string[]
  mood: 'happy' | 'neutral' | 'sad'
}
```

### Gacha 系统

```typescript
// buddy/companion.ts
export function hatchBuddy(userId: string): Buddy {
  // 使用 Mulberry32 PRNG（确定性）
  const seed = hashUserId(userId, 'friend-2026-401');
  const rng = mulberry32(seed);

  // 1. 抽稀有度
  const rarity = rollRarity(rng);

  // 2. 抽物种
  const species = selectSpecies(rarity, rng);

  // 3. 抽闪光（1% 概率）
  const isShiny = rng() < 0.01;

  // 4. 生成性格
  const personality = generatePersonality(rng);

  return {
    id: generateBuddyId(),
    species,
    personality,
    isShiny,
    level: 1,
    exp: 0
  };
}
```

## MCP (Model Context Protocol)

### 目录结构

```
services/mcp/
├── client.ts            # MCP 客户端
├── server.ts            # MCP 服务器
├── types.ts             # 类型定义
├── config.ts            # 配置管理
├── officialRegistry.ts  # 官方服务器注册表
├── utils.ts             # 工具函数
├── claudeai.ts          # Claude AI MCP 集成
├── xaaIdpLogin.ts       # XAA 认证
```

### MCP 工具调用

```typescript
// services/mcp/client.ts
export async function callMcpTool(
  serverId: string,
  toolName: string,
  args: Record<string, unknown>
): Promise<ToolResult> {
  const connection = mcpConnections.get(serverId);
  if (!connection) throw new Error(`MCP server ${serverId} not connected`);

  return connection.transport.request({
    method: 'tools/call',
    params: { name: toolName, arguments: args }
  });
}
```

## Hooks

### 目录结构

```
hooks/
├── useCanUseTool.tsx           # 工具权限检查
├── useCommandKeybindings.tsx  # 命令快捷键
├── useDiffInIDE.ts            # IDE Diff
├── useSettingsChange.ts       # 设置变更
├── useApiKeyVerification.ts   # API 密钥验证
├── useAwaySummary.ts          # 离开摘要
├── useClipboardImageHint.ts   # 剪贴板图片
├── useClaudeCodeHintRecommendation.tsx  # 提示推荐
```

### 常用 Hook

#### useCanUseTool

检查工具是否可用：

```typescript
// hooks/useCanUseTool.tsx
export function useCanUseTool(toolName: string): {
  canUse: boolean
  prompt: () => Promise<boolean>
  status: PermissionStatus
} {
  const { permissions } = useAppState();

  const toolPermission = permissions.toolPermissions[toolName];

  return {
    canUse: toolPermission?.result === 'granted',
    prompt: () => requestPermission(toolName),
    status: toolPermission?.status ?? 'unknown'
  };
}
```

#### useCommandKeybindings

处理命令快捷键：

```typescript
// hooks/useCommandKeybindings.tsx
export function useCommandKeybindings() {
  useInput((key, next) => {
    // Ctrl+C - 取消
    if (key.ctrl && key.key === 'c') {
      handleCancel();
      return;
    }

    // Ctrl+L - 清屏
    if (key.ctrl && key.key === 'l') {
      clearScreen();
      return;
    }

    next();
  });
}
```

## Vim 模式

### 目录结构

```
vim/
├── vim.ts              # Vim 引擎
├── motions.ts          #  motions（w, b, $, ^）
├── operators.ts        # 操作符（d, c, y）
├── textObjects.ts      # 文本对象（iw, aw）
└── modes.ts            # 模式（normal, insert, visual）
```

### Vim 引擎

```typescript
// vim/vim.ts
export class VimEngine {
  private mode: VimMode = 'normal'
  private register: string = ''
  private count: number = 1

  // motions
  motions = {
    'w': () => moveWordForward(),
    'b': () => moveWordBackward(),
    '$': () => moveToLineEnd(),
    '^': () => moveToLineStart(),
    '0': () => moveToColumn(0)
  }

  // operators
  operators = {
    'd': () => delete(),
    'c': () => change(),
    'y': () => yank(),
    'p': () => paste()
  }

  handleKey(key: string): void {
    if (this.mode === 'insert') {
      if (key === 'Escape') {
        this.mode = 'normal';
      } else {
        this.insertCharacter(key);
      }
      return;
    }

    if (this.mode === 'normal') {
      this.handleNormalMode(key);
    }
  }
}
```

## 迁移系统

### 目录结构

```
migrations/
├── migrate.ts          # 迁移主逻辑
├── 001_initial/         # 迁移 1
├── 002_settings/       # 迁移 2
└── runner.ts           # 迁移运行器
```

### 迁移运行器

```typescript
// migrations/runner.ts
export async function runMigrations(): Promise<void> {
  const currentVersion = await getCurrentVersion();
  const migrations = await loadMigrations();

  for (const migration of migrations) {
    if (migration.id > currentVersion) {
      console.log(`Running migration ${migration.id}...`);

      await migration.up();

      await setVersion(migration.id);
    }
  }
}
```

## 插件系统

### 目录结构

```
plugins/
├── bundled/            # 内置插件
│   └── index.ts
├── installed/          # 已安装插件
├── managed/            # 托管插件
├── cacheUtils.ts       # 缓存工具
├── installUtils.ts     # 安装工具
├── pluginLoader.ts     # 插件加载器
└── pluginTypes.ts      # 插件类型
```

### 插件接口

```typescript
// plugins/pluginTypes.ts
interface Plugin {
  id: string
  name: string
  version: string
  description?: string

  // 生命周期
  onLoad(context: PluginContext): Promise<void>
  onUnload(): Promise<void>

  // 扩展点
  tools?: Tool[]
  commands?: Command[]
  hooks?: Hook[]
  components?: { name: string; component: React.ComponentType }[]
  themes?: Theme[]
}

interface PluginContext {
  config: PluginConfig
  logger: Logger
  registerTool(tool: Tool): void
  registerCommand(command: Command): void
  registerHook(hook: Hook): void
}
```

## 任务系统

### 目录结构

```
tasks/
├── taskTypes.ts        # 任务类型
├── taskStore.ts        # 任务存储
├── taskCommands.ts     # 任务命令
└── taskRenderer.tsx    # 任务渲染
```

### 任务存储

```typescript
// tasks/taskStore.ts
export class TaskStore {
  private tasks: Map<string, Task> = new Map();

  create(task: TaskInput): Task {
    const id = generateTaskId();
    const task: Task = {
      ...task,
      id,
      status: 'pending',
      createdAt: Date.now()
    };

    this.tasks.set(id, task);
    this.persist();

    return task;
  }

  update(id: string, updates: Partial<Task>): Task {
    const task = this.tasks.get(id);
    if (!task) throw new Error(`Task ${id} not found`);

    const updated = { ...task, ...updates };
    this.tasks.set(id, updated);
    this.persist();

    return updated;
  }

  get(id: string): Task | undefined {
    return this.tasks.get(id);
  }

  list(filter?: TaskFilter): Task[] {
    return Array.from(this.tasks.values()).filter(task =>
      filter ? matchesFilter(task, filter) : true
    );
  }
}
```

## 会话存储

### 目录结构

```
utils/
├── sessionStorage.ts   # 会话存储
├── sessionHistory.ts   # 会话历史
└── conversationRecovery.ts  # 对话恢复
```

### 会话存储

```typescript
// utils/sessionStorage.ts
export async function saveSession(
  sessionId: string,
  messages: Message[]
): Promise<void> {
  const path = getSessionPath(sessionId);

  await writeFile(path, JSON.stringify({
    version: 1,
    sessionId,
    messages,
    savedAt: Date.now()
  }));
}

export async function loadSession(
  sessionId: string
): Promise<SessionData | null> {
  const path = getSessionPath(sessionId);

  if (!exists(path)) return null;

  const content = await readFile(path);
  return JSON.parse(content);
}
```

## 工具函数

### 常用工具模块

| 模块 | 用途 |
|------|------|
| `utils/log.ts` | 日志记录 |
| `utils/config.ts` | 配置管理 |
| `utils/git.ts` | Git 操作 |
| `utils/shell.ts` | Shell 执行 |
| `utils/auth.ts` | 认证 |
| `utils/model/` | 模型相关 |
| `utils/permissions/` | 权限相关 |
| `utils/plugins/` | 插件相关 |
| `utils/skills/` | 技能相关 |

---

文档完成。
