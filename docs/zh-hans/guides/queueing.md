---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:04:17.173Z'
title: 队列指南
id: queueing
---
与[限流 (Rate Limiting)](../guides/rate-limiting)、[节流 (Throttling)](../guides/throttling) 和[防抖 (Debouncing)](../guides/debouncing) 不同，队列器 (queuer) 可配置为确保每个操作都被处理。它们提供了一种管理控制操作流而不丢失任何请求的方式，这使得队列成为数据丢失不可接受场景的理想选择。队列还可设置最大容量，这对防止内存泄漏等问题很有帮助。本指南将介绍 TanStack Pacer 的队列概念。

## 队列概念

队列确保每个操作最终都会被处理，即使它们的到达速度超过处理速度。与其他会丢弃多余操作的执行控制技术不同，队列将操作缓存在有序列表中并按特定规则处理。这使得队列成为 TanStack Pacer 中唯一的"无损"执行控制技术——除非指定了 `maxSize` 导致缓冲区满时拒绝新条目。

### 队列可视化

```text
队列（每 2 个时间单位处理一个条目）
时间轴: [每个刻度代表 1 秒]
调用:      ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
队列:     [ABC]   [BC]    [BCDE]    [DE]    [E]    []
已执行:    ✅     ✅       ✅        ✅      ✅     ✅
           [=================================================================]
           ^ 与限流/节流/防抖不同，
             所有调用最终按顺序处理

           [条目排队]   [逐个处理]   [清空队列]
            忙碌时        稳定处理      完成
```

### 适用场景

当需要确保每个操作都被处理（即使引入延迟）时，队列尤为重要。这使其成为数据一致性和完整性比即时执行更重要的理想选择。使用 `maxSize` 时，它还可作为缓冲区防止系统因过多待处理操作而过载。

常见用例包括：
- 预加载数据而不使系统过载
- 处理必须记录每个交互的 UI 用户操作
- 需要保持数据一致性的数据库操作
- 必须全部成功完成的 API 请求
- 不能丢弃的后台任务协调
- 每一帧都重要的动画序列
- 需要保存每个条目的表单提交
- 使用 `maxSize` 限制容量的数据流缓冲

### 不适用场景

以下情况可能不适合使用队列：
- 即时反馈比处理每个操作更重要
- 只关心最新值（应使用[防抖 (debouncing)](../guides/debouncing)）

> [!TIP]
> 如果当前使用限流、节流或防抖但发现丢弃操作导致问题，队列很可能是您需要的解决方案。

## TanStack Pacer 中的队列

TanStack Pacer 通过简易的 `queue` 函数和更强大的 `Queuer` 类提供队列功能。虽然其他执行控制技术通常偏向函数式 API，但队列往往受益于基于类的 API 提供的额外控制。

### 基础用法：`queue` 函数

`queue` 函数提供创建自动运行队列的简单方式：

```ts
import { queue } from '@tanstack/pacer'

// 创建每秒处理条目的队列
const processItems = queue<number>({
  wait: 1000,
  maxSize: 10, // 可选：限制队列大小防止内存/时间问题
  onItemsChange: (queuer) => {
    console.log('当前队列:', queuer.getAllItems())
  }
})

// 添加待处理条目
processItems(1) // 立即处理
processItems(2) // 1 秒后处理
processItems(3) // 2 秒后处理
```

虽然 `queue` 函数简单易用，但它仅通过 `addItem` 方法提供基础自动运行队列。大多数用例需要 `Queuer` 类提供的额外控制和功能。

### 高级用法：`Queuer` 类

`Queuer` 类提供完整的队列行为控制：

```ts
import { Queuer } from '@tanstack/pacer'

// 创建每秒处理条目的队列
const queue = new Queuer<number>({
  wait: 1000, // 条目间处理间隔 1 秒
  maxSize: 5, // 可选：限制队列大小
  onItemsChange: (queuer) => {
    console.log('当前队列:', queuer.getAllItems())
  }
})

// 开始处理
queue.start()

// 添加待处理条目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// 条目将按每秒一个的间隔处理
// 输出:
// 处理: 1 (立即)
// 处理: 2 (1 秒后)
// 处理: 3 (2 秒后)
```

### 队列类型与排序

TanStack Pacer 的 Queuer 独特之处在于通过基于位置的 API 适应不同用例。同一个 Queuer 可作为传统队列 (queue)、栈 (stack) 或双端队列 (deque) 使用。

#### 先进先出队列 (FIFO)

默认行为，条目按添加顺序处理。这是最常见队列类型，遵循"先到先服务"原则。使用 `maxSize` 时，队列满将拒绝新条目。

```text
FIFO 队列可视化 (maxSize=3):

入口 →  [A][B][C] → 出口
         ⬇️     ⬆️
      新条目添加处  条目处理处
      (满时拒绝)

时间轴: [每个刻度 1 秒]
调用:      ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
队列:     [ABC]   [BC]    [C]    []
已处理:    A       B       C
已拒绝:    D      E
```

FIFO 队列适用于：
- 顺序重要的任务处理
- 需按序处理的消息队列
- 按发送顺序打印的打印队列
- 需按时间顺序处理事件的系统

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

通过指定 'back' 作为添加和获取位置，队列表现为栈。栈中最后添加的条目最先处理。使用 `maxSize` 时，栈满将拒绝新条目。

