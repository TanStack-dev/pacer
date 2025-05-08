---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:47:16.351Z'
title: 防抖指南
id: debouncing
---
# 防抖 (Debouncing) 指南

速率限制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是控制函數執行頻率的三種不同方法。每種技術以不同方式阻擋執行，使它們成為「有損」的 — 這意味當函數被要求過於頻繁執行時，某些呼叫將不會執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將涵蓋 TanStack Pacer 的防抖概念。

## 防抖概念

防抖是一種技術，它延遲函數的執行，直到指定的不活動時間過去。與允許執行突發但有限制的速率限制不同，也不同於確保均勻間隔執行的節流，防抖將多次快速函數呼叫合併為單一執行，僅在呼叫停止後發生。這使得防抖非常適合處理突發事件，其中您只關心活動停止後的最終狀態。

### 防抖視覺化

```text
防抖 (等待: 3 個刻度)
時間軸: [每秒 1 個刻度]
呼叫:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
已執行:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 在此處執行，在
                                                         無呼叫的 3 個刻度後

             [呼叫突發]     [更多呼叫]   [等待]      [新突發]
             無執行         重置計時器    [延遲執行]  [等待] [延遲執行]
```

### 何時使用防抖

當您想在採取行動前等待活動「暫停」時，防抖特別有效。這使其非常適合處理使用者輸入或其他快速觸發的事件，其中您只關心最終狀態。

常見使用案例包括：
- 搜尋輸入欄位，您希望等待使用者完成輸入
- 不應在每次按鍵時運行的表單驗證
- 計算成本高昂的視窗大小調整計算
- 編輯內容時自動儲存草稿
- 僅在使用者活動停止後才應發生的 API 呼叫
- 任何您只關心快速變化後最終值的場景

### 何時不使用防抖

防抖可能不是最佳選擇的情況：
- 您需要在特定時間段內保證執行（改用[節流](../guides/throttling)）
- 您不能承受錯過任何執行（改用[排隊](../guides/queueing)）

## TanStack Pacer 中的防抖

TanStack Pacer 分別透過 `Debouncer` 和 `AsyncDebouncer` 類別（及其對應的 `debounce` 和 `asyncDebounce` 函數）提供同步和非同步防抖。

### 使用 `debounce` 的基本用法

`debounce` 函數是為任何函數添加防抖的最簡單方法：

```ts
import { debounce } from '@tanstack/pacer'

// 防抖搜尋輸入以等待使用者停止輸入
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // 最後一次按鍵後等待 500ms
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### 使用 `Debouncer` 類別的高級用法

為了更精確控制防抖行為，您可以直接使用 `Debouncer` 類別：

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// 獲取當前狀態的資訊
console.log(searchDebouncer.getExecutionCount()) // 成功執行的次數
console.log(searchDebouncer.getIsPending()) // 是否有呼叫正在等待中

// 動態更新選項
searchDebouncer.setOptions({ wait: 1000 }) // 增加等待時間

// 取消等待中的執行
searchDebouncer.cancel()
```

### 前緣和後緣執行

同步防抖器支援前緣和後緣執行：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 在第一次呼叫時執行
  trailing: true,  // 在等待期後執行
})
```

- `leading: true` - 函數在第一次呼叫時立即執行
- `leading: false` (預設) - 第一次呼叫啟動等待計時器
- `trailing: true` (預設) - 函數在等待期後執行
- `trailing: false` - 等待期後不執行

常見模式：
- `{ leading: false, trailing: true }` - 預設，等待後執行
- `{ leading: true, trailing: false }` - 立即執行，忽略後續呼叫
- `{ leading: true, trailing: true }` - 在第一次呼叫和等待後都執行

### 最大等待時間

TanStack Pacer 防抖器特意不像其他防抖庫那樣擁有 `maxWait` 選項。如果您需要讓執行在更分散的時間段內運行，請考慮改用[節流](../guides/throttling)技術。

### 啟用/停用

`Debouncer` 類別透過 `enabled` 選項支援啟用/停用。使用 `setOptions` 方法，您可以隨時啟用/停用防抖器：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 預設停用
debouncer.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是防抖器選項具有響應性的框架適配器，您可以將 `enabled` 選項設置為條件值以動態啟用/停用防抖器：

```ts
// React 範例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 根據輸入長度啟用/停用（如果使用支援響應式選項的框架適配器）
)
```

但是，如果您使用的是 `debounce` 函數或直接使用 `Debouncer` 類別，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上是傳遞給 `Debouncer` 類別的建構函數。

```ts
// Solid 範例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 預設停用
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 根據輸入長度啟用/停用
})
```

### 回呼選項

同步和非同步防抖器都支援回呼選項以處理防抖生命週期的不同方面：

#### 同步防抖器回呼

同步 `Debouncer` 支援以下回呼：

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 在每次成功執行後呼叫
    console.log('函數已執行', debouncer.getExecutionCount())
  }
})
```

