---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:25:03.436Z'
title: 防抖指南
id: debouncing
---
節流控制 (Rate Limiting)、節流 (Throttling) 和防抖 (Debouncing) 是三種控制函式執行頻率的獨立方法。每種技術以不同方式阻擋執行，使其具有「損耗性」— 意味著當函式呼叫過於頻繁時，部分呼叫將不會執行。了解何時使用每種方法對於建構高效能且可靠的應用程式至關重要。本指南將介紹 TanStack Pacer 的防抖 (Debouncing) 概念。

## 防抖 (Debouncing) 概念

防抖 (Debouncing) 是一種延遲函式執行的技術，直到指定的無活動時間段過去為止。與允許在限制內突發執行的節流控制 (Rate Limiting) 或確保均勻間隔執行的節流 (Throttling) 不同，防抖 (Debouncing) 將多次快速函式呼叫合併為單一執行，僅在呼叫停止後觸發。這使得防抖 (Debouncing) 非常適合處理突發事件，其中您只關心活動停止後的最終狀態。

### 防抖 (Debouncing) 視覺化

```text
防抖 (Debouncing) (等待: 3 個時間單位)
時間軸: [每秒 1 個時間單位]
呼叫:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
已執行:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 在此處執行，於
                                                         無呼叫的 3 個時間單位後

             [突發呼叫]     [更多呼叫]   [等待]      [新突發]
             無執行         重置計時器    [延遲執行]  [等待] [延遲執行]
```

### 何時使用防抖 (Debouncing)

當您希望在採取行動前等待活動「暫停」時，防抖 (Debouncing) 特別有效。這使其非常適合處理使用者輸入或其他快速觸發的事件，其中您只關心最終狀態。

常見使用案例包括：
- 搜尋輸入欄位，等待使用者完成輸入
- 不應在每次按鍵時執行的表單驗證
- 計算成本高昂的視窗大小調整
- 編輯內容時自動儲存草稿
- 僅在使用者活動停止後才進行的 API 呼叫
- 任何您只關心快速變更後最終值的場景

### 何時不使用防抖 (Debouncing)

防抖 (Debouncing) 可能不是最佳選擇的情況：
- 您需要在特定時間段內保證執行（改用 [節流 (Throttling)](../guides/throttling)）
- 您無法承受錯過任何執行（改用 [佇列 (Queueing)](../guides/queueing)）

## TanStack Pacer 中的防抖 (Debouncing)

TanStack Pacer 分別透過 `Debouncer` 和 `AsyncDebouncer` 類別（及其對應的 `debounce` 和 `asyncDebounce` 函式）提供同步和非同步防抖 (Debouncing)。

### 使用 `debounce` 的基本用法

`debounce` 函式是為任何函式添加防抖 (Debouncing) 的最簡單方式：

```ts
import { debounce } from '@tanstack/pacer'

// 防抖搜尋輸入，等待使用者停止輸入
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

為了更精確控制防抖 (Debouncing) 行為，您可以直接使用 `Debouncer` 類別：

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// 取得目前狀態的資訊
console.log(searchDebouncer.getExecutionCount()) // 成功執行的次數
console.log(searchDebouncer.getIsPending()) // 是否有呼叫正在等待中

// 動態更新選項
searchDebouncer.setOptions({ wait: 1000 }) // 增加等待時間

// 取消等待中的執行
searchDebouncer.cancel()
```

### 領先和尾隨執行

同步防抖器支援領先和尾隨邊緣執行：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 在第一次呼叫時執行
  trailing: true,  // 在等待時間後執行
})
```

- `leading: true` - 函式在第一次呼叫時立即執行
- `leading: false` (預設) - 第一次呼叫啟動等待計時器
- `trailing: true` (預設) - 函式在等待時間後執行
- `trailing: false` - 等待時間後不執行

常見模式：
- `{ leading: false, trailing: true }` - 預設，等待後執行
- `{ leading: true, trailing: false }` - 立即執行，忽略後續呼叫
- `{ leading: true, trailing: true }` - 在第一次呼叫和等待後都執行

### 最大等待時間

TanStack Pacer 的防抖器不像其他防抖函式庫那樣具有 `maxWait` 選項。如果您需要讓執行在更分散的時間段內運行，請考慮改用 [節流 (Throttling)](../guides/throttling) 技術。

### 啟用/停用

`Debouncer` 類別支援透過 `enabled` 選項啟用/停用。使用 `setOptions` 方法，您可以隨時啟用/停用防抖器：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // 預設停用
debouncer.setOptions({ enabled: true }) // 隨時啟用
```

如果您使用的是防抖器選項具有反應性的框架適配器，您可以將 `enabled` 選項設為條件值以動態啟用/停用防抖器：

```ts
// React 範例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 根據輸入長度啟用/停用（如果使用支援反應式選項的框架適配器）
)
```

然而，如果您使用的是 `debounce` 函式或直接使用 `Debouncer` 類別，則必須使用 `setOptions` 方法來變更 `enabled` 選項，因為傳遞的選項實際上會傳遞給 `Debouncer` 類別的建構函式。

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
    // 在每次成功執行後呼叫
    console.log('函式已執行', debouncer.getExecutionCount())
  }
})
```

`onExecute` 回呼在防抖函式每次成功執行後呼叫，這對於追蹤執行、更新 UI 狀態或執行清理操作非常有用。

#### 非同步防抖器回呼

非同步 `AsyncDebouncer` 支援額外的錯誤處理回呼：

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // 在每次成功執行後呼叫
    console.log('非同步函式已執行', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // 如果非同步函式拋出錯誤時呼叫
    console.error('非同步函式失敗:', error)
  }
})
```

`onExecute` 回呼與同步防抖器中的功能相同，而 `onError` 回呼允許您優雅地處理錯誤而不中斷防抖鏈。這些回呼對於追蹤執行次數、更新 UI 狀態、處理錯誤、執行清理操作和記錄執行指標特別有用。

### 非同步防抖 (Debouncing)

對於非同步函式或需要錯誤處理的情況，請使用 `AsyncDebouncer` 或 `asyncDebounce`：

```ts
import { asyncDebounce } from '@tanstack/pacer'

const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    updateUI(results)
  },
  {
    wait: 500,
    onError: (error) => {
      console.error('搜尋失敗:', error)
    }
  }
)

// 僅在輸入停止後進行一次 API 呼叫
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

非同步版本提供基於 Promise 的執行追蹤、透過 `onError` 回呼的錯誤處理、待處理非同步操作的正確清理，以及可等待的 `maybeExecute` 方法。

### 框架適配器

每個框架適配器提供建立在核心防抖功能之上的鉤子，以與框架的狀態管理系統整合。每個框架都有可用的鉤子，如 `createDebouncer`、`useDebouncedCallback`、`useDebouncedState` 或 `useDebouncedValue`。

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

// 用於反應式狀態管理的基於狀態的鉤子
const [instantState, setInstantState] = useState('')
const [debouncedState, setDebouncedState] = useDebouncedValue(
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

// 用於狀態管理的基於訊號的鉤子
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('總執行次數:', debouncer.getExecutionCount())
  }
})
```