```text
LIFO 栈可视化 (maxSize=3):

     ⬆️ 处理
    [C] ← 最近添加
    [B]
    [A] ← 最早添加
     ⬇️ 入口
     (满时拒绝)

时间轴: [每个刻度 1 秒]
调用:      ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
队列:     [ABC]   [AB]    [A]    []
已处理:    C       B       A
已拒绝:    D      E
```

栈行为特别适用于：
- 应先撤销最近操作的撤销/重做系统
- 返回最近页面的浏览器历史导航
- 编程语言实现的函数调用栈
- 深度优先遍历算法

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // 默认
  getItemsFrom: 'back', // 覆盖默认实现栈行为
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// 处理顺序: 2, 然后 1

stack.getNextItem('back') // 从队列后端而非前端获取下个条目
```

#### 优先级队列

优先级队列通过允许按优先级值排序（而不仅是插入顺序）增加了排序维度。每个条目分配优先级值，队列自动维护优先级顺序。使用 `maxSize` 时，低优先级条目可能在队列满时被拒绝。

```text
优先级队列可视化 (maxSize=3):

入口 →  [P:5][P:3][P:2] → 出口
          ⬇️           ⬆️
     高优先级条目     低优先级条目
     在此处添加      最后处理
     (满时拒绝)

时间轴: [每个刻度 1 秒]
调用:      ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
队列:     [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
已处理:            5         -          3         2        1
已拒绝:                     4
```

优先级队列适用于：
- 某些任务更紧急的任务调度器
- 特定流量需优先处理的网络包路由
- 高优先级事件应先处理的系统
- 某些请求更重要的资源分配

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

`Queuer` 类通过 `start()` 和 `stop()` 方法支持启停处理，也可通过 `started` 选项自动启动：

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // 初始暂停
})

// 控制处理
queue.start() // 开始处理
queue.stop()  // 暂停处理

// 检查状态
console.log(queue.getIsRunning()) // 是否正在处理
console.log(queue.getIsIdle())    // 是否运行但队列为空
```

如果使用支持响应式选项的框架适配器，可将 `started` 设为条件值：

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // 根据连接状态启停（需框架适配器支持响应式选项）
  }
)
```

### 其他功能

Queuer 提供多种队列管理方法：

```ts
// 队列检查
queue.getPeek()           // 查看下个条目不移除
queue.getSize()          // 获取当前队列大小
queue.getIsEmpty()       // 检查是否为空
queue.getIsFull()        // 检查是否达到 maxSize
queue.getAllItems()   // 获取所有队列条目副本

// 队列操作
queue.clear()         // 移除所有条目
queue.reset()         // 重置初始状态
queue.getExecutionCount() // 获取已处理条目数

// 事件处理
queue.onItemsChange((item) => {
  console.log('已处理:', item)
})
```

### 条目过期

Queuer 支持自动使在队列中过久的条目过期，防止处理陈旧数据或实现操作超时。

```ts
const queue = new Queuer<number>({
  expirationDuration: 5000, // 条目 5 秒后过期
  onExpire: (item, queuer) => {
    console.log('条目过期:', item)
  }
})

// 或使用自定义过期检查
const queue = new Queuer<number>({
  getIsExpired: (item, addedAt) => {
    // 自定义过期逻辑
    return Date.now() - addedAt > 5000
  },
  onExpire: (item, queuer) => {
    console.log('条目过期:', item)
  }
})

// 检查过期统计
console.log(queue.getExpirationCount()) // 已过期条目数
```

过期功能特别适用于：
- 防止处理陈旧数据
- 实现队列操作超时
- 通过自动移除旧条目管理内存
- 处理有限时间内有效的临时数据

### 拒绝处理

队列达到 `maxSize` 时将拒绝新条目，Queuer 提供处理和监控方式：

```ts
const queue = new Queuer<number>({
  maxSize: 2, // 仅允许 2 个条目
  onReject: (item, queuer) => {
    console.log('队列已满。条目被拒绝:', item)
  }
})

queue.addItem(1) // 接受
queue.addItem(2) // 接受
queue.addItem(3) // 拒绝，触发 onReject 回调

console.log(queue.getRejectionCount()) // 1
```

### 初始条目

创建时可预填充队列：

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // 立即开始处理
})

// 队列初始为 [1, 2, 3] 并开始处理
```

### 动态配置

可通过 `setOptions()` 修改 Queuer 配置，通过 `getOptions()` 获取：

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// 修改配置
queue.setOptions({
  wait: 500, // 处理速度加倍
  started: true // 开始处理
})

// 获取当前配置
const options = queue.getOptions()
console.log(options.wait) // 500
```

### 性能监控

Queuer 提供性能监控方法：

```ts
const queue = new Queuer<number>()

// 添加并处理条目
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // 已处理条目数
console.log(queue.getRejectionCount()) // 已拒绝条目数
```

### 异步队列

处理多工作线程的异步操作，请参阅[异步队列指南 (Async Queueing Guide)](../guides/async-queueing) 了解 `AsyncQueuer` 类。

### 框架适配器

每个框架适配器围绕队列类构建便捷钩子和函数。如 `useQueuer` 或 `useQueueState` 等钩子可减少常见用例的样板代码。
