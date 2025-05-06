---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:04:03.475Z'
title: 异步队列指南
id: async-queueing
---
# 异步队列指南

虽然 [Queuer](../guides//queueing) 提供了带有时序控制的同步队列功能，但 `AsyncQueuer` 是专门为处理并发异步操作而设计的。它实现了传统意义上的"任务池"或"工作池"模式，允许多个操作同时处理，同时保持对并发性和时序的控制。该实现主要借鉴自 [Swimmer](https://github.com/tannerlinsley/swimmer)，这是 Tanner 自 2017 年以来一直服务于 JavaScript 社区的原生任务池工具。

## 异步队列概念

异步队列通过添加并发处理能力扩展了基本的队列概念。异步队列无需一次处理一个项目，而是可以同时处理多个项目，同时仍保持执行顺序和控制。这在处理 I/O 操作、网络请求或任何大部分时间处于等待状态而非消耗 CPU 的任务时特别有用。

### 异步队列可视化

```text
异步队列 (并发数: 2, 等待: 2 个时间单位)
时间轴: [每个时间单位 1 秒]
调用:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
队列:       [ABC]   [C]    [CDE]    [E]    []
活跃任务:    [A,B]   [B,C]  [C,D]    [D,E]  [E]
已完成:      -       A      B        C      D,E
             [=================================================================]
             ^ 与常规队列不同，多个项目
               可以并发处理

             [项目排队]   [同时处理 2 个]   [完成所有项目]
              当繁忙时     处理间有等待间隔
```

### 何时使用异步队列

异步队列在以下场景特别有效：
- 需要并发处理多个异步操作
- 控制同时进行的操作数量
- 正确处理基于 Promise 的任务及其错误处理
- 在最大化吞吐量的同时保持顺序
- 处理可以并行运行的背景任务

常见用例包括：
- 带有限流机制的并发 API 请求
- 同时处理多个文件上传
- 运行并行数据库操作
- 处理多个 WebSocket 连接
- 带背压机制的数据流处理
- 管理资源密集型背景任务

### 何时不使用异步队列

AsyncQueuer 非常通用，可以在许多情况下使用。实际上，只有当您不打算利用其所有功能时，它才不是一个好的选择。如果您不需要所有排队的执行都通过，请改用 [节流][../guides/throttling]。如果您不需要并发处理，请改用 [队列][../guides/queueing]。

## TanStack Pacer 中的异步队列

TanStack Pacer 通过简单的 `asyncQueue` 函数和更强大的 `AsyncQueuer` 类提供异步队列功能。

### 使用 `asyncQueue` 的基本用法

`asyncQueue` 函数提供了一种创建始终运行的异步队列的简单方法：

```ts
import { asyncQueue } from '@tanstack/pacer'

// 创建一个最多同时处理 2 个项目的队列
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('活跃任务:', queuer.getActiveItems().length)
  }
})

// 添加要处理的异步任务
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

`asyncQueue` 函数的使用有些受限，因为它只是 `AsyncQueuer` 类的包装器，仅暴露了 `addItem` 方法。要对队列进行更多控制，请直接使用 `AsyncQueuer` 类。

### 使用 `AsyncQueuer` 类的高级用法

`AsyncQueuer` 类提供了对异步队列行为的完全控制：

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // 同时处理 2 个项目
  wait: 1000,     // 启动新项目之间等待 1 秒
  started: true   // 立即开始处理
})

// 添加错误和成功处理程序
queue.onError((error) => {
  console.error('任务失败:', error)
})

queue.onSuccess((result) => {
  console.log('任务完成:', result)
})

// 添加异步任务
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### 队列类型和排序

AsyncQueuer 支持不同的队列策略来处理各种处理需求。每种策略决定了任务如何被添加和处理。

#### FIFO 队列 (先进先出)

FIFO 队列按照任务添加的确切顺序处理任务，非常适合保持顺序：

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // 默认
  getItemsFrom: 'front', // 默认
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// 处理顺序: first 和 second 并发
```

#### LIFO 栈 (后进先出)

LIFO 栈优先处理最近添加的任务，适用于优先处理新任务：

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // 优先处理最新项目
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// 处理顺序: second 先处理，然后 first
```

#### 优先级队列

优先级队列根据任务分配的优先级值处理任务，确保重要任务优先处理。有两种指定优先级的方式：

1. 附加到任务的静态优先级值：
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// 创建带静态优先级的任务
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

// 以任意顺序添加任务 - 它们将按优先级处理(数字越大优先级越高)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// 处理顺序: high 和 medium 并发，然后 low
```

2. 使用 `getPriority` 选项进行动态优先级计算：
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // 根据任务属性或其他因素计算优先级
    // 数字越大优先级越高
    return calculateTaskPriority(task)
  }
})

