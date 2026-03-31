# UI 组件

## 概述

项目有 146+ React 组件，用于构建 CLI 界面。

## 组件分类

| 分类 | 数量 | 示例 |
|------|------|------|
| 核心布局 | ~20 | App, FullscreenLayout, Messages |
| 消息组件 | ~30 | MessageRow, UserMessage, AssistantMessage |
| 输入组件 | ~15 | TextInput, PromptInput, SearchBox |
| 对话框 | ~25 | SettingsDialog, ConfirmDialog, Dialog |
| 状态组件 | ~15 | StatusLine, Spinner, Progress |
| 选择器 | ~10 | ModelPicker, ThemePicker, LanguagePicker |
| 设计系统 | ~20 | ThemedBox, ThemedText, Button |
| 反馈组件 | ~15 | Toast, Tooltip, ErrorMessage |

## 核心组件

### App.tsx

主应用组件：

```typescript
// components/App.tsx
export function App(): React.ReactElement {
  return (
    <AppStateProvider>
      <FullscreenLayout>
        <Messages />
        <PromptInput />
      </FullscreenLayout>
    </AppStateProvider>
  );
}
```

### FullscreenLayout

全屏布局组件：

```typescript
// components/FullscreenLayout.tsx
interface FullscreenLayoutProps {
  children: React.ReactNode
  header?: React.ReactNode
  footer?: React.ReactNode
}

export function FullscreenLayout({
  children,
  header,
  footer
}: FullscreenLayoutProps): React.ReactElement {
  return (
    <Box flexDirection="column" width="100%" height="100%">
      {header && <Box>{header}</Box>}
      <Box flexGrow={1}>{children}</Box>
      {footer && <Box>{footer}</Box>}
    </Box>
  );
}
```

## 消息组件

### MessageRow

单条消息的渲染：

```typescript
// components/MessageRow.tsx
interface MessageRowProps {
  message: Message
  onEdit?: (message: Message) => void
  onDelete?: (message: Message) => void
}

export function MessageRow({ message, onEdit, onDelete }: MessageRowProps) {
  return (
    <Box>
      {message.role === 'user' ? (
        <UserMessageContent message={message} />
      ) : (
        <AssistantMessageContent message={message} />
      )}
    </Box>
  );
}
```

### VirtualMessageList

虚拟化消息列表（支持大量消息）：

```typescript
// components/VirtualMessageList.tsx
export function VirtualMessageList({
  messages,
  onLoadMore
}: {
  messages: Message[]
  onLoadMore?: () => void
}) {
  return (
    <List
      height={getTerminalHeight()}
      rowHeight={estimateRowHeight}
      rowCount={messages.length}
      overscan={5}
      renderRow={({ index, style }) => (
        <Box style={style}>
          <MessageRow message={messages[index]} />
        </Box>
      )}
    />
  );
}
```

## 输入组件

### PromptInput

用户输入框：

```typescript
// components/PromptInput/index.tsx
export function PromptInput({
  onSubmit,
  onCancel
}: {
  onSubmit: (value: string) => void
  onCancel?: () => void
}) {
  const [value, setValue] = useState('');
  const [suggestions, setSuggestions] = useState<Suggestion[]>([]);

  // 处理输入
  const handleInput = (key: Key) => {
    if (key.key === 'enter') {
      onSubmit(value);
      setValue('');
    } else if (key.key === 'escape') {
      onCancel?.();
    }
  };

  // 补全
  const handleComplete = (suggestion: Suggestion) => {
    setValue(suggestion.text);
  };

  return (
    <Box>
      <Text color="cyan">&gt;</Text>
      <TextInput
        value={value}
        onChange={setValue}
        onKeyPress={handleInput}
      />
      {suggestions.length > 0 && (
        <SuggestionList
          suggestions={suggestions}
          onSelect={handleComplete}
        />
      )}
    </Box>
  );
}
```

### TextInput

基础文本输入：

```typescript
// components/TextInput.tsx
interface TextInputProps {
  value: string
  onChange: (value: string) => void
  placeholder?: string
  mask?: string                    // 密码遮罩
  multiline?: boolean              // 多行
  onKeyPress?: (key: Key) => void
}
```

## 对话框组件

### Dialog

通用对话框：

