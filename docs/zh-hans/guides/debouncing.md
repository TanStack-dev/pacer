---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:04:08.175Z'
title: 防抖指南
id: debouncing
---
# 防抖 (Debouncing) 指南

速率限制 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同的方式阻止执行，使它们成为"有损"的——意味着当函数被请求过于频繁地运行时，某些调用将不会执行。了解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的防抖概念。

## 防抖概念

防抖是一种延迟函数执行的技术，直到指定的不活动周期结束。与速率限制允许在限制范围内突发执行，或节流确保均匀间隔的执行不同，防抖将多个快速函数调用合并为单个执行，该执行仅在调用停止后发生。这使得防抖非常适合处理事件突发，其中您只关心活动稳定后的最终状态。

### 防抖可视化

```text
防抖 (等待: 3 个时间单位)
时间线: [每秒 1 个时间单位]
调用:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
执行:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 在此处执行，因为
                                                         3 个时间单位无调用

             [调用突发]     [更多调用]   [等待]      [新突发]
             无执行         重置计时器    [延迟执行]  [等待] [延迟执行]
```

### 何时使用防抖

当您希望在采取行动之前等待活动"暂停"时，防抖特别有效。这使得它非常适合处理用户输入或其他快速触发的事件，其中您只关心最终状态。

常见用例包括：
- 搜索输入字段，您希望等待用户完成输入
- 表单验证，不应在每次按键时运行
- 窗口大小调整计算，计算成本高
- 编辑内容时自动保存草稿
- 仅在用户活动稳定后才应进行的 API 调用
- 任何您只关心快速变化后最终值的场景

### 何时不使用防抖

在以下情况下，防抖可能不是最佳选择：
- 您需要在特定时间段内保证执行（改用[节流](../guides/throttling)）
- 您不能错过任何执行（改用[队列](../guides/queueing)）

## TanStack Pacer 中的防抖

TanStack Pacer 分别通过 `Debouncer` 和 `AsyncDebouncer` 类（及其对应的 `debounce` 和 `asyncDebounce` 函数）提供同步和异步防抖。

### 使用 `debounce` 的基本用法

`debounce` 函数是为任何函数添加防抖的最简单方法：

```ts
import { debounce } from '@tanstack/pacer'

// 防抖搜索输入，等待用户停止输入
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
console.log(searchDebouncer.getExecutionCount()) // 成功执行次数
console.log(searchDebouncer.getIsPending()) // 是否有调用待处理

// 动态更新选项
searchDebouncer.setOptions({ wait: 1000 }) // 增加等待时间

// 取消待处理的执行
searchDebouncer.cancel()
```

### 前导和尾随执行

同步防抖器支持前导和尾随边缘执行：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 在第一次调用时执行
  trailing: true,  // 在等待期后执行
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

TanStack Pacer 防抖器特意不像其他防抖库那样提供 `maxWait` 选项。如果您需要让执行在更分散的时间段内运行，请考虑改用[节流](../guides/throttling)技术。

### 启用/禁用

`Debouncer` 类通过 `enabled` 选项支持启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用防抖器：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 默认禁用
debouncer.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是支持响应式选项的框架适配器，可以将 `enabled` 选项设置为条件值以动态启用/禁用防抖器：

```ts
// React 示例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 根据输入长度启用/禁用（如果使用支持响应式选项的框架适配器）
)
```

但是，如果您使用的是 `debounce` 函数或直接使用 `Debouncer` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `Debouncer` 类构造函数的。

```ts
// Solid 示例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 默认禁用
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 根据输入长度启用/禁用
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

`onExecute` 回调在防抖函数每次成功执行后调用，对于跟踪执行、更新 UI 状态或执行清理操作非常有用。

#### 异步防抖器回调

异步 `AsyncDebouncer` 具有与同步版本不同的一组回调。

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // 每次成功执行后调用
    console.log('异步函数已执行', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // 每次执行尝试后调用
    console.log('异步函数已解决', debouncer.getSettledCount())
  },
  onError: (error) => {
    // 如果异步函数抛出错误则调用
    console.error('异步函数失败:', error)
  }
})
```

`onSuccess` 回调在防抖函数每次成功执行后调用，而 `onError` 回调在异步函数抛出错误时调用。`onSettled` 回调在每次执行尝试后调用，无论成功与否。这些回调对于跟踪执行计数、更新 UI 状态、处理错误、执行清理操作和记录执行指标特别有用。

### 异步防抖

异步防抖器提供了一种强大的方式来处理异步操作的防抖，与同步版本相比具有几个关键优势。虽然同步防抖器非常适合 UI 事件和即时反馈，但异步版本专门设计用于处理 API 调用、数据库操作和其他异步任务。

#### 与同步防抖的主要区别

1. **返回值处理**
与返回 void 的同步防抖器不同，异步版本允许您捕获和使用防抖函数的返回值。这在您需要处理 API 调用或其他异步操作的结果时特别有用。`maybeExecute` 方法返回一个 Promise，该 Promise 解析为函数的返回值，允许您等待结果并适当处理。

2. **增强的回调系统**
异步防抖器提供了比同步版本的单一 `onExecute` 回调更复杂的回调系统。该系统包括：
- `onSuccess`：当异步函数成功完成时调用，提供结果和防抖器实例
- `onError`：当异步函数抛出错误时调用，提供错误和防抖器实例
- `onSettled`：在每次执行尝试后调用，无论成功与否

3. **执行跟踪**
异步防抖器通过几种方法提供全面的执行跟踪：
- `getSuccessCount()`：成功执行次数
- `getErrorCount()`：失败执行次数
- `getSettledCount()`：已解决的执行总数（成功 + 失败）

4. **顺序执行**
异步防抖器确保后续执行等待前一次调用完成后再开始。这可以防止执行顺序混乱，并保证每次调用处理最新的数据。这在处理依赖于先前调用结果的操作或维护数据一致性至关重要时尤为重要。

例如，如果您正在更新用户的个人资料，然后立即获取其更新后的数据，异步防抖器将确保获取操作等待更新完成，防止出现获取陈旧数据的竞态条件。

#### 基本用法示例

以下是一个基本示例，展示如何将异步防抖器用于搜索操作：

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('搜索成功:', results)
    },
    onError: (error, debouncer) => {
      console.error('搜索失败:', error)
    }
  }
)

// 用法
const results = await debouncedSearch('query')
```

#### 高级模式

异步防抖器可以与各种模式结合以解决复杂问题：

1. **状态管理集成**
当将异步防抖器与状态管理系统（如 React 的 useState 或 Solid 的 createSignal）一起使用时，您可以创建强大的模式来处理加载状态、错误状态和数据更新。防抖器的回调为基于操作成功或失败更新 UI 状态提供了完美的钩子。

2. **竞态条件预防**
单次飞行变异模式自然可以防止许多场景中的竞态条件。当应用程序的多个部分尝试同时更新同一资源时，防抖器确保只有最近的更新实际发生，同时仍为所有调用者提供结果。

3. **错误恢复**
异步防抖器的错误处理能力使其非常适合实现重试逻辑和错误恢复模式。您可以使用 `onError` 回调实现自定义错误处理策略，例如指数退避或回退机制。

### 框架适配器

每个框架适配器都提供构建在核心防抖功能之上的钩子，以与框架的状态管理系统集成。每个框架都有可用的钩子，如 `createDebouncer`、`useDebouncedCallback`、`useDebouncedState` 或 `useDebouncedValue`。

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
const [debouncedValue] = useDebouncedValue(
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
