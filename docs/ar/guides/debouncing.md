---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:40:13.188Z'
title: دليل التخلص من الارتداد (Debouncing)
id: debouncing
---
# دليل إزالة الارتداد (Debouncing Guide)

تعتبر تقنيات تحديد المعدل (Rate Limiting)، والتحكم في التدفق (Throttling)، وإزالة الارتداد (Debouncing) ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بطريقة مختلفة، مما يجعلها "فاقدة" - أي أن بعض استدعاءات الدوال لن تنفذ عند طلب تشغيلها بشكل متكرر جدًا. يعد فهم الوقت المناسب لاستخدام كل طريقة أمرًا بالغ الأهمية لبناء تطبيقات عالية الأداء وموثوقة. سيغطي هذا الدليل مفاهيم إزالة الارتداد في TanStack Pacer.

## مفهوم إزالة الارتداد (Debouncing Concept)

إزالة الارتداد (Debouncing) هي تقنية تؤخر تنفيذ الدالة حتى يمر فترة محددة من عدم النشاط. على عكس تحديد المعدل الذي يسمح باندفاعات من عمليات التنفيذ حتى حد معين، أو التحكم في التدفق الذي يضمن تنفيذًا متباعدًا بشكل متساوٍ، فإن إزالة الارتداد تجمع بين استدعاءات الدوال السريعة المتعددة في تنفيذ واحد يحدث فقط بعد توقف الاستدعاءات. هذا يجعل إزالة الارتداد مثالية للتعامل مع اندفاعات الأحداث حيث تهتم فقط بالحالة النهائية بعد استقرار النشاط.

### تصور إزالة الارتداد (Debouncing Visualization)

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

### متى تستخدم إزالة الارتداد (When to Use Debouncing)

تكون إزالة الارتداد فعالة بشكل خاص عندما تريد الانتظار حتى "توقف" النشاط قبل اتخاذ إجراء. هذا يجعلها مثالية للتعامل مع إدخال المستخدم أو الأحداث الأخرى التي يتم تشغيلها بسرعة حيث تهتم فقط بالحالة النهائية.

من حالات الاستخدام الشائعة:
- حقول إدخال البحث حيث تريد الانتظار حتى ينتهي المستخدم من الكتابة
- التحقق من صحة النموذج الذي لا يجب أن يعمل مع كل ضغطة مفتاح
- حسابات تغيير حجم النافذة التي تكون مكلفة في الحساب
- الحفظ التلقائي للمسودات أثناء تحرير المحتوى
- استدعاءات API التي يجب أن تحدث فقط بعد استقرار نشاط المستخدم
- أي سيناريو حيث تهتم فقط بالقيمة النهائية بعد التغييرات السريعة

### متى لا تستخدم إزالة الارتداد (When Not to Use Debouncing)

قد لا تكون إزالة الارتداد هي الخيار الأفضل عندما:
- تحتاج إلى تنفيذ مضمون خلال فترة زمنية محددة (استخدم [التحكم في التدفق](../guides/throttling) بدلاً من ذلك)
- لا يمكنك تحمل فقدان أي عمليات تنفيذ (استخدم [الطابور](../guides/queueing) بدلاً من ذلك)

## إزالة الارتداد في TanStack Pacer (Debouncing in TanStack Pacer)

يوفر TanStack Pacer إزالة ارتداد متزامنة وغير متزامنة من خلال فئتي `Debouncer` و `AsyncDebouncer` على التوالي (ووظائفهما المقابلة `debounce` و `asyncDebounce`).

### الاستخدام الأساسي مع `debounce`

تعتبر وظيفة `debounce` أبسط طريقة لإضافة إزالة ارتداد إلى أي دالة:

```ts
import { debounce } from '@tanstack/pacer'

// إزالة ارتداد إدخال البحث للانتظار حتى يتوقف المستخدم عن الكتابة
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // الانتظار لمدة 500 مللي ثانية بعد آخر ضغطة مفتاح
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### استخدام متقدم مع فئة `Debouncer`

لمزيد من التحكم في سلوك إزالة الارتداد، يمكنك استخدام فئة `Debouncer` مباشرة:

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// الحصول على معلومات حول الحالة الحالية
console.log(searchDebouncer.getExecutionCount()) // عدد عمليات التنفيذ الناجحة
console.log(searchDebouncer.getIsPending()) // ما إذا كان هناك استدعاء معلق

// تحديث الخيارات ديناميكيًا
searchDebouncer.setOptions({ wait: 1000 }) // زيادة وقت الانتظار

// إلغاء التنفيذ المعلق
searchDebouncer.cancel()
```

### التنفيذ الأولي والنهائي (Leading and Trailing Executions)

