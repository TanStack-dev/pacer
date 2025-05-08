---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:45:08.193Z'
title: 限流指南
id: rate-limiting
---
# 速率限制 (Rate Limiting) 指南

速率限制 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同方式阻止执行，使它们具有"有损性" (lossy) - 意味着当函数被请求过于频繁时，某些调用将不会执行。了解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的速率限制概念。

> [!NOTE]
> TanStack Pacer 目前仅是一个前端库。这些是用于客户端速率限制 (client-side rate-limiting) 的工具。

## 速率限制概念

速率限制 (Rate Limiting) 是一种限制函数在特定时间窗口内执行速率的技术。它特别适用于需要防止函数被过于频繁调用的场景，例如处理 API 请求或其他外部服务调用。这是最*基础*的方法，因为它允许执行突发调用，直到达到配额限制。

### 速率限制可视化

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### 窗口类型

TanStack Pacer 支持两种速率限制窗口类型：

1. **固定窗口 (Fixed Window)** (默认)
   - 严格的窗口，在窗口周期后重置
   - 窗口内的所有执行都计入限制
   - 窗口在周期后完全重置
   - 可能导致窗口边界处的突发行为

2. **滑动窗口 (Sliding Window)**
   - 滚动窗口，当旧执行过期时允许新执行
   - 随时间提供更一致的执行速率
   - 更适合维持稳定的执行流
   - 防止窗口边界处的突发行为

以下是滑动窗口速率限制的可视化：

```text
Sliding Window Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ✅                             ✅     ✅
             [=== 3 allowed ===][=== oldest expires, new allowed ===][=== continues sliding =======]
```

关键区别在于，使用滑动窗口时，一旦最旧的执行过期，就允许新的执行。与固定窗口方法相比，这创造了更一致的执行流。

### 何时使用速率限制

速率限制在处理可能意外压倒后端服务或导致浏览器性能问题的前端操作时特别重要。

常见用例包括：
- 防止快速用户交互（如按钮点击或表单提交）导致的意外 API 滥用
- 可接受突发行为但需要限制最大速率的场景
- 防止意外的无限循环或递归操作

### 何时不使用速率限制

速率限制是控制函数执行频率的最基础方法。它是这三种技术中最不灵活和限制最多的。考虑使用[节流 (throttling)](../guides/throttling) 或[防抖 (debouncing)](../guides/debouncing) 来获得更分散的执行。

> [!TIP]
> 对于大多数用例，您可能不希望使用"速率限制"。考虑使用[节流 (throttling)](../guides/throttling) 或[防抖 (debouncing)](../guides/debouncing) 代替。

速率限制的"有损性" (lossy) 也意味着某些执行将被拒绝和丢失。如果您需要确保所有执行始终成功，这可能是个问题。如果需要确保所有执行都被排队等待执行，但通过节流延迟来减慢执行速率，请考虑使用[队列 (queueing)](../guides/queueing)。

## TanStack Pacer 中的速率限制

TanStack Pacer 通过 `RateLimiter` 和 `AsyncRateLimiter` 类（及其对应的 `rateLimit` 和 `asyncRateLimit` 函数）提供同步和异步速率限制。

### 使用 `rateLimit` 的基本用法

`rateLimit` 函数是为任何函数添加速率限制的最简单方法。它非常适合大多数只需要强制执行简单限制的用例。

```ts
import { rateLimit } from '@tanstack/pacer'

// 将 API 调用限制为每分钟 5 次
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 分钟，以毫秒为单位
    windowType: 'fixed', // 默认
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// 前 5 次调用将立即执行
rateLimitedApi('user-1') // ✅ 执行
rateLimitedApi('user-2') // ✅ 执行
rateLimitedApi('user-3') // ✅ 执行
rateLimitedApi('user-4') // ✅ 执行
rateLimitedApi('user-5') // ✅ 执行
rateLimitedApi('user-6') // ❌ 拒绝，直到窗口重置
```

### 使用 `RateLimiter` 类的高级用法

对于需要额外控制速率限制行为的更复杂场景，可以直接使用 `RateLimiter` 类。这使您可以访问其他方法和状态信息。

```ts
import { RateLimiter } from '@tanstack/pacer'

// 创建速率限制器实例
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Function executed', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// 获取当前状态信息
console.log(limiter.getRemainingInWindow()) // 当前窗口中剩余的调用次数
console.log(limiter.getExecutionCount()) // 成功执行的总次数
console.log(limiter.getRejectionCount()) // 被拒绝执行的总次数

// 尝试执行（返回表示成功的布尔值）
limiter.maybeExecute('user-1')

// 动态更新选项
limiter.setOptions({ limit: 10 }) // 增加限制

// 重置所有计数器和状态
limiter.reset()
```

