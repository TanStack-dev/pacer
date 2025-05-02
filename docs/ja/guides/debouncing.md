---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:14:58.704Z'
title: デバウンスガイド
id: debouncing
---
# デバウンスガイド

TanStack Pacerは、アプリケーション内での関数実行タイミングを制御するための高品質なユーティリティを提供するライブラリです。類似のユーティリティは他にも存在しますが、私たちは***型安全性***、***ツリーシェイキング***、一貫性のある***直感的なAPI***といった重要な詳細をすべて正しく実装することを目指しています。これらの基本原則に焦点を当て、それらを***フレームワークに依存しない***形で提供することで、これらのユーティリティとパターンがアプリケーションでもっと一般的に使用されることを願っています。適切な実行制御はアプリケーション開発において後回しにされがちで、パフォーマンス問題、競合状態、防ぐことのできたユーザー体験の低下を引き起こします。TanStack Pacerは、これらの重要なパターンを最初から正しく実装するのを助けます！

レートリミット、スロットリング、デバウンスは、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、それらを「ロスあり」にします - つまり、関数呼び出しが頻繁に要求されるときに、いくつかの呼び出しが実行されないことを意味します。各アプローチをいつ使用するかを理解することは、パフォーマンスが高く信頼性のあるアプリケーションを構築するために重要です。このガイドでは、TanStack Pacerのデバウンス概念について説明します。

## デバウンスの概念

デバウンスは、指定された期間の非アクティブ状態が発生するまで関数の実行を遅らせる技術です。制限まで実行をバーストさせるレートリミットや、均等に間隔をあけた実行を保証するスロットリングとは異なり、デバウンスは複数の高速な関数呼び出しを、呼び出しが停止した後にのみ発生する単一の実行にまとめます。これにより、デバウンスはアクティビティが落ち着いた後の最終状態のみを気にするイベントのバースト処理に理想的です。

### デバウンスの可視化

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Executes here after
                                                         3 ticks of no calls

             [Burst of calls]     [More calls]   [Wait]      [New burst]
             No execution         Resets timer    [Delayed Execute]  [Wait] [Delayed Execute]
```

### デバウンスを使用するタイミング

デバウンスは、アクションを実行する前に「一時停止」を待ちたい場合に特に効果的です。これにより、最終状態のみを気にするユーザー入力や他の高速発火イベントの処理に理想的です。

一般的な使用例:
- ユーザーが入力を終えるのを待ちたい検索入力フィールド
- キーストロークごとに実行すべきではないフォームバリデーション
- 計算コストが高いウィンドウリサイズ計算
- コンテンツ編集中の自動保存ドラフト
- ユーザーアクティビティが落ち着いた後にのみ発生すべきAPI呼び出し
- 急激な変更後の最終値のみを気にするシナリオ

### デバウンスを使用しないタイミング

デバウンスが最適でない場合:
- 特定の期間にわたって実行を保証する必要がある場合（代わりに[スロットリング](../guides/throttling)を使用）
- いかなる実行も見逃すことが許されない場合（代わりに[キューイング](../guides/queueing)を使用）

## TanStack Pacerでのデバウンス

TanStack Pacerは、`Debouncer`と`AsyncDebouncer`クラス（および対応する`debounce`と`asyncDebounce`関数）を介して、同期および非同期のデバウンスを提供します。

### `debounce`を使用した基本的な使い方

`debounce`関数は、任意の関数にデバウンスを追加する最も簡単な方法です:

```ts
import { debounce } from '@tanstack/pacer'

// ユーザーが入力を止めるのを待つために検索入力をデバウンス
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

### `Debouncer`クラスを使用した高度な使い方

デバウンス動作をより詳細に制御するには、`Debouncer`クラスを直接使用できます:

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

### 先行実行と後行実行

同期デバウンサーは先行実行と後行実行の両方をサポートします:

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // 最初の呼び出しで実行
  trailing: true,  // 待機期間後に実行
})
```

- `leading: true` - 最初の呼び出しで即時に関数を実行
- `leading: false` (デフォルト) - 最初の呼び出しが待機タイマーを開始
- `trailing: true` (デフォルト) - 待機期間後に関数を実行
- `trailing: false` - 待機期間後に実行なし

一般的なパターン:
- `{ leading: false, trailing: true }` - デフォルト、待機後に実行
- `{ leading: true, trailing: false }` - 即時実行、後続の呼び出しを無視
- `{ leading: true, trailing: true }` - 最初の呼び出し時と待機後の両方で実行

### 最大待機時間

TanStack Pacerのデバウンサーには、他のデバウンスライブラリのような`maxWait`オプションはありません。実行をより広がった期間にわたって実行させる必要がある場合は、代わりに[スロットリング](../guides/throttling)技術の使用を検討してください。

### 有効化/無効化

`Debouncer`クラスは、`enabled`オプションを介して有効化/無効化をサポートします。`setOptions`メソッドを使用して、いつでもデバウンサーを有効化/無効化できます:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
debouncer.setOptions({ enabled: true }) // いつでも有効化
```

デバウンサーオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、デバウンサーを動的に有効化/無効化できます:

```ts
// Reactの例
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // 入力長に基づいて有効化/無効化（リアクティブオプションをサポートするフレームワークアダプターを使用している場合）
)
```

ただし、`debounce`関数または`Debouncer`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`Debouncer`クラスのコンストラクターに渡されるためです。

```ts
// Solidの例
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // デフォルトで無効
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // 入力長に基づいて有効化/無効化
})
```

### コールバックオプション

同期および非同期デバウンサーの両方が、デバウンスライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期デバウンサーのコールバック

同期`Debouncer`は次のコールバックをサポートします:

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // 各成功した実行後に呼び出される
    console.log('関数が実行されました', debouncer.getExecutionCount())
  }
})
```

`onExecute`コールバックは、デバウンスされた関数の各成功した実行後に呼び出され、実行の追跡、UI状態の更新、またはクリーンアップ操作の実行に役立ちます。

#### 非同期デバウンサーのコールバック

非同期`AsyncDebouncer`は、エラー処理のための追加コールバックをサポートします:

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // 各成功した実行後に呼び出される
    console.log('非同期関数が実行されました', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // 非同期関数がエラーをスローした場合に呼び出される
    console.error('非同期関数が失敗しました:', error)
  }
})
```

`onExecute`コールバックは同期デバウンサーと同じように機能し、`onError`コールバックはデバウンスチェーンを壊すことなくエラーを優雅に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、クリーンアップ操作、および実行メトリックのロギングに特に役立ちます。

### 非同期デバウンス

非同期関数またはエラー処理が必要な場合、`AsyncDebouncer`または`asyncDebounce`を使用します:

```ts
import { asyncDebounce } from '@tanstack/pacer'

const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    updateUI(results)
  },
  {
    wait: 500,
    onError: (error) => {
      console.error('検索が失敗しました:', error)
    }
  }
)

// タイピングが停止した後にのみ1回のAPI呼び出しを行います
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

非同期バージョンは、Promiseベースの実行追跡、`onError`コールバックを介したエラー処理、保留中の非同期操作の適切なクリーンアップ、および待機可能な`maybeExecute`メソッドを提供します。

### フレームワークアダプター

各フレームワークアダプターは、フレームワークの状態管理システムと統合するために、コアのデバウンス機能の上に構築されたフックを提供します。`createDebouncer`、`useDebouncedCallback`、`useDebouncedState`、または`useDebouncedValue`などのフックが各フレームワークで利用可能です。

いくつかの例:

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
const [debouncedState, setDebouncedState] = useDebouncedValue(
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
