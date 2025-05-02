---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:25:19.496Z'
title: 非同步佇列指南
id: async-queueing
---
# 非同步佇列 (Async Queueing) 指南

雖然 [Queuer](../guides//queueing) 提供了具備時間控制的同步佇列功能，但 `AsyncQueuer` 是專門設計用來處理並行非同步操作的。它實現了傳統上稱為「任務池 (task pool)」或「工作池 (worker pool)」的模式，允許同時處理多個操作，同時保持對並行性和時間的控制。此實作大部分是從 [Swimmer](https://github.com/tannerlinsley/swimmer) 複製而來，這是 Tanner 原創的任務池工具，自 2017 年以來一直為 JavaScript 社群服務。

## 非同步佇列概念

非同步佇列透過增加並行處理能力，擴展了基本佇列概念。非同步佇列可以同時處理多個項目，而不是一次處理一個項目，同時仍保持執行順序和控制。這在處理 I/O 操作、網路請求或任何大部分時間花在等待而非消耗 CPU 的任務時特別有用。

### 非同步佇列視覺化

```text
非同步佇列 (並行數: 2, 等待: 2 ticks)
時間軸: [每秒 1 tick]
呼叫:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
佇列:       [ABC]   [C]    [CDE]    [E]    []
進行中:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
已完成:       -       A      B        C      D,E
             [=================================================================]
             ^ 與常規佇列不同，多個項目
               可以並行處理

             [項目排隊]   [同時處理 2 個]   [完成所有項目]
              當忙碌時     並在之間等待
```

### 何時使用非同步佇列

非同步佇列在以下情況特別有效：
- 需要並行處理多個非同步操作
- 控制同時進行的操作數量
- 處理基於 Promise 的任務並具備適當的錯誤處理
- 在最大化吞吐量的同時保持順序
- 處理可以平行運行的背景任務

常見使用案例包括：
- 進行具有速率限制的並行 API 請求
- 同時處理多個檔案上傳
- 執行平行資料庫操作
- 處理多個 websocket 連接
- 處理具有背壓 (backpressure) 的資料流
- 管理資源密集的背景任務

### 何時不使用非同步佇列

`AsyncQueuer` 非常多功能，可以在許多情況下使用。實際上，只有在您不打算利用其所有功能時才不適合使用。如果不需要所有排隊的執行都通過，請改用 [Throttling][../guides/throttling]。如果不需要並行處理，請改用 [Queueing][../guides/queueing]。

## TanStack Pacer 中的非同步佇列

TanStack Pacer 透過簡單的 `asyncQueue` 函式和更強大的 `AsyncQueuer` 類別提供非同步佇列功能。

### 使用 `asyncQueue` 的基本用法

`asyncQueue` 函式提供了一種簡單的方法來創建始終運行的非同步佇列：

```ts
import { asyncQueue } from '@tanstack/pacer'

// 創建一個最多同時處理 2 個項目的佇列
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('進行中任務:', queuer.getActiveItems().length)
  }
})

// 添加非同步任務進行處理
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

`asyncQueue` 函式的使用有些限制，因為它只是 `AsyncQueuer` 類別的包裝，僅公開 `addItem` 方法。要對佇列進行更多控制，請直接使用 `AsyncQueuer` 類別。

### 使用 `AsyncQueuer` 類別的高級用法

`AsyncQueuer` 類別提供了對非同步佇列行為的完整控制：

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // 同時處理 2 個項目
  wait: 1000,     // 在開始新項目之間等待 1 秒
  started: true   // 立即開始處理
})

// 添加錯誤和成功處理器
queue.onError((error) => {
  console.error('任務失敗:', error)
})

queue.onSuccess((result) => {
  console.log('任務完成:', result)
})

// 添加非同步任務
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### 佇列類型和順序

`AsyncQueuer` 支援不同的佇列策略來處理各種處理需求。每種策略決定了任務如何被添加和從佇列中處理。

#### FIFO 佇列 (先進先出)

FIFO 佇列按照任務添加的順序處理任務，非常適合保持順序：

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // 預設
  getItemsFrom: 'front', // 預設
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// 處理順序: first 和 second 並行
```

#### LIFO 堆疊 (後進先出)

LIFO 堆疊優先處理最近添加的任務，適用於優先處理新任務：

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // 優先處理最新項目
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// 處理順序: second 先, 然後 first
```

#### 優先佇列

優先佇列根據任務分配的優先值處理任務，確保重要任務優先處理。有兩種指定優先級的方式：

1. 附加到任務的靜態優先值：
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// 創建具有靜態優先值的任務
const lowPriorityTask = Object.assign(
  async () => 'low priority result',
  { priority: 1 }
)

const highPriorityTask = Object.assign(
  async () => 'high priority result',
  { priority: 3 }
)

const mediumPriorityTask = Object.assign(
  async () => 'medium priority result',
  { priority: 2 }
)

// 以任意順序添加任務 - 將按優先級處理 (數字越大優先級越高)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// 處理順序: high 和 medium 並行, 然後 low
```

2. 使用 `getPriority` 選項進行動態優先級計算：
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // 根據任務屬性或其他因素計算優先級
    // 數字越大優先級越高
    return calculateTaskPriority(task)
  }
})

