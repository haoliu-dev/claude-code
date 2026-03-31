# 入口流程

## 启动流程总览

```
main.tsx (785KB)
    │
    ├─► 前置导入 (MDM、Keychain 预读)
    │
    ├─► 初始化 (init.ts)
    │       │
    │       ├─► 配置加载
    │       ├─► Analytics 初始化
    │       └─► 插件加载
    │
    ├─► 命令解析 (Commander.js)
    │       │
    │       ├─► 全局选项 (--version, --help)
    │       └─► 子命令 (commit, branch, agent...)
    │
    └─► REPL 启动 (replLauncher.tsx)
            │
            └─► App.tsx 渲染
```

## main.tsx 详解

`main.tsx` 是整个应用的入口，约 785KB。

### 前置导入（副作用）

```typescript
// 必须最先执行，用于性能优化
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';

// MDM 配置预读（并行执行）
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();

// Keychain 预读（并行执行）
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

> 这些是**必须先执行**的副作用，用于在模块加载前启动后台进程。

### Commander.js 命令定义

```typescript
// Commander.js 定义命令
const program = new Command();

// 全局选项
program
  .option('-v, --version', '输出版本')
  .option('--model <model>', '指定模型');

// 子命令通过 commands/ 目录动态注册
```

### 启动序列

```typescript
// 1. 预检查
if (args[0] === '--version') {
  console.log(MACRO.VERSION);
  return;
}

// 2. 初始化
await init();  // entrypoints/init.ts

// 3. 命令注册
const commands = await getCommands();

// 4. 启动 REPL
await launchRepl();  // replLauncher.tsx
```

## init.ts 详解

`entrypoints/init.ts` 负责应用初始化。

### 初始化步骤

```typescript
export async function init(): Promise<void> {
  // 1. 启用配置
  enableConfigs();

  // 2. 初始化分析服务
  await initializeAnalyticsAfterTrust();

  // 3. 加载远程托管设置
  await loadRemoteManagedSettings();

  // 4. 加载策略限制
  loadPolicyLimits();

  // 5. 初始化插件
  await initBundledPlugins();
  await initBundledSkills();

  // 6. 初始化 GrowthBook
  await initializeGrowthBook();
}
```

### 初始化顺序图

```
init()
 │
 ├─► enableConfigs()
 │        │
 │        └─► 加载全局配置、验证设置
 │
 ├─► initializeTelemetryAfterTrust()
 │        │
 │        └─► 用户同意后初始化分析
 │
 ├─► loadRemoteManagedSettings()
 │        │
 │        └─► 企业环境加载托管设置
 │
 ├─► loadPolicyLimits()
 │        │
 │        └─► 加载策略限制（速率限制等）
 │
 ├─► initBundledPlugins()
 │        │
 │        └─► 加载内置插件
 │
 └─► initializeGrowthBook()
          │
          └─► 初始化 A/B 测试框架
```

## REPL 启动流程

`replLauncher.tsx` 启动交互式循环。

```typescript
// replLauncher.tsx
import { render } from './ink.js';
import { REPL } from './components/REPL.js';

export async function launchRepl(): Promise<void> {
  const instance = await render(React.createElement(REPL));

  // 保持运行
  await instance.waitUntilExit();
}
```

## App.tsx 组件树

```
App
├── AppStateProvider
│       │
│       └─► VoiceProvider (可选)
│       │
│       └─► MailboxProvider
│       │
│       └─► Children
│              │
│              ├─► FullscreenLayout
│              │       │
│              │       ├─► Header
│              │       ├─► Messages (VirtualMessageList)
│              │       │       │
│              │       │       └─► MessageRow[]
│              │       │              │
│              │       │              ├─► UserMessage
│              │       │              ├─► AssistantMessage
│              │       │              └─► SystemMessage
│              │       │
│              │       ├─► PromptInput
│              │       │       │
│              │       │       ├─► TextInput
│              │       │       └─► SuggestionList
│              │       │
│              │       └─► StatusLine
│              │               │
│              │               ├─► ModelPicker
│              │               ├─► CostTracker
│              │               └─► AgentStatus
│              │
│              └─► Dialogs (叠加层)
│                      │
│                      ├─► SettingsDialog
│                      ├─► HelpDialog
│                      └─► ConfirmDialog
```

## QueryEngine 查询流程

用户消息通过 `query.ts` 处理。

```typescript
// query.ts
export async function query(input: UserMessage): Promise<void> {
  // 1. 解析用户输入
  const parsed = parseUserInput(input);

  // 2. 构建 API 请求
  const request = buildRequest(parsed);

  // 3. 调用 Claude API
  const response = await api.messages.create(request);

  // 4. 处理工具调用
  for (const block of response.content) {
    if (block.type === 'tool_use') {
      await executeTool(block);
    }
  }

  // 5. 渲染响应
  renderMessage(response);
}
```

### API 请求构建

```typescript
// QueryEngine.ts
interface QueryOptions {
  messages: Message[]           // 对话历史
  tools: Tool[]                  // 可用工具
  systemPrompt: SystemPrompt     // 系统提示
  model?: string                 // 模型选择
}

async function buildRequest(options: QueryOptions) {
  return {
    model: options.model ?? getMainLoopModel(),
    max_tokens: getContextWindowForModel(options.model),
    messages: options.messages,
    tools: options.tools.map(tool => ({
      name: tool.name,
      description: tool.description,
      input_schema: tool.inputSchema
    })),
    system: options.systemPrompt
  };
}
```

## 快速路径优化

项目针对常见场景优化了启动速度：

| 场景 | 优化方式 |
|------|----------|
| `--version` | 零模块加载，直接输出 |
| `--help` | 最小导入 |
| 首次启动 | MDM/Keychain 并行预读 |
| 重复启动 | 缓存 GrowthBook 配置 |

---

下一章：[Ink 渲染引擎](./05-ink-renderer.md)
