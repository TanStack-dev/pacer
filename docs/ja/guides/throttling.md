---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:09:14.283Z'
title: スロットリングガイド
id: throttling
---
# スロットリングガイド

レートリミット、スロットリング、デバウンスは、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロスあり（lossy）」の動作となります。つまり、関数呼び出しが頻繁すぎる場合、一部の呼び出しは実行されません。パフォーマンスが高く信頼性のあるアプリケーションを構築するには、各アプローチを使い分けることが重要です。このガイドでは、TanStack Pacerのスロットリング概念について説明します。

## スロットリングの概念

スロットリングは、関数の実行を時間的に均等に分散させます。レートリミットが一定数までのバースト実行を許可したり、デバウンスがアクティビティが停止するのを待つのとは異なり、スロットリングは呼び出し間に一定の遅延を強制することで、よりスムーズな実行パターンを作成します。1秒に1回のスロットリングを設定した場合、呼び出しがどれだけ急速に要求されても、均等に間隔が空けられます。

### スロットリングの可視化

```text
スロットリング（3ティックごとに1回実行）
タイムライン: [1ティックあたり1秒]
呼び出し:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
実行:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ 3ティックごとに1回のみ実行許可、
               呼び出し回数に関係なく

             [最初のバースト]    [追加呼び出し]              [間隔を空けた呼び出し]
             最初に実行し       待機期間後に実行            待機期間が経過するごとに実行
             その後スロットリング
```

### スロットリングを使用する場面

スロットリングは、一貫性があり予測可能な実行タイミングが必要な場合に特に効果的です。これにより、スムーズで制御された動作が求められる頻繁なイベントや更新の処理に最適です。

一般的な使用例:
- 一定のタイミングが必要なUI更新（例: プログレスインジケーター）
- ブラウザに過負荷をかけないスクロールやリサイズイベントハンドラー
- 一定間隔が望ましいリアルタイムデータポーリング
- 安定したペースが必要なリソース集約型操作
- ゲームループの更新やアニメーションフレーム処理
- ユーザー入力時のライブ検索候補表示

### スロットリングを使用しない場面

以下の場合、スロットリングは最適な選択ではないかもしれません:
- アクティビティが停止するのを待ちたい場合（代わりに[デバウンス](../guides/debouncing)を使用）
- どの実行も見逃せない場合（代わりに[キューイング](../guides/queueing)を使用）

> [!TIP]
> スロットリングは、スムーズで一貫した実行タイミングが必要な場合に最適な選択肢となることが多いです。レートリミットよりも予測可能な実行パターンを提供し、デバウンスよりも即時のフィードバックを提供します。

## TanStack Pacerでのスロットリング

TanStack Pacerは、`Throttler`クラスと`AsyncThrottler`クラス（および対応する`throttle`と`asyncThrottle`関数）を通じて、同期型と非同期型の両方のスロットリングを提供します。

### `throttle`を使った基本的な使用法

`throttle`関数は、任意の関数にスロットリングを追加する最も簡単な方法です:

```ts
import { throttle } from '@tanstack/pacer'

// UI更新を200msごとにスロットリング
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

### `Throttler`クラスを使った高度な使用法

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

同期型スロッターは、先頭（leading）と末尾（trailing）の両方のエッジ実行をサポートします:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // 最初の呼び出しで実行（デフォルト）
  trailing: true,  // 待機期間後に実行（デフォルト）
})
```

- `leading: true` (デフォルト) - 最初の呼び出しで即時実行
- `leading: false` - 最初の呼び出しをスキップし、末尾の実行を待つ
- `trailing: true` (デフォルト) - 待機期間後に最後の呼び出しを実行
- `trailing: false` - 待機期間内であれば最後の呼び出しをスキップ

一般的なパターン:
- `{ leading: true, trailing: true }` - デフォルト、最も反応が速い
- `{ leading: false, trailing: true }` - すべての実行を遅延
- `{ leading: true, trailing: false }` - キューに入った実行をスキップ

### 有効化/無効化

