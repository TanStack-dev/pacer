---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:09:26.534Z'
title: レート制限ガイド
id: rate-limiting
---
# レートリミティングガイド

レートリミティング（Rate Limiting）、スロットリング（Throttling）、デバウンス（Debouncing）は、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロッシー（lossy）」な性質を持ちます - つまり、関数が頻繁に呼び出されるとき、一部の関数呼び出しは実行されません。パフォーマンスが高く信頼性のあるアプリケーションを構築するためには、各アプローチを使用するタイミングを理解することが重要です。このガイドでは、TanStack Pacerのレートリミティング概念について説明します。

> [!NOTE]
> TanStack Pacerは現在フロントエンドライブラリのみです。これらはクライアントサイドのレートリミティング用ユーティリティです。

## レートリミティングの概念

レートリミティングは、特定の時間枠内で関数が実行できる回数を制限する技術です。APIリクエストや他の外部サービス呼び出しを処理する際など、関数が過剰に呼び出されるのを防ぎたいシナリオで特に有用です。これは最も*単純な*アプローチであり、クォータが満たされるまで実行がバースト的に発生することを許容します。

### レートリミティングの可視化

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     �️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### レートリミティングを使用するタイミング

レートリミティングは、バックエンドサービスに過剰な負荷をかけたり、ブラウザでパフォーマンス問題を引き起こす可能性のあるフロントエンド操作を扱う際に特に重要です。

一般的な使用例:
- ユーザーの迅速な操作（ボタンクリックやフォーム送信など）による意図しないAPIスパムの防止
- バースト的な動作が許容されるが、最大レートを制限したいシナリオ
- 意図しない無限ループや再帰操作からの保護

### レートリミティングを使用しないタイミング

レートリミティングは、関数実行頻度を制御する最も単純なアプローチです。3つの手法の中で最も柔軟性が低く、制限的です。より間隔を空けた実行が必要な場合は、[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

> [!TIP]
> ほとんどのユースケースでは「レートリミティング」を使用したくないでしょう。[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

レートリミティングの「ロッシー（lossy）」な性質は、一部の実行が拒否され失われることを意味します。すべての実行が常に成功することを保証する必要がある場合、これは問題になる可能性があります。実行レートを遅くするためにスロットルされた遅延で実行されるようにすべての実行をキューイングする必要がある場合は、[キューイング](../guides/queueing)の使用を検討してください。

## TanStack Pacerでのレートリミティング

TanStack Pacerは、`RateLimiter`クラスと`AsyncRateLimiter`クラス（および対応する`rateLimit`と`asyncRateLimit`関数）を介して、同期型と非同期型のレートリミティングを提供します。

### `rateLimit`での基本的な使用法

`rateLimit`関数は、任意の関数にレートリミティングを追加する最も簡単な方法です。単純な制限を強制するだけでよいほとんどのユースケースに最適です。

```ts
import { rateLimit } from '@tanstack/pacer'

// API呼び出しを1分間に5回に制限
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1分（ミリ秒）
    onReject: (rateLimiter) => {
      console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
    }
  }
)

// 最初の5回の呼び出しは即時実行
rateLimitedApi('user-1') // ✅ 実行
rateLimitedApi('user-2') // ✅ 実行
rateLimitedApi('user-3') // ✅ 実行
rateLimitedApi('user-4') // ✅ 実行
rateLimitedApi('user-5') // ✅ 実行
rateLimitedApi('user-6') // ❌ ウィンドウがリセットされるまで拒否
```

### `RateLimiter`クラスでの高度な使用法

レートリミティングの動作をさらに制御する必要があるより複雑なシナリオでは、`RateLimiter`クラスを直接使用できます。これにより、追加のメソッドや状態情報にアクセスできます。

```ts
import { RateLimiter } from '@tanstack/pacer'

// レートリミッターインスタンスを作成
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('関数が実行されました', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
    }
  }
)

// 現在の状態に関する情報を取得
console.log(limiter.getRemainingInWindow()) // 現在のウィンドウ内の残り呼び出し回数
console.log(limiter.getExecutionCount()) // 成功した実行の総数
console.log(limiter.getRejectionCount()) // 拒否された実行の総数

// 実行を試みる（成功を示すブール値を返す）
limiter.maybeExecute('user-1')

// オプションを動的に更新
limiter.setOptions({ limit: 10 }) // 制限を増加

// すべてのカウンターと状態をリセット
limiter.reset()
```

### 有効化/無効化

`RateLimiter`クラスは、`enabled`オプションを介して有効化/無効化をサポートしています。`setOptions`メソッドを使用して、いつでもレートリミッターを有効化/無効化できます:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // デフォルトで無効
})
limiter.setOptions({ enabled: true }) // いつでも有効化
```

レートリミッターオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、レートリミッターを動的に有効化/無効化できます。ただし、`rateLimit`関数または`RateLimiter`クラスを直接使用している場合、`enabled`オプションを変更するには`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`RateLimiter`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期型と非同期型の両方のレートリミッターは、レートリミティングライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期型レートリミッターのコールバック

