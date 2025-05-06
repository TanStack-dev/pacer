---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:06:21.580Z'
title: 防抖指南
id: debouncing
---
速率限制 (Rate Limiting)、節流 (Throttling) 與防抖 (Debouncing) 是三種控制函數執行頻率的不同方法。每種技術都以不同的方式阻擋執行，使其成為「有損」的 — 意味著當函數被要求過於頻繁執行時，某些呼叫將不會被執行。了解何時使用每種方法對於構建高效能且可靠的應用程式至關重要。本指南將涵蓋 TanStack Pacer 的防抖概念。

## 防抖概念

防抖是一種技術，它會延遲函數的執行，直到指定的不活動時間過去。與速率限制 (允許在限制內爆發性執行) 或節流 (確保執行間隔均勻) 不同，防抖會將多次快速函數呼叫合併為單一執行，且僅在呼叫停止後才會發生。這使得防抖非常適合處理事件爆發的情況，當你只關心活動結束後的最終狀態時。

### 防抖視覺化

```text
防抖 (等待: 3 個刻度)
時間軸: [每秒 1 個刻度]
呼叫:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
已執行:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 在此處執行
                                                         無呼叫 3 個刻度後

             [呼叫爆發]     [更多呼叫]   [等待]      [新爆發]
             無執行         重置計時器    [延遲執行]  [等待] [延遲執行]
```

### 何時使用防抖

當你想在採取行動前等待活動「暫停」時，防抖特別有效。這使其非常適合處理使用者輸入或其他快速觸發的事件，當你只關心最終狀態時。

常見使用情境包括：
- 搜尋輸入欄位，當你想等待使用者完成輸入時
- 表單驗證，不應在每次按鍵時執行
- 視窗大小調整計算，這些計算成本高昂
- 編輯內容時自動儲存草稿
- 僅在使用者活動停止後才應發生的 API 呼叫
- 任何你只關心快速變化後最終值的情境

### 何時不使用防抖

防抖可能不是最佳選擇的情況：
- 你需要保證在特定時間段內執行 (改用 [節流](../guides/throttling))
- 你無法承受錯過任何執行 (改用 [排隊](../guides/queueing))

## TanStack Pacer 中的防抖

TanStack Pacer 分別透過 `Debouncer` 和 `AsyncDebouncer` 類別 (及其對應的 `debounce` 和 `asyncDebounce` 函數) 提供同步和非同步防抖。

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

為了更精確控制防抖行為，你可以直接使用 `Debouncer` 類別：

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// 獲取當前狀態資訊
console.log(searchDebouncer.getExecutionCount()) // 成功執行次數
console.log(searchDebouncer.getIsPending()) // 是否有呼叫待處理

// 動態更新選項
searchDebouncer.setOptions({ wait: 1000 }) // 增加等待時間

// 取消待處理的執行
searchDebouncer.cancel()
```

### 前緣與後緣執行

同步防抖器支援前緣和後緣執行：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 第一次呼叫時立即執行
  trailing: true,  // 等待期後執行
})
```

- `leading: true` - 函數在第一次呼叫時立即執行
- `leading: false` (預設) - 第一次呼叫開始等待計時器
- `trailing: true` (預設) - 函數在等待期後執行
- `trailing: false` - 等待期後不執行

常見模式：
- `{ leading: false, trailing: true }` - 預設，等待後執行
- `{ leading: true, trailing: false }` - 立即執行，忽略後續呼叫
- `{ leading: true, trailing: true }` - 在第一次呼叫和等待後都執行

### 最大等待時間

TanStack Pacer 防抖器特意不像其他防抖函式庫那樣提供 `maxWait` 選項。如果你需要讓執行在更分散的時間段內運行，請考慮改用 [節流](../guides/throttling) 技術。

### 啟用/停用

