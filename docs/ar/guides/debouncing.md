---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T06:05:27.197Z'
title: دليل التخلص من الارتداد (Debouncing)
id: debouncing
---
# دليل إزالة الارتداد (Debouncing)

تعتبر تقنيات تحديد المعدل (Rate Limiting)، والتحكم في التدفق (Throttling)، وإزالة الارتداد (Debouncing) ثلاث طرق مختلفة للتحكم في تكرار تنفيذ الدوال. كل تقنية تمنع عمليات التنفيذ بشكل مختلف، مما يجعلها "فاقدة" - أي أن بعض استدعاءات الدوال لن يتم تنفيذها عند طلب تشغيلها بشكل متكرر جدًا. فهم متى تستخدم كل تقنية أمر بالغ الأهمية لبناء تطبيقات عالية الأداء وموثوقة. سيغطي هذا الدليل مفاهيم إزالة الارتداد (Debouncing) في TanStack Pacer.

## مفهوم إزالة الارتداد (Debouncing)

إزالة الارتداد (Debouncing) هي تقنية تؤخر تنفيذ الدالة حتى مرور فترة محددة من عدم النشاط. على عكس تحديد المعدل (Rate Limiting) الذي يسمح بتنفيذ دفعات من الاستدعاءات حتى حد معين، أو التحكم في التدفق (Throttling) الذي يضمن تنفيذًا متباعدًا بشكل متساوٍ، فإن إزالة الارتداد (Debouncing) تجمع استدعاءات الدالة المتعددة السريعة في تنفيذ واحد يحدث فقط بعد توقف الاستدعاءات. هذا يجعل إزالة الارتداد (Debouncing) مثالية للتعامل مع دفعات الأحداث حيث تهتم فقط بالحالة النهائية بعد استقرار النشاط.

### تصور إزالة الارتداد (Debouncing)

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

### متى تستخدم إزالة الارتداد (Debouncing)

تكون إزالة الارتداد (Debouncing) فعالة بشكل خاص عندما تريد الانتظار حتى "توقف" النشاط قبل اتخاذ إجراء. هذا يجعلها مثالية للتعامل مع إدخال المستخدم أو الأحداث الأخرى التي يتم تشغيلها بسرعة حيث تهتم فقط بالحالة النهائية.

من حالات الاستخدام الشائعة:
- حقول إدخال البحث حيث تريد الانتظار حتى ينتهي المستخدم من الكتابة
- التحقق من صحة النموذج الذي لا يجب أن يعمل مع كل ضغطة مفتاح
- حسابات تغيير حجم النافذة التي تكون مكلفة في الحساب
- الحفظ التلقائي للمسودات أثناء تحرير المحتوى
- استدعاءات API التي يجب أن تحدث فقط بعد استقرار نشاط المستخدم
- أي سيناريو حيث تهتم فقط بالقيمة النهائية بعد التغييرات السريعة

### متى لا تستخدم إزالة الارتداد (Debouncing)

قد لا تكون إزالة الارتداد (Debouncing) هي الخيار الأفضل عندما:
- تحتاج إلى تنفيذ مضمون خلال فترة زمنية محددة (استخدم [التحكم في التدفق (Throttling)](../guides/throttling) بدلاً من ذلك)
- لا يمكنك تحمل فقدان أي عمليات تنفيذ (استخدم [الطابور (Queueing)](../guides/queueing) بدلاً من ذلك)

## إزالة الارتداد (Debouncing) في TanStack Pacer

يوفر TanStack Pacer إزالة الارتداد (Debouncing) المتزامن وغير المتزامن من خلال فئتي `Debouncer` و `AsyncDebouncer` على التوالي (ووظائفهما المقابلة `debounce` و `asyncDebounce`).

### الاستخدام الأساسي مع `debounce`

تعتبر الدالة `debounce` أبسط طريقة لإضافة إزالة الارتداد (Debouncing) إلى أي دالة:

