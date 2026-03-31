# 命令系统

## 概述

命令是用户在 CLI 输入的 slash 命令（如 `/commit`、`/branch`）。项目有 100+ 命令。

## 目录结构

```
commands/
├── commit.ts                 # git commit
├── commit-push-pr.ts         # commit + push + PR
├── branch/
│   ├── index.ts              # 分支命令入口
│   └── branch.ts             # 分支操作
├── add-dir/                  # 添加目录
├── advisor.ts                # Advisor 设置
├── agents/                   # Agent 管理
├── btw/                      # 插入注释
├── clear/                    # 清除缓存/对话
├── compact/                  # 对话压缩
├── config/                   # 配置管理
├── context/                  # 上下文管理
├── copy/                     # 复制到剪贴板
├── cost/                     # 成本查看
├── diff/                     # Git diff
├── doctor/                   # 诊断命令
├── effort/                   # 难度设置
├── exit/                     # 退出
├── export/                   # 导出对话
├── fast/                     # 快速模式
├── files/                    # 文件操作
├── feedback/                 # 反馈
├── heapdump/                 # 堆转储
├── help/                     # 帮助
├── hooks/                    # Hooks 管理
├── mcp/                      # MCP 服务器
├── model/                    # 模型选择
├── notif/                    # 通知
├── patch/                    # 应用补丁
├── resume/                   # 恢复会话
├── scratch/                  # 草稿板
├── search/                   # 搜索
├── session/                  # 会话管理
├── skills/                   # 技能管理
├── task/                     # 任务管理
├── team/                     # 团队管理
├── telescope/                # 调试
├── test/                     # 测试运行
├── theme/                    # 主题设置
├── undo/                     # 撤销
├── vim/                      # Vim 模式
├── vscode/                   # VSCode 集成
├── weave/                    # Weave 模式
└── workflow/                 # 工作流
```

## 命令结构

每个命令有两种组织方式：

### 单文件模式

```typescript
// commands/brief.ts
export const briefCommand = {
  name: 'brief',
  description: 'Generate a brief summary',

  async handler(args: string[], context: CommandContext) {
    // 实现
  }
};
```

### 目录模式（推荐）

```
commands/commit/
├── index.ts      # 命令注册
├── commit.ts     # 实现
└── validation.ts # 验证逻辑
```

```typescript
// commands/commit/index.ts
import { commit } from './commit.js';

export const commitCommand = {
  name: 'commit',
  description: 'Create a git commit',
  options: [
    { name: '-m, --message <msg>', description: 'Commit message' },
    { name: '-a, --all', description: 'Stage all changes' }
  ],

  async handler(args, context) {
    return commit(args, context);
  }
};
```

## 命令注册

命令通过 `commands.ts` 统一注册：

```typescript
// commands.ts
export function getCommands(): Command[] {
  return [
    // 动态导入所有命令
    ...(await import('./commands/commit/index.js')).default,
    ...(await import('./commands/branch/index.js')).default,
    // ...
  ];
}

// 注册到 Commander.js
for (const cmd of getCommands()) {
  program.command(cmd.name)
    .description(cmd.description)
    .action(cmd.handler);
}
```

## 命令执行流程

```
用户输入 /commit -m "fix bug"
        │
        ▼
┌─────────────────┐
│  命令解析       │
│ (Commander.js)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  参数验证       │
│ (yargs/zod)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  命令执行       │
│ (handler)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  结果展示       │
│ (格式化输出)    │
└─────────────────┘
```

## Slash Command 处理

Slash command 在 REPL 中处理：

```typescript
// hooks/useCommandKeybindings.tsx
function parseSlashCommand(input: string): {
  command: string;
  args: string[];
} | null {
  if (!input.startsWith('/')) return null;

  const parts = input.slice(1).split(/\s+/);
  return {
    command: parts[0],
    args: parts.slice(1)
  };
}
```

## 常用命令详解

### /commit - Git 提交

```typescript
// commands/commit/commit.ts
export async function commit(args, context) {
  const { message, all } = parseArgs(args);

  // 1. 验证 git 状态
  const status = await gitStatus();

  // 2. 暂存文件（如果指定 -a）
  if (all) {
    await gitAdd(['-A']);
  }

  // 3. 执行 commit
  const result = await gitCommit(['-m', message]);

  // 4. 返回结果
  return formatCommitResult(result);
}
```

### /branch - 分支管理

```typescript
// commands/branch/branch.ts
export const branchCommands = [
  {
    name: 'list',
    description: 'List all branches',
    handler: listBranches
  },
  {
    name: 'create',
    description: 'Create a new branch',
    handler: createBranch
  },
  {
    name: 'delete',
    description: 'Delete a branch',
    handler: deleteBranch
  },
  {
    name: 'checkout',
    description: 'Switch branches',
    handler: checkoutBranch
  }
];
```

### /compact - 对话压缩

压缩历史消息以节省 token：

```typescript
// commands/compact/compact.ts
export async function compact(context) {
  // 1. 获取当前对话历史
  const messages = await getConversationHistory();

  // 2. 生成压缩摘要
  const summary = await summarize(messages);

  // 3. 替换历史为摘要
  await replaceHistoryWithSummary(messages, summary);

  // 4. 显示压缩结果
  return {
    original: messages.length,
    after: 1,
    tokens: estimateTokenSavings(messages, summary)
  };
}
```

## 命令选项定义

```typescript
interface CommandOption {
  flags: string           // '-m, --message <msg>'
  description: string     // 'Commit message'
  defaultValue?: string
}

const commitOptions: CommandOption[] = [
  {
    flags: '-m, --message <msg>',
    description: 'Commit message',
    required: true
  },
  {
    flags: '-a, --all',
    description: 'Stage all modified files'
  },
  {
    flags: '--amend',
    description: 'Amend to previous commit'
  }
];
```

## 命令上下文

命令执行时接收的上下文：

```typescript
interface CommandContext {
  cwd: string                          // 当前工作目录
  sessionId: string                    // 会话 ID
  gitRoot?: string                     // Git 仓库根目录
  config: GlobalConfig                 // 全局配置
  ui: {
    prompt: (msg: string) => Promise<string>
    confirm: (msg: string) => Promise<boolean>
    showMessage: (msg: string) => void
  }
}
```

## 帮助系统

```typescript
// 自动生成帮助
export function generateHelp(command: Command): string {
  return `
Usage: /${command.name} ${command.options.map(o => o.flags).join(' ')}

${command.description}

Options:
${command.options.map(o => `  ${o.flags.padEnd(20)} ${o.description}`).join('\n')}
  `.trim();
}
```

---

下一章：[状态管理](./08-state-management.md)
