---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T06:05:07.488Z'
title: دليل التخفيف (Throttling)
id: throttling
---
# دليل التحديد (Throttling)

التحديد المعدل (Rate Limiting)، التحديد (Throttling)، والتخفيف (Debouncing) هي ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بشكل مختلف، مما يجعلها "فاقدة" - بمعنى أن بعض استدعاءات الدوال لن تنفذ عند طلب تشغيلها بشكل متكرر جدًا. فهم الوقت المناسب لاستخدام كل أسلوب أمر بالغ الأهمية لبناء تطبيقات عالية الأداء وموثوقة. سيغطي هذا الدليل مفاهيم التحديد (Throttling) في TanStack Pacer.

## مفهوم التحديد (Throttling)

يضمن التحديد (Throttling) توزيع عمليات تنفيذ الدوال بشكل متساوٍ عبر الزمن. على عكس التحديد المعدل (Rate Limiting) الذي يسمح باندفاعات من عمليات التنفيذ حتى حد معين، أو التخفيف (Debouncing) الذي ينتظر توقف النشاط، فإن التحديد (Throttling) ينشئ نمط تنفيذ أكثر سلاسة من خلال فرض تأخيرات ثابتة بين الاستدعاءات. إذا قمت بتعيين تحديد (Throttling) لتنفيذ واحد كل ثانية، فسيتم توزيع الاستدعاءات بشكل متساوٍ بغض النظر عن مدى سرعة طلبها.

### تصور التحديد (Throttling)

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

### متى تستخدم التحديد (Throttling)

التحديد (Throttling) فعال بشكل خاص عندما تحتاج إلى توقيت تنفيذ ثابت ومتوقع. هذا يجعله مثاليًا للتعامل مع الأحداث أو التحديثات المتكررة حيث تريد سلوكًا سلسًا ومسيطرًا عليه.

حالات الاستخدام الشائعة تشمل:
- تحديثات واجهة المستخدم التي تحتاج إلى توقيت ثابت (مثل مؤشرات التقدم)
- معالجات أحداث التمرير (Scroll) أو تغيير الحجم (Resize) التي لا يجب أن تثقل المتصفح
- استطلاع البيانات في الوقت الفعلي حيث تكون الفواصل الزمنية الثابتة مرغوبة
- العمليات كثيفة الموارد التي تحتاج إلى وتيرة ثابتة
- تحديثات حلقة اللعبة (Game loop) أو معالجة إطار الرسوم المتحركة
- اقتراحات البحث المباشر أثناء كتابة المستخدمين

### متى لا تستخدم التحديد (Throttling)

قد لا يكون التحديد (Throttling) هو الخيار الأفضل عندما:
- تريد الانتظار حتى يتوقف النشاط (استخدم [التخفيف (Debouncing)](../guides/debouncing) بدلاً من ذلك)
- لا يمكنك تحمل فقدان أي عمليات تنفيذ (استخدم [الطوابير (Queueing)](../guides/queueing) بدلاً من ذلك)

> [!TIP]
> غالبًا ما يكون التحديد (Throttling) هو الخيار الأفضل عندما تحتاج إلى توقيت تنفيذ سلس وثابت. فهو يوفر نمط تنفيذ أكثر قابلية للتنبؤ من التحديد المعدل (Rate Limiting) وردود فعل أكثر فورية من التخفيف (Debouncing).

## التحديد (Throttling) في TanStack Pacer

يوفر TanStack Pacer كلاً من التحديد المتزامن وغير المتزامن من خلال فئتي `Throttler` و `AsyncThrottler` على التوالي (ووظائفهما المقابلة `throttle` و `asyncThrottle`).

### الاستخدام الأساسي مع `throttle`

تعتبر دالة `throttle` أبسط طريقة لإضافة تحديد (Throttling) إلى أي دالة:

```ts
import { throttle } from '@tanstack/pacer'

// تحديد تحديثات واجهة المستخدم لمرة واحدة كل 200 مللي ثانية
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// في حلقة سريعة، ينفذ فقط كل 200 مللي ثانية
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // العديد من الاستدعاءات يتم تحديدها
}
```

### الاستخدام المتقدم مع فئة `Throttler`

لمزيد من التحكم في سلوك التحديد (Throttling)، يمكنك استخدام فئة `Throttler` مباشرة:

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

### عمليات التنفيذ الأولية والنهائية (Leading and Trailing)

يدعم المحدد المتزامن كلاً من عمليات التنفيذ على الحافة الأولية والنهائية:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // التنفيذ فورًا عند أول استدعاء (افتراضي)
  trailing: true,  // التنفيذ بعد فترة الانتظار (افتراضي)
})
```

- `leading: true` (افتراضي) - التنفيذ فورًا عند أول استدعاء
- `leading: false` - تخطي أول استدعاء، الانتظار للتنفيذ النهائي
- `trailing: true` (افتراضي) - تنفيذ آخر استدعاء بعد فترة الانتظار
- `trailing: false` - تخطي آخر استدعاء إذا كان ضمن فترة الانتظار

أنماط شائعة:
- `{ leading: true, trailing: true }` - افتراضي، أكثر استجابة
- `{ leading: false, trailing: true }` - تأخير جميع عمليات التنفيذ
- `{ leading: true, trailing: false }` - تخطي عمليات التنفيذ في قائمة الانتظار

### التمكين/التعطيل

تدعم فئة `Throttler` التمكين/التعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل المحدد في أي وقت:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // تعطيل افتراضيًا
throttler.setOptions({ enabled: true }) // تمكين في أي وقت
```

