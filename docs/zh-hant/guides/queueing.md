---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:25:22.066Z'
title: 佇列指南
id: queueing
---
與[速率限制 (Rate Limiting)](../guides/rate-limiting)、[節流 (Throttling)](../guides/throttling) 和[防抖 (Debouncing)](../guides/debouncing) 不同——這些技術會在操作過於頻繁時捨棄執行——佇列器 (queuer) 能確保每個操作都被處理。它們提供了一種管理並控制操作流的方式，且不會遺失任何請求。這使得佇列器特別適合無法接受資料遺失的場景。本指南將介紹 TanStack Pacer 的佇列 (Queueing) 概念。

## 佇列概念

佇列能確保每個操作最終都會被處理，即使它們的到達速度超過處理速度。與其他會捨棄過量操作的執行控制技術不同，佇列會將操作緩存在有序列表中，並根據特定規則處理。這使得佇列成為 TanStack Pacer 中唯一「無損 (lossless)」的執行控制技術。

### 佇列視覺化

```text
佇列 (每 2 個 tick 處理一個項目)
時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  �️
佇列:       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
已執行:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ 不同於速率限制/節流/防抖，
               所有呼叫最終都會按順序處理

             [項目排隊]   [穩定處理]   [清空佇列]
              忙碌時       逐一處理
```

### 使用佇列的時機

當您需要確保每個操作都被處理（即使這意味著引入一些延遲）時，佇列特別重要。這使得佇列非常適合資料一致性和完整性比立即執行更重要的場景。

常見使用案例包括：
- 處理使用者介面中的使用者互動，每個動作都必須被記錄
- 處理需要維持資料一致性的資料庫操作
- 管理必須全部成功完成的 API 請求
- 協調不能被捨棄的背景任務
- 動畫序列中每一幀都很重要的情況
- 表單提交中每個條目都需要保存的情況

### 不適合使用佇列的時機

在以下情況，佇列可能不是最佳選擇：
- 立即反饋比處理每個操作更重要
- 您只關心最新值（改用[防抖 (debouncing)](../guides/debouncing)）

> [!TIP]
> 如果您目前使用速率限制、節流或防抖，但發現捨棄操作導致問題，佇列很可能是您需要的解決方案。

## TanStack Pacer 中的佇列

TanStack Pacer 透過簡單的 `queue` 函式和更強大的 `Queuer` 類別提供佇列功能。雖然其他執行控制技術通常偏好基於函式的 API，但佇列通常受益於類別式 API 提供的額外控制。

### 使用 `queue` 的基本用法

`queue` 函式提供了一種簡單的方式來建立一個始終運行的佇列，並在項目加入時處理它們：

```ts
import { queue } from '@tanstack/pacer'

// 建立一個每秒處理項目的佇列
const processItems = queue<number>({
  wait: 1000,
  onItemsChange: (queuer) => {
    console.log('目前佇列:', queuer.getAllItems())
  }
})

// 加入要處理的項目
processItems(1) // 立即處理
processItems(2) // 1 秒後處理
processItems(3) // 2 秒後處理
```

雖然 `queue` 函式簡單易用，但它僅透過 `addItem` 方法提供基本的始終運行佇列。對於大多數使用案例，您會需要 `Queuer` 類別提供的額外控制和功能。

### 使用 `Queuer` 類別的高級用法

`Queuer` 類別提供對佇列行為和處理的完整控制：

```ts
import { Queuer } from '@tanstack/pacer'

// 建立一個每秒處理項目的佇列
const queue = new Queuer<number>({
  wait: 1000, // 處理項目間等待 1 秒
  onItemsChange: (queuer) => {
    console.log('目前佇列:', queuer.getAllItems())
  }
})

// 開始處理
queue.start()

// 加入要處理的項目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// 項目將逐一處理，每個間隔 1 秒
// 輸出:
// 處理中: 1 (立即)
// 處理中: 2 (1 秒後)
// 處理中: 3 (2 秒後)
```

### 佇列類型與排序

TanStack Pacer 的 Queuer 獨特之處在於它能透過基於位置的 API 適應不同使用案例。同一個 Queuer 可以表現為傳統佇列、堆疊 (stack) 或雙端佇列 (double-ended queue)，全部透過一致的介面實現。

#### FIFO 佇列 (先進先出)

