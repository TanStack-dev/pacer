---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:04:06.670Z'
title: 节流指南
id: throttling
---
# 节流 (Throttling) 指南

TanStack Pacer 是一个专注于为应用程序提供高质量函数执行时序控制工具的库。虽然类似的工具在其他地方也存在，但我们的目标是正确处理所有重要细节 —— 包括 ***类型安全 (type-safety)***、***摇树优化 (tree-shaking)*** 以及一致且 ***直观的 API (intuitive API)***。通过专注于这些基础功能并以 ***框架无关 (framework agnostic)*** 的方式提供它们，我们希望这些工具和模式能在您的应用程序中更加普及。适当的执行控制在应用程序开发中常常被忽视，从而导致性能问题、竞态条件和糟糕的用户体验，而这些本可以通过正确的执行控制来预防。TanStack Pacer 帮助您从一开始就正确实现这些关键模式！

速率限制 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同的方式阻止执行，使它们具有"有损 (lossy)"特性 —— 意味着当函数被请求过于频繁时，某些调用将不会执行。了解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的节流概念。

## 节流概念

节流确保函数执行在时间上均匀分布。与允许突发执行直到达到限制的速率限制 (rate limiting) 不同，也与等待活动停止的防抖 (debouncing) 不同，节流通过在调用之间强制执行一致的延迟来创建更平滑的执行模式。如果您设置每秒执行一次的节流，无论请求多么频繁，调用都将均匀间隔。

### 节流可视化

```text
节流 (每 3 个 tick 执行一次)
时间线: [每秒一个 tick]
调用:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
执行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 个 tick 只允许一次执行，
               无论进行了多少次调用

             [第一次突发]    [更多调用]              [间隔调用]
             首次执行后      等待周期后执行          每次等待周期
             开始节流                           结束后执行
```

### 何时使用节流

节流在您需要一致、可预测的执行时序时特别有效。这使得它非常适合处理频繁事件或更新，在这些情况下您希望获得平滑、受控的行为。

常见用例包括：
- 需要一致时序的 UI 更新（例如进度指示器）
- 不应使浏览器不堪重负的滚动或调整大小事件处理程序
- 需要一致间隔的实时数据轮询
- 需要稳定节奏的资源密集型操作
- 游戏循环更新或动画帧处理
- 用户输入时的实时搜索建议

### 何时不使用节流

在以下情况下，节流可能不是最佳选择：
- 您想等待活动停止（改用[防抖](../guides/debouncing)）
- 您不能错过任何执行（改用[队列](../guides/queueing)）

> [!TIP]
> 当您需要平滑、一致的执行时序时，节流通常是最佳选择。它提供了比速率限制更可预测的执行模式，比防抖更即时的反馈。

## TanStack Pacer 中的节流

TanStack Pacer 分别通过 `Throttler` 和 `AsyncThrottler` 类（以及它们对应的 `throttle` 和 `asyncThrottle` 函数）提供同步和异步节流。

### 使用 `throttle` 的基本用法

`throttle` 函数是为任何函数添加节流的最简单方法：

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

为了更精细地控制节流行为，您可以直接使用 `Throttler` 类：

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 获取执行状态信息
console.log(updateThrottler.getExecutionCount()) // 成功执行的次数
console.log(updateThrottler.getLastExecutionTime()) // 最后一次执行的时间戳

// 取消任何待执行的调用
updateThrottler.cancel()
```

### 前缘和后缘执行

同步节流器支持前缘和后缘执行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 在第一次调用时立即执行（默认）
  trailing: true,  // 在等待周期后执行（默认）
})
```

- `leading: true`（默认）- 在第一次调用时立即执行
- `leading: false` - 跳过第一次调用，等待后缘执行
- `trailing: true`（默认）- 在等待周期后执行最后一次调用
- `trailing: false` - 如果在等待周期内则跳过最后一次调用

常见模式：
- `{ leading: true, trailing: true }` - 默认，响应最快
- `{ leading: false, trailing: true }` - 延迟所有执行
- `{ leading: true, trailing: false }` - 跳过排队的执行

### 启用/禁用

`Throttler` 类支持通过 `enabled` 选项启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用节流器：

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // 默认禁用
throttler.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是节流器选项具有响应性的框架适配器，您可以将 `enabled` 选项设置为条件值以动态启用/禁用节流器。但是，如果您直接使用 `throttle` 函数或 `Throttler` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `Throttler` 类的构造函数。

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
    // 如果异步函数抛出错误则调用
    console.error('异步函数失败:', error)
  }
})
```

`onExecute` 回调的工作方式与同步节流器相同，而 `onError` 回调允许您优雅地处理错误而不中断节流链。这些回调对于跟踪执行计数、更新 UI 状态、处理错误、执行清理操作和记录执行指标特别有用。

### 异步节流

异步节流器提供了一种强大的方式来处理带节流的异步操作，与同步版本相比具有几个关键优势。虽然同步节流器非常适合 UI 事件和即时反馈，但异步版本专门设计用于处理 API 调用、数据库操作和其他异步任务。

#### 与同步节流的主要区别

1. **返回值处理**
与返回 void 的同步节流器不同，异步版本允许您捕获和使用节流函数的返回值。这在您需要使用 API 调用或其他异步操作的结果时特别有用。`maybeExecute` 方法返回一个 Promise，该 Promise 解析为函数的返回值，允许您等待结果并适当地处理它。

2. **增强的回调系统**
异步节流器提供了比同步版本的单一 `onExecute` 回调更复杂的回调系统。该系统包括：
- `onSuccess`：当异步函数成功完成时调用，提供结果和节流器实例
- `onError`：当异步函数抛出错误时调用，提供错误和节流器实例
- `onSettled`：在每次执行尝试后调用，无论成功或失败

3. **执行跟踪**
异步节流器通过几种方法提供全面的执行跟踪：
- `getSuccessCount()`：成功执行的次数
- `getErrorCount()`：失败的执行次数
- `getSettledCount()`：已完成的执行总数（成功 + 失败）

4. **顺序执行**
异步节流器确保后续执行等待前一个调用完成后再开始。这可以防止执行顺序混乱，并保证每个调用处理最新的数据。这在处理依赖于先前调用结果的操作或维护数据一致性至关重要时尤为重要。

例如，如果您正在更新用户的个人资料并立即获取其更新后的数据，异步节流器将确保获取操作等待更新完成，防止您可能获取到过时数据的竞态条件。

#### 基本用法示例

以下是一个显示如何使用异步节流器进行搜索操作的基本示例：

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

#### 高级模式

异步节流器可以与各种模式结合以解决复杂问题：

1. **状态管理集成**
当将异步节流器与状态管理系统（如 React 的 useState 或 Solid 的 createSignal）一起使用时，您可以创建强大的模式来处理加载状态、错误状态和数据更新。节流器的回调为基于操作的成功或失败更新 UI 状态提供了完美的钩子。

2. **竞态条件预防**
节流模式自然可以防止许多场景中的竞态条件。当应用程序的多个部分尝试同时更新同一资源时，节流器确保更新以受控速率进行，同时仍向所有调用者提供结果。

3. **错误恢复**
异步节流器的错误处理能力使其成为实现重试逻辑和错误恢复模式的理想选择。您可以使用 `onError` 回调实现自定义错误处理策略，例如指数退避或回退机制。

### 框架适配器

每个框架适配器都提供构建在核心节流功能之上的钩子，以与框架的状态管理系统集成。每个框架都有可用的钩子，如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue`。

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
