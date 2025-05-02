---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:40:53.720Z'
title: ديل تحديد المعدل (Rate Limiting)
id: rate-limiting
---
# دليل تحديد معدل التنفيذ (Rate Limiting)

تعتبر تقنيات تحديد المعدل (Rate Limiting)، والتحكم في التدفق (Throttling)، وإلغاء الاهتزاز (Debouncing) ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بشكل مختلف، مما يجعلها "فاقدة" (lossy) - أي أن بعض استدعاءات الدوال لن تنفذ عند طلب تشغيلها بكثرة. فهم متى تستخدم كل تقنية أمر بالغ الأهمية لبناء تطبيقات ذات أداء موثوق. سيغطي هذا الدليل مفاهيم تحديد المعدل في TanStack Pacer.

> [!ملاحظة]
> TanStack Pacer حاليًا مكتبة للواجهة الأمامية فقط. هذه الأدوات مخصصة لتحديد المعدل من جانب العميل (client-side).

## مفهوم تحديد المعدل (Rate Limiting)

تحديد المعدل (Rate Limiting) هو تقنية تحد من المعدل الذي يمكن للدالة أن تنفذ خلال نافذة زمنية محددة. إنها مفيدة بشكل خاص في السيناريوهات التي تريد منع استدعاء دالة بكثرة، مثل التعامل مع طلبات واجهة برمجة التطبيقات (API) أو استدعاءات الخدمات الخارجية. إنها الطريقة الأكثر *بساطة*، حيث تسمح بتنفيذ الدوال في دفعات حتى يتم استنفاد الحصة المحددة.

### تصور تحديد المعدل

```text
تحديد المعدل (حد: 3 استدعاءات لكل نافذة)
الخط الزمني: [1 ثانية لكل علامة]
                                        النافذة 1                  |    النافذة 2            
الاستدعاءات:   ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
المنفذة:       ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 مسموح بها ===][=== محظورة حتى تنتهي النافذة ===][=== نافذة جديدة =======]
```

### متى تستخدم تحديد المعدل

تحديد المعدل مهم بشكل خاص عند التعامل مع عمليات الواجهة الأمامية التي قد تثقل كاهل خدمات الخلفية عن غير قصد أو تسبب مشاكل في الأداء في المتصفح.

من حالات الاستخدام الشائعة:
- منع إرسال طلبات واجهة برمجة التطبيقات (API) بكثرة من تفاعلات المستخدم السريعة (مثل النقر على الأزرار أو إرسال النماذج)
- السيناريوهات التي يكون فيها السلوك المتقطع مقبولاً ولكنك تريد تحديد الحد الأقصى للمعدل
- الحماية من الحلقات اللانهائية غير المقصودة أو العمليات المتكررة

### متى لا تستخدم تحديد المعدل

تحديد المعدل هو الطريقة الأكثر بساطة للتحكم في تكرار تنفيذ الدوال. إنها الأقل مرونة والأكثر تقييدًا بين التقنيات الثلاث. فكر في استخدام [التحكم في التدفق (Throttling)](../guides/throttling) أو [إلغاء الاهتزاز (Debouncing)](../guides/debouncing) بدلاً من ذلك لتنفيذ أكثر تباعدًا.

> [!نصيحة]
> على الأرجح لا تريد استخدام "تحديد المعدل" لمعظم حالات الاستخدام. فكر في استخدام [التحكم في التدفق (Throttling)](../guides/throttling) أو [إلغاء الاهتزاز (Debouncing)](../guides/debouncing) بدلاً من ذلك.

طبيعة تحديد المعدل "الفاقدة" تعني أيضًا أن بعض عمليات التنفيذ سيتم رفضها وفقدانها. يمكن أن يكون هذا مشكلة إذا كنت بحاجة إلى التأكد من أن جميع عمليات التنفيذ ناجحة دائمًا. فكر في استخدام [الانتظار في قائمة (Queueing)](../guides/queueing) إذا كنت بحاجة إلى التأكد من أن جميع عمليات التنفيذ تنتظر في قائمة للتنفيذ، ولكن مع تأخير للتحكم في التدفق لإبطاء معدل التنفيذ.

## تحديد المعدل في TanStack Pacer

يوفر TanStack Pacer تحديد المعدل المتزامن وغير المتزامن من خلال فئتي `RateLimiter` و `AsyncRateLimiter` على التوالي (ووظائفهما المقابلة `rateLimit` و `asyncRateLimit`).

### الاستخدام الأساسي مع `rateLimit`

وظيفة `rateLimit` هي أبسط طريقة لإضافة تحديد معدل إلى أي دالة. إنها مثالية لمعظم حالات الاستخدام حيث تحتاج فقط إلى فرض حد بسيط.

```ts
import { rateLimit } from '@tanstack/pacer'

// تحديد معدل استدعاءات واجهة برمجة التطبيقات إلى 5 في الدقيقة
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 دقيقة بالميلي ثانية
    onReject: (rateLimiter) => {
      console.log(`تم تجاوز حد المعدل. حاول مرة أخرى بعد ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// أول 5 استدعاءات ستنفذ فورًا
rateLimitedApi('user-1') // ✅ تنفذ
rateLimitedApi('user-2') // ✅ تنفذ
rateLimitedApi('user-3') // ✅ تنفذ
rateLimitedApi('user-4') // ✅ تنفذ
rateLimitedApi('user-5') // ✅ تنفذ
rateLimitedApi('user-6') // ❌ مرفوض حتى إعادة تعيين النافذة
```

### الاستخدام المتقدم مع فئة `RateLimiter`

للحالات الأكثر تعقيدًا حيث تحتاج إلى تحكم إضافي في سلوك تحديد المعدل، يمكنك استخدام فئة `RateLimiter` مباشرة. هذا يمنحك الوصول إلى طرق ومعلومات حالة إضافية.

```ts
import { RateLimiter } from '@tanstack/pacer'

