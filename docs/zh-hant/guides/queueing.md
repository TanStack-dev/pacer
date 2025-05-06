---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:06:43.086Z'
title: 佇列指南
id: queueing
---
與[速率限制 (Rate Limiting)](../guides/rate-limiting)、[節流 (Throttling)](../guides/throttling)和[防抖 (Debouncing)](../guides/debouncing)不同，這些技術會在操作過於頻繁時捨棄執行，而排隊器 (queuer) 可配置為確保每個操作都能被處理。它們提供了一種管理和控制操作流程的方式，且不會遺失任何請求。這使得排隊器非常適合無法接受資料遺失的情境。排隊器也可設定最大容量，這對於防止記憶體洩漏或其他問題很有幫助。本指南將介紹 TanStack Pacer 的排隊概念。

## 排隊概念

排隊確保每個操作最終都會被處理，即使它們的到達速度比處理速度還快。與其他會捨棄過多操作的執行控制技術不同，排隊會將操作緩衝在一個有序列表中，並根據特定規則處理它們。這使得排隊成為 TanStack Pacer 中唯一「無損 (lossless)」的執行控制技術，除非指定了 `maxSize`，這會在緩衝區滿時導致項目被拒絕。

### 排隊視覺化

```text
排隊 (每 2 個 tick 處理一個項目)
時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
佇列:       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
已執行:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ 與速率限制/節流/防抖不同，
               所有呼叫最終都會按順序處理

             [項目排隊]   [穩定處理]   [清空佇列]
              忙碌時       逐一處理
```

### 何時使用排隊

當您需要確保每個操作都被處理時，排隊尤其重要，即使這意味著引入一些延遲。這使其非常適合資料一致性和完整性比立即執行更重要的情境。使用 `maxSize` 時，它還可以作為緩衝區，防止系統因過多待處理操作而超載。

常見使用案例包括：
- 在需要資料前預先取得，避免系統過載
- 處理 UI 中的使用者互動，每個動作都必須被記錄
- 處理需要維持資料一致性的資料庫操作
- 管理必須全部成功完成的 API 請求
- 協調不能捨棄的背景任務
- 動畫序列中每一幀都很重要
- 表單提交中每個條目都需要保存
- 使用 `maxSize` 緩衝有固定容量的資料流

### 何時不使用排隊

排隊可能不是最佳選擇的情況：
- 立即回饋比處理每個操作更重要
- 您只關心最新值（改用[防抖 (debouncing)](../guides/debouncing)）

> [!TIP]
> 如果您目前使用速率限制、節流或防抖，但發現捨棄操作導致問題，排隊可能是您需要的解決方案。

## TanStack Pacer 中的排隊

TanStack Pacer 透過簡單的 `queue` 函式和更強大的 `Queuer` 類別提供排隊功能。雖然其他執行控制技術通常偏愛其基於函式的 API，但排隊通常受益於基於類別的 API 提供的額外控制。

### 使用 `queue` 的基本用法

`queue` 函式提供了一種簡單的方式來創建一個始終運行的佇列，並在項目添加時處理它們：

```ts
import { queue } from '@tanstack/pacer'

// 創建一個每秒處理項目的佇列
const processItems = queue<number>({
  wait: 1000,
  maxSize: 10, // 可選：限制佇列大小以防止記憶體或時間問題
  onItemsChange: (queuer) => {
    console.log('當前佇列:', queuer.getAllItems())
  }
})

// 添加要處理的項目
processItems(1) // 立即處理
processItems(2) // 1 秒後處理
processItems(3) // 2 秒後處理
```

雖然 `queue` 函式易於使用，但它僅透過 `addItem` 方法提供基本的始終運行佇列。對於大多數使用案例，您會需要 `Queuer` 類別提供的額外控制和功能。

### 使用 `Queuer` 類別的高級用法

`Queuer` 類別提供了對佇列行為和處理的完整控制：