// 添加任務 - 優先級將動態計算
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

優先佇列在以下情況至關重要：
- 任務有不同的重要性級別
- 關鍵操作需要優先執行
- 需要基於優先級的靈活任務排序
- 資源分配應優先考慮重要任務
- 需要根據任務屬性或外部因素動態確定優先級

### 錯誤處理

`AsyncQueuer` 提供了全面的錯誤處理能力，以確保穩健的任務處理。您可以在佇列級別和單個任務級別處理錯誤：

```ts
const queue = new AsyncQueuer<string>()

// 全局處理錯誤
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('任務失敗:', error)
  },
  onSuccess: (result) => {
    console.log('任務成功:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('任務失敗:', result)
    } else {
      console.log('任務成功:', result)
    }
  }
})

// 按任務處理錯誤
queue.addItem(async () => {
  throw new Error('任務失敗')
}).catch(error => {
  console.error('單個任務錯誤:', error)
})
```

### 佇列管理

`AsyncQueuer` 提供了多種方法來監控和控制佇列狀態：

```ts
// 佇列檢查
queue.getPeek()           // 查看下一個項目而不移除它
queue.getSize()          // 獲取當前佇列大小
queue.getIsEmpty()       // 檢查佇列是否為空
queue.getIsFull()        // 檢查佇列是否已達到 maxSize
queue.getAllItems()   // 獲取所有排隊項目的副本
queue.getActiveItems() // 獲取當前正在處理的項目
queue.getPendingItems() // 獲取等待處理的項目

// 佇列操作
queue.clear()         // 移除所有項目
queue.reset()         // 重置為初始狀態
queue.getExecutionCount() // 獲取已處理項目的數量

// 處理控制
queue.start()         // 開始處理項目
queue.stop()          // 暫停處理
queue.getIsRunning()     // 檢查佇列是否正在處理
queue.getIsIdle()        // 檢查佇列是否為空且未處理
```

### 任務回調

`AsyncQueuer` 提供了三種類型的回調來監控任務執行：

```ts
const queue = new AsyncQueuer<string>()

// 處理成功的任務完成
const unsubSuccess = queue.onSuccess((result) => {
  console.log('任務成功:', result)
})

// 處理任務錯誤
const unsubError = queue.onError((error) => {
  console.error('任務失敗:', error)
})

// 無論成功或失敗都處理任務完成
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('任務失敗:', result)
  } else {
    console.log('任務成功:', result)
  }
})

// 不再需要時取消訂閱回調
unsubSuccess()
unsubError()
unsubSettled()
```

### 拒絕處理

當佇列達到其最大大小 (由 `maxSize` 選項設定) 時，新任務將被拒絕。`AsyncQueuer` 提供了處理和監控這些拒絕的方法：

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // 只允許 2 個任務在佇列中
  onReject: (task, queuer) => {
    console.log('佇列已滿。任務被拒絕:', task)
  }
})

queue.addItem(async () => 'first') // 接受
queue.addItem(async () => 'second') // 接受
queue.addItem(async () => 'third') // 拒絕, 觸發 onReject 回調

console.log(queue.getRejectionCount()) // 1
```

### 初始任務

您可以在創建非同步佇列時預先填充初始任務：

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // 立即開始處理
})

// 佇列開始時有三個任務並開始處理它們
```

### 動態配置

`AsyncQueuer` 的選項可以在創建後使用 `setOptions()` 修改，並使用 `getOptions()` 獲取：

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// 更改配置
queue.setOptions({
  concurrency: 4, // 同時處理更多任務
  started: true // 開始處理
})

// 獲取當前配置
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### 活動中和待處理任務

`AsyncQueuer` 提供了監控活動中和待處理任務的方法：

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// 添加一些任務
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// 監控任務狀態
console.log(queue.getActiveItems().length) // 當前正在處理的任務
console.log(queue.getPendingItems().length) // 等待處理的任務
```

### 框架適配器

每個框架適配器都圍繞非同步佇列類別構建了方便的鉤子和函式。像 `useAsyncQueuer` 或 `useAsyncQueuerState` 這樣的鉤子是小型包裝器，可以減少您自己代碼中一些常見用例所需的樣板代碼。
