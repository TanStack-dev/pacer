---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:09:04.500Z'
title: 非同期キューイングガイド
id: async-queueing
---
# 非同期キューイングガイド

[Queuer](../guides//queueing)がタイミング制御付きの同期キューイングを提供する一方で、`AsyncQueuer`は同時実行される非同期操作を処理するために特別に設計されています。これは従来「タスクプール」または「ワーカープール」パターンとして知られるものを実装しており、複数の操作を同時に処理しながら並列性とタイミングを制御できます。この実装は主に[Swimmer](https://github.com/tannerlinsley/swimmer)からコピーされたもので、Tannerが2017年からJavaScriptコミュニティに提供しているオリジナルのタスクプーリングユーティリティです。

## 非同期キューイングの概念

非同期キューイングは基本的なキューイングの概念を拡張し、並列処理機能を追加します。一度に1つのアイテムを処理する代わりに、非同期キューは複数のアイテムを同時に処理しながら、実行の順序と制御を維持できます。これは特にI/O操作、ネットワークリクエスト、またはCPUよりも待機時間の長いタスクを扱う場合に有用です。

### 非同期キューイングの可視化

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ 通常のキューイングとは異なり、複数のアイテムが
               同時に処理可能

             [アイテムがキューに追加]   [一度に2つ処理]   [すべて完了]
              ビジー時                 待機時間あり        全アイテム
```

### 非同期キューイングを使用する場合

非同期キューイングは特に以下の場合に有効です:
- 複数の非同期操作を並列に処理する必要がある場合
- 同時実行数を制御する必要がある場合
- Promiseベースのタスクを適切なエラーハンドリングで扱う場合
- スループットを最大化しながら順序を維持する場合
- 並列実行可能なバックグラウンドタスクを処理する場合

一般的なユースケース:
- レート制限付きでAPIリクエストを並列実行
- 複数のファイルアップロードを同時に処理
- 並列データベース操作の実行
- 複数のWebsocket接続の処理
- バックプレッシャー付きのデータストリーム処理
- リソース集約的なバックグラウンドタスクの管理

### 非同期キューイングを使用しない場合

AsyncQueuerは非常に汎用性が高く、多くの状況で使用できます。実際、そのすべての機能を活用する予定がない場合にのみ適していません。キューに追加されたすべての実行を通過させる必要がない場合は、代わりに[スロットリング][../guides/throttling]を使用してください。並列処理が必要ない場合は、代わりに[キューイング][../guides/queueing]を使用してください。

## TanStack Pacerにおける非同期キューイング

TanStack Pacerは、シンプルな`asyncQueue`関数とより強力な`AsyncQueuer`クラスを通じて非同期キューイングを提供します。

### `asyncQueue`による基本的な使用法

`asyncQueue`関数は、常に実行される非同期キューを作成する簡単な方法を提供します:

```ts
import { asyncQueue } from '@tanstack/pacer'

// 最大2つのアイテムを同時に処理するキューを作成
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('Active tasks:', queuer.getActiveItems().length)
  }
})

// 処理する非同期タスクを追加
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

`asyncQueue`関数の使用は少し制限されています。これは`AsyncQueuer`クラスをラップしたもので、`addItem`メソッドのみを公開しています。キューのより詳細な制御が必要な場合は、`AsyncQueuer`クラスを直接使用してください。

### `AsyncQueuer`クラスによる高度な使用法

`AsyncQueuer`クラスは非同期キューの動作を完全に制御できます:

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // 一度に2つのアイテムを処理
  wait: 1000,     // 新しいアイテムを開始する間に1秒待機
  started: true   // すぐに処理を開始
})

// エラーと成功ハンドラを追加
queue.onError((error) => {
  console.error('Task failed:', error)
})

queue.onSuccess((result) => {
  console.log('Task completed:', result)
})

// 非同期タスクを追加
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### キューの種類と順序付け

AsyncQueuerはさまざまな処理要件に対応するため、異なるキューイング戦略をサポートしています。各戦略はタスクがキューに追加され、処理される方法を決定します。

#### FIFOキュー (先入れ先出し)

FIFOキューはタスクが追加された正確な順序で処理され、シーケンスを維持するのに理想的です:

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // デフォルト
  getItemsFrom: 'front', // デフォルト
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// 処理: firstとsecondを並列に
```

#### LIFOスタック (後入れ先出し)

LIFOスタックは最も最近に追加されたタスクを最初に処理し、新しいタスクを優先するのに有用です:

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // 新しいアイテムを最初に処理
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// 処理: secondを最初に、次にfirst
```

#### 優先度キュー

優先度キューはタスクに割り当てられた優先度値に基づいてタスクを処理し、重要なタスクが最初に処理されるようにします。優先度を指定する方法は2つあります:

1. タスクに付属する静的優先度値:
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// 静的優先度値を持つタスクを作成
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