```ts
import { Queuer } from '@tanstack/pacer'

// 創建一個每秒處理項目的佇列
const queue = new Queuer<number>({
  wait: 1000, // 處理項目間等待 1 秒
  maxSize: 5, // 可選：限制佇列大小以防止記憶體或時間問題
  onItemsChange: (queuer) => {
    console.log('當前佇列:', queuer.getAllItems())
  }
})

// 開始處理
queue.start()

// 添加要處理的項目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// 項目將逐一處理，每個之間有 1 秒延遲
// 輸出：
// 處理中: 1 (立即)
// 處理中: 2 (1 秒後)
// 處理中: 3 (2 秒後)
```

### 佇列類型和順序

TanStack Pacer 的 Queuer 獨特之處在於其透過基於位置的 API 適應不同使用案例的能力。同一個 Queuer 可以表現為傳統佇列、堆疊或雙端佇列，全部透過相同的統一介面。

#### FIFO 佇列 (先進先出)

預設行為，項目按添加順序處理。這是最常見的佇列類型，遵循先添加的項目應先處理的原則。使用 `maxSize` 時，如果佇列已滿，新項目將被拒絕。

```text
FIFO 佇列視覺化 (maxSize=3):

進入 →  [A][B][C] → 離開
         ⬇️     ⬆️
      新項目在此   項目在此
      添加       處理
      (滿時拒絕)

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
佇列:       [ABC]   [BC]    [C]    []
已處理:    A       B       C
已拒絕:     D      E
```

FIFO 佇列適合：
- 順序重要的任務處理
- 需要按順序處理的訊息佇列
- 文件應按發送順序列印的列印佇列
- 必須按時間順序處理事件的系統

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // 預設
  getItemsFrom: 'front', // 預設
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// 處理順序: 1, 然後 2
```

#### LIFO 堆疊 (後進先出)

透過將添加和檢索項目的位置都指定為 'back'，Queuer 的行為類似堆疊。在堆疊中，最近添加的項目會先被處理。使用 `maxSize` 時，如果堆疊已滿，新項目將被拒絕。

```text
LIFO 堆疊視覺化 (maxSize=3):

     ⬆️ 處理
    [C] ← 最近添加
    [B]
    [A] ← 最先添加
     ⬇️ 進入
     (滿時拒絕)

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
佇列:       [ABC]   [AB]    [A]    []
已處理:    C       B       A
已拒絕:     D      E
```

堆疊行為特別適用於：
- 應先復原最近動作的復原/重做系統
- 瀏覽器歷史導航，您希望返回最近頁面
- 程式語言實現中的函式呼叫堆疊
- 深度優先遍歷演算法

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // 預設
  getItemsFrom: 'back', // 覆寫預設以實現堆疊行為
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// 項目處理順序: 2, 然後 1

stack.getNextItem('back') // 從佇列後端而非前端獲取下一個項目
```

#### 優先佇列

優先佇列透過允許項目根據其優先級而非僅插入順序排序，為佇列順序增加了另一個維度。每個項目被分配一個優先級值，佇列會自動按優先級順序維護項目。使用 `maxSize` 時，如果佇列已滿，優先級較低的項目可能會被拒絕。

```text
優先佇列視覺化 (maxSize=3):

進入 →  [P:5][P:3][P:2] → 離開
          ⬇️           ⬆️
     高優先級項目      低優先級項目
     在此            最後處理
     (滿時拒絕)

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
佇列:       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
已處理:              5         -          3         2        1
已拒絕:                         4
```

優先佇列對於以下情況至關重要：
- 某些任務比其他任務更緊急的任務排程器
- 需要優先處理特定類型流量的網路封包路由
- 高優先級事件應在低優先級事件之前處理的事件系統
- 某些請求比其他請求更重要的資源分配

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // 數字越大優先級越高
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// 處理順序: 3, 2, 然後 1
```

### 啟動和停止

`Queuer` 類別支援透過 `start()` 和 `stop()` 方法啟動和停止處理，並可配置為自動啟動的 `started` 選項：

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // 暫停啟動
})

// 控制處理
queue.start() // 開始處理項目
queue.stop()  // 暫停處理

// 檢查處理狀態
console.log(queue.getIsRunning()) // 佇列是否正在處理
console.log(queue.getIsIdle())    // 佇列是否正在運行但為空
```

