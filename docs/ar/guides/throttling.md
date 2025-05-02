---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:40:45.347Z'
title: ديل التخفيف (Throttling)
id: throttling
---
# دليل التخفيض (Throttling)

الحد من المعدل (Rate Limiting)، التخفيض (Throttling)، وإلغاء الاهتزاز (Debouncing) هي ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بطريقة مختلفة، مما يجعلها "فاقدة" (lossy) - أي أن بعض استدعاءات الدوال لن تنفذ عند طلب تشغيلها بشكل متكرر جدًا. فهم متى تستخدم كل طريقة أمر بالغ الأهمية لبناء تطبيقات عالية الأداء وموثوقة. سيغطي هذا الدليل مفاهيم التخفيض (Throttling) في TanStack Pacer.

## مفهوم التخفيض (Throttling)

يضمن التخفيض (Throttling) توزيع تنفيذ الدوال بشكل متساوٍ عبر الزمن. على عكس الحد من المعدل (Rate Limiting) الذي يسمح باندفاعات من عمليات التنفيذ حتى حد معين، أو إلغاء الاهتزاز (Debouncing) الذي ينتظر توقف النشاط، فإن التخفيض (Throttling) ينشئ نمط تنفيذ أكثر سلاسة من خلال فرض تأخيرات ثابتة بين الاستدعاءات. إذا قمت بتعيين تخفيض (Throttling) لتنفيذ واحد في الثانية، فسيتم تباعد الاستدعاءات بشكل متساوٍ بغض النظر عن مدى سرعة طلبها.

### تصور التخفيض (Throttling)

```text
Throttling (one execution per 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Executed:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Only one execution allowed per 3 ticks,
               regardless of how many calls are made

             [First burst]    [More calls]              [Spaced calls]
             Execute first    Execute after             Execute each time
             then throttle    wait period               wait period passes
```

### متى تستخدم التخفيض (Throttling)

التخفيض (Throttling) فعال بشكل خاص عندما تحتاج إلى توقيت تنفيذ متسق ومتوقع. هذا يجعله مثاليًا للتعامل مع الأحداث أو التحديثات المتكررة حيث تريد سلوكًا سلسًا ومسيطرًا عليه.

حالات الاستخدام الشائعة تشمل:
- تحديثات واجهة المستخدم التي تحتاج إلى توقيت ثابت (مثل مؤشرات التقدم)
- معالجات أحداث التمرير (Scroll) أو تغيير الحجم (Resize) التي لا يجب أن تثقل متصفح الويب
- جلب البيانات في الوقت الفعلي حيث تكون الفواصل الزمنية المتسقة مرغوبة
- العمليات المكثفة للموارد التي تحتاج إلى وتيرة ثابتة
- تحديثات حلقة اللعبة أو معالجة إطارات الرسوم المتحركة
- اقتراحات البحث المباشر أثناء كتابة المستخدمين

### متى لا تستخدم التخفيض (Throttling)

قد لا يكون التخفيض (Throttling) الخيار الأفضل عندما:
- تريد الانتظار حتى يتوقف النشاط (استخدم [إلغاء الاهتزاز (Debouncing)](../guides/debouncing) بدلاً من ذلك)
- لا يمكنك تحمل فقدان أي عمليات تنفيذ (استخدم [الانتظار في قائمة (Queueing)](../guides/queueing) بدلاً من ذلك)

> [!TIP]
> غالبًا ما يكون التخفيض (Throttling) هو الخيار الأفضل عندما تحتاج إلى توقيت تنفيذ سلس ومتسق. فهو يوفر نمط تنفيذ أكثر قابلية للتنبؤ من الحد من المعدل (Rate Limiting) وردود فعل أكثر فورية من إلغاء الاهتزاز (Debouncing).

## التخفيض (Throttling) في TanStack Pacer

يوفر TanStack Pacer كلاً من التخفيض (Throttling) المتزامن وغير المتزامن من خلال فئتي `Throttler` و `AsyncThrottler` على التوالي (ووظائفهما المقابلة `throttle` و `asyncThrottle`).

### الاستخدام الأساسي مع `throttle`

تعتبر دالة `throttle` أبسط طريقة لإضافة التخفيض (Throttling) إلى أي دالة:

```ts
import { throttle } from '@tanstack/pacer'

// تخفيض تحديثات واجهة المستخدم إلى مرة واحدة كل 200 مللي ثانية
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// في حلقة سريعة، ينفذ فقط كل 200 مللي ثانية
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // العديد من الاستدعاءات يتم تخفيضها (Throttled)
}
```

### استخدام متقدم مع فئة `Throttler`

لمزيد من التحكم في سلوك التخفيض (Throttling)، يمكنك استخدام فئة `Throttler` مباشرة:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// الحصول على معلومات حول حالة التنفيذ
console.log(updateThrottler.getExecutionCount()) // عدد عمليات التنفيذ الناجحة
console.log(updateThrottler.getLastExecutionTime()) // الطابع الزمني لآخر تنفيذ