`onExecute` 回呼在防抖函數每次成功執行後呼叫，這對於追蹤執行、更新 UI 狀態或執行清理操作非常有用。

#### 非同步防抖器回呼

非同步 `AsyncDebouncer` 與同步版本相比具有一組不同的回呼。

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // 在每次成功執行後呼叫
    console.log('非同步函數已執行', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // 在每次執行嘗試後呼叫
    console.log('非同步函數已結算', debouncer.getSettledCount())
  },
  onError: (error) => {
    // 如果非同步函數拋出錯誤則呼叫
    console.error('非同步函數失敗:', error)
  }
})
```

`onSuccess` 回呼在防抖函數每次成功執行後呼叫，而 `onError` 回呼在非同步函數拋出錯誤時呼叫。`onSettled` 回呼在每次執行嘗試後呼叫，無論成功或失敗。這些回呼對於追蹤執行計數、更新 UI 狀態、處理錯誤、執行清理操作和記錄執行指標特別有用。

### 非同步防抖

非同步防抖器提供了一種強大的方式來處理非同步操作與防抖，相比同步版本具有幾個關鍵優勢。雖然同步防抖器非常適合 UI 事件和即時反饋，但非同步版本專門設計用於處理 API 呼叫、數據庫操作和其他非同步任務。

#### 與同步防抖的主要區別

1. **返回值處理**
與返回 void 的同步防抖器不同，非同步版本允許您捕獲和使用防抖函數的返回值。這在您需要使用 API 呼叫或其他非同步操作的結果時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函數的返回值，允許您等待結果並適當處理。

2. **不同的回呼**
`AsyncDebouncer` 支援以下回呼，而不僅僅是同步版本中的 `onExecute`：
- `onSuccess`：在每次成功執行後呼叫，提供防抖器實例
- `onSettled`：在每次執行後呼叫，提供防抖器實例
- `onError`：如果非同步函數拋出錯誤則呼叫，提供錯誤和防抖器實例

3. **順序執行**
由於防抖器的 `maybeExecute` 方法返回一個 Promise，您可以選擇在開始下一次執行前等待每次執行。這讓您可以控制執行順序並確保每次呼叫處理最新的數據。這在處理依賴先前呼叫結果的操作或維護數據一致性至關重要時特別有用。

例如，如果您正在更新使用者的個人資料並立即獲取其更新後的數據，您可以在開始獲取前等待更新操作：

#### 基本用法範例

以下是一個基本範例，展示如何使用非同步防抖器進行搜尋操作：

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('搜尋成功:', results)
    },
    onError: (error, debouncer) => {
      console.error('搜尋失敗:', error)
    }
  }
)

// 用法
const results = await debouncedSearch('query')
```

### 框架適配器

每個框架適配器都提供建立在核心防抖功能之上的鉤子，以與框架的狀態管理系統集成。每個框架都有可用的鉤子，如 `createDebouncer`、`useDebouncedCallback`、`useDebouncedState` 或 `useDebouncedValue`。

以下是一些範例：

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// 用於完全控制的低階鉤子
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 用於基本用例的簡單回呼鉤子
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// 用於響應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState('')
const [debouncedValue] = useDebouncedValue(
  instantState, // 要防抖的值
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// 用於完全控制的低階鉤子
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 用於狀態管理的基於信號的鉤子
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('總執行次數:', debouncer.getExecutionCount())
  }
})
```
