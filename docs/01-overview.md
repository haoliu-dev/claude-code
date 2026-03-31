# Claude Code 项目概览

## 项目背景

Claude Code 是 Anthropic 官方推出的 AI 编程 CLI 工具。本代码库是其完整源代码，于 2026 年 3 月通过 npm 包中的 source map 文件意外泄露。

> **源码来源**: https://github.com/kuberwastaken/claude-code

## 技术栈

| 层级 | 技术 | 作用 |
|------|------|------|
| 运行时 | Bun | JavaScript 运行时 + bundler |
| 语言 | TypeScript | 类型安全 |
| CLI 渲染 | Ink | React for terminals |
| UI 组件 | React | 组件化开发 |
| 状态管理 | React Context + useSyncExternalStore | 应用状态 |
| API 调用 | @anthropic-ai/sdk | Claude API |
| MCP | @modelcontextprotocol/sdk | Model Context Protocol |

### 关键技术选型说明

**为什么用 Ink 而不是原生 Node.js readline？**
- Ink 是专为 CLI 设计的 React 渲染引擎，支持类 React 的组件化开发
- 保持与前端一致的开发体验（组件、props、hooks）
- 内置 terminal UI 组件（Box、Text、Spinner 等）

**为什么用 Bun 而不是 Node.js？**
- 更快的启动速度和打包速度
- 内置 TypeScript 支持
- `feature()` 函数支持编译时特性开关（DCE）

## 目录结构

```
claude-code/
├── main.tsx                    # 应用入口 (~785KB)
├── ink.ts                       # Ink 渲染封装
├── ink/                         # Ink 渲染引擎核心
│   ├── ink.tsx                  # 主渲染逻辑 (~250KB)
│   ├── screen.ts                # 屏幕管理
│   ├── render-node-to-output.ts # 渲染输出
│   ├── components/              # Ink 基础组件
│   └── hooks/                   # Ink 内置 hooks
├── components/                   # 业务组件 (~146个)
│   ├── App.tsx                  # 主应用组件
│   ├── Messages.tsx             # 消息列表
│   ├── MessageRow.tsx           # 单条消息
│   ├── PromptInput/             # 输入框组件
│   ├── design-system/           # 设计系统
│   └── ...
├── tools/                       # 工具系统 (~40+工具)
│   ├── BashTool/                # Bash 执行
│   ├── FileEditTool/            # 文件编辑
│   ├── AgentTool/               # Agent 创建
│   └── ...
├── commands/                    # CLI 命令 (~100+)
│   ├── commit.ts                # Git commit
│   ├── branch/                  # 分支管理
│   └── ...
├── services/                    # 服务层
│   ├── api/                     # API 调用
│   ├── mcp/                     # MCP 协议
│   └── analytics/               # 分析服务
├── state/                       # 状态管理
│   ├── AppState.tsx             # 主应用状态
│   └── AppStateStore.ts         # 状态存储
├── bridge/                      # IDE 桥接
│   ├── bridgeMain.ts            # 桥接主逻辑
│   └── ...
├── hooks/                       # React Hooks (~50个)
├── utils/                       # 工具函数 (~330个)
├── entrypoints/                 # 入口点
│   ├── cli.tsx                 # CLI 入口
│   └── init.ts                 # 初始化逻辑
├── context.ts                  # 全局上下文
├── query.ts                    # 查询引擎
├── Tool.ts                     # 工具基类
├── Task.ts                     # 任务模型
├── buddy/                      # 宠物系统 (Tamagotchi)
├── cli/                        # CLI 传输层
├── migrations/                 # 数据迁移
├── plugins/                    # 插件系统
├── schemas/                    # 数据 schema
├── screens/                    # 全屏界面
├── server/                     # 服务端
├── skills/                     # 技能系统
├── tasks/                      # 任务系统
└── types/                      # 类型定义
```

## 核心模块关系图

```
┌─────────────────────────────────────────────────────────────┐
│                         main.tsx                           │
│                    (入口，协调各模块)                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  entrypoints/ │ │   commands/   │ │    tools/    │
│    cli.tsx    │ │   (~100+)    │ │    (~40+)    │
└───────┬───────┘ └───────────────┘ └───────────────┘
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│     ink/      │ │   components/ │ │     hooks/    │
│   渲染引擎    │ │   (~146个)    │ │    (~50个)    │
└───────────────┘ └───────────────┘ └───────────────┘
        │                 │                 │
        └────────────┬────┴─────────────────┘
                     ▼
              ┌─────────────┐
              │  state/     │
              │ AppState    │
              └─────────────┘
```

## 快速定位指南

| 想找... | 去这里 |
|--------|--------|
| 应用入口 | `main.tsx` |
| CLI 命令定义 | `commands/` |
| 工具实现 | `tools/` |
| UI 组件 | `components/` |
| 状态管理 | `state/AppState.tsx` |
| API 调用 | `services/api/` |
| Ink 渲染 | `ink/` |
| 工具类型定义 | `Tool.ts` |
| 查询引擎 | `query.ts` |
| IDE 桥接 | `bridge/` |

## 编译时特性开关

项目使用 Bun 的 `feature()` 实现编译时 DCE（Dead Code Elimination）：

```typescript
// 条件引入，外部构建会被消除
const module = feature('FEATURE_NAME') ? require('./module.js') : null;
```

常见特性开关：
- `COORDINATOR_MODE` - 协调者模式
- `KAIROS` - 助手模式
- `BUDDY` - 宠物系统
- `VOICE_MODE` - 语音模式
- `DAEMON` - 守护进程
- `CHICAGO_MCP` - 计算机使用 MCP

---

下一章：[系统架构](./02-architecture.md)
