---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:06:28.311Z'
title: 速率限制指南
id: rate-limiting
---
# 速率限制指南 (Rate Limiting Guide)

速率限制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是三種控制函數執行頻率的不同方法。每種技術以不同的方式阻擋執行，使它們具有「損耗性」— 這意味著當函數被要求過於頻繁執行時，某些呼叫將不會執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的速率限制概念。

> [!NOTE]
> TanStack Pacer 目前僅是一個前端函式庫 (front-end library)。這些是用於客戶端速率限制 (client-side rate-limiting) 的實用工具。

## 速率限制概念 (Rate Limiting Concept)

速率限制是一種技術，用於限制函數在特定時間窗口內可以執行的速率。它特別適用於您希望防止函數被過於頻繁呼叫的場景，例如處理 API 請求或其他外部服務呼叫時。這是最*基礎* (naive) 的方法，因為它允許執行在達到配額前以突發方式發生。

### 速率限制視覺化 (Rate Limiting Visualization)

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### 何時使用速率限制 (When to Use Rate Limiting)

速率限制在處理可能意外壓垮後端服務或導致瀏覽器效能問題的前端操作時特別重要。

常見使用案例包括：
- 防止快速使用者互動 (如按鈕點擊或表單提交) 導致的意外 API 濫發 (API spam)
- 可接受突發行為但希望限制最大速率的場景
- 防止意外無限迴圈或遞迴操作

### 何時不應使用速率限制 (When Not to Use Rate Limiting)

速率限制是控制函數執行頻率最基礎的方法。它是這三種技術中最不靈活且限制最多的。考慮使用[節流](../guides/throttling)或[防抖](../guides/debouncing)來獲得更分散的執行。

> [!TIP]
> 在大多數使用案例中，您可能不希望使用「速率限制」。考慮改用[節流](../guides/throttling)或[防抖](../guides/debouncing)。

速率限制的「損耗性」也意味著某些執行將被拒絕並丟失。如果您需要確保所有執行始終成功，這可能會成為問題。考慮使用[佇列](../guides/queueing)，如果您需要確保所有執行都被排隊等待執行，但通過節流延遲來減慢執行速率。

## TanStack Pacer 中的速率限制 (Rate Limiting in TanStack Pacer)

TanStack Pacer 通過 `RateLimiter` 和 `AsyncRateLimiter` 類別 (及其對應的 `rateLimit` 和 `asyncRateLimit` 函數) 提供同步和非同步速率限制功能。

### 使用 `rateLimit` 的基本用法 (Basic Usage with `rateLimit`)

`rateLimit` 函數是為任何函數添加速率限制的最簡單方法。它非常適合大多數只需要執行簡單限制的使用案例。

```ts
import { rateLimit } from '@tanstack/pacer'

// 將 API 呼叫限制為每分鐘 5 次
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 分鐘，以毫秒為單位
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

### 使用 `RateLimiter` 類別的高級用法 (Advanced Usage with `RateLimiter` Class)

對於需要對速率限制行為進行額外控制的更複雜場景，您可以直接使用 `RateLimiter` 類別。這使您能夠訪問其他方法和狀態資訊。

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

// 獲取當前狀態資訊
console.log(limiter.getRemainingInWindow()) // 當前窗口中剩餘的呼叫次數
console.log(limiter.getExecutionCount()) // 成功執行的總次數
console.log(limiter.getRejectionCount()) // 被拒絕執行的總次數

// 嘗試執行 (返回表示成功的布林值)
limiter.maybeExecute('user-1')

// 動態更新選項
limiter.setOptions({ limit: 10 }) // 增加限制

// 重置所有計數器和狀態
limiter.reset()
```

### 啟用/停用 (Enabling/Disabling)

`RateLimiter` 類別通過 `enabled` 選項支援啟用/停用功能。使用 `setOptions` 方法，您可以隨時啟用/停用速率限制器：

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // 預設停用
})
limiter.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是框架適配器 (framework adapter)，其中速率限制器選項是響應式的 (reactive)，您可以將 `enabled` 選項設置為條件值以動態啟用/停用速率限制器。但是，如果您直接使用 `rateLimit` 函數或 `RateLimiter` 類別，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `RateLimiter` 類別的建構函數。

### 回調選項 (Callback Options)

同步和非同步速率限制器都支援回調選項來處理速率限制生命週期的不同方面：

#### 同步速率限制器回調 (Synchronous Rate Limiter Callbacks)

同步 `RateLimiter` 支援以下回調：

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

