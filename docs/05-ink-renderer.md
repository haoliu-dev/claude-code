# Ink 渲染引擎

## 概述

Ink 是一个 React for CLI 框架，允许使用 React 组件构建终端 UI。项目在 `ink/` 目录实现了自定义 Ink 渲染引擎。

### 与标准 React 的区别

| 方面 | 标准 React (DOM) | Ink (Terminal) |
|------|------------------|---------------|
| 渲染目标 | HTML DOM | Terminal 字符输出 |
| 更新机制 | Virtual DOM diff | 最小化重渲染 |
| 布局 | CSS Flexbox | ANSI 排版 |
| 事件 | DOM Events | Keypress/Mouse |
| 输出 | HTML | ANSI 转义序列 |

## 目录结构

```
ink/
├── ink.tsx                    # 主渲染器 (~250KB)
├── screen.ts                   # 屏幕管理
├── render-node-to-output.ts   # 节点渲染
├── render-to-screen.ts        # 输出渲染
├── reconciler.ts             # React Reconciler
├── components/                # Ink 内置组件
│   ├── Box.tsx               # 容器组件
│   ├── Text.tsx              # 文本组件
│   ├── Button.tsx            # 按钮组件
│   └── ...
├── hooks/                     # Ink 内置 hooks
│   ├── use-input.ts           # 输入处理
│   ├── use-app.ts            # App 实例
│   └── ...
├── termio/                    # 终端 IO
│   ├── termio.ts             # 读写封装
│   ├── dec.ts               # DEC 序列
│   └── osc.ts               # Operating System Command
└── events/                   # 事件系统
    ├── input-event.ts        # 输入事件
    ├── click-event.ts        # 点击事件
    └── terminal-focus-event.ts
```

## 渲染管线

```
React Component Tree
        │
        ▼
┌─────────────────┐
│   ink.tsx       │  React.createElement()
│   render()      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Reconciler    │  Fiber tree 构造
│ (reconciler.ts) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ renderNode()    │  计算差异
│ (render-node-   │
│  to-output.ts)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Output Buffer   │  ANSI 转义序列
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Terminal       │  写入 stdout
└─────────────────┘
```

## 核心组件

### Box - 容器

```typescript
// ink/components/Box.tsx
interface BoxProps {
  flexDirection?: 'row' | 'column'
  flexGrow?: number
  flexShrink?: number
  width?: number | string
  height?: number | string
  padding?: number
  margin?: number
  borderStyle?: 'single' | 'double' | 'round'
  backgroundColor?: string
}

// 使用示例
<Box flexDirection="row" padding={1}>
  <Text>Hello</Text>
  <Text>World</Text>
</Box>
```

### Text - 文本

```typescript
// ink/components/Text.tsx
interface TextProps {
  color?: string
  backgroundColor?: string
  bold?: boolean
  italic?: boolean
  underline?: boolean
}

// 使用示例
<Text color="green" bold>
  Success!
</Text>
```

### Spinner - 加载指示器

项目有自定义的 Spinner 组件在 `components/Spinner/` 目录。

## 输入处理

### useInput Hook

```typescript
// ink/hooks/use-input.ts
function useInput(
  handler: (key: Key, next: () => void) => void,
  options?: { isActive?: boolean }
): void

// 使用示例
function MyComponent() {
  useInput((key, next) => {
    if (key.key === 'q') {
      exit();
    } else {
      next(); // 传递给下一个处理器
    }
  });
}
```

### Key 对象结构

```typescript
interface Key {
  key: string           // 'a', 'enter', 'escape'
  shift?: boolean       // Shift 修饰符
  ctrl?: boolean        // Ctrl 修饰符
  meta?: boolean        // Meta/Alt 修饰符
}
```

## ANSI 颜色支持

Ink 通过 `ink/colorize.ts` 支持 ANSI 颜色：

```typescript
// 支持的颜色
'black', 'red', 'green', 'yellow', 'blue', 'magenta', 'cyan', 'white'
'gray', 'grey'
'redBright', 'greenBright', ...

// 使用
<Text color="green">Green Text</Text>
<Box backgroundColor="cyan">Cyan Background</Box>
```

## 焦点管理

```typescript
// ink/focus.ts
export function FocusManager({ children }) {
  // 管理焦点顺序
}

// 使用 useApp 获取焦点状态
function MyInput() {
  const { isFocusIn } = useApp();

  return isFocusIn ? <TextInput /> : <Text>Disabled</Text>;
}
```

## Ink 与业务组件的桥接

项目在 `ink.ts` 中封装了 Ink：

```typescript
// ink.ts
export async function render(node: ReactNode, options?: RenderOptions) {
  // 1. 包装 ThemeProvider
  const themed = <ThemeProvider>{node}</ThemeProvider>;

  // 2. 调用 Ink 渲染
  return inkRender(themed, options);
}

export function createRoot(options?: RenderOptions): Promise<Root> {
  // 返回自定义 root，包装了 ThemeProvider
}
```

## 性能优化

### 差异计算

Ink 的 reconciler (`ink/reconciler.ts`) 针对终端进行了优化：

```typescript
// 只重渲染变化的节点
function reconcile(oldTree, newTree) {
  if (sameType(oldTree, newTree) && !hasStyleChange(oldTree, newTree)) {
    // 复用子树
    return updateChildren(oldTree, newTree);
  }
  // 重新渲染
  return createNewTree(newTree);
}
```

### 输出缓冲

```typescript
// ink/output.ts
class OutputBuffer {
  private buffer: string = '';

  write(text: string) {
    this.buffer += text;
  }

  flush(stream: WriteStream) {
    // 批量写入，减少 syscall
    stream.write(this.buffer);
    this.buffer = '';
  }
}
```

## 自定义组件开发指南

```typescript
// 创建自定义 Ink 组件
import { Box, Text } from '../ink.js';

interface Props {
  label: string;
  value: string;
}

export function StatusBadge({ label, value }: Props) {
  return (
    <Box borderStyle="round" padding={1}>
      <Text bold>{label}: </Text>
      <Text color="cyan">{value}</Text>
    </Box>
  );
}
```

---

下一章：[工具系统](./06-tools-system.md)