// 任意の順序でタスクを追加 - 優先度で処理される（数値が大きい方が優先）
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// 処理: highとmediumを並列に、次にlow
```

2. `getPriority`オプションを使用した動的優先度計算:
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // タスクのプロパティや他の要因に基づいて優先度を計算
    // 数値が大きい方が優先
    return calculateTaskPriority(task)
  }
})

// タスクを追加 - 優先度は動的に計算される
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

優先度キューは以下の場合に不可欠です:
- タスクに異なる重要度レベルがある場合
- 重要な操作を最初に実行する必要がある場合
- 優先度に基づいた柔軟なタスク順序付けが必要な場合
- リソース割り当てを重要なタスクに優先させる必要がある場合
- タスクのプロパティや外部要因に基づいて優先度を動的に決定する必要がある場合

### エラーハンドリング

AsyncQueuerは堅牢なタスク処理を確保するための包括的なエラーハンドリング機能を提供します。キューレベルと個々のタスクレベルでエラーを処理できます:

```ts
const queue = new AsyncQueuer<string>()

// グローバルにエラーを処理
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('Task failed:', error)
  },
  onSuccess: (result) => {
    console.log('Task succeeded:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('Task failed:', result)
    } else {
      console.log('Task succeeded:', result)
    }
  }
})

// タスクごとにエラーを処理
queue.addItem(async () => {
  throw new Error('Task failed')
}).catch(error => {
  console.error('Individual task error:', error)
})
```

### キューの管理

AsyncQueuerはキューの状態を監視および制御するためのいくつかのメソッドを提供します:

```ts
// キューの検査
queue.getPeek()           // 削除せずに次のアイテムを表示
queue.getSize()          // 現在のキューサイズを取得
queue.getIsEmpty()       // キューが空かどうかを確認
queue.getIsFull()        // キューがmaxSizeに達したかどうかを確認
queue.getAllItems()   // キュー内のすべてのアイテムのコピーを取得
queue.getActiveItems() // 現在処理中のアイテムを取得
queue.getPendingItems() // 処理待ちのアイテムを取得

// キューの操作
queue.clear()         // すべてのアイテムを削除
queue.reset()         // 初期状態にリセット
queue.getExecutionCount() // 処理されたアイテム数を取得

// 処理制御
queue.start()         // アイテムの処理を開始
queue.stop()          // 処理を一時停止
queue.getIsRunning()     // キューが処理中かどうかを確認
queue.getIsIdle()        // キューが空で処理中でないかどうかを確認
```

### タスクコールバック

AsyncQueuerはタスク実行を監視するための3種類のコールバックを提供します:

```ts
const queue = new AsyncQueuer<string>()

// タスクの成功完了を処理
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Task succeeded:', result)
})

// タスクエラーを処理
const unsubError = queue.onError((error) => {
  console.error('Task failed:', error)
})

// 成功/失敗に関係なくタスク完了を処理
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('Task failed:', result)
  } else {
    console.log('Task succeeded:', result)
  }
})

// 不要になったらコールバックを解除
unsubSuccess()
unsubError()
unsubSettled()
```

### 拒否処理

キューが最大サイズ(`maxSize`オプションで設定)に達すると、新しいタスクは拒否されます。AsyncQueuerはこれらの拒否を処理および監視する方法を提供します:

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // キュー内のタスクを2つまでに制限
  onReject: (task, queuer) => {
    console.log('Queue is full. Task rejected:', task)
  }
})

queue.addItem(async () => 'first') // 受理
queue.addItem(async () => 'second') // 受理
queue.addItem(async () => 'third') // 拒否、onRejectコールバックをトリガー

console.log(queue.getRejectionCount()) // 1
```

### 初期タスク

AsyncQueuerを作成時に初期タスクで事前に設定できます:

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // すぐに処理を開始
})

// キューは3つのタスクで開始され、すぐに処理を開始
```

### 動的設定

AsyncQueuerのオプションは、作成後に`setOptions()`を使用して変更し、`getOptions()`を使用して取得できます:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// 設定を変更
queue.setOptions({
  concurrency: 4, // より多くのタスクを同時に処理
  started: true // 処理を開始
})

// 現在の設定を取得
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### アクティブおよび保留中のタスク

AsyncQueuerはアクティブなタスクと保留中のタスクを監視するメソッドを提供します:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// いくつかのタスクを追加
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// タスクの状態を監視
console.log(queue.getActiveItems().length) // 現在処理中のタスク
console.log(queue.getPendingItems().length) // 処理待ちのタスク
```

### フレームワークアダプター

各フレームワークアダプターは非同期キューアクラスの周りに便利なフックや関数を構築します。`useAsyncQueuer`や`useAsyncQueuedState`のようなフックは、一般的なユースケースで必要な定型コードを削減できる小さなラッパーです。
