---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:22:36.353Z'
title: ديل التخفيف (Throttling)
id: throttling
---
# دليل التحديد (Throttling)  

الحد من المعدل (Rate Limiting)، التحديد (Throttling)، وإلغاء الاهتزاز (Debouncing) هي ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بطريقة مختلفة، مما يجعلها "فاقدة" (lossy) - أي أن بعض استدعاءات الدوال لن تنفذ عند طلب تشغيلها بشكل متكرر جدًا. فهم متى تستخدم كل طريقة أمر بالغ الأهمية لبناء تطبيقات عالية الأداء وموثوقة. سيتناول هذا الدليل مفاهيم التحديد (Throttling) في TanStack Pacer.

## مفهوم التحديد (Throttling)  

يضمن التحديد (Throttling) توزيع عمليات تنفيذ الدوال بشكل متساوٍ عبر الزمن. على عكس الحد من المعدل (Rate Limiting) الذي يسمح بتنفيذ دفعات من الاستدعاءات حتى حد معين، أو إلغاء الاهتزاز (Debouncing) الذي ينتظر توقف النشاط، فإن التحديد (Throttling) ينشئ نمط تنفيذ أكثر سلاسة من خلال فرض تأخيرات ثابتة بين الاستدعاءات. إذا قمت بتعيين تحديد (Throttling) لتنفيذ واحد كل ثانية، فسيتم توزيع الاستدعاءات بشكل متساوٍ بغض النظر عن مدى سرعة طلبها.

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

التحديد (Throttling) فعال بشكل خاص عندما تحتاج إلى توقيت تنفيذ متسق ومتوقع. هذا يجعله مثاليًا للتعامل مع الأحداث أو التحديثات المتكررة حيث تريد سلوكًا سلسًا ومسيطرًا عليه.  

من حالات الاستخدام الشائعة:  
- تحديثات واجهة المستخدم التي تحتاج إلى توقيت متسق (مثل مؤشرات التقدم)  
- معالجات أحداث التمرير (Scroll) أو تغيير الحجم (Resize) التي لا يجب أن تثقل المتصفح  
- جلب البيانات في الوقت الفعلي (Real-time polling) حيث تكون الفواصل الزمنية المتسقة مطلوبة  
- العمليات كثيفة الاستهلاك للموارد التي تحتاج إلى وتيرة ثابتة  
- تحديثات حلقة اللعبة (Game loop) أو معالجة إطارات الرسوم المتحركة  
- اقتراحات البحث المباشر أثناء كتابة المستخدم  

### متى لا تستخدم التحديد (Throttling)  

قد لا يكون التحديد (Throttling) هو الخيار الأفضل عندما:  
- تريد الانتظار حتى يتوقف النشاط (استخدم [إلغاء الاهتزاز (Debouncing)](../guides/debouncing) بدلاً من ذلك)  
- لا يمكنك تحمل فقدان أي عمليات تنفيذ (استخدم [الطابور (Queueing)](../guides/queueing) بدلاً من ذلك)  

> [!TIP]  
> غالبًا ما يكون التحديد (Throttling) هو الخيار الأفضل عندما تحتاج إلى توقيت تنفيذ سلس ومتسق. فهو يوفر نمط تنفيذ أكثر قابلية للتنبؤ من الحد من المعدل (Rate Limiting) وردود فعل أكثر فورية من إلغاء الاهتزاز (Debouncing).  

## التحديد (Throttling) في TanStack Pacer  

يوفر TanStack Pacer التحديد (Throttling) المتزامن وغير المتزامن من خلال الفئات `Throttler` و `AsyncThrottler` على التوالي (والدوال المقابلة لهما `throttle` و `asyncThrottle`).  

### الاستخدام الأساسي مع `throttle`  

تعد دالة `throttle` أبسط طريقة لإضافة التحديد (Throttling) إلى أي دالة:  

```ts
import { throttle } from '@tanstack/pacer'

// تحديد تحديثات واجهة المستخدم لمرة واحدة كل 200 مللي ثانية
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// في حلقة سريعة، يتم التنفيذ فقط كل 200 مللي ثانية
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // العديد من الاستدعاءات يتم تحديدها (Throttled)
}
```

### استخدام متقدم مع فئة `Throttler`  

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

### التنفيذ الأولي والنهائي  