```ts
import { debounce } from '@tanstack/pacer'

// إزالة الارتداد (Debouncing) لإدخال البحث للانتظار حتى يتوقف المستخدم عن الكتابة
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // الانتظار 500 مللي ثانية بعد آخر ضغطة مفتاح
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### الاستخدام المتقدم مع فئة `Debouncer`

لمزيد من التحكم في سلوك إزالة الارتداد (Debouncing)، يمكنك استخدام فئة `Debouncer` مباشرة:

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

### التنفيذ الأولي والنهائي

يدعم مزيل الارتداد (Debouncer) المتزامن عمليات التنفيذ على الحافة الأولية والنهائية:

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
- `{ leading: true, trailing: true }` - التنفيذ عند أول استدعاء وبعد الانتظار

### وقت الانتظار الأقصى

لا يحتوي مزيل الارتداد (Debouncer) في TanStack Pacer عمدًا على خيار `maxWait` مثل مكتبات إزالة الارتداد (Debouncing) الأخرى. إذا كنت بحاجة إلى السماح بتنفيذ عمليات على فترة زمنية أكثر انتشارًا، ففكر في استخدام تقنية [التحكم في التدفق (Throttling)](../guides/throttling) بدلاً من ذلك.

### التمكين/التعطيل

تدعم فئة `Debouncer` التمكين/التعطيل عبر خيار `enabled`. باستخدام طريقة `setOptions`، يمكنك تمكين/تعطيل مزيل الارتداد (Debouncer) في أي وقت:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // التعطيل افتراضيًا
debouncer.setOptions({ enabled: true }) // التمكين في أي وقت
```

إذا كنت تستخدم محول إطار عمل حيث تكون خيارات مزيل الارتداد (Debouncer) تفاعلية، يمكنك تعيين خيار `enabled` إلى قيمة شرطية لتمكين/تعطيل مزيل الارتداد (Debouncer) على الفور:

```ts
// مثال React
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // تمكين/تعطيل بناءً على طول الإدخال IF باستخدام محول إطار عمل يدعم الخيارات التفاعلية
)
```

ومع ذلك، إذا كنت تستخدم الدالة `debounce` أو فئة `Debouncer` مباشرة، فيجب عليك استخدام طريقة `setOptions` لتغيير خيار `enabled`، حيث أن الخيارات التي يتم تمريرها يتم تمريرها بالفعل إلى مُنشئ فئة `Debouncer`.

```ts
// مثال Solid
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // التعطيل افتراضيًا
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // تمكين/تعطيل بناءً على طول الإدخال
})
```

### خيارات ردود النداء

يدعم كل من مزيل الارتداد (Debouncer) المتزامن وغير المتزامن خيارات ردود النداء للتعامل مع جوانب مختلفة من دورة حياة إزالة الارتداد (Debouncing):

#### ردود نداء مزيل الارتداد (Debouncer) المتزامن

يدعم `Debouncer` المتزامن رد النداء التالي:

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة', debouncer.getExecutionCount())
  }
})
```

يتم استدعاء رد النداء `onExecute` بعد كل تنفيذ ناجح للدالة التي تمت إزالة ارتدادها، مما يجعله مفيدًا لتتبع عمليات التنفيذ، أو تحديث حالة واجهة المستخدم، أو تنفيذ عمليات التنظيف.

#### ردود نداء مزيل الارتداد (Debouncer) غير المتزامن

يحتوي `AsyncDebouncer` على مجموعة مختلفة من ردود النداء مقارنة بالإصدار المتزامن.

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // يتم استدعاؤه بعد كل تنفيذ ناجح
    console.log('تم تنفيذ الدالة غير المتزامنة', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // يتم استدعاؤه بعد كل محاولة تنفيذ
    console.log('تم تسوية الدالة غير المتزامنة', debouncer.getSettledCount())
  },
  onError: (error) => {
    // يتم استدعاؤه إذا ألقت الدالة غير المتزامنة خطأ
    console.error('فشلت الدالة غير المتزامنة:', error)
  }
})
```

يتم استدعاء رد النداء `onSuccess` بعد كل تنفيذ ناجح للدالة التي تمت إزالة ارتدادها، بينما يتم استدعاء `onError` إذا ألقت الدالة غير المتزامنة خطأ. يتم استدعاء `onSettled` بعد كل محاولة تنفيذ، بغض النظر عن النجاح أو الفشل. تعتبر ردود النداء هذه مفيدة بشكل خاص لتتبع عدد عمليات التنفيذ، وتحديث حالة واجهة المستخدم، والتعامل مع الأخطاء، وتنفيذ عمليات التنظيف، وتسجيل مقاييس التنفيذ.

