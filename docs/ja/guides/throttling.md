---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:50:16.098Z'
title: スロットリングガイド
id: throttling
---
# スロットリングガイド

レートリミット (Rate Limiting)、スロットリング (Throttling)、デバウンス (Debouncing) は、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロスあり (lossy)」の動作となります - つまり、関数呼び出しが頻繁に行われすぎると、一部の呼び出しが実行されません。パフォーマンスが高く信頼性のあるアプリケーションを構築するには、各アプローチを使用するタイミングを理解することが重要です。このガイドでは、TanStack Pacerのスロットリング概念について説明します。

## スロットリングの概念

スロットリングは、関数の実行が時間的に均等に間隔を空けて行われることを保証します。制限までバースト実行を許可するレートリミットや、アクティビティが停止するのを待つデバウンスとは異なり、スロットリングは呼び出し間に一貫した遅延を強制することで、よりスムーズな実行パターンを作成します。1秒に1回のスロットリングを設定した場合、呼び出しがどれだけ急速に要求されても、均等に間隔が空けられます。

### スロットリングの可視化

```text
スロットリング (3ティックに1回実行)
タイムライン: [1ティック=1秒]
呼び出し:      ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
実行:         ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 3ティックごとに1回のみ実行許可、
               呼び出し回数に関係なく

             [最初のバースト]    [追加呼び出し]        [間隔を空けた呼び出し]
             最初に実行後       待機期間後に実行      待機期間が経過するたびに実行
             スロットリング
```

### スロットリングを使用するタイミング

スロットリングは、一貫性があり予測可能な実行タイミングが必要な場合に特に効果的です。これにより、スムーズで制御された動作が必要な頻繁なイベントや更新の処理に最適です。

一般的な使用例:
- 一貫したタイミングが必要なUI更新（例: プログレスインジケーター）
- ブラウザに過負荷をかけないようにするスクロールやリサイズイベントハンドラ
- 一定間隔が望ましいリアルタイムデータポーリング
- 安定したペースが必要なリソース集約型操作
- ゲームループの更新やアニメーションフレーム処理
- ユーザーが入力する際のライブ検索候補表示

### スロットリングを使用しない方が良い場合

以下の場合はスロットリングが最適な選択ではないかもしれません:
- アクティビティが停止するのを待ちたい場合（代わりに[デバウンス](../guides/debouncing)を使用）
- いかなる実行も見逃したくない場合（代わりに[キューイング](../guides/queueing)を使用）

> [!TIP]
> スロットリングは、スムーズで一貫した実行タイミングが必要な場合に最適な選択となることが多いです。レートリミットよりも予測可能な実行パターンを提供し、デバウンスよりも即時のフィードバックを提供します。

## TanStack Pacerでのスロットリング

TanStack Pacerは、`Throttler`クラスと`AsyncThrottler`クラス（および対応する`throttle`と`asyncThrottle`関数）を通じて、同期型と非同期型のスロットリングを提供します。

### `throttle`を使った基本的な使用法

`throttle`関数は、任意の関数にスロットリングを追加する最も簡単な方法です:

```ts
import { throttle } from '@tanstack/pacer'

// UI更新を200msに1回にスロットリング
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// 高速ループでも200msごとにのみ実行
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // 多くの呼び出しがスロットリングされる
}
```

### `Throttler`クラスを使った高度な使用法

スロットリング動作をより細かく制御するには、`Throttler`クラスを直接使用できます:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 実行状態に関する情報を取得
console.log(updateThrottler.getExecutionCount()) // 成功した実行回数
console.log(updateThrottler.getLastExecutionTime()) // 最後の実行のタイムスタンプ

// 保留中の実行をキャンセル
updateThrottler.cancel()
```

### 先頭と末尾の実行

同期スロッターは先頭（leading）と末尾（trailing）の両方の実行をサポートします:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 最初の呼び出しで即時実行（デフォルト）
  trailing: true,  // 待機期間後に実行（デフォルト）
})
```

- `leading: true` (デフォルト) - 最初の呼び出しで即時実行
- `leading: false` - 最初の呼び出しをスキップし、末尾の実行を待つ
- `trailing: true` (デフォルト) - 待機期間後に最後の呼び出しを実行
- `trailing: false` - 待機期間内の場合、最後の呼び出しをスキップ

