---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:47:45.610Z'
title: 速率限制指南
id: rate-limiting
---
# 速率限制指南 (Rate Limiting Guide)

TanStack Pacer 是一個專注於提供高品質工具函式庫，用於控制應用程式中函式執行時機。雖然其他地方也有類似工具，但我們的目標是確保所有重要細節都正確無誤 - 包括 ***型別安全 (type-safety)***、***樹搖優化 (tree-shaking)*** 以及一致且 ***直觀的 API (intuitive API)***。通過專注於這些基礎並以 ***框架無關 (framework agnostic)*** 的方式提供，我們希望這些工具和模式能在您的應用程式中更普及。在應用程式開發中，適當的執行控制常常被忽視，導致可能預防的效能問題、競爭條件 (race conditions) 和使用者體驗不佳。TanStack Pacer 幫助您從一開始就正確實現這些關鍵模式！

速率限制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是三種控制函式執行頻率的截然不同的方法。每種技術以不同方式阻擋執行，使它們成為「有損耗的 (lossy)」- 這意味著當函式被要求過於頻繁執行時，某些呼叫將不會執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將涵蓋 TanStack Pacer 的速率限制概念。

> [!NOTE]
> TanStack Pacer 目前僅是一個前端函式庫。這些是用於客戶端速率限制的工具。

## 速率限制概念 (Rate Limiting Concept)

速率限制是一種技術，限制函式在特定時間窗口內可以執行的速率。它特別適用於您希望防止函式被過於頻繁呼叫的場景，例如處理 API 請求或其他外部服務呼叫時。這是最 *基礎 (naive)* 的方法，因為它允許執行在達到配額前以突發方式發生。

### 速率限制視覺化 (Rate Limiting Visualization)

```text
速率限制 (限制: 每個窗口 3 次呼叫)
時間軸: [每秒一個刻度]
                                        窗口 1                  |    窗口 2            
呼叫:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
已執行:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 允許 3 次 ===][=== 阻擋直到窗口結束 ===][=== 新窗口 =======]
```

### 窗口類型 (Window Types)

TanStack Pacer 支援兩種速率限制窗口類型：

1. **固定窗口 (Fixed Window)** (預設)
   - 嚴格的窗口，在窗口期後重置
   - 窗口內的所有執行都計入限制
   - 窗口期後完全重置
   - 可能在窗口邊界處導致突發行為

2. **滑動窗口 (Sliding Window)**
   - 滾動窗口，允許舊執行到期後新執行
   - 隨時間提供更一致的執行速率
   - 更適合維持穩定的執行流
   - 防止窗口邊界處的突發行為

以下是滑動窗口速率限制的視覺化：

```text
滑動窗口速率限制 (限制: 每個窗口 3 次呼叫)
時間軸: [每秒一個刻度]
                                        窗口 1                  |    窗口 2            
呼叫:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
已執行:     ✅     ✅     ✅     ❌     ✅                             ✅     ✅
             [=== 允許 3 次 ===][=== 舊執行到期，新執行允許 ===][=== 繼續滑動 =======]
```

關鍵差異在於，使用滑動窗口時，一旦最舊的執行到期，就允許新的執行。這創造了比固定窗口方法更一致的執行流。

### 何時使用速率限制 (When to Use Rate Limiting)

速率限制在處理可能意外壓垮後端服務或導致瀏覽器效能問題的前端操作時特別重要。

常見使用案例包括：
- 防止快速使用者互動（例如按鈕點擊或表單提交）導致的意外 API 濫發
- 可接受突發行為但希望限制最大速率的場景
- 防止意外無限迴圈或遞迴操作

### 何時不使用速率限制 (When Not to Use Rate Limiting)

速率限制是控制函式執行頻率最基礎的方法。它是三種技術中最不靈活且限制最多的。考慮使用 [節流 (throttling)](../guides/throttling) 或 [防抖 (debouncing)](../guides/debouncing) 來實現更分散的執行。

> [!TIP]
> 對於大多數使用案例，您很可能不希望使用「速率限制」。考慮使用 [節流 (throttling)](../guides/throttling) 或 [防抖 (debouncing)](../guides/debouncing) 代替。

速率限制的「有損耗 (lossy)」特性也意味著某些執行將被拒絕並丟失。如果您需要確保所有執行始終成功，這可能會是個問題。考慮使用 [隊列 (queueing)](../guides/queueing) 如果您需要確保所有執行都被排隊等待執行，但通過節流延遲來減慢執行速率。

## TanStack Pacer 中的速率限制 (Rate Limiting in TanStack Pacer)

TanStack Pacer 分別通過 `RateLimiter` 和 `AsyncRateLimiter` 類別（以及它們對應的 `rateLimit` 和 `asyncRateLimit` 函式）提供同步和非同步速率限制。

### 使用 `rateLimit` 的基本用法 (Basic Usage with `rateLimit`)

`rateLimit` 函式是為任何函式添加速率限制的最簡單方法。它非常適合您只需要執行簡單限制的大多數使用案例。

```ts
import { rateLimit } from '@tanstack/pacer'

// 將 API 呼叫限制為每分鐘 5 次
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 分鐘，以毫秒為單位
    windowType: 'fixed', // 預設
    onReject: (rateLimiter) => {
      console.log(`超過速率限制。請在 ${rateLimiter.getMsUntilNextWindow()} 毫秒後重試`)
    }
  }
)

// 前 5 次呼叫將立即執行
rateLimitedApi('user-1') // ✅ 執行
rateLimitedApi('user-2') // ✅ 執行
rateLimitedApi('user-3') // ✅ 執行
rateLimitedApi('user-4') // ✅ 執行
rateLimitedApi('user-5') // ✅ 執行
rateLimitedApi('user-6') // ❌ 拒絕直到窗口重置
```

