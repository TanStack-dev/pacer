---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:25:00.387Z'
title: 速率限制指南
id: rate-limiting
---
速率限制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是三種控制函式執行頻率的不同方法。每種技術以不同方式阻擋執行，使其具有「損耗性」——意味著當函式被要求過於頻繁執行時，部分呼叫將不會執行。了解何時使用每種方法對於建構高效能且可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的速率限制概念。

> [!NOTE]
> TanStack Pacer 目前僅是一個前端函式庫。這些是用於客戶端速率限制的實用工具。

## 速率限制概念

速率限制是一種技術，用於限制函式在特定時間窗口內的執行速率。它特別適用於需要防止函式被過於頻繁呼叫的場景，例如處理 API 請求或其他外部服務呼叫。這是最「基礎」的方法，因為它允許執行在達到配額前以突發方式進行。

### 速率限制視覺化

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### 何時使用速率限制

速率限制在處理可能意外壓垮後端服務或導致瀏覽器效能問題的前端操作時特別重要。

常見使用案例包括：
- 防止快速使用者互動（例如按鈕點擊或表單提交）導致的意外 API 濫發
- 可接受突發行為但需要限制最大速率的場景
- 防止意外無限迴圈或遞迴操作

### 何時不應使用速率限制

速率限制是控制函式執行頻率最基礎的方法。它是這三種技術中最不靈活且限制最多的。考慮改用[節流](../guides/throttling)或[防抖](../guides/debouncing)以獲得更分散的執行。

> [!TIP]
> 大多數情況下，您可能不希望使用「速率限制」。考慮改用[節流](../guides/throttling)或[防抖](../guides/debouncing)。

速率限制的「損耗性」也意味著部分執行將被拒絕並丟失。如果您需要確保所有執行始終成功，這可能會成為問題。考慮使用[佇列](../guides/queueing)來確保所有執行都被排隊等待執行，但透過節流延遲來減緩執行速率。

## TanStack Pacer 中的速率限制

TanStack Pacer 分別透過 `RateLimiter` 和 `AsyncRateLimiter` 類別（及其對應的 `rateLimit` 和 `asyncRateLimit` 函式）提供同步和非同步速率限制功能。

### 使用 `rateLimit` 的基本用法

`rateLimit` 函式是為任何函式添加速率限制的最簡單方法。它非常適合大多數只需強制執行簡單限制的使用案例。

```ts
import { rateLimit } from '@tanstack/pacer'

// 將 API 呼叫限制為每分鐘 5 次
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 分鐘（毫秒）
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// 前 5 次呼叫將立即執行
rateLimitedApi('user-1') // ✅ 執行
rateLimitedApi('user-2') // ✅ 執行
rateLimitedApi('user-3') // ✅ 執行
rateLimitedApi('user-4') // ✅ 執行
rateLimitedApi('user-5') // ✅ 執行
rateLimitedApi('user-6') // ❌ 拒絕，直到窗口重置
```

### 使用 `RateLimiter` 類別的高級用法

對於需要額外控制速率限制行為的更複雜場景，您可以直接使用 `RateLimiter` 類別。這讓您可以存取其他方法和狀態資訊。

```ts
import { RateLimiter } from '@tanstack/pacer'

// 建立速率限制器實例
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

// 取得當前狀態的資訊
console.log(limiter.getRemainingInWindow()) // 當前窗口中剩餘的呼叫次數
console.log(limiter.getExecutionCount()) // 成功執行的總次數
console.log(limiter.getRejectionCount()) // 被拒絕的執行總次數

// 嘗試執行（返回表示成功與否的布林值）
limiter.maybeExecute('user-1')

// 動態更新選項
limiter.setOptions({ limit: 10 }) // 提高限制

// 重置所有計數器和狀態
limiter.reset()
```

### 啟用/停用

`RateLimiter` 類別支援透過 `enabled` 選項啟用/停用。使用 `setOptions` 方法，您可以隨時啟用/停用速率限制器：

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // 預設停用
})
limiter.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是框架適配器，其中速率限制器選項是響應式的，您可以將 `enabled` 選項設為條件值以動態啟用/停用速率限制器。然而，如果您直接使用 `rateLimit` 函式或 `RateLimiter` 類別，則必須使用 `setOptions` 方法來變更 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `RateLimiter` 類別的建構函式。

### 回呼選項

同步和非同步速率限制器都支援回呼選項，以處理速率限制生命週期的不同方面：

#### 同步速率限制器回呼

同步 `RateLimiter` 支援以下回呼：

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 每次成功執行後呼叫
    console.log('Function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 當執行被拒絕時呼叫
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

`onExecute` 回呼在每次成功執行速率限制函式後呼叫，而 `onReject` 回呼在執行因速率限制被拒絕時呼叫。這些回呼對於追蹤執行、更新 UI 狀態或向使用者提供回饋非常有用。

#### 非同步速率限制器回呼

非同步 `AsyncRateLimiter` 支援額外的錯誤處理回呼：

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 每次成功執行後呼叫
    console.log('Async function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 當執行被拒絕時呼叫
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // 如果非同步函式拋出錯誤時呼叫
    console.error('Async function failed:', error)
  }
})
```

`onExecute` 和 `onReject` 回呼的工作方式與同步速率限制器相同，而 `onError` 回呼讓您可以優雅地處理錯誤而不中斷速率限制鏈。這些回呼對於追蹤執行次數、更新 UI 狀態、處理錯誤和向使用者提供回饋特別有用。

### 非同步速率限制

在以下情況使用 `AsyncRateLimiter`：
- 您的速率限制函式返回 Promise
- 您需要處理非同步函式的錯誤
- 您希望確保即使非同步函式需要時間完成，也能正確進行速率限制

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
      console.error('API call failed:', error)
    }
  }
)

// 返回 Promise<boolean> - 解析為 true 表示已執行，false 表示被拒絕
const wasExecuted = await rateLimited('123')
```

非同步版本提供基於 Promise 的執行追蹤、透過 `onError` 回呼的錯誤處理、待處理非同步操作的正確清理，以及可等待的 `maybeExecute` 方法。

### 框架適配器

每個框架適配器都提供建立在核心速率限制功能之上的鉤子，以與框架的狀態管理系統整合。每個框架都有可用的鉤子，例如 `createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState` 或 `useRateLimitedValue`。

以下是一些範例：

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// 用於完整控制的低階鉤子
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用於基本使用案例的簡單回呼鉤子
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用於響應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // 要進行速率限制的值
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// 用於完整控制的低階鉤子
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用於狀態管理的基於 Signal 的鉤子
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
