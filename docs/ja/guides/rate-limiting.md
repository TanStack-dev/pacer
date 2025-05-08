---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:50:45.049Z'
title: レート制限ガイド
id: rate-limiting
---
# レートリミティングガイド

レートリミティング (Rate Limiting)、スロットリング (Throttling)、デバウンス (Debouncing) は、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックするため、「ロッシー (lossy)」、つまり関数呼び出しが頻繁に要求されると一部の実行が行われない可能性があります。パフォーマンスが高く信頼性のあるアプリケーションを構築するためには、各アプローチを使用するタイミングを理解することが重要です。このガイドでは、TanStack Pacerのレートリミティング概念について説明します。

> [!NOTE]
> TanStack Pacerは現在フロントエンドライブラリのみです。これらはクライアントサイドのレートリミティング用ユーティリティです。

## レートリミティングの概念

レートリミティングは、特定の時間枠内で関数が実行できる回数を制限する技術です。APIリクエストや他の外部サービス呼び出しを処理する際など、関数が頻繁に呼び出されるのを防ぎたいシナリオで特に有用です。これは最も*単純な*アプローチであり、クォータが満たされるまで実行がバースト的に発生することを許容します。

### レートリミティングの可視化

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### ウィンドウタイプ

TanStack Pacerは2種類のレートリミティングウィンドウをサポートしています:

1. **固定ウィンドウ (Fixed Window)** (デフォルト)
   - ウィンドウ期間後にリセットされる厳密なウィンドウ
   - ウィンドウ内のすべての実行が制限にカウントされる
   - 期間後にウィンドウが完全にリセットされる
   - ウィンドウ境界でバースト的な動作が発生する可能性がある

2. **スライディングウィンドウ (Sliding Window)**
   - 古い実行が期限切れになると新しい実行を許可するローリングウィンドウ
   - 時間経過に伴ってより一貫した実行レートを提供
   - 実行の安定したフローを維持するのに適している
   - ウィンドウ境界でのバースト的な動作を防ぐ

スライディングウィンドウレートリミティングの可視化:

```text
Sliding Window Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ✅                             ✅     ✅
             [=== 3 allowed ===][=== oldest expires, new allowed ===][=== continues sliding =======]
```

重要な違いは、スライディングウィンドウでは最も古い実行が期限切れになるとすぐに新しい実行が許可されることです。これにより、固定ウィンドウアプローチと比較してより一貫した実行フローが作成されます。

### レートリミティングを使用するタイミング

レートリミティングは、バックエンドサービスに過剰な負荷をかけたり、ブラウザでパフォーマンス問題を引き起こす可能性のあるフロントエンド操作を扱う際に特に重要です。

一般的な使用例:
- ユーザーの迅速な操作（ボタンクリックやフォーム送信など）による誤ったAPIスパムの防止
- バースト的な動作が許容可能だが最大レートを制限したいシナリオ
- 誤った無限ループや再帰操作からの保護

### レートリミティングを使用しないタイミング