### 使用 `RateLimiter` 類別的高級用法 (Advanced Usage with `RateLimiter` Class)

對於需要額外控制速率限制行為的更複雜場景，您可以直接使用 `RateLimiter` 類別。這讓您可以訪問其他方法和狀態資訊。

```ts
import { RateLimiter } from '@tanstack/pacer'

// 建立速率限制器實例
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('函式已執行', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`超過速率限制。請在 ${rateLimiter.getMsUntilNextWindow()} 毫秒後重試`)
    }
  }
)

// 獲取當前狀態資訊
console.log(limiter.getRemainingInWindow()) // 當前窗口中剩餘的呼叫次數
console.log(limiter.getExecutionCount()) // 成功執行的總次數
console.log(limiter.getRejectionCount()) // 被拒絕執行的總次數

// 嘗試執行（返回表示成功的布林值）
limiter.maybeExecute('user-1')

// 動態更新選項
limiter.setOptions({ limit: 10 }) // 增加限制

// 重置所有計數器和狀態
limiter.reset()
```

### 啟用/禁用 (Enabling/Disabling)

`RateLimiter` 類別通過 `enabled` 選項支援啟用/禁用。使用 `setOptions` 方法，您可以隨時啟用/禁用速率限制器：

> [!NOTE]
> `enabled` 選項啟用/禁用實際的函式執行。禁用速率限制器並不會關閉速率限制，它只是完全阻止函式執行。

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // 預設禁用
})
limiter.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是框架適配器，其中速率限制器選項是響應式的，您可以將 `enabled` 選項設置為條件值以動態啟用/禁用速率限制器。但是，如果您使用 `rateLimit` 函式或直接使用 `RateLimiter` 類別，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `RateLimiter` 類別的建構函式。

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
    console.log('函式已執行', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 當執行被拒絕時呼叫
    console.log(`超過速率限制。請在 ${rateLimiter.getMsUntilNextWindow()} 毫秒後重試`)
  }
})
```

`onExecute` 回調在速率限制函式每次成功執行後呼叫，而 `onReject` 回調在執行因速率限制被拒絕時呼叫。這些回調對於追蹤執行、更新 UI 狀態或向使用者提供反饋非常有用。

#### 非同步速率限制器回調 (Asynchronous Rate Limiter Callbacks)

非同步 `AsyncRateLimiter` 支援額外的錯誤處理回調：

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 每次成功執行後呼叫
    console.log('非同步函式已執行', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 當執行被拒絕時呼叫
    console.log(`超過速率限制。請在 ${rateLimiter.getMsUntilNextWindow()} 毫秒後重試`)
  },
  onError: (error) => {
    // 如果非同步函式拋出錯誤時呼叫
    console.error('非同步函式失敗:', error)
  }
})
```

`onExecute` 和 `onReject` 回調的工作方式與同步速率限制器相同，而 `onError` 回調允許您優雅地處理錯誤而不中斷速率限制鏈。這些回調對於追蹤執行計數、更新 UI 狀態、處理錯誤和向使用者提供反饋特別有用。

### 非同步速率限制 (Asynchronous Rate Limiting)

非同步速率限制器提供了一種強大的方式來處理帶有速率限制的非同步操作，與同步版本相比具有幾個關鍵優勢。雖然同步速率限制器非常適合 UI 事件和即時反饋，但非同步版本專門設計用於處理 API 呼叫、資料庫操作和其他非同步任務。

#### 與同步速率限制的主要差異 (Key Differences from Synchronous Rate Limiting)

1. **返回值處理 (Return Value Handling)**
與返回表示成功與否的布林值的同步速率限制器不同，非同步版本允許您捕獲並使用來自速率限制函式的返回值。這在您需要使用 API 呼叫或其他非同步操作的結果時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函式的返回值，允許您等待結果並適當地處理它。

2. **不同的回調 (Different Callbacks)**
`AsyncRateLimiter` 支援以下回調，而不僅僅是同步版本中的 `onExecute`：
- `onSuccess`：每次成功執行後呼叫，提供速率限制器實例
- `onSettled`：每次執行後呼叫，提供速率限制器實例
- `onError`：如果非同步函式拋出錯誤時呼叫，提供錯誤和速率限制器實例

非同步和同步速率限制器都支援 `onReject` 回調來處理被阻擋的執行。

3. **順序執行 (Sequential Execution)**
由於速率限制器的 `maybeExecute` 方法返回一個 Promise，您可以選擇在開始下一個執行之前等待每個執行。這讓您可以控制執行順序並確保每個呼叫處理最新的數據。這在處理依賴於先前呼叫結果的操作或當維護數據一致性至關重要時特別有用。

例如，如果您正在更新使用者的個人資料然後立即獲取其更新後的數據，您可以在開始獲取之前等待更新操作：

#### 基本用法範例 (Basic Usage Example)

以下是一個基本範例，展示如何使用非同步速率限制器進行 API 操作：

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
      console.log('API 呼叫成功:', limiter.getExecutionCount())
    },
    onReject: (limiter) => {
      console.log(`超過速率限制。請在 ${limiter.getMsUntilNextWindow()} 毫秒後重試`)
    },
    onError: (error, limiter) => {
      console.error('API 呼叫失敗:', error)
    }
  }
)

// 使用方式
const result = await rateLimitedApi('123')
```

### 框架適配器 (Framework Adapters)

每個框架適配器都提供建立在核心速率限制功能之上的鉤子，以與框架的狀態管理系統集成。每個框架都有可用的鉤子，如 `createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState` 或 `useRateLimitedValue`。

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

// 用於響應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // 要速率限制的值
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

// 用於狀態管理的基於信號的鉤子
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('總執行次數:', limiter.getExecutionCount())
  }
})
```
