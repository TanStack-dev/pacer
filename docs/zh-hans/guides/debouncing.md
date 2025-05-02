---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:19:53.011Z'
title: 防抖指南
id: debouncing
---
# 防抖 (Debouncing) 指南

TanStack Pacer 是一个专注于为应用程序提供高质量函数执行时序控制工具的库。虽然类似的工具在其他地方也存在，但我们的目标是正确处理所有重要细节——包括**类型安全 (type-safety)**、**摇树优化 (tree-shaking)** 以及一致且**直观的 API (intuitive API)**。通过专注于这些基础并以**框架无关 (framework agnostic)** 的方式提供它们，我们希望这些工具和模式能在您的应用中更加普及。正确的执行控制在应用开发中常常被忽视，导致性能问题、竞态条件和糟糕的用户体验，而这些本可以避免。TanStack Pacer 帮助您从一开始就正确实现这些关键模式！

## 速率限制 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同的方式阻止执行，使它们成为"有损 (lossy)"的——意味着当函数被请求过于频繁地运行时，某些调用将不会执行。了解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的防抖概念。

## 防抖概念

防抖是一种延迟函数执行的技术，直到指定的不活动时间段过去。与允许执行突发但有限制的速率限制不同，也与确保均匀间隔执行的节流不同，防抖将多个快速函数调用合并为单个执行，该执行仅在调用停止后发生。这使得防抖非常适合处理事件突发，其中您只关心活动稳定后的最终状态。

### 防抖可视化

```text
防抖 (等待: 3 个时间单位)
时间线: [每秒一个时间单位]
调用:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
执行:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 在此处执行
                                                         无调用 3 个时间单位后

             [调用突发]     [更多调用]   [等待]      [新突发]
             无执行         重置计时器    [延迟执行]  [等待] [延迟执行]
```

### 何时使用防抖

当您希望在采取行动之前等待活动"暂停"时，防抖特别有效。这使其非常适合处理用户输入或其他快速触发的事件，其中您只关心最终状态。

常见用例包括：
- 搜索输入字段，您希望等待用户完成输入
- 不应在每次按键时运行的表单验证
- 计算成本高的窗口大小调整计算
- 编辑内容时的自动保存草稿
- 仅在用户活动稳定后才应进行的 API 调用
- 任何您只关心快速变化后最终值的场景

### 何时不使用防抖

防抖可能不是最佳选择的情况：
- 您需要在特定时间段内保证执行（改用[节流](../guides/throttling)）
- 您不能错过任何执行（改用[队列](../guides/queueing)）

## TanStack Pacer 中的防抖

TanStack Pacer 分别通过 `Debouncer` 和 `AsyncDebouncer` 类（及其对应的 `debounce` 和 `asyncDebounce` 函数）提供同步和异步防抖。

### 使用 `debounce` 的基本用法

`debounce` 函数是为任何函数添加防抖的最简单方法：

```ts
import { debounce } from '@tanstack/pacer'

// 防抖搜索输入以等待用户停止输入
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // 最后一次按键后等待 500ms
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### 使用 `Debouncer` 类的高级用法

为了更精细地控制防抖行为，您可以直接使用 `Debouncer` 类：

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// 获取当前状态信息
console.log(searchDebouncer.getExecutionCount()) // 成功执行的次数
console.log(searchDebouncer.getIsPending()) // 是否有调用待处理

// 动态更新选项
searchDebouncer.setOptions({ wait: 1000 }) // 增加等待时间

// 取消待处理的执行
searchDebouncer.cancel()
```

### 前缘和后缘执行

同步防抖器支持前缘和后缘执行：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 第一次调用时立即执行
  trailing: true,  // 等待期后执行
})
```

- `leading: true` - 函数在第一次调用时立即执行
- `leading: false` (默认) - 第一次调用启动等待计时器
- `trailing: true` (默认) - 函数在等待期后执行
- `trailing: false` - 等待期后不执行

常见模式：
- `{ leading: false, trailing: true }` - 默认，等待后执行
- `{ leading: true, trailing: false }` - 立即执行，忽略后续调用
- `{ leading: true, trailing: true }` - 在第一次调用和等待后都执行

### 最大等待时间

TanStack Pacer 的防抖器不像其他防抖库那样有 `maxWait` 选项。如果您需要让执行在更分散的时间段内运行，请考虑改用[节流](../guides/throttling)技术。

### 启用/禁用

`Debouncer` 类通过 `enabled` 选项支持启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用防抖器：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 默认禁用
debouncer.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是防抖器选项具有响应性的框架适配器，您可以将 `enabled` 选项设置为条件值以动态启用/禁用防抖器：

```ts
// React 示例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 基于输入长度启用/禁用（如果使用支持响应式选项的框架适配器）
)
```

但是，如果您使用 `debounce` 函数或直接使用 `Debouncer` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `Debouncer` 类的构造函数。

```ts
// Solid 示例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 默认禁用
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 基于输入长度启用/禁用
})
```

### 回调选项

同步和异步防抖器都支持回调选项来处理防抖生命周期的不同方面：

#### 同步防抖器回调

同步 `Debouncer` 支持以下回调：

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 每次成功执行后调用
    console.log('函数已执行', debouncer.getExecutionCount())
  }
})
```

`onExecute` 回调在防抖函数每次成功执行后被调用，对于跟踪执行、更新 UI 状态或执行清理操作非常有用。

#### 异步防抖器回调

异步 `AsyncDebouncer` 支持额外的错误处理回调：

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // 每次成功执行后调用
    console.log('异步函数已执行', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // 如果异步函数抛出错误则调用
    console.error('异步函数失败:', error)
  }
})
```

`onExecute` 回调的工作方式与同步防抖器中的相同，而 `onError` 回调允许您优雅地处理错误而不中断防抖链。这些回调对于跟踪执行计数、更新 UI 状态、处理错误、执行清理操作和记录执行指标特别有用。

### 异步防抖

对于异步函数或当您需要错误处理时，使用 `AsyncDebouncer` 或 `asyncDebounce`：

```ts
import { asyncDebounce } from '@tanstack/pacer'

const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    updateUI(results)
  },
  {
    wait: 500,
    onError: (error) => {
      console.error('搜索失败:', error)
    }
  }
)

// 仅在输入停止后进行一次 API 调用
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

异步版本提供基于 Promise 的执行跟踪、通过 `onError` 回调的错误处理、待处理异步操作的正确清理以及可等待的 `maybeExecute` 方法。

### 框架适配器

每个框架适配器都提供建立在核心防抖功能之上的钩子，以与框架的状态管理系统集成。每个框架都有可用的钩子，如 `createDebouncer`、`useDebouncedCallback`、`useDebouncedState` 或 `useDebouncedValue`。

以下是一些示例：

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// 用于完全控制的低级钩子
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 用于基本用例的简单回调钩子
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// 用于响应式状态管理的基于状态的钩子
const [instantState, setInstantState] = useState('')
const [debouncedState, setDebouncedState] = useDebouncedValue(
  instantState, // 要防抖的值
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// 用于完全控制的低级钩子
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 用于状态管理的基于信号的钩子
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('总执行次数:', debouncer.getExecutionCount())
  }
})
```