```typescript
// components/Dialog.tsx
interface DialogProps {
  title: string
  children: React.ReactNode
  onClose: () => void
  width?: number | string
  footer?: React.ReactNode
}

export function Dialog({
  title,
  children,
  onClose,
  width = '50%',
  footer
}: DialogProps) {
  return (
    <Box
      borderStyle="round"
      borderColor="cyan"
      padding={1}
      width={width}
    >
      {/* 标题栏 */}
      <Box borderStyle="bold" borderBottom>
        <Text bold>{title}</Text>
        <Spacer />
        <Text color="gray" onDimmed>X</Text>
      </Box>

      {/* 内容 */}
      <Box padding={1}>{children}</Box>

      {/* 页脚 */}
      {footer && <Box borderTop>{footer}</Box>}
    </Box>
  );
}
```

### ConfirmDialog

确认对话框：

```typescript
// components/ConfirmDialog.tsx
interface ConfirmDialogProps {
  title: string
  message: string
  onConfirm: () => void
  onCancel: () => void
  confirmText?: string
  cancelText?: string
  severity?: 'warning' | 'danger' | 'info'
}
```

### SettingsDialog

设置对话框：

```typescript
// components/Settings/index.ts
interface SettingsDialogProps {
  onClose: () => void
}

// 子设置面板
export interface SettingsSection {
  id: string
  title: string
  icon: string
  component: React.ComponentType
}
```

## 状态组件

### StatusLine

状态栏：

```typescript
// components/StatusLine.tsx
export function StatusLine(): React.ReactElement {
  const { model, cost, tokens } = useCostTracker();

  return (
    <Box borderStyle="single" borderTop>
      <StatusItem icon="M" label={model} />
      <StatusItem icon="$" label={`${cost.toFixed(2)}`} />
      <StatusItem icon="T" label={`${tokens}/128k`} />
      <Spacer />
      <AgentStatusIndicator />
    </Box>
  );
}
```

### Spinner

加载指示器：

```typescript
// components/Spinner.tsx
interface SpinnerProps {
  label?: string
  color?: string
  frames?: number
}

// 使用
<Spinner label="Loading..." />
```

## 选择器组件

### ModelPicker

模型选择器：

```typescript
// components/ModelPicker.tsx
interface ModelPickerProps {
  selected: string
  onSelect: (model: string) => void
}

export function ModelPicker({ selected, onSelect }: ModelPickerProps) {
  const models = useAvailableModels();

  return (
    <Box>
      {models.map(model => (
        <Box
          key={model.id}
          onSelect={() => onSelect(model.id)}
        >
          <Text
            color={model.id === selected ? 'cyan' : 'white'}
            bold={model.id === selected}
          >
            {model.name}
          </Text>
        </Box>
      ))}
    </Box>
  );
}
```

## 设计系统

### ThemedBox

主题化盒子：

```typescript
// components/design-system/ThemedBox.tsx
interface BoxProps {
  backgroundColor?: ThemeColor
  borderColor?: ThemeColor
  textColor?: ThemeColor
  padding?: number
  margin?: number
}

// 使用主题变量而非硬编码颜色
<ThemedBox backgroundColor="primary" padding={2}>
  <ThemedText color="onPrimary">Content</ThemedText>
</ThemedBox>
```

### ThemeProvider

主题提供者：

```typescript
// components/design-system/ThemeProvider.tsx
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const theme = useThemeSetting();

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

// 主题定义
interface Theme {
  name: ThemeName
  colors: Record<ThemeColor, string>
  fonts: Record<FontFamily, string>
  spacing: typeof defaultSpacing
}
```

## 组件开发指南

### 创建新组件

```typescript
// components/MyComponent.tsx
import { Box, Text } from '../ink.js';

interface MyComponentProps {
  title: string
  children?: React.ReactNode
}

export function MyComponent({ title, children }: MyComponentProps) {
  return (
    <Box flexDirection="column" padding={1}>
      <Text bold color="cyan">{title}</Text>
      {children && <Box marginTop={1}>{children}</Box>}
    </Box>
  );
}
```

### 组件测试

```typescript
// components/__tests__/MyComponent.test.tsx
import { render } from '../test-utils';
import { MyComponent } from '../MyComponent';

test('renders title', () => {
  const { getByText } = render(<MyComponent title="Hello" />);
  expect(getByText('Hello')).toBeDefined();
});
```

---

下一章：[其他模块](./12-modules.md)