يدعم مزيل الارتداد المتزامن التنفيذ على الحافتين الأولية والنهائية:

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // التنفيذ عند أول استدعاء
  trailing: true,  // التنفيذ بعد فترة الانتظار
})
```

- `leading: true` - يتم تنفيذ الدالة فورًا عند أول استدعاء
- `leading: false` (الافتراضي) - يبدأ أول استدعاء مؤقت الانتظار
- `trailing: true` (الافتراضي) - يتم تنفيذ الدالة بعد فترة الانتظار
- `trailing: false` - لا يوجد تنفيذ بعد فترة الانتظار

أنماط شائعة:
- `{ leading: false, trailing: true }` - الافتراضي، التنفيذ بعد الانتظار
- `{ leading: true, trailing: false }` - التنفيذ فورًا، تجاهل الاستدعاءات اللاحقة
- `{ leading: true, trailing: true }` - التنفيذ عند أول استدعاء وبعد فترة الانتظار

### وقت الانتظار الأقصى (Max Wait Time)

لا يحتوي مزيل الارتداد في TanStack Pacer على خيار `maxWait` مثل مكتبات إزالة الارتداد الأخرى. إذا كنت بحاجة إلى السماح بتنفيذ عمليات على فترة زمنية أكثر انتشارًا، ففكر في استخدام تقنية [التحكم في التدفق](../guides/throttling) بدلاً من ذلك.

### التمكين/التعطيل (Enabling/Disabling)

تدعم فئة `Debouncer` التمكين/التعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل مزيل الارتداد في أي وقت:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // التعطيل افتراضيًا
debouncer.setOptions({ enabled: true }) // التمكين في أي وقت
```

إذا كنت تستخدم محول إطار عمل حيث تكون خيارات مزيل الارتداد تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل مزيل الارتداد على الفور:

```ts
// مثال React
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // تمكين/تعطيل بناءً على طول الإدخال IF باستخدام محول إطار عمل يدعم الخيارات التفاعلية
)
```

ومع ذلك، إذا كنت تستخدم وظيفة `debounce` أو فئة `Debouncer` مباشرة، فيجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث يتم تمرير الخيارات في الواقع إلى منشئ فئة `Debouncer`.

```ts
// مثال Solid
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // التعطيل افتراضيًا
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // تمكين/تعطيل بناءً على طول الإدخال
})
```

### خيارات رد الاتصال (Callback Options)

يدعم كل من مزيل الارتداد المتزامن وغير المتزامن خيارات رد الاتصال للتعامل مع جوانب مختلفة من دورة حياة إزالة الارتداد:

#### ردود الاتصال لمزيل الارتداد المتزامن

يدعم `Debouncer` المتزامن رد الاتصال التالي:

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', debouncer.getExecutionCount())
  }
})
```

يتم استدعاء رد الاتصال `onExecute` بعد كل تنفيذ ناجح للدالة التي تمت إزالة ارتدادها، مما يجعله مفيدًا لتتبع عمليات التنفيذ، تحديث حالة واجهة المستخدم، أو تنفيذ عمليات التنظيف.

#### ردود الاتصال لمزيل الارتداد غير المتزامن

يدعم `AsyncDebouncer` ردود اتصال إضافية للتعامل مع الأخطاء:

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة غير المتزامنة', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // يتم استدعاؤه إذا ألقت الدالة غير المتزامنة خطأ
    console.error('فشلت الدالة غير المتزامنة:', error)
  }
})
```

يعمل رد الاتصال `onExecute` بنفس الطريقة كما في مزيل الارتداد المتزامن، بينما يسمح رد الاتصال `onError` بالتعامل مع الأخطاء بسلاسة دون كسر سلسلة إزالة الارتداد. تعتبر ردود الاتصال هذه مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، تحديث حالة واجهة المستخدم، التعامل مع الأخطاء، تنفيذ عمليات التنظيف، وتسجيل مقاييس التنفيذ.

### إزالة الارتداد غير المتزامنة (Asynchronous Debouncing)

للدوال غير المتزامنة أو عندما تحتاج إلى التعامل مع الأخطاء، استخدم `AsyncDebouncer` أو `asyncDebounce`:

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
      console.error('فشل البحث:', error)
    }
  }
)

// سيقوم بعمل استدعاء API واحد فقط بعد توقف الكتابة
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

توفر النسخة غير المتزامنة تتبع تنفيذ قائم على الوعد، التعامل مع الأخطاء عبر رد الاتصال `onError`، تنظيفًا مناسبًا للعمليات غير المتزامنة المعلقة، وطريقة `maybeExecute` التي يمكن انتظارها.

### محولات الأطر (Framework Adapters)

يوفر كل محول إطار عمل خطافات تبني على وظائف إزالة الارتداد الأساسية للاندماج مع نظام إدارة حالة الإطار. تتوفر خطافات مثل `createDebouncer`، `useDebouncedCallback`، `useDebouncedState`، أو `useDebouncedValue` لكل إطار عمل.

فيما يلي بعض الأمثلة:

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// خطاف رد اتصال بسيط لحالات الاستخدام الأساسية
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// خطاف قائم على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState('')
const [debouncedState, setDebouncedState] = useDebouncedValue(
  instantState, // القيمة المراد إزالة ارتدادها
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// خطاف إشارة لإدارة الحالة
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('إجمالي عمليات التنفيذ:', debouncer.getExecutionCount())
  }
})
```
