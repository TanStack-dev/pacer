---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:14:57.779Z'
title: レート制限ガイド
id: rate-limiting
---
レートリミティング、スロットリング、デバウンスは、関数の実行頻度を制御する3つの異なるアプローチです。各手法は実行を異なる方法でブロックし、「ロスあり（lossy）」の動作となります - つまり、関数が頻繁に呼び出されすぎると、一部の呼び出しが実行されません。パフォーマンスが高く信頼性のあるアプリケーションを構築するには、各アプローチをいつ使用するかを理解することが重要です。このガイドでは、TanStack Pacerのレートリミティングの概念について説明します。

> [!NOTE]
> TanStack Pacerは現在、フロントエンドライブラリのみです。これらはクライアントサイドのレートリミティング用のユーティリティです。

## レートリミティングの概念

レートリミティングは、特定の時間ウィンドウ内で関数が実行できる回数を制限する手法です。APIリクエストや他の外部サービス呼び出しを処理する際など、関数が頻繁に呼び出されるのを防ぎたいシナリオで特に有用です。これは最も*単純な（naive）*アプローチであり、クォータに達するまでバースト的に実行を許可します。

### レートリミティングの可視化

```text
レートリミティング（制限: ウィンドウあたり3回）
タイムライン: [1秒ごとに1ティック]
                                       ウィンドウ1                  |    ウィンドウ2            
呼び出し:     ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
実行:         ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3回許可 ===][=== ウィンドウ終了までブロック ===][=== 新しいウィンドウ =======]
```

### レートリミティングを使用する場合

レートリミティングは、バックエンドサービスに過剰な負荷をかけたり、ブラウザでパフォーマンスの問題を引き起こす可能性のあるフロントエンド操作を扱う際に特に重要です。

一般的な使用例:
- ユーザーの迅速な操作（ボタンクリックやフォーム送信など）による誤ったAPIスパムの防止
- バースト的な動作が許容されるが、最大レートを制限したいシナリオ
- 誤った無限ループや再帰操作からの保護

### レートリミティングを使用しない場合

レートリミティングは、関数の実行頻度を制御する最も単純なアプローチです。3つの手法の中で最も柔軟性が低く、制限的です。より間隔を空けた実行が必要な場合は、[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

> [!TIP]
> ほとんどのユースケースでは「レートリミティング」を使用したくないでしょう。[スロットリング](../guides/throttling)または[デバウンス](../guides/debouncing)の使用を検討してください。

レートリミティングの「ロスあり」の性質は、一部の実行が拒否され失われることを意味します。すべての実行が常に成功する必要がある場合は、[キューイング](../guides/queueing)の使用を検討してください。これにより、すべての実行がキューに入れられ、実行レートを遅らせるためにスロットリングされた遅延で実行されます。

## TanStack Pacerでのレートリミティング

TanStack Pacerは、`RateLimiter`クラスと`AsyncRateLimiter`クラス（および対応する`rateLimit`関数と`asyncRateLimit`関数）を介して、同期型と非同期型のレートリミティングを提供します。

### `rateLimit`での基本的な使用法

`rateLimit`関数は、任意の関数にレートリミティングを追加する最も簡単な方法です。単純な制限を適用する必要があるほとんどのユースケースに最適です。

```ts
import { rateLimit } from '@tanstack/pacer'

// API呼び出しを1分あたり5回に制限
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // ミリ秒単位で1分
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

レートリミティングの動作をさらに制御する必要がある複雑なシナリオでは、`RateLimiter`クラスを直接使用できます。これにより、追加のメソッドや状態情報にアクセスできます。

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

// 実行を試みる（成功したかどうかを示すブール値を返す）
limiter.maybeExecute('user-1')

// オプションを動的に更新
limiter.setOptions({ limit: 10 }) // 制限を増やす

// すべてのカウンターと状態をリセット
limiter.reset()
```

### 有効化/無効化

`RateLimiter`クラスは、`enabled`オプションを介して有効化/無効化をサポートしています。`setOptions`メソッドを使用して、いつでもレートリミッターを有効化/無効化できます:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // デフォルトで無効化
})
limiter.setOptions({ enabled: true }) // いつでも有効化
```

レートリミッターオプションがリアクティブなフレームワークアダプターを使用している場合、`enabled`オプションを条件付きの値に設定して、レートリミッターを動的に有効化/無効化できます。ただし、`rateLimit`関数または`RateLimiter`クラスを直接使用している場合は、`setOptions`メソッドを使用して`enabled`オプションを変更する必要があります。渡されるオプションは実際には`RateLimiter`クラスのコンストラクターに渡されるためです。

### コールバックオプション

同期型と非同期型の両方のレートリミッターは、レートリミティングのライフサイクルのさまざまな側面を処理するためのコールバックオプションをサポートしています:

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
    // 実行が拒否されたときに呼び出される
    console.log(`レート制限を超えました。${rateLimiter.getMsUntilNextWindow()}ms後に再試行してください`)
  }
})
```

`onExecute`コールバックは、レート制限された関数が成功裏に実行されるたびに呼び出され、`onReject`コールバックは、レートリミティングにより実行が拒否されたときに呼び出されます。これらのコールバックは、実行の追跡、UI状態の更新、またはユーザーへのフィードバックの提供に役立ちます。

#### 非同期型レートリミッターのコールバック

非同期型`AsyncRateLimiter`は、エラー処理のための追加のコールバックをサポートします:

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

`onExecute`および`onReject`コールバックは同期型レートリミッターと同じように機能し、`onError`コールバックにより、レートリミティングのチェーンを壊すことなくエラーを適切に処理できます。これらのコールバックは、実行回数の追跡、UI状態の更新、エラー処理、およびユーザーへのフィードバックの提供に特に有用です。

### 非同期型レートリミティング

`AsyncRateLimiter`は以下の場合に使用します:
- レート制限された関数がPromiseを返す場合
- 非同期関数からのエラーを処理する必要がある場合
- 非同期関数の完了に時間がかかる場合でも適切なレートリミティングを確保したい場合

```ts
import { asyncRateLimit } from '@tanstack/pacer'

const rateLimited = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onError: (error) => {
      console.error('API呼び出しが失敗しました:', error)
    }
  }
)

// Promise<boolean>を返す - 実行された場合はtrue、拒否された場合はfalseに解決
const wasExecuted = await rateLimited('123')
```

非同期バージョンは、Promiseベースの実行追跡、`onError`コールバックを介したエラー処理、保留中の非同期操作の適切なクリーンアップ、および待機可能な`maybeExecute`メソッドを提供します。

### フレームワークアダプター

各フレームワークアダプターは、コアのレートリミティング機能を基盤として構築され、フレームワークの状態管理システムと統合するためのフックを提供します。`createRateLimiter`、`useRateLimitedCallback`、`useRateLimitedState`、または`useRateLimitedValue`などのフックが各フレームワークで利用可能です。

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

// リアクティブな状態管理のためのステートベースのフック
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
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
  { limit: 5, window: 1000 }
)

// 状態管理のためのシグナルベースのフック
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('総実行回数:', limiter.getExecutionCount())
  }
})
```