`Throttler`クラスは、`enabled`オプションを通じて有効化/無効化をサポートします。`setOptions`メソッドを使用すると、いつでもスロッターを有効化/無効化できます:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // デフォルトで無効
throttler.setOptions({ enabled: true }) // いつでも有効化
```

フレームワークアダプターを使用していてスロッターオプションがリアクティブな場合、`enabled`オプションを条件付きの値に設定することで、スロッターを動的に有効化/無効化できます。ただし、`throttle`関数や`Throttler`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`Throttler`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期型と非同期型の両方のスロッターが、スロットリングライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期型スロッターのコールバック

同期型`Throttler`は以下のコールバックをサポートします:

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

`onExecute`コールバックは同期型スロッターと同じように機能し、`onError`コールバックではスロットリングチェーンを壊すことなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、および実行メトリクスのロギングに特に役立ちます。

### 非同期スロットリング

非同期スロッターは、スロットリングを使用して非同期操作を処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期スロッターがUIイベントや即時のフィードバックに適しているのに対し、非同期バージョンはAPI呼び出し、データベース操作、およびその他の非同期タスクの処理に特化して設計されています。

#### 同期型スロットリングとの主な違い

1. **戻り値の処理**
同期スロッターがvoidを返すのとは異なり、非同期バージョンではスロットリングされた関数からの戻り値をキャプチャして使用できます。これは、API呼び出しや他の非同期操作の結果を操作する必要がある場合に特に便利です。`maybeExecute`メソッドは、関数の戻り値で解決されるPromiseを返し、結果を待機して適切に処理できるようにします。

2. **強化されたコールバックシステム**
非同期スロッターは、同期バージョンの単一の`onExecute`コールバックに比べて、より洗練されたコールバックシステムを提供します。このシステムには以下が含まれます:
- `onSuccess`: 非同期関数が正常に完了したときに呼び出され、結果とスロッターインスタンスの両方を提供
- `onError`: 非同期関数がエラーをスローしたときに呼び出され、エラーとスロッターインスタンスの両方を提供
- `onSettled`: 成功または失敗にかかわらず、すべての実行試行後に呼び出される

3. **実行追跡**
非同期スロッターは、いくつかの方法を通じて包括的な実行追跡を提供します:
- `getSuccessCount()`: 成功した実行回数
- `getErrorCount()`: 失敗した実行回数
- `getSettledCount()`: 完了した実行の総数（成功 + 失敗）

4. **順次実行**
非同期スロッターは、後続の実行が前の呼び出しの完了を待ってから開始することを保証します。これにより、順不同の実行が防止され、各呼び出しが最新のデータを処理することが保証されます。これは、前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に重要です。

たとえば、ユーザーのプロファイルを更新してからすぐに更新されたデータを取得する場合、非同期スロッターは取得操作が更新の完了を待つようにし、古いデータを取得する可能性のある競合状態を防ぎます。

#### 基本的な使用例

以下は、検索操作に非同期スロッターを使用する基本的な例です:

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

#### 高度なパターン

非同期スロッターは、複雑な問題を解決するためにさまざまなパターンと組み合わせることができます:

1. **状態管理との統合**
非同期スロッターを状態管理システム（ReactのuseStateやSolidのcreateSignalなど）と組み合わせると、ローディング状態、エラー状態、およびデータ更新を処理する強力なパターンを作成できます。スロッターのコールバックは、操作の成功または失敗に基づいてUI状態を更新するための完璧なフックを提供します。

2. **競合状態の防止**
スロットリングパターンは、多くのシナリオで自然に競合状態を防止します。アプリケーションの複数の部分が同時に同じリソースを更新しようとする場合、スロッターは更新が制御された速度で行われることを保証し、すべての呼び出し元に結果を提供します。

3. **エラー回復**
非同期スロッターのエラー処理機能は、リトライロジックやエラー回復パターンを実装するのに理想的です。`onError`コールバックを使用して、指数バックオフやフォールバックメカニズムなどのカスタムエラー処理戦略を実装できます。

### フレームワークアダプター

各フレームワークアダプターは、コアのスロットリング機能を基盤として構築され、フレームワークの状態管理システムと統合するためのフックを提供します。`createThrottler`、`useThrottledCallback`、`useThrottledState`、`useThrottledValue`などのフックが各フレームワークで利用可能です。

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

// リアクティブな状態管理のための状態ベースフック
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

// 状態管理のためのシグナルベースフック
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('総実行回数:', throttler.getExecutionCount())
  }
})
```

各フレームワークアダプターは、コアのスロットリング機能を維持しながら、フレームワークの状態管理システムと統合するフックを提供します。