### 启用/禁用

`RateLimiter` 类通过 `enabled` 选项支持启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用速率限制器：

> [!NOTE]
> `enabled` 选项启用/禁用实际函数执行。禁用速率限制器并不会关闭速率限制，它只是完全阻止函数的执行。

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // 默认禁用
})
limiter.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是框架适配器，其中速率限制器选项是响应式的，您可以将 `enabled` 选项设置为条件值以动态启用/禁用速率限制器。但是，如果您直接使用 `rateLimit` 函数或 `RateLimiter` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `RateLimiter` 类的构造函数。

### 回调选项

同步和异步速率限制器都支持回调选项来处理速率限制生命周期的不同方面：

#### 同步速率限制器回调

同步 `RateLimiter` 支持以下回调：

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 每次成功执行后调用
    console.log('Function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 当执行被拒绝时调用
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

`onExecute` 回调在速率限制函数每次成功执行后调用，而 `onReject` 回调在因速率限制而拒绝执行时调用。这些回调对于跟踪执行、更新 UI 状态或向用户提供反馈非常有用。

#### 异步速率限制器回调

异步 `AsyncRateLimiter` 支持额外的错误处理回调：

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 每次成功执行后调用
    console.log('Async function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 当执行被拒绝时调用
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // 如果异步函数抛出错误则调用
    console.error('Async function failed:', error)
  }
})
```

`onExecute` 和 `onReject` 回调的工作方式与同步速率限制器中的相同，而 `onError` 回调允许您优雅地处理错误而不中断速率限制链。这些回调对于跟踪执行计数、更新 UI 状态、处理错误和向用户提供反馈特别有用。

### 异步速率限制

异步速率限制器提供了一种强大的方法来处理带有限速的异步操作，与同步版本相比具有几个关键优势。虽然同步速率限制器非常适合 UI 事件和即时反馈，但异步版本专门设计用于处理 API 调用、数据库操作和其他异步任务。

#### 与同步速率限制的主要区别

1. **返回值处理**
与返回表示成功的布尔值的同步速率限制器不同，异步版本允许您捕获和使用来自速率限制函数的返回值。这在您需要使用 API 调用或其他异步操作的结果时特别有用。`maybeExecute` 方法返回一个 Promise，该 Promise 解析为函数的返回值，允许您等待结果并适当处理它。

2. **不同的回调**
`AsyncRateLimiter` 支持以下回调，而不仅仅是同步版本中的 `onExecute`：
- `onSuccess`：每次成功执行后调用，提供速率限制器实例
- `onSettled`：每次执行后调用，提供速率限制器实例
- `onError`：如果异步函数抛出错误则调用，提供错误和速率限制器实例

异步和同步速率限制器都支持 `onReject` 回调来处理被阻止的执行。

3. **顺序执行**
由于速率限制器的 `maybeExecute` 方法返回一个 Promise，您可以选择在开始下一个执行之前等待每个执行。这使您可以控制执行顺序，并确保每次调用处理最新的数据。这在处理依赖于先前调用结果的操作或维护数据一致性至关重要时特别有用。

例如，如果您正在更新用户配置文件然后立即获取其更新后的数据，您可以在开始获取之前等待更新操作：

#### 基本用法示例

以下是显示如何将异步速率限制器用于 API 操作的基本示例：

```ts
const rateLimitedApi = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onExecute: (limiter) => {
      console.log('API call succeeded:', limiter.getExecutionCount())
    },
    onReject: (limiter) => {
      console.log(`Rate limit exceeded. Try again in ${limiter.getMsUntilNextWindow()}ms`)
    },
    onError: (error, limiter) => {
      console.error('API call failed:', error)
    }
  }
)

// 用法
const result = await rateLimitedApi('123')
```

### 框架适配器

每个框架适配器都提供构建在核心速率限制功能之上的钩子，以与框架的状态管理系统集成。每个框架都有可用的钩子，如 `createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState` 或 `useRateLimitedValue`。

以下是一些示例：

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// 用于完全控制的低级钩子
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用于基本用例的简单回调钩子
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用于响应式状态管理的基于状态的钩子
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // 要限制速率的值
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// 用于完全控制的低级钩子
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用于状态管理的基于信号的钩子
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