يدعم المحدد (Throttler) المتزامن عمليات التنفيذ على الحافة الأولية (Leading) والنهائية (Trailing):  

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // التنفيذ على أول استدعاء (افتراضي)
  trailing: true,  // التنفيذ بعد فترة الانتظار (افتراضي)
})
```

- `leading: true` (افتراضي) - التنفيذ فورًا عند أول استدعاء  
- `leading: false` - تخطي أول استدعاء، انتظر التنفيذ النهائي  
- `trailing: true` (افتراضي) - تنفيذ آخر استدعاء بعد فترة الانتظار  
- `trailing: false` - تخطي آخر استدعاء إذا كان ضمن فترة الانتظار  

أنماط شائعة:  
- `{ leading: true, trailing: true }` - الافتراضي، الأكثر استجابة  
- `{ leading: false, trailing: true }` - تأخير جميع عمليات التنفيذ  
- `{ leading: true, trailing: false }` - تخطي عمليات التنفيذ في الطابور  

### التمكين/التعطيل  

تدعم فئة `Throttler` التمكين/التعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل المحدد (Throttler) في أي وقت:  

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // تعطيل افتراضيًا
throttler.setOptions({ enabled: true }) // تمكين في أي وقت
```

إذا كنت تستخدم أداة تكيف إطار عمل حيث تكون خيارات المحدد (Throttler) تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل المحدد (Throttler) على الفور. ومع ذلك، إذا كنت تستخدم دالة `throttle` أو فئة `Throttler` مباشرة، فيجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث أن الخيارات التي يتم تمريرها يتم تمريرها بالفعل إلى مُنشئ فئة `Throttler`.  

### خيارات رد النداء (Callback)  

يدعم كل من المحدد (Throttler) المتزامن وغير المتزامن خيارات رد النداء (Callback) للتعامل مع جوانب مختلفة من دورة حياة التحديد (Throttling):  

#### ردود نداء المحدد (Throttler) المتزامن  

يدعم المحدد (Throttler) المتزامن رد النداء التالي:  

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', throttler.getExecutionCount())
  }
})
```

يتم استدعاء رد النداء `onExecute` بعد كل تنفيذ ناجح للدالة المحددة (Throttled)، مما يجعله مفيدًا لتتبع عمليات التنفيذ، تحديث حالة واجهة المستخدم، أو تنفيذ عمليات التنظيف.  

#### ردود نداء المحدد (AsyncThrottler) غير المتزامن  

يدعم المحدد (AsyncThrottler) غير المتزامن ردود نداء إضافية للتعامل مع الأخطاء:  

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

يعمل رد النداء `onExecute` بنفس الطريقة كما في المحدد (Throttler) المتزامن، بينما يسمح رد النداء `onError` بالتعامل مع الأخطاء بسهولة دون كسر سلسلة التحديد (Throttling). تعد ردود النداء هذه مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، تحديث حالة واجهة المستخدم، التعامل مع الأخطاء، تنفيذ عمليات التنظيف، وتسجيل مقاييس التنفيذ.  

### التحديد غير المتزامن (Async Throttling)  

يوفر المحدد غير المتزامن (AsyncThrottler) طريقة قوية للتعامل مع العمليات غير المتزامنة مع التحديد (Throttling)، ويقدم عدة مزايا رئيسية مقارنة بالإصدار المتزامن. بينما يكون المحدد المتزامن رائعًا لأحداث واجهة المستخدم والردود الفورية، فإن الإصدار غير المتزامن مصمم خصيصًا للتعامل مع استدعاءات API، عمليات قاعدة البيانات، والمهام غير المتزامنة الأخرى.  

#### الاختلافات الرئيسية عن التحديد المتزامن  

1. **معالجة قيمة الإرجاع**  
على عكس المحدد المتزامن الذي يُرجع `void`، يسمح الإصدار غير المتزامن بالتقاط واستخدام قيمة الإرجاع من الدالة المحددة (Throttled). هذا مفيد بشكل خاص عندما تحتاج إلى العمل مع نتائج استدعاءات API أو العمليات غير المتزامنة الأخرى. تُرجع طريقة `maybeExecute` وعدًا (Promise) يحل بقيمة إرجاع الدالة، مما يسمح لك بالانتظار (await) للنتيجة والتعامل معها بشكل مناسب.  

2. **نظام رد نداء محسن**  
يوفر المحدد غير المتزامن (AsyncThrottler) نظام رد نداء أكثر تطورًا مقارنة برد النداء الفردي `onExecute` في الإصدار المتزامن. يتضمن هذا النظام:  
- `onSuccess`: يتم استدعاؤه عند اكتمال الدالة غير المتزامنة بنجاح، مع توفير كل من النتيجة ومثيل المحدد (Throttler)  
- `onError`: يتم استدعاؤه عند حدوث خطأ في الدالة غير المتزامنة، مع توفير كل من الخطأ ومثيل المحدد (Throttler)  
- `onSettled`: يتم استدعاؤه بعد كل محاولة تنفيذ، بغض النظر عن النجاح أو الفشل  

3. **تتبع التنفيذ**  
يوفر المحدد غير المتزامن (AsyncThrottler) تتبعًا شاملاً للتنفيذ من خلال عدة طرق:  
- `getSuccessCount()`: عدد عمليات التنفيذ الناجحة  
- `getErrorCount()`: عدد عمليات التنفيذ الفاشلة  
- `getSettledCount()`: إجمالي عدد عمليات التنفيذ المكتملة (النجاح + الفشل)  

4. **التنفيذ التسلسلي**  
يضمن المحدد غير المتزامن (AsyncThrottler) أن عمليات التنفيذ اللاحقة تنتظر اكتمال الاستدعاء السابق قبل البدء. هذا يمنع التنفيذ خارج الترتيب ويضمن أن كل استدعاء يعالج أحدث البيانات. هذا مهم بشكل خاص عند التعامل مع العمليات التي تعتمد على نتائج الاستدعاءات السابقة أو عندما يكون الحفاظ على تناسق البيانات أمرًا بالغ الأهمية.  

على سبيل المثال، إذا كنت تقوم بتحديث ملف تعريف المستخدم ثم جلب بياناته المحدثة على الفور، فإن المحدد غير المتزامن (AsyncThrottler) سيضمن أن عملية الجلب تنتظر اكتمال التحديث، مما يمنع حالات السباق (Race conditions) حيث قد تحصل على بيانات قديمة.  

#### مثال على الاستخدام الأساسي  

إليك مثالًا أساسيًا يوضح كيفية استخدام المحدد غير المتزامن (AsyncThrottler) لعملية بحث:  

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('نجاح البحث:', results)
    },
    onError: (error, throttler) => {
      console.error('فشل البحث:', error)
    }
  }
)

// الاستخدام
const results = await throttledSearch('query')
```

