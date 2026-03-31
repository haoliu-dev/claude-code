# 核心概念

## Tool - 工具

Tool 是 Agent 与外部世界交互的接口。每个工具执行特定操作（读文件、执行命令等）。

### 工具基类

```typescript
// Tool.ts
export interface Tool {
  name: string              // 工具名称
  description: string       // 描述（供 LLM 理解）
  inputSchema: ToolInputJSONSchema  // 输入参数 schema

  // 执行入口
  execute(
    input: unknown,
    context: ToolContext
  ): Promise<ToolResult>
}
```

### 工具分类

| 分类 | 工具 | 用途 |
|------|------|------|
| 文件操作 | FileReadTool, FileEditTool, FileWriteTool | 读写编辑文件 |
| 代码搜索 | GrepTool, GlobTool | 搜索代码 |
| 命令执行 | BashTool, PowerShellTool | 执行 shell 命令 |
| 子 Agent | AgentTool | 创建和管理子 Agent |
| 任务管理 | TaskCreateTool, TaskListTool, TaskUpdateTool | 任务系统 |
| MCP | MCPTool, McpAuthTool | 调用 MCP 服务器 |
| REPL | REPLTool | 交互式解释器 |
| 其他 | WebFetchTool, WebSearchTool, NotebookEditTool | 杂项功能 |

### 工具执行流程

```
QueryEngine 调用工具
        │
        ▼
┌─────────────────┐
│ Tool.execute()  │
│   │             │
│   ▼             │
│ 权限检查        │
│   │             │
│ ▼             │
│ 参数验证       │
│   │             │
│ ▼             │
│ 执行操作       │
│   │             │
│ ▼             │
│ 结果格式化     │
└─────────────────┘
        │
        ▼
    ToolResult
```

### 工具权限模型

工具执行需要权限检查：

```typescript
// 权限模式
enum PermissionMode {
  Auto = 'auto',        // 自动批准
  Ask = 'ask',          // 每次询问
  Bypass = 'bypass'     // 绕过（需特殊条件）
}
```

## Command - 命令

Command 是用户在 CLI 输入的 slash 命令（如 `/commit`、`/branch`）。

### 命令结构

```
commands/
├── commit.ts           # 命令定义
├── commit/
│   ├── index.ts        # 注册入口
│   └── commit.ts       # 实现
```

### 命令注册

```typescript
// commands/commit/index.ts
export const commitCommand = {
  name: 'commit',
  description: 'Create a git commit',

  async handler(args, context) {
    // 执行逻辑
  }
}
```

### Slash Command 处理

当用户输入 `/commit` 时：

1. `commands.ts` 解析输入
2. 匹配到 `commit` 命令
3. 调用 `commitCommand.handler(args, context)`
4. 返回结果并显示

## Agent - 代理

Agent 是能够自主执行任务的子进程。

### Agent 类型

```typescript
// 内置 Agent
enum AgentType {
  Sum = 'sum',           // 求和 Agent
  Critic = 'critic',     // 审查 Agent
  SpecWriter = 'spec-writer'  // 规范编写
}

// 自定义 Agent（从 agents/ 目录加载）
interface AgentDefinition {
  name: string
  description: string
  systemPrompt: string
  tools: string[]
}
```

### Agent 创建流程

```
AgentTool.execute()
        │
        ▼
┌─────────────────┐
│ 解析 Agent 配置 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 创建子进程      │
│ (spawn Agent)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 消息通信       │
│ (stdio/json)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 合并结果       │
└─────────────────┘
```

### Agent 通信协议

Agent 使用 JSON over stdio 通信：

```typescript
// 发送给 Agent
{ type: 'user_message', content: '...' }

// 接收自 Agent
{ type: 'tool_call', name: 'Bash', input: {...} }
{ type: 'result', content: '...' }
```

## Task - 任务

Task 是需要长期跟踪的工作单元。

### Task 状态机

```
┌────────┐     create      ┌──────────┐
│ none   │ ──────────────► │ pending  │
└────────┘                 └────┬─────┘
                                 │
                     start       │
                      ┌──────────┴──────────┐
                      ▼                     │
               ┌──────────┐                 │
               │ in_progress              │
               └────┬─────┘                 │
                    │                       │
        ┌───────────┼───────────┐           │
        │ complete  │ skip      │ cancel   │
        ▼           ▼           ▼           │
   ┌─────────┐ ┌────────┐ ┌──────────┐    │
   │completed│ │ skipped │ │ cancelled│    │
   └─────────┘ └────────┘ └──────────┘    │
                                           │
                        block              │
                      ┌────────────────────┘
                      ▼
                 ┌─────────┐
                 │ blocked │
                 └─────────┘
```

### Task 属性

```typescript
interface Task {
  id: string
  subject: string           // 任务描述
  status: TaskStatus        // 状态
  parentId?: string          // 父任务
  dependsOn?: string[]       // 依赖任务
  effort?: 'small' | 'medium' | 'large'
}
```

## 概念间关系

```
┌─────────────────────────────────────────────────────────────┐
│                      User (Terminal)                        │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Command                                │
│  (用户输入 /commit，触发命令执行)                              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      QueryEngine                            │
│  (处理用户消息，决定调用哪些 Tool)                              │
└────────────────────────────┬────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Tool     │     │   Agent     │     │   Task      │
│  (执行操作)  │     │ (创建子进程) │     │ (跟踪状态)   │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 调用链示例

用户输入：`/commit -m "fix bug"`

```
Command (/commit)
    │
    ▼
Tool (可能调用 Git)
    │
    ▼
Agent (可选，用于复杂逻辑)
    │
    ▼
Task (可选，用于跟踪)
```

---

下一章：[入口流程](./04-entrypoint-flow.md)
