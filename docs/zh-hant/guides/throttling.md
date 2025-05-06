---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:06:13.842Z'
title: 節流指南
id: throttling
---
節流控制指南 (Throttling Guide)

速率限制 (Rate Limiting)、節流控制 (Throttling) 和防抖動 (Debouncing) 是三種控制函數執行頻率的技術。每種技術以不同方式阻擋執行，屬於「有損」機制 — 這意味當函數被頻繁呼叫時，部分呼叫將不會執行。了解何時使用每種技術對於構建高效可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的節流控制概念。

## 節流控制概念

節流控制確保函數執行在時間上均勻分佈。與允許短時間內突發執行的速率限制不同，也不同於等待活動停止的防抖動技術，節流控制通過在呼叫之間強制保持一致的間隔來創建更平滑的執行模式。如果您設定每秒執行一次的節流，無論呼叫請求多麼頻繁，執行都會保持均勻間隔。

### 節流控制視覺化

```text
節流控制 (每 3 個 tick 執行一次)
時間軸: [每秒一個 tick]
呼叫:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
實際執行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 個 tick 只允許執行一次，
               無論進行了多少次呼叫

             [首次突發]    [更多呼叫]              [間隔呼叫]
             首次執行後     等待週期結束後         每次等待週期
             開始節流       執行                  結束後執行
```

### 何時使用節流控制

當您需要一致且可預測的執行時機時，節流控制特別有效。這使其成為處理頻繁事件或更新的理想選擇，可實現平滑可控的行為。

常見使用場景包括：
- 需要定時更新的 UI 元件（例如進度指示器）
- 不應讓瀏覽器過載的滾動或調整大小事件處理程序
- 需要固定間隔的即時資料輪詢
- 需要穩定節奏的資源密集型操作
- 遊戲循環更新或動畫幀處理
- 使用者輸入時的即時搜尋建議

### 何時不應使用節流控制

以下情況可能不適合使用節流控制：
- 您希望等待活動停止（改用 [防抖動](../guides/debouncing)）
- 不能承受任何執行遺漏（改用 [佇列處理](../guides/queueing)）

> [!TIP]
> 當您需要平滑一致的執行時機時，節流控制通常是最佳選擇。它比速率限制提供更可預測的執行模式，比防抖動提供更即時的回饋。

## TanStack Pacer 中的節流控制

TanStack Pacer 通過 `Throttler` 和 `AsyncThrottler` 類別（及其對應的 `throttle` 和 `asyncThrottle` 函數）提供同步和非同步節流控制功能。

### 使用 `throttle` 的基本用法

`throttle` 函數是為任何函數添加節流控制的最簡單方式：

```ts
import { throttle } from '@tanstack/pacer'

// 將 UI 更新節流為每 200 毫秒一次
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 在快速循環中，只會每 200 毫秒執行一次
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 多數呼叫會被節流
}
```

### 使用 `Throttler` 類別的高級用法

要更精確控制節流行為，可以直接使用 `Throttler` 類別：

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 獲取執行狀態資訊
console.log(updateThrottler.getExecutionCount()) // 成功執行次數
console.log(updateThrottler.getLastExecutionTime()) // 最後執行時間戳

// 取消任何待處理的執行
updateThrottler.cancel()
```

### 前緣與後緣執行

同步節流器支援前緣和後緣執行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 首次呼叫立即執行 (預設)
  trailing: true,  // 等待週期後執行 (預設)
})
```

- `leading: true` (預設) - 首次呼叫立即執行
- `leading: false` - 跳過首次呼叫，等待後緣執行
- `trailing: true` (預設) - 等待週期後執行最後一次呼叫
- `trailing: false` - 如果在等待週期內則跳過最後一次呼叫

常見模式：
- `{ leading: true, trailing: true }` - 預設，回應最靈敏
- `{ leading: false, trailing: true }` - 延遲所有執行
- `{ leading: true, trailing: false }` - 跳過排隊的執行

### 啟用/停用功能

`Throttler` 類別通過 `enabled` 選項支援啟用/停用功能。使用 `setOptions` 方法可以隨時啟用或停用節流器：

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // 預設停用
throttler.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的框架適配器支援響應式節流器選項，可以將 `enabled` 選項設為條件值以動態啟用/停用節流器。但是，如果直接使用 `throttle` 函數或 `Throttler` 類別，則必須使用 `setOptions` 方法更改 `enabled` 選項，因為傳遞的選項實際上是傳遞給 `Throttler` 類別的建構函數。

### 回呼選項