預設行為，項目按加入順序處理。這是最常見的佇列類型，遵循「先加入的項目應先處理」的原則。

```text
FIFO 佇列視覺化:

進入 →  [A][B][C][D] → 離開
         ⬇️         ⬆️
      新項目加入   項目從此處
      此處         處理

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️
佇列:       [ABC]   [BC]    [C]    []
已處理:    A       B       C
```

FIFO 佇列適合：
- 順序重要的任務處理
- 需要按順序處理的訊息佇列
- 文件應按發送順序列印的列印佇列
- 事件必須按時間順序處理的事件處理系統

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

透過將加入和擷取項目的位置都指定為 'back'，佇列器會表現得像堆疊。在堆疊中，最近加入的項目會最先被處理。

```text
LIFO 堆疊視覺化:

     ⬆️ 處理
    [D] ← 最近加入
    [C]
    [B]
    [A] ← 最先加入
     ⬇️ 進入

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️  ⬇️  ⬇️     ⬇️
佇列:       [ABC]   [AB]    [A]    []
已處理:    C       B       A
```

堆疊行為特別適合：
- 最近動作應最先復原的復原/重做系統
- 想返回最近頁面的瀏覽器歷史導航
- 程式語言實作中的函式呼叫堆疊
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

#### 優先佇列 (Priority Queue)

優先佇列透過允許項目根據優先級而非僅加入順序排序，為佇列排序增加了另一個維度。每個項目被賦予一個優先級值，佇列會自動按優先級維護項目順序。

```text
優先佇列視覺化:

進入 →  [P:5][P:3][P:2][P:1] → 離開
          ⬇️           ⬆️
     高優先級項目     低優先級項目
     在此處          最後處理

時間軸: [每個 tick 1 秒]
呼叫:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
佇列:       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
已處理:              5         -          3         2        1
```

優先佇列對以下情況至關重要：
- 某些任務比其他更緊急的任務排程器
- 特定類型流量需要優先處理的網路封包路由
- 高優先級事件應在低優先級之前處理的事件系統
- 某些請求比其他更重要的資源分配

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // 數字越大優先級越高
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// 處理順序: 3, 2, 然後 1
```

### 啟動與停止

`Queuer` 類別支援透過 `start()` 和 `stop()` 方法啟動和停止處理，並可透過 `started` 選項配置為自動啟動：

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
console.log(queue.getIsIdle())    // 佇列是否在運行但為空
```

如果您使用的框架適配器支援反應式選項，可以將 `started` 選項設為條件值：

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // 根據連線狀態啟動/停止（需使用支援反應式選項的框架適配器）
  }
)
```

### 其他功能

Queuer 提供了幾個有用的佇列管理方法：

```ts
// 佇列檢查
queue.getPeek()           // 查看下一個項目而不移除
queue.getSize()          // 獲取當前佇列大小
queue.getIsEmpty()       // 檢查佇列是否為空
queue.getIsFull()        // 檢查佇列是否已達 maxSize
queue.getAllItems()   // 獲取所有佇列項目的副本

// 佇列操作
queue.clear()         // 移除所有項目
queue.reset()         // 重置為初始狀態
queue.getExecutionCount() // 獲取已處理項目數

// 事件處理
queue.onItemsChange((item) => {
  console.log('已處理:', item)
})
```

### 拒絕處理

當佇列達到最大大小（由 `maxSize` 選項設定）時，新項目將被拒絕。Queuer 提供了處理和監控這些拒絕的方法：

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

建立佇列時可以預先填入初始項目：

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // 立即開始處理
})

// 佇列以 [1, 2, 3] 開始並立即處理
```

### 動態配置

Queuer 的選項可以在建立後使用 `setOptions()` 修改，並使用 `getOptions()` 獲取：

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

// 加入並處理一些項目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // 已處理項目數
console.log(queue.getRejectionCount()) // 被拒絕項目數
```

### 非同步佇列

有關使用多個工作處理器處理非同步操作，請參閱[非同步佇列指南 (Async Queueing Guide)](../guides/async-queueing)，該指南涵蓋了 `AsyncQueuer` 類別。

### 框架適配器

每個框架適配器都圍繞佇列類別建構了方便的鉤子和函式。像 `useQueuer` 或 `useQueueState` 這樣的鉤子是小型包裝器，可以減少您自己程式碼中一些常見使用案例所需的樣板程式碼。
