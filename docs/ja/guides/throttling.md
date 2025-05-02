---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:14:52.687Z'
title: スロットリングガイド
id: throttling
---
スロットリングガイド

TanStack Pacerは、アプリケーション内での関数実行タイミングを制御するための高品質なユーティリティを提供するライブラリです。他の類似ユーティリティも存在しますが、私たちは***タイプセーフティ***、***ツリーシェイキング***、一貫性のある***直感的なAPI***など、重要な詳細をすべて正しく実装することを目指しています。これらの基本機能に焦点を当て、***フレームワークに依存しない***方法で提供することで、これらのユーティリティとパターンがアプリケーションでより一般的に使用されることを願っています。適切な実行制御はアプリケーション開発で後回しにされがちで、パフォーマンス問題、競合状態、防げたはずのユーザー体験の低下を引き起こします。TanStack Pacerはこれらの重要なパターンを最初から正しく実装するのに役立ちます！

レートリミット、スロットリング、デバウンスは、関数実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロスあり」の動作となります - つまり、関数呼び出しが頻繁に要求されると、一部の呼び出しは実行されません。各アプローチを使用する適切なタイミングを理解することは、パフォーマンスが高く信頼性のあるアプリケーションを構築するために重要です。このガイドでは、TanStack Pacerのスロットリング概念について説明します。

## スロットリングの概念

スロットリングは、関数実行が時間的に均等に間隔をあけて行われることを保証します。制限までバースト実行を許可するレートリミットや、アクティビティが停止するのを待つデバウンスとは異なり、スロットリングは呼び出し間に一貫した遅延を強制することで、よりスムーズな実行パターンを作成します。1秒に1回のスロットリングを設定した場合、呼び出しがどれだけ急速に要求されても、均等に間隔をあけて実行されます。

### スロットリングの可視化

```text
スロットリング (3ティックに1回実行)
タイムライン: [1秒/ティック]
呼び出し:     ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
実行:        ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
            [=================================================================]
            ^ 3ティックごとに1回のみ実行許可、
              呼び出し回数に関係なく

            [最初のバースト]    [追加呼び出し]       [間隔をあけた呼び出し]
            最初に実行後       待機期間後に実行     待機期間が経過するたびに実行
            スロットリング
```

### スロットリングを使用するタイミング

スロットリングは、一貫性があり予測可能な実行タイミングが必要な場合に特に効果的です。これにより、スムーズで制御された動作が求められる頻繁なイベントや更新の処理に最適です。

一般的な使用例:
- 一定のタイミングが必要なUI更新（例: プログレスインジケータ）
- ブラウザに過負荷をかけないようにするスクロールやリサイズイベントハンドラ
- 一定間隔が望ましいリアルタイムデータポーリング
- 安定したペースが必要なリソース集約的な操作
- ゲームループ更新やアニメーションフレーム処理
- ユーザー入力時のライブ検索サジェスト

### スロットリングを使用しないタイミング

スロットリングが最適でない場合:
- アクティビティが停止するのを待ちたい場合（代わりに[デバウンス](../guides/debouncing)を使用）
- いかなる実行も見逃せない場合（代わりに[キューイング](../guides/queueing)を使用）

> [!TIP]
> スロットリングは、スムーズで一貫した実行タイミングが必要な場合に最適な選択肢となることが多いです。レートリミットよりも予測可能な実行パターンを提供し、デバウンスよりも即時のフィードバックを提供します。

## TanStack Pacerでのスロットリング

TanStack Pacerは、`Throttler`と`AsyncThrottler`クラス（および対応する`throttle`と`asyncThrottle`関数）を通じて、同期型と非同期型のスロットリングを提供します。

### `throttle`の基本的な使用法

`throttle`関数は、任意の関数にスロットリングを追加する最も簡単な方法です:

```ts
import { throttle } from '@tanstack/pacer'

// UI更新を200msごとに1回にスロットリング
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 高速ループ内では、200msごとにのみ実行
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 多くの呼び出しがスロットリングされる
}
```

### `Throttler`クラスによる高度な使用法