同步和非同步節流器都支援回呼選項來處理節流生命週期的不同方面：

#### 同步節流器回呼

同步 `Throttler` 支援以下回呼：

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // 每次成功執行後呼叫
    console.log('函數已執行', throttler.getExecutionCount())
  }
})
```

`onExecute` 回呼在每次成功執行節流函數後呼叫，可用於追蹤執行次數、更新 UI 狀態或執行清理操作。

#### 非同步節流器回呼

非同步 `AsyncThrottler` 支援額外的錯誤處理回呼：

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // 每次成功執行後呼叫
    console.log('非同步函數已執行', throttler.getExecutionCount())
  },
  onError: (error) => {
    // 非同步函數拋出錯誤時呼叫
    console.error('非同步函數執行失敗:', error)
  }
})
```

`onExecute` 回呼的工作方式與同步節流器相同，而 `onError` 回呼允許您優雅地處理錯誤而不中斷節流鏈。這些回呼特別適用於追蹤執行次數、更新 UI 狀態、處理錯誤、執行清理操作和記錄執行指標。

### 非同步節流控制

非同步節流器提供了一種強大的方式來處理非同步操作的節流控制，相比同步版本具有幾個關鍵優勢。雖然同步節流器適用於 UI 事件和即時回饋，但非同步版本專為處理 API 呼叫、資料庫操作和其他非同步任務而設計。

#### 與同步節流的主要區別

1. **返回值處理**
與返回 void 的同步節流器不同，非同步版本允許您捕獲和使用節流函數的返回值。這在需要處理 API 呼叫結果或其他非同步操作時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函數的返回值，允許您等待結果並適當處理。

2. **增強的回呼系統**
非同步節流器提供比同步版本的單一 `onExecute` 回呼更複雜的回呼系統。該系統包括：
- `onSuccess`: 非同步函數成功完成時呼叫，提供結果和節流器實例
- `onError`: 非同步函數拋出錯誤時呼叫，提供錯誤和節流器實例
- `onSettled`: 每次執行嘗試後呼叫，無論成功與否

3. **執行追蹤**
非同步節流器通過多種方法提供全面的執行追蹤：
- `getSuccessCount()`: 成功執行次數
- `getErrorCount()`: 失敗執行次數
- `getSettledCount()`: 已完成的執行總數 (成功 + 失敗)

4. **順序執行**
非同步節流器確保後續執行等待前一次呼叫完成後才開始。這可防止執行順序錯亂，並保證每次呼叫處理的都是最新資料。這在處理依賴前次呼叫結果的操作或維護資料一致性至關重要時特別重要。

例如，如果您正在更新使用者個人資料並立即獲取其更新後的資料，非同步節流器將確保獲取操作等待更新完成，防止可能獲取過時資料的競爭條件。

#### 基本用法範例

以下是展示如何使用非同步節流器進行搜尋操作的基本範例：

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('搜尋成功:', results)
    },
    onError: (error, throttler) => {
      console.error('搜尋失敗:', error)
    }
  }
)

// 用法
const results = await throttledSearch('query')
```

#### 高級模式

非同步節流器可以與各種模式結合以解決複雜問題：

1. **狀態管理整合**
當將非同步節流器與狀態管理系統（如 React 的 useState 或 Solid 的 createSignal）結合使用時，可以創建強大的模式來處理載入狀態、錯誤狀態和資料更新。節流器的回呼提供了根據操作成功或失敗更新 UI 狀態的完美鉤子。

2. **競爭條件預防**
節流模式自然預防了許多場景中的競爭條件。當應用程式的多個部分嘗試同時更新同一資源時，節流器確保更新以受控速率進行，同時仍向所有呼叫者提供結果。

3. **錯誤恢復**
非同步節流器的錯誤處理能力使其成為實現重試邏輯和錯誤恢復模式的理想選擇。您可以使用 `onError` 回呼實現自定義錯誤處理策略，例如指數退避或回退機制。

### 框架適配器

每個框架適配器都提供建立在核心節流功能之上的鉤子，以與框架的狀態管理系統整合。每個框架都提供如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue` 等鉤子。

以下是一些範例：

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 用於完全控制的低階鉤子
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於基本用例的簡單回呼鉤子
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於響應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // 要節流的值
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// 用於完全控制的低階鉤子
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於狀態管理的基於信號的鉤子
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('總執行次數:', throttler.getExecutionCount())
  }
})
```

每個框架適配器都提供與框架狀態管理系統整合的鉤子，同時保持核心節流功能。