同期型`RateLimiter`は以下のコールバックをサポートします:

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 各成功した実行後に呼び出される
    console.log('関数が実行されました', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 実行がレートリミティングにより拒否されたときに呼び出される
    console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
  }
})
```

`onExecute`コールバックは、レート制限された関数が正常に実行されるたびに呼び出され、`onReject`コールバックは、レートリミティングにより実行が拒否されたときに呼び出されます。これらのコールバックは、実行の追跡、UI状態の更新、またはユーザーへのフィードバックの提供に役立ちます。

#### 非同期型レートリミッターのコールバック

非同期型`AsyncRateLimiter`は、エラー処理のための追加コールバックをサポートします:

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // 各成功した実行後に呼び出される
    console.log('非同期関数が実行されました', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // 実行がレートリミティングにより拒否されたときに呼び出される
    console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
  },
  onError: (error) => {
    // 非同期関数がエラーをスローした場合に呼び出される
    console.error('非同期関数が失敗しました:', error)
  }
})
```

`onExecute`および`onReject`コールバックは同期型レートリミッターと同じように機能し、`onError`コールバックは、レートリミティングチェーンを壊すことなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、およびユーザーへのフィードバックの提供に特に役立ちます。

### 非同期レートリミティング

非同期レートリミッターは、レートリミティングを使用して非同期操作を処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期レートリミッターがUIイベントと即時のフィードバックに適しているのに対し、非同期バージョンはAPI呼び出し、データベース操作、およびその他の非同期タスクの処理に特化して設計されています。

#### 同期型レートリミティングとの主な違い

1. **戻り値の処理**
同期型レートリミッターが成功を示すブール値を返すのとは異なり、非同期バージョンではレート制限された関数からの戻り値をキャプチャして使用できます。これは、API呼び出しやその他の非同期操作の結果を操作する必要がある場合に特に便利です。`maybeExecute`メソッドは、関数の戻り値で解決されるPromiseを返し、結果を適切に処理できるようにします。

2. **強化されたコールバックシステム**
非同期レートリミッターは、同期バージョンのコールバックに比べてより洗練されたコールバックシステムを提供します。このシステムには以下が含まれます:
- `onExecute`: 各成功した実行後に呼び出され、レートリミッターインスタンスを提供
- `onReject`: レートリミティングにより実行が拒否されたときに呼び出され、レートリミッターインスタンスを提供
- `onError`: 非同期関数がエラーをスローした場合に呼び出され、エラーとレートリミッターインスタンスの両方を提供

3. **実行追跡**
非同期レートリミッターは、いくつかのメソッドを通じて包括的な実行追跡を提供します:
- `getExecutionCount()`: 成功した実行の数
- `getRejectionCount()`: 拒否された実行の数
- `getRemainingInWindow()`: 現在のウィンドウ内の残り実行回数
- `getMsUntilNextWindow()`: 次のウィンドウが開始するまでのミリ秒数

4. **順次実行**
非同期レートリミッターは、後続の実行が前の呼び出しが完了するのを待ってから開始することを保証します。これにより、順不同の実行が防止され、各呼び出しが最新のデータを処理することが保証されます。これは、前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に重要です。

たとえば、ユーザーのプロファイルを更新してからすぐに更新されたデータを取得する場合、非同期レートリミッターは、更新が完了するまで取得操作が待機することを保証し、古いデータを取得する可能性のある競合状態を防止します。

#### 基本的な使用例

API操作に非同期レートリミッターを使用する基本的な例を以下に示します:

```ts
const rateLimitedApi = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onExecute: (limiter) => {
      console.log('API呼び出しが成功しました:', limiter.getExecutionCount())
    },
    onReject: (limiter) => {
      console.log(`レート制限を超えました。${limiter.getMsUntilNextWindow()}ms後に再試行してください`)
    },
    onError: (error, limiter) => {
      console.error('API呼び出しが失敗しました:', error)
    }
  }
)

// 使用法
const result = await rateLimitedApi('123')
```

#### 高度なパターン

非同期レートリミッターは、さまざまなパターンと組み合わせて複雑な問題を解決できます:

1. **状態管理との統合**
非同期レートリミッターを状態管理システム（ReactのuseStateやSolidのcreateSignalなど）と組み合わせて使用すると、ローディング状態、エラー状態、およびデータ更新を処理する強力なパターンを作成できます。レートリミッターのコールバックは、操作の成功または失敗に基づいてUI状態を更新するための完璧なフックを提供します。

2. **競合状態の防止**
レートリミティングパターンは、多くのシナリオで自然に競合状態を防止します。アプリケーションの複数の部分が同時に同じリソースを更新しようとする場合、レートリミッターは更新が構成された制限内で発生することを保証しながら、すべての呼び出し元に結果を提供します。

3. **エラー回復**
非同期レートリミッターのエラー処理機能は、再試行ロジックとエラー回復パターンを実装するのに理想的です。`onError`コールバックを使用して、指数バックオフやフォールバックメカニズムなどのカスタムエラー処理戦略を実装できます。

### フレームワークアダプター

各フレームワークアダプターは、コアのレートリミティング機能を基に構築され、フレームワークの状態管理システムと統合するためのフックを提供します。`createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState`、`useRateLimitedValue`などのフックが各フレームワークで利用可能です。

以下にいくつかの例を示します:

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// 完全な制御のための低レベルフック
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// 基本的なユースケースのためのシンプルなコールバックフック
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// リアクティブな状態管理のための状態ベースのフック
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // レート制限する値
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// 完全な制御のための低レベルフック
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  {
