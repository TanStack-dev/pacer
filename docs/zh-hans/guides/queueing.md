---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:20:03.266Z'
title: 队列指南
id: queueing
---
与[速率限制 (Rate Limiting)](../guides/rate-limiting)、[节流 (Throttling)](../guides/throttling) 和[防抖 (Debouncing)](../guides/debouncing) 不同——这些技术会在操作过于频繁时丢弃执行，而队列器 (queuer) 能确保每个操作都被处理。它们提供了一种管理控制操作流的方法，且不会丢失任何请求。这使得队列成为数据丢失不可接受场景的理想选择。本指南将介绍 TanStack Pacer 的队列 (Queueing) 概念。

## 队列概念

队列技术确保每个操作最终都会被处理，即使它们的到达速度超过了处理能力。与其他会丢弃超额操作的执行控制技术不同，队列将操作缓冲在一个有序列表中，并根据特定规则进行处理。这使得队列成为 TanStack Pacer 中唯一"无损"的执行控制技术。

### 队列可视化

```text
队列 (每 2 个时间单位处理一个项目)
时间轴: [每个刻度代表 1 秒]
调用:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
队列:       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
已执行:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ 与速率限制/节流/防抖不同，
               所有调用最终都会按顺序处理

             [积压时排队]   [逐个稳定处理]   [队列清空]
              忙碌时         依次处理        空队列
```

### 适用场景

当您需要确保每个操作都被处理（即使这意味着引入一些延迟）时，队列技术尤为重要。这使得它非常适合数据一致性和完整性比立即执行更重要的场景。

常见用例包括：
- 必须记录每个操作的 UI 用户交互处理
- 需要保持数据一致性的数据库操作
- 必须全部成功完成的 API 请求处理
- 不能丢弃的后台任务协调
- 每一帧都很重要的动画序列
- 需要保存每个条目的表单提交

### 不适用场景

以下情况可能不适合使用队列：
- 即时反馈比处理每个操作更重要时
- 只关心最新值时（改用[防抖 (debouncing)](../guides/debouncing)）

> [!TIP]
> 如果您当前正在使用速率限制、节流或防抖，但发现丢弃操作导致问题，队列很可能是您需要的解决方案。

## TanStack Pacer 中的队列

TanStack Pacer 通过简单的 `queue` 函数和更强大的 `Queuer` 类提供队列功能。虽然其他执行控制技术通常偏向基于函数的 API，但队列通常受益于基于类的 API 提供的额外控制。

### 使用 `queue` 基础用法

`queue` 函数提供了一种简单的方法来创建一个始终运行的队列，在添加项目时立即处理：

```ts
import { queue } from '@tanstack/pacer'

// 创建一个每秒处理项目的队列
const processItems = queue<number>({
  wait: 1000,
  onItemsChange: (queuer) => {
    console.log('当前队列:', queuer.getAllItems())
  }
})

// 添加待处理项目
processItems(1) // 立即处理
processItems(2) // 1 秒后处理
processItems(3) // 2 秒后处理
```

虽然 `queue` 函数使用简单，但它仅通过 `addItem` 方法提供基本的始终运行队列。对于大多数用例，您会需要 `Queuer` 类提供的额外控制和功能。

### 使用 `Queuer` 类高级用法

`Queuer` 类提供对队列行为和处理的完全控制：

```ts
import { Queuer } from '@tanstack/pacer'

// 创建一个每秒处理项目的队列
const queue = new Queuer<number>({
  wait: 1000, // 每个项目处理间隔 1 秒
  onItemsChange: (queuer) => {
    console.log('当前队列:', queuer.getAllItems())
  }
})

// 开始处理
queue.start()

// 添加待处理项目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// 项目将逐个处理，间隔 1 秒
// 输出:
// 处理: 1 (立即)
// 处理: 2 (1 秒后)
// 处理: 3 (2 秒后)
```

### 队列类型与排序

TanStack Pacer 的 Queuer 独特之处在于它能够通过基于位置的 API 适应不同的用例。同一个 Queuer 可以通过一致的接口表现为传统队列、栈或双端队列。

#### 先进先出队列 (FIFO)

默认行为，项目按添加顺序处理。这是最常见的队列类型，遵循先添加的项目应该先处理的原则。

```text
FIFO 队列可视化:

入口 →  [A][B][C][D] → 出口
         ⬇️         ⬆️
      新项目在此   项目在此
      添加        处理

时间轴: [每个刻度代表 1 秒]
调用:        ⬇️  ⬇️  ⬇️     ⬇️
队列:       [ABC]   [BC]    [C]    []
已处理:    A       B       C
```

