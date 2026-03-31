# 工具系统

## 概述

工具是 Agent 与外部世界交互的唯一接口。项目实现了 40+ 工具，分布在 `tools/` 目录。

## 目录结构

```
tools/
├── BashTool/                 # Bash 命令执行
├── PowerShellTool/          # PowerShell 执行
├── FileReadTool/            # 读取文件
├── FileEditTool/            # 编辑文件
├── FileWriteTool/           # 写入文件
├── GlobTool/                # 文件模式匹配
├── GrepTool/                # 代码搜索
├── AgentTool/               # 创建子 Agent
├── TaskCreateTool/          # 创建任务
├── TaskListTool/            # 列出任务
├── TaskUpdateTool/          # 更新任务
├── MCPTool/                 # MCP 工具调用
├── REPLTool/                # 交互式解释器
├── WebFetchTool/            # HTTP 请求
├── WebSearchTool/           # 网页搜索
├── NotebookEditTool/        # Jupyter 笔记编辑
├── LSPTool/                 # 语言服务器协议
├── EnterPlanModeTool/       # 进入计划模式
├── ExitPlanModeTool/        # 退出计划模式
├── SkillTool/               # 技能调用
├── SendMessageTool/         # 发送消息
├── TeamCreateTool/          # 创建团队
├── ExitWorktreeTool/       # 退出 worktree
├── EnterWorktreeTool/      # 进入 worktree
└── utils.ts                 # 工具基类
```

## 工具基类

所有工具继承自统一接口：

```typescript
// tools/utils.ts
export interface Tool {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema

  // 核心方法
  execute(
    input: unknown,
    context: ToolContext
  ): Promise<ToolResult>
}

export interface ToolContext {
  abortSignal: AbortSignal
  sessionId: string
  cwd: string
  env: Record<string, string>
  // ... 其他上下文
}
```

## 工具分类

### 1. 文件操作工具

| 工具 | 用途 | 关键参数 |
|------|------|----------|
| FileReadTool | 读取文件 | `filePath`, `offset`, `limit` |
| FileEditTool | 编辑文件 | `filePath`, `oldString`, `newString` |
| FileWriteTool | 写入文件 | `filePath`, `content` |
| GlobTool | 模式匹配 | `pattern`, `cwd` |
| GrepTool | 搜索内容 | `pattern`, `path`, `glob` |

### 2. 命令执行工具

| 工具 | 用途 | 特点 |
|------|------|------|
| BashTool | 执行 Bash | 超时控制、working directory |
| PowerShellTool | 执行 PowerShell | Windows 兼容 |
| REPLTool | 交互式解释器 | 保持状态 |

### 3. Agent 管理工具

| 工具 | 用途 |
|------|------|
| AgentTool | 创建子 Agent |
| SkillTool | 调用技能 |

### 4. 任务管理工具

| 工具 | 用途 |
|------|------|
| TaskCreateTool | 创建任务 |
| TaskListTool | 列出任务 |
| TaskUpdateTool | 更新任务状态 |
| TaskOutputTool | 获取任务输出 |
| TaskStopTool | 停止任务 |

### 5. MCP 工具

| 工具 | 用途 |
|------|------|
| MCPTool | 调用 MCP 服务器工具 |
| McpAuthTool | MCP 认证 |
| ListMcpResourcesTool | 列出 MCP 资源 |

### 6. 其他工具

| 工具 | 用途 |
|------|------|
| WebFetchTool | HTTP GET/POST |
| WebSearchTool | 网页搜索 |
| NotebookEditTool | Jupyter 编辑 |
| LSPTool | LSP 请求 |

## 工具执行流程

```
QueryEngine 决定调用工具
        │
        ▼
┌─────────────────┐
│  权限检查       │
│ (useCanUseTool) │
└────────┬────────┘
         │拒绝
         ▼
    显示权限请求
         │
        允许
         ▼
┌─────────────────┐
│  参数验证       │
│ (schema check)  │
└────────┬────────┘
         │无效
         ▼
    返回错误
         │
        有效
         ▼
┌─────────────────┐
│  工具执行       │
│ (execute)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  结果格式化     │
│ (ToolResult)    │
└────────┬────────┘
         │
         ▼
   返回给 Agent
```

## BashTool 详解

BashTool 是最复杂的工具之一。

### 执行流程

```typescript
// tools/BashTool/index.ts
class BashTool implements Tool {
  name = 'Bash'
  description = 'Execute bash commands'

  async execute(input: BashInput, context: ToolContext): Promise<ToolResult> {
    // 1. 构建命令
    const { command, timeout, cwd } = input;

    // 2. 创建子进程
    const child = spawn('bash', ['-c', command], {
      cwd: cwd ?? context.cwd,
      env: context.env,
      signal: context.abortSignal
    });

    // 3. 收集输出
    const { stdout, stderr } = await collectOutput(child, timeout);

    // 4. 返回结果
    return {
      content: stdout,
      error: stderr ? formatError(stderr) : undefined
    };
  }
}
```

### 安全机制

| 机制 | 说明 |
|------|------|
| 权限检查 | 工具需要用户授权才能执行 |
| 路径限制 | 只能在允许的目录下执行 |
| 超时控制 | 防止命令无限运行 |
| 危险命令警告 | rm -rf 等命令需要额外确认 |

## FileEditTool 详解

文件编辑工具支持智能编辑。

### Diff 生成

```typescript
interface FileEditInput {
  filePath: string
  oldString: string      // 要替换的内容
  newString: string      // 新内容
  algorithm?: 'diff' | 'screenshot'  // diff 算法
}
```

### 寻址机制

工具支持模糊匹配：

```typescript
// 1. 精确匹配
oldString = 'const x = 1;'

// 2. 多行匹配
oldString = `function foo() {
  return 1;
}`

// 3. 带结构的匹配
oldString = `class Foo {
  // ... 其他方法
}`
```

## AgentTool 详解

AgentTool 允许创建子 Agent。

### Agent 类型

```typescript
interface AgentConfig {
  name: string
  model?: string
  systemPrompt?: string
  tools?: string[]      // 允许的工具列表
  maxTokens?: number
}
```

### 通信协议

```typescript
// 发送到 Agent
type AgentMessage =
  | { type: 'user_message'; content: string }
  | { type: 'tool_result'; tool: string; result: ToolResult }
  | { type: 'stop' }

// 接收自 Agent
type AgentResponse =
  | { type: 'tool_call'; name: string; input: unknown }
  | { type: 'text'; content: string }
  | { type: 'done' }
```

## 工具注册

工具通过 `tools.ts` 统一注册：

```typescript
// tools.ts
export function getTools(): Tool[] {
  return [
    new BashTool(),
    new FileReadTool(),
    new FileEditTool(),
    // ... 其他工具
  ];
}
```

## 工具发现

工具可以被动态发现和加载：

```typescript
// 从目录加载工具
export function loadToolsFromDir(dir: string): Tool[] {
  const files = readdirSync(dir);
  return files
    .filter(f => f.endsWith('Tool/index.ts'))
    .map(f => import(f).then(m => new m.default()));
}
```

---

下一章：[命令系统](./07-commands-system.md)
