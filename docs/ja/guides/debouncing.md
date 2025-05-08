---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:50:36.107Z'
title: デバウンスガイド
id: debouncing
---
# デバウンスガイド

TanStack Pacerは、アプリケーション内での関数実行タイミングを制御するための高品質なユーティリティを提供するライブラリです。類似のユーティリティは他にも存在しますが、私たちは***型安全性***、***ツリーシェイキング***、一貫性のある***直感的なAPI***など、重要な詳細をすべて正しく実装することを目指しています。これらの基本機能に焦点を当て、***フレームワークに依存しない***方法で提供することで、これらのユーティリティとパターンがアプリケーションでより一般的に使用されることを願っています。適切な実行制御はアプリケーション開発において後回しにされがちで、パフォーマンス問題、競合状態、回避可能だったユーザーエクスペリエンスの低下を引き起こします。TanStack Pacerは、これらの重要なパターンを最初から正しく実装するのに役立ちます！

レートリミット、スロットリング、デバウンスは、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロッシー（損失あり）」になります - つまり、関数呼び出しが頻繁に行われすぎた場合、一部の呼び出しは実行されません。各アプローチをいつ使用するかを理解することは、パフォーマンスが高く信頼性のあるアプリケーションを構築するために重要です。このガイドでは、TanStack Pacerのデバウンス概念について説明します。

## デバウンスの概念

デバウンスは、指定された期間の非アクティブ状態が発生するまで関数の実行を遅延させる技術です。制限まで実行のバーストを許可するレートリミットや、均等に間隔を空けた実行を保証するスロットリングとは異なり、デバウンスは複数の高速な関数呼び出しを、呼び出しが停止した後にのみ発生する単一の実行にまとめます。これにより、デバウンスはアクティビティが落ち着いた後の最終状態のみを気にするイベントのバースト処理に理想的です。

### デバウンスの可視化

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ 呼び出しが3ティック
                                                         ない後にここで実行

             [呼び出しのバースト]     [さらに呼び出し]   [待機]      [新しいバースト]
             実行なし               タイマーリセット     [遅延実行]  [待機] [遅延実行]
```

### デバウンスを使用すべき場合

デバウンスは、アクションを実行する前にアクティビティの「一時停止」を待ちたい場合に特に効果的です。これは、最終状態のみを気にするユーザー入力やその他の高速発火イベントの処理に理想的です。

一般的な使用例：
- ユーザーが入力を終えるのを待ちたい検索入力フィールド
- キーストロークごとに実行すべきではないフォームバリデーション
- 計算コストが高いウィンドウリサイズ計算
- コンテンツ編集中の自動ドラフト保存
- ユーザーアクティビティが落ち着いた後にのみ発生すべきAPI呼び出し
- 急速な変更後の最終値のみを気にするシナリオ

### デバウンスを使用すべきでない場合

デバウンスが最適な選択でない場合：
- 特定の期間にわたって確実に実行する必要がある場合（代わりに[スロットリング](../guides/throttling)を使用）
- いかなる実行も見逃すことが許されない場合（代わりに[キューイング](../guides/queueing)を使用）

## TanStack Pacerでのデバウンス

TanStack Pacerは、`Debouncer`と`AsyncDebouncer`クラス（および対応する`debounce`と`asyncDebounce`関数）を介して、同期および非同期のデバウンスを提供します。

### `debounce`を使った基本的な使用法

`debounce`関数は、任意の関数にデバウンスを追加する最も簡単な方法です：

```ts
import { debounce } from '@tanstack/pacer'

// ユーザーが入力をやめるのを待つ検索入力のデバウンス
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // 最後のキーストロークから500ms待機
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### `Debouncer`クラスを使った高度な使用法

デバウンス動作をより詳細に制御するには、`Debouncer`クラスを直接使用できます：

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// 現在の状態に関する情報を取得
console.log(searchDebouncer.getExecutionCount()) // 成功した実行回数
console.log(searchDebouncer.getIsPending()) // 呼び出しが保留中かどうか

// オプションを動的に更新
searchDebouncer.setOptions({ wait: 1000 }) // 待機時間を増やす

// 保留中の実行をキャンセル
searchDebouncer.cancel()
```

### 先頭と末尾の実行

同期デバウンサーは、先頭エッジと末尾エッジの両方の実行をサポートします：

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 最初の呼び出しで実行
  trailing: true,  // 待機期間後に実行
})
```

- `leading: true` - 最初の呼び出しで即時実行
- `leading: false` (デフォルト) - 最初の呼び出しで待機タイマー開始
- `trailing: true` (デフォルト) - 待機期間後に実行
- `trailing: false` - 待機期間後に実行なし

一般的なパターン：
- `{ leading: false, trailing: true }` - デフォルト、待機後に実行
- `{ leading: true, trailing: false }` - 即時実行、後続の呼び出しを無視
- `{ leading: true, trailing: true }` - 最初の呼び出し時と待機後の両方で実行

### 最大待機時間