إذا كنت تستخدم أداة تكيف إطار عمل حيث تكون خيارات المحدد تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل المحدد على الفور. ومع ذلك، إذا كنت تستخدم دالة `throttle` أو فئة `Throttler` مباشرة، يجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث أن الخيارات التي يتم تمريرها يتم تمريرها بالفعل إلى منشئ فئة `Throttler`.

### خيارات رد النداء (Callback Options)

يدعم كل من المحدد المتزامن وغير المتزامن خيارات رد النداء للتعامل مع جوانب مختلفة من دورة حياة التحديد (Throttling):

#### ردود نداء المحدد المتزامن

يدعم `Throttler` المتزامن رد النداء التالي:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', throttler.getExecutionCount())
  }
})
```

يتم استدعاء رد النداء `onExecute` بعد كل تنفيذ ناجح للدالة المحددة، مما يجعله مفيدًا لتتبع عمليات التنفيذ، تحديث حالة واجهة المستخدم، أو تنفيذ عمليات التنظيف.

#### ردود نداء المحدد غير المتزامن

يدعم `AsyncThrottler` غير المتزامن ردود نداء إضافية للتعامل مع الأخطاء:

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

يعمل رد النداء `onExecute` بنفس الطريقة كما في المحدد المتزامن، بينما يسمح رد النداء `onError` لك بالتعامل مع الأخطاء بأمان دون كسر سلسلة التحديد (Throttling). تعد ردود النداء هذه مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، تحديث حالة واجهة المستخدم، التعامل مع الأخطاء، تنفيذ عمليات التنظيف، وتسجيل مقاييس التنفيذ.

### التحديد غير المتزامن (Asynchronous Throttling)

يوفر المحدد غير المتزامن طريقة قوية للتعامل مع العمليات غير المتزامنة مع التحديد (Throttling)، حيث يقدم عدة مزايا رئيسية مقارنة بالإصدار المتزامن. بينما يكون المحدد المتزامن رائعًا لأحداث واجهة المستخدم وردود الفعل الفورية، فإن الإصدار غير المتزامن مصمم خصيصًا للتعامل مع استدعاءات API، عمليات قاعدة البيانات، والمهام غير المتزامنة الأخرى.

#### الاختلافات الرئيسية عن التحديد المتزامن

1. **معالجة القيمة المرجعة**
على عكس المحدد المتزامن الذي يُرجع void، يسمح الإصدار غير المتزامن لك بالتقاط واستخدام القيمة المرجعة من دالتك المحددة. هذا مفيد بشكل خاص عندما تحتاج إلى العمل مع نتائج استدعاءات API أو العمليات غير المتزامنة الأخرى. تُرجع طريقة `maybeExecute` وعدًا (Promise) يحل بقيمة إرجاع الدالة، مما يسمح لك بانتظار النتيجة والتعامل معها بشكل مناسب.

2. **ردود نداء مختلفة**
يدعم `AsyncThrottler` ردود النداء التالية بدلاً من `onExecute` فقط في الإصدار المتزامن:
- `onSuccess`: يتم استدعاؤه بعد كل تنفيذ ناجح، مع توفير مثيل المحدد
- `onSettled`: يتم استدعاؤه بعد كل تنفيذ، مع توفير مثيل المحدد
- `onError`: يتم استدعاؤه إذا ألقت الدالة غير المتزامنة خطأ، مع توفير كل من الخطأ ومثيل المحدد

كلا المحددين المتزامن وغير المتزامن يدعمان رد النداء `onExecute` للتعامل مع عمليات التنفيذ الناجحة.

3. **التنفيذ المتسلسل**
نظرًا لأن طريقة `maybeExecute` للمحدد تُرجع وعدًا (Promise)، يمكنك اختيار انتظار كل تنفيذ قبل بدء التالي. وهذا يمنحك التحكم في ترتيب التنفيذ ويضمن معالجة كل استدعاء لأحدث البيانات. هذا مفيد بشكل خاص عند التعامل مع العمليات التي تعتمد على نتائج الاستدعاءات السابقة أو عندما يكون الحفاظ على اتساق البيانات أمرًا بالغ الأهمية.

على سبيل المثال، إذا كنت تقوم بتحديث ملف تعريف المستخدم ثم جلب بياناته المحدثة على الفور، يمكنك انتظار عملية التحديث قبل بدء الجلب:

#### مثال على الاستخدام الأساسي

إليك مثالاً أساسيًا يوضح كيفية استخدام المحدد غير المتزامن لعملية بحث:

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('نجح البحث:', results)
    },
    onError: (error, throttler) => {
      console.error('فشل البحث:', error)
    }
  }
)

// الاستخدام
const results = await throttledSearch('query')
```

### أدوات تكيف الأطر (Framework Adapters)

توفر كل أداة تكيف إطار خطاطيف (hooks) تعتمد على وظائف التحديد الأساسية للدمج مع نظام إدارة الحالة الخاص بالإطار. تتوفر خطاطيف مثل `createThrottler`، `useThrottledCallback`، `useThrottledState`، أو `useThrottledValue` لكل إطار.

إليك بعض الأمثلة:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// خطاف رد نداء بسيط لحالات الاستخدام الأساسية
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// خطاف قائم على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // القيمة المطلوب تحديدها
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

توفر كل أداة تكيف إطار خطاطيف تدمج مع نظام إدارة الحالة الخاص بالإطار مع الحفاظ على وظائف التحديد الأساسية.