// إنشاء مثيل محدد معدل
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('تم تنفيذ الدالة', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`تم تجاوز حد المعدل. حاول مرة أخرى بعد ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// الحصول على معلومات عن الحالة الحالية
console.log(limiter.getRemainingInWindow()) // عدد الاستدعاءات المتبقية في النافذة الحالية
console.log(limiter.getExecutionCount()) // إجمالي عدد عمليات التنفيذ الناجحة
console.log(limiter.getRejectionCount()) // إجمالي عدد عمليات التنفيذ المرفوضة

// محاولة التنفيذ (تُرجع قيمة منطقية تشير إلى النجاح)
limiter.maybeExecute('user-1')

// تحديث الخيارات ديناميكيًا
limiter.setOptions({ limit: 10 }) // زيادة الحد

// إعادة تعيين جميع العدادات والحالة
limiter.reset()
```

### التمكين/التعطيل

تدعم فئة `RateLimiter` التمكين والتعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل محدد المعدل في أي وقت:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // تعطيل افتراضيًا
})
limiter.setOptions({ enabled: true }) // تمكين في أي وقت
```

إذا كنت تستخدم أداة تكييف إطار عمل حيث تكون خيارات محدد المعدل تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل محدد المعدل على الفور. ومع ذلك، إذا كنت تستخدم وظيفة `rateLimit` أو فئة `RateLimiter` مباشرة، يجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث أن الخيارات التي يتم تمريرها يتم تمريرها بالفعل إلى منشئ فئة `RateLimiter`.

### خيارات ردود النداء (Callbacks)

يدعم كل من محددات المعدل المتزامنة وغير المتزامنة خيارات ردود النداء للتعامل مع جوانب مختلفة من دورة حياة تحديد المعدل:

#### ردود نداء محدد المعدل المتزامن

يدعم `RateLimiter` المتزامن ردود النداء التالية:

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // يتم استدعاؤها بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // يتم استدعاؤها عند رفض تنفيذ
    console.log(`تم تجاوز حد المعدل. حاول مرة أخرى بعد ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

يتم استدعاء رد النداء `onExecute` بعد كل تنفيذ ناجح للدالة المحددة بمعدل، بينما يتم استدعاء رد النداء `onReject` عند رفض تنفيذ بسبب تحديد المعدل. هذه ردود النداء مفيدة لتتبع عمليات التنفيذ، تحديث حالة واجهة المستخدم، أو تقديم ملاحظات للمستخدمين.

#### ردود نداء محدد المعدل غير المتزامن

يدعم `AsyncRateLimiter` غير المتزامن ردود نداء إضافية للتعامل مع الأخطاء:

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // يتم استدعاؤها بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة غير المتزامنة', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // يتم استدعاؤها عند رفض تنفيذ
    console.log(`تم تجاوز حد المعدل. حاول مرة أخرى بعد ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // يتم استدعاؤها إذا ألقت الدالة غير المتزامنة خطأ
    console.error('فشلت الدالة غير المتزامنة:', error)
  }
})
```

تعمل ردود النداء `onExecute` و `onReject` بنفس الطريقة كما في محدد المعدل المتزامن، بينما يسمح رد النداء `onError` لك بالتعامل مع الأخطاء بسهولة دون كسر سلسلة تحديد المعدل. هذه ردود النداء مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، تحديث حالة واجهة المستخدم، التعامل مع الأخطاء، وتقديم ملاحظات للمستخدمين.

### تحديد المعدل غير المتزامن

استخدم `AsyncRateLimiter` عندما:
- ترجع الدالة المحددة بمعدل وعدًا (Promise)
- تحتاج إلى التعامل مع أخطاء من الدالة غير المتزامنة
- تريد ضمان تحديد المعدل بشكل صحيح حتى إذا استغرقت الدالة غير المتزامنة وقتًا لإكمالها

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
      console.error('فشل استدعاء واجهة برمجة التطبيقات:', error)
    }
  }
)

// تُرجع Promise<boolean> - تحل إلى true إذا تم التنفيذ، false إذا تم الرفض
const wasExecuted = await rateLimited('123')
```

توفر النسخة غير المتزامنة تتبع التنفيذ القائم على الوعد، التعامل مع الأخطاء عبر رد النداء `onError`، تنظيفًا صحيحًا للعمليات غير المتزامنة المعلقة، وطريقة `maybeExecute` قابلة للانتظار.

### أدوات تكييف إطار العمل

توفر كل أداة تكييف لإطار عمل خطافات (hooks) تبني على وظيفة تحديد المعدل الأساسية للتكامل مع نظام إدارة حالة الإطار. تتوفر خطافات مثل `createRateLimiter`، `useRateLimitedCallback`، `useRateLimitedState`، أو `useRateLimitedValue` لكل إطار عمل.

فيما يلي بعض الأمثلة:

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// خطاف رد نداء بسيط لحالات الاستخدام الأساسية
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// خطاف قائم على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // القيمة المحددة بمعدل
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// خطاف قائم على الإشارة (Signal) لإدارة الحالة
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('إجمالي عمليات التنفيذ:', limiter.getExecutionCount())
  }
})
```
