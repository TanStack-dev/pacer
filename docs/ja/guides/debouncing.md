---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:09:25.958Z'
title: デバウンスガイド
id: debouncing
---
以下は翻訳されたドキュメントです。

# デバウンスガイド

TanStack Pacerは、アプリケーション内での関数実行タイミングを制御するための高品質なユーティリティを提供するライブラリです。類似のユーティリティは他にも存在しますが、私たちは***タイプセーフ***、***ツリーシェイキング***、一貫性のある***直感的なAPI***など、重要な詳細をすべて正しく実装することを目指しています。これらの基本機能に焦点を当て、***フレームワークに依存しない***形で提供することで、これらのユーティリティとパターンがアプリケーションでより一般的に使用されることを願っています。適切な実行制御はアプリケーション開発において後回しにされがちで、パフォーマンス問題、競合状態、回避可能だったユーザー体験の低下を引き起こします。TanStack Pacerを使えば、これらの重要なパターンを最初から正しく実装できます！

レートリミット、スロットリング、デバウンスは、関数実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロッシー」（損失あり）な特性を持ちます - つまり、関数呼び出しが頻繁に行われた場合、一部の呼び出しは実行されません。各アプローチを使用する適切なタイミングを理解することは、パフォーマンスが高く信頼性のあるアプリケーションを構築する上で重要です。このガイドでは、TanStack Pacerのデバウンス概念について説明します。

## デバウンスの概念

デバウンスは、指定された期間アクティビティが発生しなくなるまで関数の実行を遅延させる技術です。一定限度まで実行のバーストを許可するレートリミットや、均等な間隔で実行を保証するスロットリングとは異なり、デバウンスは複数の高速な関数呼び出しを、呼び出しが停止した後にのみ発生する単一の実行にまとめます。これにより、デバウンスはアクティビティが落ち着いた後の最終状態のみを気にするイベントのバースト処理に理想的です。

### デバウンスの可視化

```text
デバウンス（待機時間: 3ティック）
タイムライン: [1秒/ティック]
呼び出し:     ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
実行:        ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ ここで実行
                                                         呼び出しが3ティックない後

             [呼び出しのバースト]     [さらに呼び出し]   [待機]      [新しいバースト]
             実行なし               タイマーリセット     [遅延実行]  [待機] [遅延実行]
```

### デバウンスを使用するタイミング

デバウンスは、アクションを実行する前に「一時停止」を待ちたい場合に特に効果的です。これは、最終状態のみを気にするユーザー入力やその他の高速発生イベントの処理に理想的です。

一般的な使用例:
- ユーザーが入力を終えるのを待ちたい検索入力フィールド
- キーストロークごとに実行すべきでないフォームバリデーション
- 計算コストが高いウィンドウリサイズ計算
- コンテンツ編集中の自動ドラフト保存
- ユーザーアクティビティが落ち着いた後にのみ発生すべきAPI呼び出し
- 急速な変更後の最終値のみを気にするシナリオ

### デバウンスを使用すべきでない場合

デバウンスが最適でない場合:
- 特定期間での実行保証が必要な場合（代わりに[スロットリング](../guides/throttling)を使用）
- いかなる実行も見逃せない場合（代わりに[キューイング](../guides/queueing)を使用）

## TanStack Pacerでのデバウンス

TanStack Pacerは、`Debouncer`クラスと`AsyncDebouncer`クラス（および対応する`debounce`と`asyncDebounce`関数）を介して、同期および非同期のデバウンスを提供します。

### `debounce`を使った基本的な使用法

`debounce`関数は、任意の関数にデバウンスを追加する最も簡単な方法です:

```ts
import { debounce } from '@tanstack/pacer'

// ユーザーが入力を止めるのを待つ検索入力のデバウンス
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

デバウンス動作をより細かく制御するには、`Debouncer`クラスを直接使用できます:

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
searchDebouncer.setOptions({ wait: 1000 }) // 待機時間を増加

// 保留中の実行をキャンセル
searchDebouncer.cancel()
```

### 先頭と末尾の実行

同期デバウンサーは先頭（leading）と末尾（trailing）の両方の実行をサポートします:

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
- `trailing: false` - 待機期間後の実行なし

一般的なパターン:
- `{ leading: false, trailing: true }` - デフォルト、待機後に実行
- `{ leading: true, trailing: false }` - 即時実行、後続の呼び出しを無視
- `{ leading: true, trailing: true }` - 最初の呼び出し時と待機後の両方で実行

### 最大待機時間

TanStack Pacerのデバウンサーは意図的に、他のデバウンスライブラリのような`maxWait`オプションを持っていません。実行をより分散させたい場合は、代わりに[スロットリング](../guides/throttling)技術の使用を検討してください。

### 有効/無効化