TanStack Pacerのデバウンサーは、他のデバウンスライブラリのような`maxWait`オプションを意図的に持っていません。実行をより分散させたい場合は、代わりに[スロットリング](../guides/throttling)技術の使用を検討してください。

### 有効化/無効化

`Debouncer`クラスは、`enabled`オプションを介して有効化/無効化をサポートします。`setOptions`メソッドを使用すると、いつでもデバウンサーを有効化/無効化できます：

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
debouncer.setOptions({ enabled: true }) // いつでも有効化
```

デバウンサーオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、デバウンサーを動的に有効化/無効化できます：

```ts
// Reactの例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 入力長に基づいて有効化/無効化（リアクティブオプションをサポートするフレームワークアダプターを使用している場合）
)
```

ただし、`debounce`関数または`Debouncer`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`Debouncer`クラスのコンストラクタに渡されるためです。

```ts
// Solidの例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 入力長に基づいて有効化/無効化
})
```

### コールバックオプション

同期および非同期のデバウンサーは、デバウンスライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています：

#### 同期デバウンサーのコールバック

同期`Debouncer`は次のコールバックをサポートします：

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 各成功実行後に呼び出される
    console.log('関数が実行されました', debouncer.getExecutionCount())
  }
})
```

`onExecute`コールバックは、デバウンスされた関数の各成功実行後に呼び出され、実行の追跡、UI状態の更新、またはクリーンアップ操作の実行に役立ちます。

#### 非同期デバウンサーのコールバック

非同期`AsyncDebouncer`は、同期バージョンとは異なる一連のコールバックを持っています。

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // 各成功実行後に呼び出される
    console.log('非同期関数が実行されました', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // 各実行試行後に呼び出される
    console.log('非同期関数が完了しました', debouncer.getSettledCount())
  },
  onError: (error) => {
    // 非同期関数がエラーをスローした場合に呼び出される
    console.error('非同期関数が失敗しました:', error)
  }
})
```

`onSuccess`コールバックはデバウンスされた関数の各成功実行後に呼び出され、`onError`コールバックは非同期関数がエラーをスローした場合に呼び出されます。`onSettled`コールバックは、成功または失敗に関係なく、各実行試行後に呼び出されます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、および実行メトリクスのロギングに特に役立ちます。

### 非同期デバウンス

非同期デバウンサーは、非同期操作をデバウンスで処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期デバウンサーがUIイベントと即時のフィードバックに適しているのに対し、非同期バージョンはAPI呼び出し、データベース操作、およびその他の非同期タスクの処理に特化して設計されています。

#### 同期デバウンスとの主な違い

1. **戻り値の処理**
同期デバウンサーがvoidを返すのとは異なり、非同期バージョンではデバウンスされた関数からの戻り値をキャプチャして使用できます。これはAPI呼び出しやその他の非同期操作の結果を操作する必要がある場合に特に便利です。`maybeExecute`メソッドは、関数の戻り値で解決されるPromiseを返し、結果を待機して適切に処理できるようにします。

2. **異なるコールバック**
`AsyncDebouncer`は、同期バージョンの`onExecute`だけでなく、次のコールバックをサポートします：
- `onSuccess`: 各成功実行後に呼び出され、デバウンサーインスタンスを提供
- `onSettled`: 各実行後に呼び出され、デバウンサーインスタンスを提供
- `onError`: 非同期関数がエラーをスローした場合に呼び出され、エラーとデバウンサーインスタンスの両方を提供

3. **順次実行**
デバウンサーの`maybeExecute`メソッドはPromiseを返すため、次の実行を開始する前に各実行を待機することを選択できます。これにより、実行順序を制御し、各呼び出しが最新のデータを処理することを保証できます。これは、以前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に役立ちます。

たとえば、ユーザーのプロファイルを更新してからすぐに更新されたデータを取得する場合、取得操作を開始する前に更新操作を待機できます：

#### 基本的な使用例

検索操作に非同期デバウンサーを使用する基本的な例を次に示します：

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('検索が成功しました:', results)
    },
    onError: (error, debouncer) => {
      console.error('検索が失敗しました:', error)
    }
  }
)

// 使用法
const results = await debouncedSearch('query')
```

### フレームワークアダプター

各フレームワークアダプターは、コアのデバウンス機能を基に構築され、フレームワークの状態管理システムと統合するためのフックを提供します。`createDebouncer`、`useDebouncedCallback`、`useDebouncedState`、`useDebouncedValue`などのフックが各フレームワークで利用可能です。

いくつかの例：

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// 完全な制御のための低レベルフック
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 基本的な使用例のためのシンプルなコールバックフック
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// リアクティブな状態管理のための状態ベースのフック
const [instantState, setInstantState] = useState('')
const [debouncedValue] = useDebouncedValue(
  instantState, // デバウンスする値
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// 完全な制御のための低レベルフック
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// 状態管理のためのシグナルベースのフック
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('総実行回数:', debouncer.getExecutionCount())
  }
})
```