// إلغاء أي تنفيذ معلق
updateThrottler.cancel()
```

### التنفيذ الأولي والنهائي (Leading and Trailing)

يدعم المخفض (Throttler) المتزامن كلاً من التنفيذ على الحافة الأولية (Leading) والنهائية (Trailing):

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // التنفيذ عند أول استدعاء (افتراضي)
  trailing: true,  // التنفيذ بعد فترة الانتظار (افتراضي)
})
```

- `leading: true` (افتراضي) - التنفيذ فورًا عند أول استدعاء
- `leading: false` - تخطي أول استدعاء، انتظر التنفيذ النهائي (Trailing)
- `trailing: true` (افتراضي) - تنفيذ آخر استدعاء بعد فترة الانتظار
- `trailing: false` - تخطي آخر استدعاء إذا كان ضمن فترة الانتظار

أنماط شائعة:
- `{ leading: true, trailing: true }` - افتراضي، أكثر استجابة
- `{ leading: false, trailing: true }` - تأخير جميع عمليات التنفيذ
- `{ leading: true, trailing: false }` - تخطي عمليات التنفيذ في قائمة الانتظار

### تمكين/تعطيل

تدعم فئة `Throttler` التمكين والتعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل المخفض (Throttler) في أي وقت:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // تعطيل افتراضيًا
throttler.setOptions({ enabled: true }) // تمكين في أي وقت
```

إذا كنت تستخدم أداة تكيف إطار عمل حيث تكون خيارات المخفض (Throttler) تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل المخفض (Throttler) على الفور. ومع ذلك، إذا كنت تستخدم دالة `throttle` أو فئة `Throttler` مباشرة، فيجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث أن الخيارات التي يتم تمريرها يتم تمريرها بالفعل إلى منشئ فئة `Throttler`.

### خيارات رد الاتصال (Callback Options)

يدعم كل من المخفض (Throttler) المتزامن وغير المتزامن خيارات رد الاتصال للتعامل مع جوانب مختلفة من دورة حياة التخفيض (Throttling):

#### ردود الاتصال للمخفض (Throttler) المتزامن

يدعم المخفض (Throttler) المتزامن رد الاتصال التالي:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', throttler.getExecutionCount())
  }
})
```

يتم استدعاء رد الاتصال `onExecute` بعد كل تنفيذ ناجح للدالة المخفضة (Throttled)، مما يجعله مفيدًا لتتبع عمليات التنفيذ، تحديث حالة واجهة المستخدم، أو تنفيذ عمليات التنظيف.

#### ردود الاتصال للمخفض (AsyncThrottler) غير المتزامن

يدعم المخفض (AsyncThrottler) غير المتزامن ردود اتصال إضافية للتعامل مع الأخطاء:

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة غير المتزامنة', throttler.getExecutionCount())
  },
  onError: (error) => {
    // يتم استدعاؤه إذا ألقت الدالة غير المتزامنة خطأ
    console.error('فشلت الدالة غير المتزامنة:', error)
  }
})
```

يعمل رد الاتصال `onExecute` بنفس الطريقة كما في المخفض (Throttler) المتزامن، بينما يسمح لك رد الاتصال `onError` بالتعامل مع الأخطاء بأمان دون كسر سلسلة التخفيض (Throttling). تعد ردود الاتصال هذه مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، تحديث حالة واجهة المستخدم، التعامل مع الأخطاء، تنفيذ عمليات التنظيف، وتسجيل مقاييس التنفيذ.

### التخفيض (Throttling) غير المتزامن

للدوال غير المتزامنة أو عندما تحتاج إلى التعامل مع الأخطاء، استخدم `AsyncThrottler` أو `asyncThrottle`:

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
      console.error('فشل استدعاء API:', error)
    }
  }
)

// سيقوم بعمل استدعاء API واحد فقط في الثانية
await throttledFetch('123')
```

توفر النسخة غير المتزامنة تتبع التنفيذ القائم على الوعد (Promise)، التعامل مع الأخطاء من خلال رد الاتصال `onError`، تنظيفًا مناسبًا للعمليات غير المتزامنة المعلقة، وطريقة `maybeExecute` القابلة للانتظار (awaitable).

### أدوات تكيف الأطر (Framework Adapters)

توفر كل أداة تكيف إطار (Framework Adapter) خطافات (Hooks) تبني على وظائف التخفيض (Throttling) الأساسية للتكامل مع نظام إدارة الحالة الخاص بالإطار. تتوفر خطافات مثل `createThrottler`، `useThrottledCallback`، `useThrottledState`، أو `useThrottledValue` لكل إطار.

فيما يلي بعض الأمثلة:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// خطاف رد اتصال بسيط لحالات الاستخدام الأساسية
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// خطاف قائم على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // القيمة المراد تخفيضها (Throttle)
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// خطاف قائم على الإشارة (Signal) لإدارة الحالة
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('إجمالي عمليات التنفيذ:', throttler.getExecutionCount())
  }
})
```

توفر كل أداة تكيف إطار (Framework Adapter) خطافات تتكامل مع نظام إدارة الحالة الخاص بالإطار مع الحفاظ على وظائف التخفيض (Throttling) الأساسية.
