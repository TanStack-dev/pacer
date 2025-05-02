---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:15:08.207Z'
title: 非同期キューイングガイド
id: async-queueing
---
[Queuer](../guides//queueing)がタイミング制御付きの同期キューイングを提供するのに対し、`AsyncQueuer`は並列非同期操作を処理するために特別に設計されています。これは従来「タスクプール」または「ワーカープール」パターンとして知られるものを実装しており、複数の操作を同時に処理しながら並列性とタイミングを制御できます。この実装は主に[Swimmer](https://github.com/tannerlinsley/swimmer)からコピーされたもので、2017年からJavaScriptコミュニティで使われているTannerのオリジナルのタスクプーリングユーティリティです。

## 非同期キューイングの概念

非同期キューイングは基本的なキューイングの概念を拡張し、並列処理機能を追加します。一度に1つのアイテムを処理する代わりに、非同期キューは複数のアイテムを同時に処理しながら、実行の順序と制御を維持できます。これは特にI/O操作、ネットワークリクエスト、またはCPUを使用するよりも待機時間の長いタスクを扱う場合に有用です。

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

             [アイテムがキューに登録]   [一度に2つ処理]   [全アイテム完了]
              ビジー時                待機時間を挟む
```

### 非同期キューイングを使用する場合

非同期キューイングは以下の場合に特に有効です:
- 複数の非同期操作を並列に処理する必要がある場合
- 同時実行数を制御する必要がある場合
- Promiseベースのタスクを適切なエラーハンドリングで処理する場合
- スループットを最大化しながら順序を維持する場合
- 並列実行可能なバックグラウンドタスクを処理する場合

一般的なユースケース:
- レート制限付きでAPIリクエストを並列実行
- 複数のファイルアップロードを同時に処理
- 並列データベース操作の実行
- 複数のWebSocket接続の処理
- バックプレッシャー付きのデータストリーム処理
- リソース集約型バックグラウンドタスクの管理

### 非同期キューイングを使用しない場合

`AsyncQueuer`は非常に汎用性が高く、多くの状況で使用できます。実際、そのすべての機能を活用する予定がない場合にのみ適していません。キューに登録されたすべての実行を通過させる必要がない場合は、代わりに[スロットリング][../guides/throttling]を使用してください。並列処理が必要ない場合は、代わりに[キューイング][../guides/queueing]を使用してください。

## TanStack Pacerでの非同期キューイング

TanStack Pacerは、シンプルな`asyncQueue`関数とより強力な`AsyncQueuer`クラスを通じて非同期キューイングを提供します。

### `asyncQueue`を使った基本的な使用法

`asyncQueue`関数は、常に実行される非同期キューを作成する簡単な方法を提供します:

```ts
import { asyncQueue } from '@tanstack/pacer'

// 最大2つのアイテムを並列処理するキューを作成
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

`asyncQueue`関数の使用は少し制限されています。これは`AsyncQueuer`クラスのラッパーであり、`addItem`メソッドのみを公開しているためです。キューをより詳細に制御するには、`AsyncQueuer`クラスを直接使用してください。

### `AsyncQueuer`クラスを使った高度な使用法

`AsyncQueuer`クラスは非同期キューの動作を完全に制御できます:

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // 一度に2つのアイテムを処理
  wait: 1000,     // 新しいアイテム開始前に1秒待機
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

`AsyncQueuer`はさまざまな処理要件に対応するため、異なるキューイング戦略をサポートしています。各戦略はタスクがキューに追加され、処理される方法を決定します。

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

LIFOスタックは最も最近追加されたタスクを最初に処理し、新しいタスクを優先するのに有用です:

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

1. タスクに静的な優先度値を割り当て:
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// 静的な優先度値を持つタスクを作成
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

// 任意の順序でタスクを追加 - 優先度順に処理（数値が大きいほど優先）
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// 処理: highとmediumを並列に、次にlow
```

2. `getPriority`オプションを使った動的優先度計算:
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // タスクのプロパティや他の要因に基づいて優先度を計算
    // 数値が大きいほど優先
    return calculateTaskPriority(task)
  }
})

// タスクを追加 - 優先度は動的に計算
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
- リソース割り当てを重要なタスクに優先させたい場合
- タスクのプロパティや外部要因に基づいて動的に優先度を決定する必要がある場合

### エラーハンドリング

`AsyncQueuer`は堅牢なタスク処理を確保するための包括的なエラーハンドリング機能を提供します。キューレベルと個々のタスクレベルでエラーを処理できます:

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

### キュー管理

`AsyncQueuer`はキュー状態を監視・制御するためのいくつかのメソッドを提供します:

```ts
// キュー検査
queue.getPeek()           // 削除せずに次のアイテムを表示
queue.getSize()          // 現在のキューサイズを取得
queue.getIsEmpty()       // キューが空かどうかを確認
queue.getIsFull()        // キューがmaxSizeに達したかどうかを確認
queue.getAllItems()   // キュー内のすべてのアイテムのコピーを取得
queue.getActiveItems() // 現在処理中のアイテムを取得
queue.getPendingItems() // 処理待ちのアイテムを取得

// キュー操作
queue.clear()         // すべてのアイテムを削除
queue.reset()         // 初期状態にリセット
queue.getExecutionCount() // 処理済みアイテム数を取得

// 処理制御
queue.start()         // アイテム処理を開始
queue.stop()          // 処理を一時停止
queue.getIsRunning()     // キューが処理中かどうかを確認
queue.getIsIdle()        // キューが空で処理中でないかどうかを確認
```

### タスクコールバック

`AsyncQueuer`はタスク実行を監視するための3種類のコールバックを提供します:

```ts
const queue = new AsyncQueuer<string>()

// タスク成功時の処理
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Task succeeded:', result)
})

// タスクエラー時の処理
const unsubError = queue.onError((error) => {
  console.error('Task failed:', error)
})

// 成功/失敗に関係なくタスク完了時の処理
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

キューが最大サイズ（`maxSize`オプションで設定）に達すると、新しいタスクは拒否されます。`AsyncQueuer`はこれらの拒否を処理・監視する方法を提供します:

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // キュー内のタスクを2つまでに制限
  onReject: (task, queuer) => {
    console.log('Queue is full. Task rejected:', task)
  }
})

queue.addItem(async () => 'first') // 受理
queue.addItem(async () => 'second') // 受理
queue.addItem(async () => 'third') // 拒否、onRejectコールバックがトリガー

console.log(queue.getRejectionCount()) // 1
```

### 初期タスク

`AsyncQueuer`を作成時に初期タスクで事前に設定できます:

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

`AsyncQueuer`のオプションは、作成後に`setOptions()`で変更し、`getOptions()`で取得できます:

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

`AsyncQueuer`はアクティブなタスクと保留中のタスクを監視するメソッドを提供します:

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

// タスク状態を監視
console.log(queue.getActiveItems().length) // 現在処理中のタスク
console.log(queue.getPendingItems().length) // 処理待ちのタスク
```

### フレームワークアダプター

各フレームワークアダプターは非同期キューアラウンドクラスの周りに便利なフックや関数を構築します。`useAsyncQueuer`や`useAsyncQueuerState`のようなフックは、一般的なユースケースで必要なボイラープレートコードを削減できる小さなラッパーです。