如果您使用的是選項為響應式的框架適配器，可以將 `started` 選項設為條件值：

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // 根據連線狀態啟動/停止（僅在使用支援響應式選項的框架適配器時有效）
  }
)
```

### 其他功能

Queuer 提供了幾種有用的方法來管理佇列：

```ts
// 佇列檢查
queue.getPeek()           // 查看下一個項目而不移除它
queue.getSize()          // 獲取當前佇列大小
queue.getIsEmpty()       // 檢查佇列是否為空
queue.getIsFull()        // 檢查佇列是否已達 maxSize
queue.getAllItems()   // 獲取所有佇列項目的副本

// 佇列操作
queue.clear()         // 移除所有項目
queue.reset()         // 重置為初始狀態
queue.getExecutionCount() // 獲取已處理項目的數量

// 事件處理
queue.onItemsChange((item) => {
  console.log('已處理:', item)
})
```

### 項目過期

Queuer 支援自動過期在佇列中停留過久的項目。這對於防止處理過時資料或對排隊操作實施超時很有用。

```ts
const queue = new Queuer<number>({
  expirationDuration: 5000, // 項目在 5 秒後過期
  onExpire: (item, queuer) => {
    console.log('項目已過期:', item)
  }
})

// 或使用自定義過期檢查
const queue = new Queuer<number>({
  getIsExpired: (item, addedAt) => {
    // 自定義過期邏輯
    return Date.now() - addedAt > 5000
  },
  onExpire: (item, queuer) => {
    console.log('項目已過期:', item)
  }
})

// 檢查過期統計
console.log(queue.getExpirationCount()) // 已過期項目的數量
```

過期功能特別適用於：
- 防止處理過時資料
- 對排隊操作實施超時
- 透過自動移除舊項目管理記憶體使用
- 處理應僅在有限時間內有效的臨時資料

### 拒絕處理

當佇列達到其最大容量（由 `maxSize` 選項設定）時，新項目將被拒絕。Queuer 提供了處理和監控這些拒絕的方式：

```ts
const queue = new Queuer<number>({
  maxSize: 2, // 僅允許 2 個項目在佇列中
  onReject: (item, queuer) => {
    console.log('佇列已滿。項目被拒絕:', item)
  }
})

queue.addItem(1) // 接受
queue.addItem(2) // 接受
queue.addItem(3) // 拒絕，觸發 onReject 回調

console.log(queue.getRejectionCount()) // 1
```

### 初始項目

您可以在創建佇列時預先填入初始項目：

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // 立即開始處理
})

// 佇列以 [1, 2, 3] 開始並開始處理
```

### 動態配置

Queuer 的選項可以在創建後使用 `setOptions()` 修改，並使用 `getOptions()` 檢索：

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// 變更配置
queue.setOptions({
  wait: 500, // 以兩倍速度處理項目
  started: true // 開始處理
})

// 獲取當前配置
const options = queue.getOptions()
console.log(options.wait) // 500
```

### 效能監控

Queuer 提供了監控其效能的方法：

```ts
const queue = new Queuer<number>()

// 添加並處理一些項目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // 已處理項目的數量
console.log(queue.getRejectionCount()) // 已拒絕項目的數量
```

### 非同步排隊

有關處理具有多個工作線程的非同步操作，請參閱[非同步排隊指南 (Async Queueing Guide)](../guides/async-queueing)，其中涵蓋了 `AsyncQueuer` 類別。

### 框架適配器

每個框架適配器都圍繞排隊器類別構建了方便的鉤子和函式。像 `useQueuer` 或 `useQueueState` 這樣的鉤子是小型包裝器，可以減少您自己程式碼中某些常見使用案例所需的樣板程式碼。