### إزالة الارتداد (Debouncing) غير المتزامنة

يوفر مزيل الارتداد (Debouncer) غير المتزامن طريقة قوية للتعامل مع العمليات غير المتزامنة مع إزالة الارتداد (Debouncing)، حيث يقدم عدة مزايا رئيسية مقارنة بالإصدار المتزامن. بينما يعتبر مزيل الارتداد (Debouncer) المتزامن رائعًا لأحداث واجهة المستخدم والملاحظات الفورية، فإن الإصدار غير المتزامن مصمم خصيصًا للتعامل مع استدعاءات API، وعمليات قاعدة البيانات، والمهام غير المتزامنة الأخرى.

#### الاختلافات الرئيسية عن إزالة الارتداد (Debouncing) المتزامنة

1. **معالجة القيمة المرجعة**
على عكس مزيل الارتداد (Debouncer) المتزامن الذي يُرجع void، يسمح الإصدار غير المتزامن بالتقاط واستخدام القيمة المرجعة من الدالة التي تمت إزالة ارتدادها. هذا مفيد بشكل خاص عندما تحتاج إلى العمل مع نتائج استدعاءات API أو العمليات غير المتزامنة الأخرى. تُرجع طريقة `maybeExecute` وعدًا (Promise) يتم حله بقيمة إرجاع الدالة، مما يسمح لك بالانتظار للنتائج والتعامل معها بشكل مناسب.

2. **ردود نداء مختلفة**
يدعم `AsyncDebouncer` ردود النداء التالية بدلاً من `onExecute` فقط في الإصدار المتزامن:
- `onSuccess`: يتم استدعاؤه بعد كل تنفيذ ناجح، مع توفير مثيل مزيل الارتداد (Debouncer)
- `onSettled`: يتم استدعاؤه بعد كل تنفيذ، مع توفير مثيل مزيل الارتداد (Debouncer)
- `onError`: يتم استدعاؤه إذا ألقت الدالة غير المتزامنة خطأ، مع توفير كل من الخطأ ومثيل مزيل الارتداد (Debouncer)

3. **التنفيذ المتسلسل**
نظرًا لأن طريقة `maybeExecute` لمزيل الارتداد (Debouncer) تُرجع وعدًا (Promise)، يمكنك اختيار انتظار كل تنفيذ قبل بدء التالي. هذا يمنحك التحكم في ترتيب التنفيذ ويضمن معالجة كل استدعاء لأحدث البيانات. هذا مفيد بشكل خاص عند التعامل مع العمليات التي تعتمد على نتائج الاستدعاءات السابقة أو عندما يكون الحفاظ على اتساق البيانات أمرًا بالغ الأهمية.

على سبيل المثال، إذا كنت تقوم بتحديث ملف تعريف المستخدم ثم تقوم على الفور جلب بياناته المحدثة، يمكنك انتظار عملية التحديث قبل بدء عملية الجلب:

#### مثال الاستخدام الأساسي

إليك مثالاً أساسيًا يوضح كيفية استخدام مزيل الارتداد (Debouncer) غير المتزامن لعملية بحث:

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('نجح البحث:', results)
    },
    onError: (error, debouncer) => {
      console.error('فشل البحث:', error)
    }
  }
)

// الاستخدام
const results = await debouncedSearch('query')
```

### محولات الأطر

يوفر كل محول إطار خطافات (hooks) تعتمد على وظائف إزالة الارتداد (Debouncing) الأساسية للاندماج مع نظام إدارة الحالة الخاص بالإطار. تتوفر خطافات مثل `createDebouncer`، و `useDebouncedCallback`، و `useDebouncedState`، أو `useDebouncedValue` لكل إطار.

إليك بعض الأمثلة:

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// خطاف منخفض المستوى للتحكم الكامل
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// خطاف رد نداء بسيط لحالات الاستخدام الأساسية
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// خطاف قائم على الحالة لإدارة الحالة التفاعلية
const [instantState, setInstantState] = useState('')
const [debouncedValue] = useDebouncedValue(
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

// خطاف إشارة (Signal) لإدارة الحالة
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('إجمالي عمليات التنفيذ:', debouncer.getExecutionCount())
  }
})
```