一般的なパターン:
- `{ leading: true, trailing: true }` - デフォルト、最も反応が良い
- `{ leading: false, trailing: true }` - すべての実行を遅延
- `{ leading: true, trailing: false }` - キューに入った実行をスキップ

### 有効化/無効化

`Throttler`クラスは`enabled`オプションによる有効化/無効化をサポートしています。`setOptions`メソッドを使用して、いつでもスロッターを有効/無効にできます:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // デフォルトで無効
throttler.setOptions({ enabled: true }) // いつでも有効化
```

スロッターオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、スロッターを動的に有効/無効にできます。ただし、`throttle`関数や`Throttler`クラスを直接使用している場合は、`setOptions`メソッドを使用して`enabled`オプションを変更する必要があります。渡されるオプションは実際には`Throttler`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期型と非同期型の両方のスロッターが、スロットリングライフサイクルのさまざまな側面を処理するコールバックオプションをサポートしています:

#### 同期スロッターのコールバック

同期`Throttler`は以下のコールバックをサポートします:

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

#### 非同期スロッターのコールバック

非同期`AsyncThrottler`は、エラー処理のための追加コールバックをサポートします:

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

`onExecute`コールバックは同期スロッターと同じように機能し、`onError`コールバックでは、スロットリングチェーンを壊すことなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、実行メトリックのロギングに特に役立ちます。

### 非同期スロットリング

非同期スロッターは、スロットリングを使用して非同期操作を処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期スロッターがUIイベントと即時のフィードバックに適しているのに対し、非同期バージョンはAPI呼び出し、データベース操作、およびその他の非同期タスクの処理に特化して設計されています。

#### 同期スロットリングとの主な違い

1. **戻り値の処理**
同期スロッターがvoidを返すのとは異なり、非同期バージョンではスロットリングされた関数からの戻り値をキャプチャして使用できます。これは、API呼び出しやその他の非同期操作の結果を操作する必要がある場合に特に便利です。`maybeExecute`メソッドは、関数の戻り値で解決されるPromiseを返し、結果を待機して適切に処理できるようにします。

2. **異なるコールバック**
`AsyncThrottler`は、同期バージョンの`onExecute`だけでなく、以下のコールバックをサポートします:
- `onSuccess`: 各成功実行後に呼び出され、スロッターインスタンスを提供
- `onSettled`: 各実行後に呼び出され、スロッターインスタンスを提供
- `onError`: 非同期関数がエラーをスローした場合に呼び出され、エラーとスロッターインスタンスの両方を提供

非同期と同期の両方のスロッターが、成功した実行を処理するための`onExecute`コールバックをサポートしています。

3. **順次実行**
スロッターの`maybeExecute`メソッドはPromiseを返すため、次の実行を開始する前に各実行を待機することを選択できます。これにより、実行順序を制御し、各呼び出しが最新のデータを処理することを保証できます。これは、前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に役立ちます。

たとえば、ユーザーのプロファイルを更新してからすぐに更新されたデータを取得する場合、フェッチ操作を開始する前に更新操作を待機できます:

#### 基本的な使用例

検索操作に非同期スロッターを使用する基本的な例を以下に示します:

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('検索が成功しました:', results)
    },
    onError: (error, throttler) => {
      console.error('検索が失敗しました:', error)
    }
  }
)

// 使用法
const results = await throttledSearch('query')
```

### フレームワークアダプター

各フレームワークアダプターは、コアのスロットリング機能を基盤として構築され、フレームワークの状態管理システムと統合するフックを提供します。`createThrottler`、`useThrottledCallback`、`useThrottledState`、`useThrottledValue`などのフックが各フレームワークで利用可能です。

以下にいくつかの例を示します:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// 完全な制御のための低レベルフック
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 基本的な使用例のためのシンプルなコールバックフック
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// リアクティブな状態管理のための状態ベースのフック
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // スロットリングする値
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// 完全な制御のための低レベルフック
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// 状態管理のためのシグナルベースのフック
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('総実行回数:', throttler.getExecutionCount())
  }
})
```

各フレームワークアダプターは、コアのスロットリング機能を維持しながら、フレームワークの状態管理システムと統合するフックを提供します。