#### أنماط متقدمة  

يمكن دمج المحدد غير المتزامن (AsyncThrottler) مع أنماط مختلفة لحل المشكلات المعقدة:  

1. **تكامل إدارة الحالة**  
عند استخدام المحدد غير المتزامن (AsyncThrottler) مع أنظمة إدارة الحالة (مثل useState في React أو createSignal في Solid)، يمكنك إنشاء أنماط قوية للتعامل مع حالات التحميل، حالات الخطأ، وتحديثات البيانات. توفر ردود نداء المحدد (Throttler) خطافات مثالية لتحديث حالة واجهة المستخدم بناءً على نجاح أو فشل العمليات.  

2. **منع حالات السباق (Race Conditions)**  
يمنع نمط التحديد (Throttling) حالات السباق (Race Conditions) في العديد من السيناريوهات. عندما تحاول أجزاء متعددة من تطبيقك تحديث نفس المورد في نفس الوقت، يضمن المحدد (Throttler) حدوث التحديثات بمعدل مسيطر عليه، مع توفير النتائج لجميع المستدعين.  

3. **استعادة الخطأ**  
تجعل قدرات التعامل مع الأخطاء في المحدد غير المتزامن (AsyncThrottler) مثاليًا لتنفيذ منطق إعادة المحاولة وأنماط استعادة الخطأ. يمكنك استخدام رد النداء `onError` لتنفيذ استراتيجيات التعامل مع الأخطاء المخصصة، مثل التراجع الأسي (Exponential backoff) أو آليات الاحتياط.  

### أدوات تكيف الأطر (Framework Adapters)  

يوفر كل أداة تكيف إطار خطافات (Hooks) تعتمد على وظائف التحديد (Throttling) الأساسية للتكامل مع نظام إدارة الحالة الخاص بالإطار. تتوفر خطافات مثل `createThrottler`، `useThrottledCallback`، `useThrottledState`، أو `useThrottledValue` لكل إطار.  

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

// خطاف يعتمد على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // القيمة المطلوب تحديدها (Throttle)
  { wait: 200 }
)
```

#### Solid  

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// خطاف منخ
