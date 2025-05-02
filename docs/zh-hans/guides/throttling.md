---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:19:39.371Z'
title: 节流指南
id: throttling
---
限流 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术都以不同方式阻止执行，使其具有"有损性"——这意味着当函数被请求过于频繁时，部分调用将不会执行。理解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的节流 (Throttling) 概念。

## 节流概念

节流确保函数执行在时间上均匀分布。与允许突发执行直到达到限制的限流 (Rate Limiting) 不同，也不同于等待活动停止的防抖 (Debouncing)，节流通过在调用之间强制执行一致的延迟来创建更平滑的执行模式。如果设置每秒执行一次的节流，无论请求多么频繁，调用都会均匀间隔。

### 节流可视化

```text
节流 (每 3 个 tick 执行一次)
时间线: [每秒一个 tick]
调用:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
执行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 个 tick 只允许一次执行，
               无论进行了多少次调用

             [首次突发]    [更多调用]              [间隔调用]
             首次执行后    等待周期后执行         每次等待周期过后执行
             开始节流
```

### 何时使用节流

节流在需要一致、可预测的执行时间时特别有效。这使得它非常适合处理需要平滑、受控行为的频繁事件或更新。

常见用例包括：
- 需要一致定时的 UI 更新（例如进度指示器）
- 不应使浏览器不堪重负的滚动或调整大小事件处理程序
- 需要固定间隔的实时数据轮询
- 需要稳定节奏的资源密集型操作
- 游戏循环更新或动画帧处理
- 用户输入时的实时搜索建议

### 何时不使用节流

节流可能不是最佳选择的情况：
- 您希望等待活动停止（改用[防抖](../guides/debouncing)）
- 不能错过任何执行（改用[队列](../guides/queueing)）

> [!TIP]
> 当需要平滑、一致的执行时间时，节流通常是最佳选择。它提供了比限流 (Rate Limiting) 更可预测的执行模式，比防抖 (Debouncing) 更即时的反馈。

## TanStack Pacer 中的节流

TanStack Pacer 通过 `Throttler` 和 `AsyncThrottler` 类（及其对应的 `throttle` 和 `asyncThrottle` 函数）提供同步和异步节流。

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

// 在快速循环中，每 200ms 只执行一次
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 许多调用被节流
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
console.log(updateThrottler.getLastExecutionTime()) // 最后一次执行的时间戳

// 取消任何待处理的执行
updateThrottler.cancel()
```

### 前导和尾随执行

同步节流器支持前导和尾随边缘执行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 首次调用时执行（默认）
  trailing: true,  // 等待周期后执行（默认）
})
```

- `leading: true`（默认）- 首次调用时立即执行
- `leading: false` - 跳过首次调用，等待尾随执行
- `trailing: true`（默认）- 等待周期后执行最后一次调用
- `trailing: false` - 如果在等待周期内则跳过最后一次调用

常见模式：
- `{ leading: true, trailing: true }` - 默认，响应最快
- `{ leading: false, trailing: true }` - 延迟所有执行
- `{ leading: true, trailing: false }` - 跳过排队的执行

### 启用/禁用

`Throttler` 类支持通过 `enabled` 选项启用/禁用。使用 `setOptions` 方法，可以随时启用/禁用节流器：

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // 默认禁用
throttler.setOptions({ enabled: true }) // 随时启用
```

如果在框架适配器中使用节流器选项是响应式的，可以将 `enabled` 选项设置为条件值以动态启用/禁用节流器。但是，如果直接使用 `throttle` 函数或 `Throttler` 类，则必须使用 `setOptions` 方法更改 `enabled` 选项，因为传递的选项实际上是传递给 `Throttler` 类的构造函数。

### 回调选项

同步和异步节流器都支持回调选项以处理节流生命周期的不同方面：

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

`onExecute` 回调在每次成功执行节流函数后调用，可用于跟踪执行、更新 UI 状态或执行清理操作。

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
    // 如果异步函数抛出错误则调用
    console.error('异步函数失败:', error)
  }
})
```

`onExecute` 回调与同步节流器中的工作方式相同，而 `onError` 回调允许您优雅地处理错误而不会中断节流链。这些回调对于跟踪执行计数、更新 UI 状态、处理错误、执行清理操作和记录执行指标特别有用。

### 异步节流

对于异步函数或需要错误处理的情况，使用 `AsyncThrottler` 或 `asyncThrottle`：

```ts
import { asyncThrottle } from '@tanstack/pacer'

const throttledFetch = asyncThrottle(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    wait: 1000,
    onError: (error) => {
      console.error('API 调用失败:', error)
    }
  }
)

// 每秒只进行一次 API 调用
await throttledFetch('123')
```

异步版本提供基于 Promise 的执行跟踪、通过 `onError` 回调的错误处理、待处理异步操作的适当清理以及可等待的 `maybeExecute` 方法。

### 框架适配器

每个框架适配器都提供建立在核心节流功能之上的钩子，以与框架的状态管理系统集成。每个框架都有可用的钩子，如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue`。

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
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // 要节流的值
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