FIFO 队列适用于：
- 顺序重要的任务处理
- 需要按顺序处理的消息队列
- 文档应按发送顺序打印的打印队列
- 必须按时间顺序处理事件的事件处理系统

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // 默认
  getItemsFrom: 'front', // 默认
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// 处理顺序: 1, 然后 2
```

#### 后进先出栈 (LIFO)

通过将添加和检索位置都指定为 'back'，队列器表现为栈。在栈中，最近添加的项目最先被处理。

```text
LIFO 栈可视化:

     ⬆️ 处理
    [D] ← 最近添加
    [C]
    [B]
    [A] ← 最先添加
     ⬇️ 入口

时间轴: [每个刻度代表 1 秒]
调用:        ⬇️  ⬇️  ⬇️     ⬇️
队列:       [ABC]   [AB]    [A]    []
已处理:    C       B       A
```

栈行为特别适用于：
- 应该先撤销最近操作的撤销/重做系统
- 想要返回最近页面的浏览器历史导航
- 编程语言实现中的函数调用栈
- 深度优先遍历算法

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // 默认
  getItemsFrom: 'back', // 覆盖默认以实现栈行为
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// 项目处理顺序: 2, 然后 1

stack.getNextItem('back') // 从队列尾部而非头部获取下一个项目
```

#### 优先级队列

优先级队列通过允许项目根据优先级而非仅插入顺序排序，为队列排序增加了另一个维度。每个项目被分配一个优先级值，队列自动按优先级顺序维护项目。

```text
优先级队列可视化:

入口 →  [P:5][P:3][P:2][P:1] → 出口
          ⬇️           ⬆️
     高优先级       低优先级
     项目在此       最后处理

时间轴: [每个刻度代表 1 秒]
调用:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
队列:       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
已处理:              5         -          3         2        1
```

优先级队列对于以下场景至关重要：
- 某些任务比其他任务更紧急的任务调度器
- 某些类型流量需要优先处理的网络数据包路由
- 高优先级事件应在低优先级事件之前处理的事件系统
- 某些请求比其他请求更重要的资源分配

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // 数值越大优先级越高
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// 处理顺序: 3, 2, 然后 1
```

### 启动与停止

`Queuer` 类通过 `start()` 和 `stop()` 方法支持启动和停止处理，并可通过 `started` 选项配置为自动启动：

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // 初始暂停
})

// 控制处理
queue.start() // 开始处理项目
queue.stop()  // 暂停处理

// 检查处理状态
console.log(queue.getIsRunning()) // 队列当前是否正在处理
console.log(queue.getIsIdle())    // 队列是否正在运行但为空
```

如果您使用的框架适配器支持响应式选项，可以将 `started` 选项设置为条件值：

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // 基于连接状态启动/停止（使用支持响应式选项的框架适配器时）
  }
)
```

### 附加功能

Queuer 提供了几个有用的队列管理方法：

```ts
// 队列检查
queue.getPeek()           // 查看下一个项目而不移除
queue.getSize()          // 获取当前队列大小
queue.getIsEmpty()       // 检查队列是否为空
queue.getIsFull()        // 检查队列是否达到 maxSize
queue.getAllItems()   // 获取所有排队项目的副本

// 队列操作
queue.clear()         // 移除所有项目
queue.reset()         // 重置到初始状态
queue.getExecutionCount() // 获取已处理项目数量

// 事件处理
queue.onItemsChange((item) => {
  console.log('已处理:', item)
})
```

### 拒绝处理

当队列达到其最大大小（由 `maxSize` 选项设置）时，新项目将被拒绝。Queuer 提供了处理和监控这些拒绝的方法：

```ts
const queue = new Queuer<number>({
  maxSize: 2, // 只允许 2 个项目在队列中
  onReject: (item, queuer) => {
    console.log('队列已满。项目被拒绝:', item)
  }
})

queue.addItem(1) // 接受
queue.addItem(2) // 接受
queue.addItem(3) // 拒绝，触发 onReject 回调

console.log(queue.getRejectionCount()) // 1
```

### 初始项目

创建队列时可以预填充初始项目：

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // 立即开始处理
})

// 队列初始为 [1, 2, 3] 并开始处理
```

### 动态配置

Queuer 的选项可以在创建后使用 `setOptions()` 修改，并使用 `getOptions()` 检索：

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// 更改配置
queue.setOptions({
  wait: 500, // 以两倍速度处理项目
  started: true // 开始处理
})

// 获取当前配置
const options = queue.getOptions()
console.log(options.wait) // 500
```

### 性能监控

Queuer 提供了监控其性能的方法：

```ts
const queue = new Queuer<number>()

// 添加并处理一些项目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // 已处理项目数量
console.log(queue.getRejectionCount()) // 被拒绝项目数量
```

### 异步队列

要处理具有多个工作线程的异步操作，请参阅[异步队列指南 (Async Queueing Guide)](../guides/async-queueing)，其中介绍了 `AsyncQueuer` 类。

### 框架适配器

每个框架适配器都在队列器类周围构建了方便的钩子和函数。像 `useQueuer` 或 `useQueueState` 这样的钩子是小型包装器，可以减少您自己代码中一些常见用例所需的样板文件。
