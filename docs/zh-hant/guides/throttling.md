---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:24:53.816Z'
title: 節流指南
id: throttling
---
節流 (Throttling) 指南

速率限制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是控制函數執行頻率的三種不同方法。每種技術以不同方式阻擋執行，使它們具有「損耗性」——意味著當函數被要求過於頻繁執行時，某些呼叫將不會執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的節流概念。

## 節流概念

節流確保函數執行在時間上均勻分佈。與允許突發執行直到達到限制的速率限制不同，也不同於等待活動停止的防抖，節流通過在呼叫之間強制一致的延遲來創建更平滑的執行模式。如果將節流設置為每秒執行一次，無論呼叫請求多麼頻繁，呼叫都會均勻分佈。

### 節流可視化

```text
節流 (每 3 個 tick 執行一次)
時間軸: [每秒一個 tick]
呼叫:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
已執行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 個 tick 只允許一次執行，
               無論進行了多少次呼叫

             [第一次突發]    [更多呼叫]              [間隔呼叫]
             首次執行後     等待週期後執行          每次等待週期
             開始節流                          結束後執行
```

### 何時使用節流

當需要一致且可預測的執行時機時，節流特別有效。這使其成為處理頻繁事件或更新的理想選擇，您希望這些事件或更新具有平滑、受控的行為。

常見使用情境包括：
- 需要一致時機的 UI 更新（例如進度指示器）
- 不應壓垮瀏覽器的滾動或調整大小事件處理程序
- 希望保持固定間隔的即時資料輪詢
- 需要穩定節奏的資源密集型操作
- 遊戲循環更新或動畫影格處理
- 使用者輸入時的即時搜尋建議

### 何時不使用節流

在以下情況下，節流可能不是最佳選擇：
- 您希望等待活動停止（改用[防抖](../guides/debouncing)）
- 您不能錯過任何執行（改用[佇列](../guides/queueing)）

> [!TIP]
> 當您需要平滑、一致的執行時機時，節流通常是最佳選擇。它提供比速率限制更可預測的執行模式，並比防抖提供更即時的回饋。

## TanStack Pacer 中的節流

TanStack Pacer 分別通過 `Throttler` 和 `AsyncThrottler` 類（以及對應的 `throttle` 和 `asyncThrottle` 函數）提供同步和非同步節流。

### 使用 `throttle` 的基本用法

`throttle` 函數是為任何函數添加節流的最簡單方法：

```ts
import { throttle } from '@tanstack/pacer'

// 將 UI 更新節流為每 200ms 一次
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 在快速循環中，僅每 200ms 執行一次
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 許多呼叫被節流
}
```

### 使用 `Throttler` 類的高級用法

為了更精確控制節流行為，可以直接使用 `Throttler` 類：

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 獲取執行狀態資訊
console.log(updateThrottler.getExecutionCount()) // 成功執行的次數
console.log(updateThrottler.getLastExecutionTime()) // 最後執行的時間戳

// 取消任何待處理的執行
updateThrottler.cancel()
```

### 前緣和後緣執行

同步節流器支援前緣和後緣執行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 首次呼叫時執行（預設）
  trailing: true,  // 等待週期後執行（預設）
})
```

- `leading: true`（預設） - 首次呼叫時立即執行
- `leading: false` - 跳過首次呼叫，等待後緣執行
- `trailing: true`（預設） - 等待週期後執行最後一次呼叫
- `trailing: false` - 如果在等待週期內則跳過最後一次呼叫

常見模式：
- `{ leading: true, trailing: true }` - 預設，回應最靈敏
- `{ leading: false, trailing: true }` - 延遲所有執行
- `{ leading: true, trailing: false }` - 跳過排隊的執行

### 啟用/停用

`Throttler` 類通過 `enabled` 選項支援啟用/停用。使用 `setOptions` 方法，您可以隨時啟用/停用節流器：

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // 預設停用
throttler.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是節流器選項具有反應性的框架適配器，可以將 `enabled` 選項設置為條件值以動態啟用/停用節流器。但是，如果您直接使用 `throttle` 函數或 `Throttler` 類，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `Throttler` 類的構造函數。

### 回呼選項

同步和非同步節流器都支援回呼選項，以處理節流生命週期的不同方面：

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

`onExecute` 回呼在每次成功執行節流函數後呼叫，這對於追蹤執行、更新 UI 狀態或執行清理操作非常有用。

#### 非同步節流器回呼

非同步 `AsyncThrottler` 支援用於錯誤處理的額外回呼：

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
    // 如果非同步函數拋出錯誤則呼叫
    console.error('非同步函數失敗:', error)
  }
})
```

`onExecute` 回呼與同步節流器中的工作方式相同，而 `onError` 回呼允許您優雅地處理錯誤而不中斷節流鏈。這些回呼對於追蹤執行計數、更新 UI 狀態、處理錯誤、執行清理操作和記錄執行指標特別有用。

### 非同步節流

對於非同步函數或需要錯誤處理的情況，請使用 `AsyncThrottler` 或 `asyncThrottle`：

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
      console.error('API 呼叫失敗:', error)
    }
  }
)

// 每秒只會進行一次 API 呼叫
await throttledFetch('123')
```

非同步版本提供基於 Promise 的執行追蹤、通過 `onError` 回呼的錯誤處理、待處理非同步操作的正確清理，以及可等待的 `maybeExecute` 方法。

### 框架適配器

每個框架適配器都提供建立在核心節流功能之上的鉤子，以與框架的狀態管理系統集成。每個框架都有可用的鉤子，如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue`。

以下是一些範例：

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 用於完全控制的低階鉤子
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於基本使用情境的簡單回呼鉤子
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於反應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
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

每個框架適配器都提供與框架狀態管理系統集成的鉤子，同時保持核心節流功能。