スロットリング動作をより詳細に制御するには、`Throttler`クラスを直接使用できます:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 実行状態に関する情報を取得
console.log(updateThrottler.getExecutionCount()) // 成功した実行回数
console.log(updateThrottler.getLastExecutionTime()) // 最後の実行タイムスタンプ

// 保留中の実行をキャンセル
updateThrottler.cancel()
```

### 先頭と末尾の実行

同期型スロッターは先頭と末尾の両方のエッジ実行をサポートします:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 最初の呼び出しで実行（デフォルト）
  trailing: true,  // 待機期間後に実行（デフォルト）
})
```

- `leading: true` (デフォルト) - 最初の呼び出しで即時実行
- `leading: false` - 最初の呼び出しをスキップし、末尾の実行を待機
- `trailing: true` (デフォルト) - 待機期間後に最後の呼び出しを実行
- `trailing: false` - 待機期間内の最後の呼び出しをスキップ

一般的なパターン:
- `{ leading: true, trailing: true }` - デフォルト、最も反応性が高い
- `{ leading: false, trailing: true }` - すべての実行を遅延
- `{ leading: true, trailing: false }` - キューされた実行をスキップ

### 有効化/無効化

`Throttler`クラスは`enabled`オプションによる有効化/無効化をサポートしています。`setOptions`メソッドを使用して、いつでもスロッターを有効/無効にできます:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // デフォルトで無効
throttler.setOptions({ enabled: true }) // いつでも有効化
```

スロッターオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、スロッターを動的に有効/無効にできます。ただし、`throttle`関数や`Throttler`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`Throttler`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期型と非同期型の両方のスロッターが、スロットリングライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期型スロッターのコールバック

同期型`Throttler`は次のコールバックをサポートします:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // 各成功実行後に呼び出される
    console.log('関数が実行されました', throttler.getExecutionCount())
  }
})
```

`onExecute`コールバックは、スロットリングされた関数の各成功実行後に呼び出され、実行の追跡、UI状態の更新、またはクリーンアップ操作の実行に役立ちます。

#### 非同期型スロッターのコールバック

非同期型`AsyncThrottler`は、エラー処理のための追加コールバックをサポートします:

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // 各成功実行後に呼び出される
    console.log('非同期関数が実行されました', throttler.getExecutionCount())
  },
  onError: (error) => {
    // 非同期関数がエラーをスローした場合に呼び出される
    console.error('非同期関数が失敗しました:', error)
  }
})
```

`onExecute`コールバックは同期型スロッターと同じように機能し、`onError`コールバックはスロットリングチェーンを中断することなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、実行メトリクスのロギングに特に役立ちます。

### 非同期スロットリング

非同期関数やエラー処理が必要な場合は、`AsyncThrottler`または`asyncThrottle`を使用します:

```ts
import { asyncThrottle } from '@tanstack/pacer'

const throttledFetch = asyncThrottle(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    wait: 1000,
    onError: (error) => {
      console.error('API呼び出しが失敗しました:', error)
    }
  }
)

// 1秒に1回のみAPI呼び出しを実行
await throttledFetch('123')
```

非同期バージョンは、Promiseベースの実行追跡、`onError`コールバックによるエラー処理、保留中の非同期操作の適切なクリーンアップ、および待機可能な`maybeExecute`メソッドを提供します。

### フレームワークアダプター

各フレームワークアダプターは、コアのスロットリング機能を基盤として、フレームワークの状態管理システムと統合するフックを提供します。`createThrottler`、`useThrottledCallback`、`useThrottledState`、`useThrottledValue`などのフックが各フレームワークで利用可能です。

いくつかの例を示します:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 詳細な制御のための低レベルフック
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 基本的な使用例のためのシンプルなコールバックフック
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// リアクティブな状態管理のためのステートベースフック
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // スロットリングする値
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// 詳細な制御のための低レベルフック
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 状態管理のためのシグナルベースフック
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('総実行回数:', throttler.getExecutionCount())
  }
})
```

各フレームワークアダプターは、コアのスロットリング機能を維持しながら、フレームワークの状態管理システムと統合するフックを提供します。