`Debouncer` 類別支援透過 `enabled` 選項啟用/停用。使用 `setOptions` 方法，你可以隨時啟用/停用防抖器：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 預設停用
debouncer.setOptions({ enabled: true }) // 隨時啟用
```

如果你使用的是框架適配器，其中防抖器選項是響應式的，你可以將 `enabled` 選項設置為條件值以動態啟用/停用防抖器：

```ts
// React 範例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 根據輸入長度啟用/停用 IF 使用支援響應式選項的框架適配器
)
```

然而，如果你使用的是 `debounce` 函數或直接使用 `Debouncer` 類別，則必須使用 `setOptions` 方法來更改 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `Debouncer` 類別的建構函數。

```ts
// Solid 範例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 預設停用
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 根據輸入長度啟用/停用
})
```

### 回呼選項

同步和非同步防抖器都支援回呼選項，以處理防抖生命週期的不同方面：

#### 同步防抖器回呼

同步 `Debouncer` 支援以下回呼：

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 每次成功執行後呼叫
    console.log('函數已執行', debouncer.getExecutionCount())
  }
})
```

`onExecute` 回呼在防抖函數每次成功執行後呼叫，這對於追蹤執行、更新 UI 狀態或執行清理操作非常有用。

#### 非同步防抖器回呼

非同步 `AsyncDebouncer` 的回呼集與同步版本不同。

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // 每次成功執行後呼叫
    console.log('非同步函數已執行', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // 每次執行嘗試後呼叫
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

非同步防抖器提供了一種強大的方式來處理非同步操作與防抖，相比同步版本提供了幾個關鍵優勢。雖然同步防抖器非常適合 UI 事件和即時反饋，但非同步版本專門設計用於處理 API 呼叫、資料庫操作和其他非同步任務。

#### 與同步防抖的主要差異

1. **返回值處理**
與返回 void 的同步防抖器不同，非同步版本允許你捕獲並使用防抖函數的返回值。這在需要處理 API 呼叫或其他非同步操作的結果時特別有用。`maybeExecute` 方法返回一個 Promise，該 Promise 解析為函數的返回值，允許你等待結果並適當處理。

2. **增強的回呼系統**
非同步防抖器提供比同步版本的單一 `onExecute` 回呼更複雜的回呼系統。此系統包括：
- `onSuccess`：當非同步函數成功完成時呼叫，提供結果和防抖器實例
- `onError`：當非同步函數拋出錯誤時呼叫，提供錯誤和防抖器實例
- `onSettled`：在每次執行嘗試後呼叫，無論成功或失敗

3. **執行追蹤**
非同步防抖器透過多種方法提供全面的執行追蹤：
- `getSuccessCount()`：成功執行次數
- `getErrorCount()`：失敗執行次數
- `getSettledCount()`：已結算的執行總數 (成功 + 失敗)

4. **順序執行**
非同步防抖器確保後續執行等待前一次呼叫完成後才開始。這防止了執行順序錯亂，並保證每次呼叫處理的都是最新的資料。這在處理依賴於先前呼叫結果的操作或當保持資料一致性至關重要時特別重要。

例如，如果你正在更新使用者個人資料，然後立即獲取其更新後的資料，非同步防抖器將確保獲取操作等待更新完成，防止可能獲取過時資料的競態條件。

#### 基本用法範例

這是一個基本範例，展示如何將非同步防抖器用於搜尋操作：

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

#### 高級模式

非同步防抖器可以與各種模式結合以解決複雜問題：

1. **狀態管理整合**
當將非同步防抖器與狀態管理系統 (如 React 的 useState 或 Solid 的 createSignal) 一起使用時，你可以創建強大的模式來處理載入狀態、錯誤狀態和資料更新。防抖器的回呼提供了根據操作成功或失敗更新 UI 狀態的完美鉤點。

2. **競態條件預防**
單次飛行突變模式自然防止了許多情境下的競態條件。當應用程式的多個部分同時嘗試更新同一資源時，防抖器確保只有最近的更新實際發生，同時仍向所有呼叫者提供結果。

3. **錯誤恢復**
非同步防抖器的錯誤處理能力使其非常適合實現重試邏輯和錯誤恢復模式。你可以使用 `onError` 回呼來實現自定義錯誤處理策略，例如指數退避或後備機制。

### 框架適配器

每個框架適配器都提供建立在核心防抖功能之上的鉤子，以與框架的狀態管理系統整合。每個框架都有可用的鉤子，如 `createDebouncer`、`useDebouncedCallback`、`useDebouncedState` 或 `useDebouncedValue`。

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
