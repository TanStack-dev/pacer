---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:44:37.020Z'
title: 节流指南
id: throttling
---
限流 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同方式阻止执行，使其具有"有损"特性 —— 意味着当函数被请求过于频繁时，部分调用将不会执行。理解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将重点介绍 TanStack Pacer 的节流概念。

## 节流概念

节流确保函数执行在时间上均匀分布。与允许突发执行直到达到限制的限流不同，也与等待活动停止的防抖不同，节流通过在调用之间强制执行一致的延迟来创建更平滑的执行模式。如果设置每秒执行一次的节流，无论请求多么频繁，调用都将均匀间隔。

### 节流可视化

```text
节流（每 3 个时间单位执行一次）
时间轴：[每秒一个刻度]
调用：       ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
执行：      ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 个刻度仅允许一次执行，
               无论进行多少次调用

             [首次突发]    [更多调用]              [间隔调用]
             首次执行后     等待周期结束后         每次等待周期
             开始节流       执行                  结束后执行
```

### 适用场景

节流在需要一致、可预测的执行时机时特别有效。这使其成为处理频繁事件或更新的理想选择，在这些场景中您希望获得平滑、受控的行为。

常见用例包括：
- 需要定时一致的 UI 更新（例如进度指示器）
- 不应使浏览器过载的滚动或调整大小事件处理程序
- 需要固定间隔的实时数据轮询
- 需要稳定节奏的资源密集型操作
- 游戏循环更新或动画帧处理
- 用户输入时的实时搜索建议

### 不适用场景

以下情况可能不适合使用节流：
- 您希望等待活动停止（改用[防抖](../guides/debouncing)）
- 不能错过任何执行（改用[队列](../guides/queueing)）

> [!TIP]
> 当需要平滑、一致的执行时机时，节流通常是最佳选择。它比限流提供更可预测的执行模式，比防抖提供更即时的反馈。

## TanStack Pacer 中的节流

TanStack Pacer 通过 `Throttler` 和 `AsyncThrottler` 类（及其对应的 `throttle` 和 `asyncThrottle` 函数）分别提供同步和异步节流功能。

### 使用 `throttle` 的基本用法

`throttle` 函数是为任何函数添加节流的最简单方式：

```ts
import { throttle } from '@tanstack/pacer'

// 将 UI 更新节流为每 200ms 一次
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 在快速循环中，每 200ms 仅执行一次
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 许多调用会被节流
}
```

### 使用 `Throttler` 类的高级用法

为了更精细地控制节流行为，可以直接使用 `Throttler` 类：

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 获取执行状态信息
console.log(updateThrottler.getExecutionCount()) // 成功执行次数
console.log(updateThrottler.getLastExecutionTime()) // 上次执行时间戳

// 取消任何待执行操作
updateThrottler.cancel()
```

### 前缘与后缘执行

同步节流器支持前缘和后缘执行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 首次调用立即执行（默认）
  trailing: true,  // 等待周期后执行（默认）
})
```

- `leading: true`（默认）- 首次调用立即执行
- `leading: false` - 跳过首次调用，等待后缘执行
- `trailing: true`（默认）- 等待周期后执行最后一次调用
- `trailing: false` - 如果在等待周期内则跳过最后一次调用

常见模式：
- `{ leading: true, trailing: true }` - 默认模式，响应最快
- `{ leading: false, trailing: true }` - 延迟所有执行
- `{ leading: true, trailing: false }` - 跳过排队执行

### 启用/禁用

`Throttler` 类支持通过 `enabled` 选项启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用节流器：

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // 默认禁用
throttler.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是节流器选项具有响应性的框架适配器，可以将 `enabled` 选项设置为条件值以动态启用/禁用节流器。但是，如果直接使用 `throttle` 函数或 `Throttler` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `Throttler` 类的构造函数。

### 回调选项

同步和异步节流器都支持回调选项来处理节流生命周期的不同方面：

#### 同步节流器回调

同步 `Throttler` 支持以下回调：

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // 每次成功执行后调用
    console.log('函数已执行', throttler.getExecutionCount())
  }
})
```

`onExecute` 回调在每次成功执行节流函数后被调用，可用于跟踪执行、更新 UI 状态或执行清理操作。

#### 异步节流器回调

异步 `AsyncThrottler` 支持额外的错误处理回调：

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // 每次成功执行后调用
    console.log('异步函数已执行', throttler.getExecutionCount())
  },
  onError: (error) => {
    // 异步函数抛出错误时调用
    console.error('异步函数失败:', error)
  }
})
```

`onExecute` 回调的工作方式与同步节流器相同，而 `onError` 回调允许您优雅地处理错误而不中断节流链。这些回调特别适用于跟踪执行次数、更新 UI 状态、处理错误、执行清理操作和记录执行指标。

### 异步节流

异步节流器提供了一种强大的方式来处理带节流的异步操作，相比同步版本具有几个关键优势。虽然同步节流器适用于 UI 事件和即时反馈，但异步版本专为处理 API 调用、数据库操作和其他异步任务而设计。

#### 与同步节流的主要区别

1. **返回值处理**
与返回 void 的同步节流器不同，异步版本允许您捕获和使用节流函数的返回值。这在需要处理 API 调用或其他异步操作的结果时特别有用。`maybeExecute` 方法返回一个 Promise，该 Promise 解析为函数的返回值，允许您等待结果并适当处理。

2. **不同的回调**
`AsyncThrottler` 支持以下回调，而不仅仅是同步版本中的 `onExecute`：
- `onSuccess`：每次成功执行后调用，提供节流器实例
- `onSettled`：每次执行后调用，提供节流器实例
- `onError`：如果异步函数抛出错误则调用，提供错误和节流器实例

异步和同步节流器都支持 `onExecute` 回调来处理成功执行。

3. **顺序执行**
由于节流器的 `maybeExecute` 方法返回一个 Promise，您可以选择在开始下一个执行之前等待每个执行完成。这使您可以控制执行顺序，并确保每次调用处理最新的数据。这在处理依赖于先前调用结果的操作或维护数据一致性至关重要时特别有用。

例如，如果您正在更新用户的个人资料然后立即获取其更新后的数据，可以在开始获取之前等待更新操作完成：

#### 基本用法示例

以下是展示如何使用异步节流器进行搜索操作的基本示例：

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('搜索成功:', results)
    },
    onError: (error, throttler) => {
      console.error('搜索失败:', error)
    }
  }
)

// 用法
const results = await throttledSearch('query')
```

### 框架适配器

每个框架适配器都提供建立在核心节流功能之上的钩子，以与框架的状态管理系统集成。每个框架都提供诸如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue` 等钩子。

以下是一些示例：

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 用于完全控制的低级钩子
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用于基本用例的简单回调钩子
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用于响应式状态管理的基于状态的钩子
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // 需要节流的值
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// 用于完全控制的低级钩子
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用于状态管理的基于信号的钩子
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('总执行次数:', throttler.getExecutionCount())
  }
})
```

每个框架适配器都提供与框架状态管理系统集成的钩子，同时保持核心节流功能。