`Debouncer`クラスは`enabled`オプションを介した有効/無効化をサポートします。`setOptions`メソッドを使用して、いつでもデバウンサーを有効/無効にできます:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
debouncer.setOptions({ enabled: true }) // いつでも有効化
```

デバウンサーオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションに条件付きの値を設定して、デバウンサーを動的に有効/無効にできます:

```ts
// Reactの例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 入力長に基づいて有効/無効化（リアクティブオプションをサポートするフレームワークアダプターを使用している場合）
)
```

ただし、`debounce`関数または`Debouncer`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`Debouncer`クラスのコンストラクターに渡されるためです。

```ts
// Solidの例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 入力長に基づいて有効/無効化
})
```

### コールバックオプション

同期および非同期のデバウンサーは、デバウンスライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期デバウンサーのコールバック

同期`Debouncer`は以下のコールバックをサポートします:

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 各成功実行後に呼び出される
    console.log('関数が実行されました', debouncer.getExecutionCount())
  }
})
```

`onExecute`コールバックは、デバウンスされた関数が正常に実行されるたびに呼び出され、実行の追跡、UI状態の更新、またはクリーンアップ操作の実行に役立ちます。

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

`onSuccess`コールバックはデバウンスされた関数が正常に実行されるたびに呼び出され、`onError`コールバックは非同期関数がエラーをスローした場合に呼び出されます。`onSettled`コールバックは、成功または失敗に関係なく、各実行試行後に呼び出されます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、および実行メトリクスのロギングに特に役立ちます。

### 非同期デバウンス

非同期デバウンサーは、非同期操作をデバウンスで処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期デバウンサーがUIイベントと即時フィードバックに適しているのに対し、非同期バージョンはAPI呼び出し、データベース操作、およびその他の非同期タスクの処理に特化して設計されています。

#### 同期デバウンスとの主な違い

1. **戻り値の処理**
同期デバウンサーがvoidを返すのとは異なり、非同期バージョンではデバウンスされた関数からの戻り値をキャプチャして使用できます。これはAPI呼び出しやその他の非同期操作の結果を扱う必要がある場合に特に便利です。`maybeExecute`メソッドは関数の戻り値で解決するPromiseを返し、結果を適切に処理できます。

2. **強化されたコールバックシステム**
非同期デバウンサーは、同期バージョンの単一の`onExecute`コールバックに比べて、より洗練されたコールバックシステムを提供します。このシステムには以下が含まれます:
- `onSuccess`: 非同期関数が正常に完了したときに呼び出され、結果とデバウンサーインスタンスの両方を提供
- `onError`: 非同期関数がエラーをスローしたときに呼び出され、エラーとデバウンサーインスタンスの両方を提供
- `onSettled`: 成功または失敗に関係なく、すべての実行試行後に呼び出される

3. **実行追跡**
非同期デバウンサーは、いくつかのメソッドを通じて包括的な実行追跡を提供します:
- `getSuccessCount()`: 成功した実行回数
- `getErrorCount()`: 失敗した実行回数
- `getSettledCount()`: 完了した実行の総数（成功 + 失敗）

4. **順次実行**
非同期デバウンサーは、後続の実行が前の呼び出しの完了を待つことを保証します。これにより、順不同の実行が防止され、各呼び出しが最新のデータを処理することが保証されます。これは、前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に重要です。

例えば、ユーザーのプロファイルを更新し、すぐに更新されたデータを取得する場合、非同期デバウンサーは取得操作が更新の完了を待つようにし、古いデータを取得する可能性のある競合状態を防ぎます。

#### 基本的な使用例

以下は、検索操作に非同期デバウンサーを使用する基本的な例です:

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

#### 高度なパターン

非同期デバウンサーは、複雑な問題を解決するためにさまざまなパターンと組み合わせることができます:

1. **状態管理との統合**
非同期デバウンサーを状態管理システム（ReactのuseStateやSolidのcreateSignalなど）と使用する場合、ローディング状態、エラー状態、およびデータ更新を処理するための強力なパターンを作成できます。デバウンサーのコールバックは、操作の成功または失敗に基づいてUI状態を更新するための完璧なフックを提供します。

2. **競合状態の防止**
シングルフライトミューテーションパターンは、多くのシナリオで自然に競合状態を防ぎます。アプリケーションの複数の部分が同時に同じリソースを更新しようとした場合、デバウンサーは最新の更新のみが実際に発生するようにしつつ、すべての呼び出し元に結果を提供します。

3. **エラー回復**
非同期デバウンサーのエラー処理機能は、リトロジックやエラー回復パターンの実装に理想的です。`onError`コールバックを使用して、指数バックオフやフォールバックメカニズムなどのカスタムエラー処理戦略を実装できます。

### フレームワークアダプター

各フレームワークアダプターは、コアのデバウンス機能を基に構築され、フレームワークの状態管理システムと統合するためのフックを提供します。`createDebouncer`、`useDebouncedCallback`、`useDebouncedState`、または`useDebouncedValue`などのフックが各フレームワークで利用可能です。

以下にいくつかの例を示します:

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
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-p