// 添加任务 - 优先级将动态计算
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

优先级队列在以下场景至关重要：
- 任务有不同的重要性级别
- 关键操作需要优先运行
- 需要基于优先级的灵活任务排序
- 资源分配应优先考虑重要任务
- 需要根据任务属性或外部因素动态确定优先级

### 错误处理

AsyncQueuer 提供了全面的错误处理能力，确保稳健的任务处理。您可以在队列级别和单个任务级别处理错误：

```ts
const queue = new AsyncQueuer<string>()

// 全局错误处理
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('任务失败:', error)
  },
  onSuccess: (result) => {
    console.log('任务成功:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('任务失败:', result)
    } else {
      console.log('任务成功:', result)
    }
  }
})

// 按任务处理错误
queue.addItem(async () => {
  throw new Error('任务失败')
}).catch(error => {
  console.error('单个任务错误:', error)
})
```

### 队列管理

AsyncQueuer 提供了多种方法来监控和控制队列状态：

```ts
// 队列检查
queue.getPeek()           // 查看下一个项目而不移除它
queue.getSize()          // 获取当前队列大小
queue.getIsEmpty()       // 检查队列是否为空
queue.getIsFull()        // 检查队列是否达到 maxSize
queue.getAllItems()   // 获取所有排队项目的副本
queue.getActiveItems() // 获取当前正在处理的项目
queue.getPendingItems() // 获取等待处理的项目

// 队列操作
queue.clear()         // 移除所有项目
queue.reset()         // 重置到初始状态
queue.getExecutionCount() // 获取已处理项目的数量

// 处理控制
queue.start()         // 开始处理项目
queue.stop()          // 暂停处理
queue.getIsRunning()     // 检查队列是否正在处理
queue.getIsIdle()        // 检查队列是否为空且未处理
```

### 任务回调

AsyncQueuer 提供了三种类型的回调来监控任务执行：

```ts
const queue = new AsyncQueuer<string>()

// 处理任务成功完成
const unsubSuccess = queue.onSuccess((result) => {
  console.log('任务成功:', result)
})

// 处理任务错误
const unsubError = queue.onError((error) => {
  console.error('任务失败:', error)
})

// 无论成功/失败都处理任务完成
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('任务失败:', result)
  } else {
    console.log('任务成功:', result)
  }
})

// 不再需要时取消订阅回调
unsubSuccess()
unsubError()
unsubSettled()
```

### 拒绝处理

当队列达到其最大大小(由 `maxSize` 选项设置)时，新任务将被拒绝。AsyncQueuer 提供了处理和监控这些拒绝的方法：

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // 只允许 2 个任务在队列中
  onReject: (task, queuer) => {
    console.log('队列已满。任务被拒绝:', task)
  }
})

queue.addItem(async () => 'first') // 接受
queue.addItem(async () => 'second') // 接受
queue.addItem(async () => 'third') // 拒绝，触发 onReject 回调

console.log(queue.getRejectionCount()) // 1
```

### 初始任务

您可以在创建异步队列时预填充初始任务：

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // 立即开始处理
})

// 队列以三个任务开始并立即开始处理它们
```

### 动态配置

AsyncQueuer 的选项可以在创建后使用 `setOptions()` 修改，并使用 `getOptions()` 检索：

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// 更改配置
queue.setOptions({
  concurrency: 4, // 同时处理更多任务
  started: true // 开始处理
})

// 获取当前配置
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### 活跃和待处理任务

AsyncQueuer 提供了监控活跃和待处理任务的方法：

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// 添加一些任务
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// 监控任务状态
console.log(queue.getActiveItems().length) // 当前正在处理的任务
console.log(queue.getPendingItems().length) // 等待处理的任务
```

### 框架适配器

每个框架适配器都围绕异步队列类构建了便捷的钩子和函数。像 `useAsyncQueuer` 或 `useAsyncQueuedState` 这样的钩子是小型包装器，可以减少您自己代码中一些常见用例所需的样板代码。