`onExecute` 回調在每次成功執行速率限制函數後呼叫，而 `onReject` 回調在因速率限制而拒絕執行時呼叫。這些回調對於追蹤執行、更新 UI 狀態或向使用者提供反饋非常有用。

#### 非同步速率限制器回調 (Asynchronous Rate Limiter Callbacks)

非同步 `AsyncRateLimiter` 支援用於錯誤處理的額外回調：

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
    // 如果非同步函數拋出錯誤時呼叫
    console.error('Async function failed:', error)
  }
})
```

`onExecute` 和 `onReject` 回調的工作方式與同步速率限制器相同，而 `onError` 回調允許您優雅地處理錯誤而不中斷速率限制鏈。這些回調對於追蹤執行計數、更新 UI 狀態、處理錯誤和向使用者提供反饋特別有用。

### 非同步速率限制 (Asynchronous Rate Limiting)

非同步速率限制器提供了一種強大的方式來處理具有速率限制的非同步操作，與同步版本相比具有幾個關鍵優勢。雖然同步速率限制器非常適合 UI 事件和即時反饋，但非同步版本專門設計用於處理 API 呼叫、資料庫操作和其他非同步任務。

#### 與同步速率限制的主要差異 (Key Differences from Synchronous Rate Limiting)

1. **返回值處理**
與同步速率限制器返回表示成功的布林值不同，非同步版本允許您捕獲並使用來自速率限制函數的返回值。這在您需要處理 API 呼叫或其他非同步操作的結果時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函數的返回值，允許您等待結果並適當處理。

2. **增強的回調系統**
非同步速率限制器提供比同步版本更複雜的回調系統。此系統包括：
- `onExecute`：每次成功執行後呼叫，提供速率限制器實例
- `onReject`：當因速率限制而拒絕執行時呼叫，提供速率限制器實例
- `onError`：如果非同步函數拋出錯誤時呼叫，提供錯誤和速率限制器實例

3. **執行追蹤**
非同步速率限制器通過以下幾種方法提供全面的執行追蹤：
- `getExecutionCount()`：成功執行的次數
- `getRejectionCount()`：被拒絕執行的次數
- `getRemainingInWindow()`：當前窗口中剩餘的執行次數
- `getMsUntilNextWindow()`：距離下一個窗口開始的毫秒數

4. **順序執行**
非同步速率限制器確保後續執行等待前一次呼叫完成後才開始。這防止了執行順序錯亂，並保證每次呼叫處理最新的資料。這在處理依賴於先前呼叫結果的操作或維護資料一致性至關重要時特別重要。

例如，如果您正在更新使用者的個人資料，然後立即獲取其更新後的資料，非同步速率限制器將確保獲取操作等待更新完成，防止可能獲取過時資料的競爭條件 (race conditions)。

#### 基本用法範例 (Basic Usage Example)

這是一個基本範例，展示如何將非同步速率限制器用於 API 操作：

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

// 使用方式
const result = await rateLimitedApi('123')
```

#### 高級模式 (Advanced Patterns)

非同步速率限制器可以與各種模式結合以解決複雜問題：

1. **狀態管理整合**
當將非同步速率限制器與狀態管理系統 (如 React 的 useState 或 Solid 的 createSignal) 一起使用時，您可以創建強大的模式來處理載入狀態、錯誤狀態和資料更新。速率限制器的回調為根據操作的成功或失敗更新 UI 狀態提供了完美的鉤子。

2. **競爭條件預防**
速率限制模式自然防止了許多場景中的競爭條件。當應用程式的多個部分嘗試同時更新同一資源時，速率限制器確保更新在配置的限制內發生，同時仍向所有呼叫者提供結果。

3. **錯誤恢復**
非同步速率限制器的錯誤處理能力使其成為實現重試邏輯和錯誤恢復模式的理想選擇。您可以使用 `onError` 回調來實現自訂錯誤處理策略，例如指數退避 (exponential backoff) 或回退機制 (fallback mechanisms)。

### 框架適配器 (Framework Adapters)

每個框架適配器都提供建立在核心速率限制功能之上的鉤子 (hooks)，以與框架的狀態管理系統集成。每個框架都提供如 `createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState` 或 `useRateLimitedValue` 等鉤子。

以下是一些範例：

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// 用於完全控制的低階鉤子
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 用於基本使用案例的簡單回調鉤子
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 基於狀態的鉤子，用於響應式狀態管理
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // 要進行速率限制的值
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// 用於完全控制的低階鉤子
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 基於信號 (Signal) 的鉤子，用於狀態管理
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
