---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:19:47.836Z'
title: 限流指南
id: rate-limiting
---
# 速率限制 (Rate Limiting) 指南

速率限制 (Rate Limiting)、节流 (Throttling) 和防抖 (Debouncing) 是控制函数执行频率的三种不同方法。每种技术以不同方式阻止执行，使它们具有"有损性"——意味着当函数被请求过于频繁时，某些调用将不会执行。了解何时使用每种方法对于构建高性能和可靠的应用程序至关重要。本指南将介绍 TanStack Pacer 的速率限制概念。

> [!NOTE]
> TanStack Pacer 目前仅是一个前端库。这些是用于客户端速率限制的实用工具。

## 速率限制概念

速率限制是一种技术，用于限制函数在特定时间窗口内的执行速率。它特别适用于需要防止函数被过于频繁调用的场景，例如处理 API 请求或其他外部服务调用。这是最*简单*的方法，因为它允许执行突发，直到达到配额为止。

### 速率限制可视化

```text
速率限制 (限制: 每个窗口 3 次调用)
时间线: [每秒一个刻度]
                                        窗口 1                  |    窗口 2            
调用:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
已执行:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 允许 3 次 ===][=== 阻止直到窗口结束 ===][=== 新窗口 =======]
```

### 何时使用速率限制

速率限制在处理可能意外压倒后端服务或导致浏览器性能问题的前端操作时特别重要。

常见用例包括：
- 防止快速用户交互（如按钮点击或表单提交）导致的意外 API 滥用
- 可接受突发行为但希望限制最大速率的场景
- 防止意外的无限循环或递归操作

### 何时不使用速率限制

速率限制是控制函数执行频率的最简单方法。它是这三种技术中最不灵活且限制最多的。考虑使用[节流](../guides/throttling)或[防抖](../guides/debouncing)来获得更分散的执行。

> [!TIP]
> 对于大多数用例，您可能不想使用"速率限制"。考虑使用[节流](../guides/throttling)或[防抖](../guides/debouncing)替代。

速率限制的"有损"特性也意味着某些执行将被拒绝和丢失。如果您需要确保所有执行始终成功，这可能是个问题。如果需要确保所有执行都被排队等待执行，但通过节流延迟来减慢执行速率，请考虑使用[队列](../guides/queueing)。

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
    onReject: (rateLimiter) => {
      console.log(`超过速率限制。请在 ${rateLimiter.getMsUntilNextWindow()}ms 后重试`)
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
      console.log('函数已执行', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`超过速率限制。请在 ${rateLimiter.getMsUntilNextWindow()}ms 后重试`)
    }
  }
)

// 获取当前状态信息
console.log(limiter.getRemainingInWindow()) // 当前窗口中剩余的调用次数
console.log(limiter.getExecutionCount()) // 成功执行的总次数
console.log(limiter.getRejectionCount()) // 被拒绝执行的总次数

// 尝试执行（返回布尔值表示成功）
limiter.maybeExecute('user-1')

// 动态更新选项
limiter.setOptions({ limit: 10 }) // 增加限制

// 重置所有计数器和状态
limiter.reset()
```

### 启用/禁用

`RateLimiter` 类支持通过 `enabled` 选项启用/禁用。使用 `setOptions` 方法，您可以随时启用/禁用速率限制器：

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // 默认禁用
})
limiter.setOptions({ enabled: true }) // 随时启用
```

如果您使用的是框架适配器，其中速率限制器选项是响应式的，您可以将 `enabled` 选项设置为条件值以动态启用/禁用速率限制器。但是，如果您使用的是 `rateLimit` 函数或直接使用 `RateLimiter` 类，则必须使用 `setOptions` 方法来更改 `enabled` 选项，因为传递的选项实际上是传递给 `RateLimiter` 类的构造函数。

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
    console.log('函数已执行', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 当执行被拒绝时调用
    console.log(`超过速率限制。请在 ${rateLimiter.getMsUntilNextWindow()}ms 后重试`)
  }
})
```

`onExecute` 回调在每次成功执行速率限制函数后调用，而 `onReject` 回调在由于速率限制而拒绝执行时调用。这些回调对于跟踪执行、更新 UI 状态或向用户提供反馈非常有用。

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
    console.log('异步函数已执行', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 当执行被拒绝时调用
    console.log(`超过速率限制。请在 ${rateLimiter.getMsUntilNextWindow()}ms 后重试`)
  },
  onError: (error) => {
    // 如果异步函数抛出错误时调用
    console.error('异步函数失败:', error)
  }
})
```

`onExecute` 和 `onReject` 回调的工作方式与同步速率限制器相同，而 `onError` 回调允许您优雅地处理错误而不中断速率限制链。这些回调对于跟踪执行计数、更新 UI 状态、处理错误和向用户提供反馈特别有用。

### 异步速率限制

在以下情况下使用 `AsyncRateLimiter`：
- 您的速率限制函数返回 Promise
- 您需要处理来自异步函数的错误
- 您希望确保即使异步函数需要时间完成也能正确进行速率限制

```ts
import { asyncRateLimit } from '@tanstack/pacer'

const rateLimited = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onError: (error) => {
      console.error('API 调用失败:', error)
    }
  }
)

// 返回 Promise<boolean> - 解析为 true 表示已执行，false 表示被拒绝
const wasExecuted = await rateLimited('123')
```

异步版本提供基于 Promise 的执行跟踪、通过 `onError` 回调的错误处理、待处理异步操作的正确清理以及可等待的 `maybeExecute` 方法。

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
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // 要速率限制的值
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
    console.log('总执行次数:', limiter.getExecutionCount())
  }
})
```
