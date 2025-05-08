---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:47:08.677Z'
title: 節流指南
id: throttling
---
節流控制指南 (Throttling Guide)

速率限制 (Rate Limiting)、節流控制 (Throttling) 和防抖動 (Debouncing) 是三種控制函數執行頻率的不同方法。每種技術以不同方式阻擋執行，使它們成為「有損」的 - 這意味著當函數呼叫過於頻繁時，部分呼叫將不會執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的節流控制概念。

## 節流控制概念

節流控制確保函數執行在時間上均勻分佈。與允許突發執行直到達到限制的速率限制 (Rate Limiting) 不同，也不同於等待活動停止的防抖動 (Debouncing)，節流控制通過在呼叫之間強制一致的延遲來創建更平滑的執行模式。如果您設定每秒執行一次的節流控制，無論請求多麼頻繁，呼叫都會均勻間隔。

### 節流控制視覺化

```text
節流控制 (每 3 個刻度執行一次)
時間軸: [每秒一個刻度]
呼叫:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  �️             ⬇️
已執行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 每 3 個刻度只允許一次執行，
               無論進行了多少次呼叫

             [第一次突發]    [更多呼叫]              [間隔呼叫]
             第一次執行     等待週期後執行          每次等待週期
             然後節流                          通過後執行
```

### 何時使用節流控制

當您需要一致且可預測的執行時機時，節流控制特別有效。這使其成為處理頻繁事件或更新的理想選擇，您希望獲得平滑、受控的行為。

常見使用場景包括：
- 需要一致時機的 UI 更新（例如進度指示器）
- 不應壓垮瀏覽器的滾動或調整大小事件處理程序
- 需要一致間隔的即時資料輪詢
- 需要穩定節奏的資源密集型操作
- 遊戲循環更新或動畫幀處理
- 使用者輸入時的即時搜尋建議

### 何時不使用節流控制

在以下情況下，節流控制可能不是最佳選擇：
- 您想等待活動停止（改用 [防抖動](../guides/debouncing)）
- 您不能錯過任何執行（改用 [排隊](../guides/queueing)）

> [!TIP]
> 當您需要平滑、一致的執行時機時，節流控制通常是最佳選擇。它提供比速率限制更可預測的執行模式，比防抖動更即時的回饋。

## TanStack Pacer 中的節流控制

TanStack Pacer 分別通過 `Throttler` 和 `AsyncThrottler` 類（及其對應的 `throttle` 和 `asyncThrottle` 函數）提供同步和非同步節流控制。

### 使用 `throttle` 的基本用法

`throttle` 函數是為任何函數添加節流控制的最簡單方法：

```ts
import { throttle } from '@tanstack/pacer'

// 將 UI 更新節流為每 200 毫秒一次
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 在快速循環中，僅每 200 毫秒執行一次
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 許多呼叫被節流
}
```

### 使用 `Throttler` 類的高級用法

為了更精確控制節流行為，您可以直接使用 `Throttler` 類：

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

### 前導和尾隨執行

同步節流器支援前導和尾隨邊緣執行：

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 第一次呼叫時執行（預設）
  trailing: true,  // 等待週期後執行（預設）
})
```

- `leading: true` (預設) - 第一次呼叫時立即執行
- `leading: false` - 跳過第一次呼叫，等待尾隨執行
- `trailing: true` (預設) - 等待週期後執行最後一次呼叫
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

如果您使用的是節流器選項具有反應性的框架適配器，您可以將 `enabled` 選項設定為條件值以動態啟用/停用節流器。但是，如果您直接使用 `throttle` 函數或 `Throttler` 類，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `Throttler` 類的構造函數。

### 回調選項

同步和非同步節流器都支援回調選項，以處理節流生命週期的不同方面：

#### 同步節流器回調

同步 `Throttler` 支援以下回調：

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // 每次成功執行後呼叫
    console.log('函數已執行', throttler.getExecutionCount())
  }
})
```

`onExecute` 回調在節流函數每次成功執行後呼叫，這對於追蹤執行、更新 UI 狀態或執行清理操作非常有用。

#### 非同步節流器回調

非同步 `AsyncThrottler` 支援額外的錯誤處理回調：

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

`onExecute` 回調的工作方式與同步節流器相同，而 `onError` 回調允許您優雅地處理錯誤而不中斷節流鏈。這些回調對於追蹤執行計數、更新 UI 狀態、處理錯誤、執行清理操作和記錄執行指標特別有用。

### 非同步節流控制

非同步節流器提供了一種強大的方式來處理具有節流控制的非同步操作，與同步版本相比具有幾個關鍵優勢。雖然同步節流器非常適合 UI 事件和即時回饋，但非同步版本專門設計用於處理 API 呼叫、數據庫操作和其他非同步任務。

#### 與同步節流控制的主要區別

1. **返回值處理**
與返回 void 的同步節流器不同，非同步版本允許您捕獲和使用節流函數的返回值。這在您需要使用 API 呼叫或其他非同步操作的結果時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函數的返回值，允許您等待結果並適當處理。

2. **不同的回調**
`AsyncThrottler` 支援以下回調，而不僅僅是同步版本中的 `onExecute`：
- `onSuccess`：每次成功執行後呼叫，提供節流器實例
- `onSettled`：每次執行後呼叫，提供節流器實例
- `onError`：如果非同步函數拋出錯誤則呼叫，提供錯誤和節流器實例

非同步和同步節流器都支援 `onExecute` 回調以處理成功執行。

3. **順序執行**
由於節流器的 `maybeExecute` 方法返回一個 Promise，您可以選擇在開始下一個執行之前等待每個執行。這使您可以控制執行順序，並確保每個呼叫處理最新的數據。這在處理依賴於先前呼叫結果的操作或維護數據一致性至關重要時特別有用。

例如，如果您正在更新用戶的個人資料，然後立即獲取其更新的數據，您可以在開始獲取之前等待更新操作：

#### 基本用法示例

以下是一個基本示例，展示如何使用非同步節流器進行搜尋操作：

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

### 框架適配器

每個框架適配器都提供建立在核心節流功能之上的鉤子，以與框架的狀態管理系統集成。每個框架都有可用的鉤子，如 `createThrottler`、`useThrottledCallback`、`useThrottledState` 或 `useThrottledValue`。

以下是一些示例：

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 用於完全控制的低階鉤子
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於基本用例的簡單回調鉤子
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 用於反應式狀態管理的基於狀態的鉤子
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

每個框架適配器都提供與框架狀態管理系統集成的鉤子，同時保持核心節流功能。