レートリミティングは、関数実行頻度を制御する最も単純なアプローチです。3つの手法の中で最も柔軟性が低く、制限的です。より間隔を空けた実行が必要な場合は、代わりに[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

> [!TIP]
> ほとんどのユースケースでは「レートリミティング」を使用したくないでしょう。代わりに[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

レートリミティングの「ロッシー (lossy)」な性質は、一部の実行が拒否され失われることを意味します。すべての実行が常に成功することを保証する必要がある場合、これは問題になる可能性があります。実行レートを遅くするためにスロットルされた遅延で実行されるようにすべての実行をキューに入れる必要がある場合は、[キューイング](../guides/queueing)の使用を検討してください。

## TanStack Pacerでのレートリミティング

TanStack Pacerは、`RateLimiter`クラスと`AsyncRateLimiter`クラス（および対応する`rateLimit`と`asyncRateLimit`関数）を介して、同期および非同期のレートリミティングを提供します。

### `rateLimit`を使用した基本的な使用法

`rateLimit`関数は、任意の関数にレートリミティングを追加する最も簡単な方法です。単純な制限を適用する必要があるほとんどのユースケースに最適です。

```ts
import { rateLimit } from '@tanstack/pacer'

// API呼び出しを1分間に5回に制限
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1分（ミリ秒）
    windowType: 'fixed', // デフォルト
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

### `RateLimiter`クラスを使用した高度な使用法

レートリミティングの動作を追加で制御する必要があるより複雑なシナリオでは、`RateLimiter`クラスを直接使用できます。これにより、追加のメソッドや状態情報にアクセスできます。

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
limiter.setOptions({ limit: 10 }) // 制限を増やす

// すべてのカウンターと状態をリセット
limiter.reset()
```

### 有効化/無効化

`RateLimiter`クラスは、`enabled`オプションを介して有効化/無効化をサポートしています。`setOptions`メソッドを使用すると、いつでもレートリミッターを有効化/無効化できます:

> [!NOTE]
> `enabled`オプションは実際の関数実行を有効化/無効化します。レートリミッターを無効にしてもレートリミティングはオフになりませんが、関数がまったく実行されなくなります。

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // デフォルトで無効
})
limiter.setOptions({ enabled: true }) // いつでも有効化
```

レートリミッターオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、レートリミッターを動的に有効化/無効化できます。ただし、`rateLimit`関数または`RateLimiter`クラスを直接使用している場合は、`enabled`オプションを変更するために`setOptions`メソッドを使用する必要があります。渡されるオプションは実際には`RateLimiter`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期および非同期のレートリミッターは、レートリミティングライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

#### 同期レートリミッターのコールバック

同期`RateLimiter`は次のコールバックをサポートします:

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

`onExecute`コールバックは、レート制限された関数の各成功した実行後に呼び出され、`onReject`コールバックは、レートリミティングにより実行が拒否されたときに呼び出されます。これらのコールバックは、実行の追跡、UI状態の更新、またはユーザーへのフィードバックの提供に役立ちます。

#### 非同期レートリミッターのコールバック

非同期`AsyncRateLimiter`は、エラー処理のための追加のコールバックをサポートします:

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
    // 実行が拒否されたときに呼び出される
    console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
  },
  onError: (error) => {
    // 非同期関数がエラーをスローした場合に呼び出される
    console.error('非同期関数が失敗しました:', error)
  }
})
```

`onExecute`および`onReject`コールバックは同期レートリミッターと同じように機能し、`onError`コールバックにより、レートリミティングチェーンを壊すことなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、およびユーザーへのフィードバックの提供に特に役立ちます。

### 非同期レートリミティング

非同期レートリミッターは、非同期操作をレートリミティングで処理する強力な方法を提供し、同期バージョンに比べていくつかの重要な利点があります。同期レートリミッターがUIイベントと即時のフィードバックに適しているのに対し、非同期バージョンは特にAPI呼び出し、データベース操作、およびその他の非同期タスクの処理用に設計されています。

#### 同期レートリミティングとの主な違い

1. **戻り値の処理**
成功を示すブール値を返す同期レートリミッターとは異なり、非同期バージョンではレート制限された関数からの戻り値をキャプチャして使用できます。これは、API呼び出しやその他の非同期操作の結果を操作する必要がある場合に特に便利です。`maybeExecute`メソッドは、関数の戻り値で解決されるPromiseを返すため、結果を待機して適切に処理できます。

2. **異なるコールバック**
`AsyncRateLimiter`は、同期バージョンの`onExecute`だけでなく、次のコールバックをサポートします:
- `onSuccess`: 各成功した実行後に呼び出され、レートリミッターインスタンスを提供
- `onSettled`: 各実行後に呼び出され、レートリミッターインスタンスを提供
- `onError`: 非同期関数がエラーをスローした場合に呼び出され、エラーとレートリミッターインスタンスの両方を提供

非同期および同期レートリミッターの両方が、ブロックされた実行を処理するための`onReject`コールバックをサポートしています。

3. **順次実行**
レートリミッターの`maybeExecute`メソッドはPromiseを返すため、各実行を待機してから次の実行を開始することを選択できます。これにより、実行順序を制御し、各呼び出しが最新のデータを処理することを保証できます。これは、前の呼び出しの結果に依存する操作や、データの一貫性を維持することが重要な場合に特に役立ちます。

たとえば、ユーザーのプロファイルを更新してからすぐに更新されたデータを取得する場合、取得操作を開始する前に更新操作を待機できます:

#### 基本的な使用例

API操作に非同期レートリミッターを使用する方法を示す基本的な例:

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

### フレームワークアダプター

各フレームワークアダプターは、フレームワークの状態管理システムと統合するために、コアのレートリミティング機能の上に構築されたフックを提供します。`createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState`、または`useRateLimitedValue`などのフックが各フレームワークで利用可能です。

いくつかの例を示します:

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

// リアクティブな状態管理のためのステートベースのフック
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
const limiter = createRateLim
