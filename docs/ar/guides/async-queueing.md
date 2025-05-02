---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:41:07.116Z'
title: ديل الطابور غير المتزامن (Async Queueing)
id: async-queueing
---
# دليل الطابور غير المتزامن (Asynchronous Queueing Guide)

بينما يوفر [Queuer](../guides//queueing) طابورًا متزامنًا مع ضوابط التوقيت، فإن `AsyncQueuer` مصمم خصيصًا للتعامل مع العمليات غير المتزامنة المتزامنة. فهو ينفذ ما يُعرف تقليديًا بنمط "تجمع المهام" أو "تجمع العمال" (task pool or worker pool)، مما يسمح بمعالجة عمليات متعددة في نفس الوقت مع الحفاظ على التحكم في التزامن والتوقيت. يتم نسخ التنفيذ في الغالب من [Swimmer](https://github.com/tannerlinsley/swimmer)، أداة تجميع المهام الأصلية لتانر التي تخدم مجتمع جافا سكريبت منذ عام 2017.

## مفهوم الطابور غير المتزامن (Async Queueing Concept)

يمتد مفهوم الطابور غير المتزامن لمفهوم الطابور الأساسي بإضافة قدرات المعالجة المتزامنة. بدلاً من معالجة عنصر واحد في كل مرة، يمكن لطابور غير متزامن معالجة عناصر متعددة في نفس الوقت مع الحفاظ على الترتيب والتحكم في التنفيذ. هذا مفيد بشكل خاص عند التعامل مع عمليات الإدخال/الإخراج (I/O)، أو طلبات الشبكة، أو أي مهام تقضي معظم وقتها في الانتظار بدلاً من استهلاك وحدة المعالجة المركزية.

### تصور الطابور غير المتزامن (Async Queueing Visualization)

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ Unlike regular queueing, multiple items
               can be processed concurrently

             [Items queue up]   [Process 2 at once]   [Complete]
              when busy         with wait between      all items
```

### متى تستخدم الطابور غير المتزامن (When to Use Async Queueing)

يكون الطابور غير المتزامن فعالاً بشكل خاص عندما تحتاج إلى:
- معالجة عمليات غير متزامنة متعددة بشكل متزامن
- التحكم في عدد العمليات المتزامنة
- التعامل مع مهام تعتمد على الوعود (Promise-based tasks) مع معالجة الأخطاء المناسبة
- الحفاظ على الترتيب مع تعظيم الإنتاجية
- معالجة مهام الخلفية التي يمكن تشغيلها بالتوازي

تشمل حالات الاستخدام الشائعة:
- تقديم طلبات API متزامنة مع تحديد معدل (rate limiting)
- معالجة تحميل ملفات متعددة في نفس الوقت
- تشغيل عمليات قاعدة بيانات متوازية
- التعامل مع اتصالات متعددة لبروتوكول Websocket
- معالجة تدفقات البيانات مع ضغط عكسي (backpressure)
- إدارة مهام الخلفية التي تستهلك الكثير من الموارد

### متى لا تستخدم الطابور غير المتزامن (When Not to Use Async Queueing)

إن `AsyncQueuer` متعدد الاستخدامات ويمكن استخدامه في العديد من المواقف. في الحقيقة، إنه ليس مناسبًا فقط عندما لا تخطط للاستفادة من جميع ميزاته. إذا كنت لا تحتاج إلى تمرير جميع عمليات التنفيذ التي يتم وضعها في الطابور، استخدم [الخنق (Throttling)][../guides/throttling] بدلاً من ذلك. إذا كنت لا تحتاج إلى معالجة متزامنة، استخدم [الطابور (Queueing)][../guides/queueing] بدلاً من ذلك.

## الطابور غير المتزامن في TanStack Pacer

يوفر TanStack Pacer الطابور غير المتزامن من خلال الدالة البسيطة `asyncQueue` والفئة الأكثر قوة `AsyncQueuer`.

### الاستخدام الأساسي مع `asyncQueue`

توفر الدالة `asyncQueue` طريقة بسيطة لإنشاء طابور غير متزامن يعمل دائمًا:

```ts
import { asyncQueue } from '@tanstack/pacer'

// إنشاء طابور يعالج حتى عنصرين في نفس الوقت
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('المهام النشطة:', queuer.getActiveItems().length)
  }
})

// إضافة مهام غير متزامنة للمعالجة
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

استخدام الدالة `asyncQueue` محدود بعض الشيء، حيث أنها مجرد غلاف حول فئة `AsyncQueuer` الذي يعرض فقط طريقة `addItem`. لمزيد من التحكم في الطابور، استخدم فئة `AsyncQueuer` مباشرة.

### الاستخدام المتقدم مع فئة `AsyncQueuer`

توفر فئة `AsyncQueuer` تحكمًا كاملاً في سلوك الطابور غير المتزامن:

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // معالجة عنصرين في نفس الوقت
  wait: 1000,     // الانتظار ثانية واحدة قبل بدء عناصر جديدة
  started: true   // بدء المعالجة فورًا
})

// إضافة معالجات للأخطاء والنجاح
queue.onError((error) => {
  console.error('فشلت المهمة:', error)
})

queue.onSuccess((result) => {
  console.log('تم إكمال المهمة:', result)
})

// إضافة مهام غير متزامنة
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### أنواع الطوابير والترتيب (Queue Types and Ordering)

يدعم `AsyncQueuer` استراتيجيات مختلفة للطوابير للتعامل مع متطلبات المعالجة المتنوعة. تحدد كل استراتيجية كيفية إضافة المهام ومعالجتها من الطابور.

#### طابور FIFO (أول داخل أول خارج) (FIFO Queue)

تعالج طوابير FIFO المهام بنفس الترتيب الذي تمت إضافتها به، مما يجعلها مثالية للحفاظ على التسلسل:

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // افتراضي
  getItemsFrom: 'front', // افتراضي
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// المعالجة: first و second بشكل متزامن
```

#### مكدس LIFO (آخر داخل أول خارج) (LIFO Stack)

تعالج أكوام LIFO أحدث المهام المضافة أولاً، وهو مفيد لإعطاء الأولوية للمهام الأحدث:

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // معالجة أحدث العناصر أولاً
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// المعالجة: second أولاً، ثم first
```

#### طابور الأولوية (Priority Queue)

تعالج طوابير الأولوية المهام بناءً على قيم الأولوية المخصصة لها، مما يضمن معالجة المهام المهمة أولاً. هناك طريقتان لتحديد الأولويات:

1. قيم أولوية ثابتة مرفقة بالمهام:
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// إنشاء مهام بقيم أولوية ثابتة
const lowPriorityTask = Object.assign(
  async () => 'نتيجة أولوية منخفضة',
  { priority: 1 }
)

const highPriorityTask = Object.assign(
  async () => 'نتيجة أولوية عالية',
  { priority: 3 }
)

const mediumPriorityTask = Object.assign(
  async () => 'نتيجة أولوية متوسطة',
  { priority: 2 }
)

// إضافة المهام بأي ترتيب - سيتم معالجتها حسب الأولوية (الأرقام الأعلى أولاً)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// المعالجة: high و medium بشكل متزامن، ثم low
```

2. حساب الأولوية الديناميكي باستخدام خيار `getPriority`:
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // حساب الأولوية بناءً على خصائص المهمة أو عوامل أخرى
    // الأرقام الأعلى لها أولوية أعلى
    return calculateTaskPriority(task)
  }
})

// إضافة مهام - سيتم حساب الأولوية ديناميكيًا
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

طوابير الأولوية ضرورية عندما:
- يكون للمهام مستويات أهمية مختلفة
- تحتاج العمليات الحرجة إلى التشغيل أولاً
- تحتاج إلى ترتيب مرن للمهام بناءً على الأولوية
- يجب تخصيص الموارد لصالح المهام المهمة
- يجب تحديد الأولوية ديناميكيًا بناءً على خصائص المهمة أو العوامل الخارجية

### معالجة الأخطاء (Error Handling)

يوفر `AsyncQueuer` قدرات شاملة لمعالجة الأخطاء لضمان معالجة قوية للمهام. يمكنك التعامل مع الأخطاء على مستوى الطابور ومستوى المهمة الفردية:

```ts
const queue = new AsyncQueuer<string>()

// التعامل مع الأخطاء عالميًا
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('فشلت المهمة:', error)
  },
  onSuccess: (result) => {
    console.log('نجحت المهمة:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('فشلت المهمة:', result)
    } else {
      console.log('نجحت المهمة:', result)
    }
  }
})

// التعامل مع الأخطاء لكل مهمة
queue.addItem(async () => {
  throw new Error('فشلت المهمة')
}).catch(error => {
  console.error('خطأ المهمة الفردية:', error)
})
```

### إدارة الطابور (Queue Management)

يوفر `AsyncQueuer` عدة طرق لمراقبة وحالة الطابور والتحكم فيها:

```ts
// فحص الطابور
queue.getPeek()           // عرض العنصر التالي دون إزالته
queue.getSize()          // الحصول على حجم الطابور الحالي
queue.getIsEmpty()       // التحقق مما إذا كان الطابور فارغًا
queue.getIsFull()        // التحقق مما إذا كان الطابور قد وصل إلى الحد الأقصى للحجم
queue.getAllItems()   // الحصول على نسخة من جميع العناصر في الطابور
queue.getActiveItems() // الحصول على العناصر قيد المعالجة حاليًا
queue.getPendingItems() // الحصول على العناصر المنتظرة للمعالجة

// معالجة الطابور
queue.clear()         // إزالة جميع العناصر
queue.reset()         // إعادة التعيين إلى الحالة الأولية
queue.getExecutionCount() // الحصول على عدد العناصر المعالجة

// التحكم في المعالجة
queue.start()         // بدء معالجة العناصر
queue.stop()          // إيقاف المعالجة مؤقتًا
queue.getIsRunning()     // التحقق مما إذا كان الطابور يعالج
queue.getIsIdle()        // التحقق مما إذا كان الطابور فارغًا ولا يعالج
```

### ردود النداء للمهام (Task Callbacks)

يوفر `AsyncQueuer` ثلاثة أنواع من ردود النداء لمراقبة تنفيذ المهام:

```ts
const queue = new AsyncQueuer<string>()

// التعامل مع إكمال المهمة بنجاح
const unsubSuccess = queue.onSuccess((result) => {
  console.log('نجحت المهمة:', result)
})

// التعامل مع أخطاء المهام
const unsubError = queue.onError((error) => {
  console.error('فشلت المهمة:', error)
})

// التعامل مع إكمال المهمة بغض النظر عن النجاح/الفشل
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('فشلت المهمة:', result)
  } else {
    console.log('نجحت المهمة:', result)
  }
})

// إلغاء الاشتراك من ردود النداء عند عدم الحاجة إليها
unsubSuccess()
unsubError()
unsubSettled()
```

### التعامل مع الرفض (Rejection Handling)

عندما يصل الطابور إلى الحد الأقصى لحجمه (المحدد بخيار `maxSize`)، سيتم رفض المهام الجديدة. يوفر `AsyncQueuer` طرقًا للتعامل مع هذه الرفضات ومراقبتها:

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // السماح فقط بمهمتين في الطابور
  onReject: (task, queuer) => {
    console.log('الطابور ممتلئ. تم رفض المهمة:', task)
  }
})

queue.addItem(async () => 'first') // مقبول
queue.addItem(async () => 'second') // مقبول
queue.addItem(async () => 'third') // مرفوض، يشغل رد النداء onReject

console.log(queue.getRejectionCount()) // 1
```

### المهام الأولية (Initial Tasks)

يمكنك تعبئة طابور غير متزامن مسبقًا بمهام أولية عند إنشائه:

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // بدء المعالجة فورًا
})

// يبدأ الطابور بثلاث مهام ويبدأ في معالجتها
```

### التكوين الديناميكي (Dynamic Configuration)

يمكن تعديل خيارات `AsyncQueuer` بعد الإنشاء باستخدام `setOptions()` واسترجاعها باستخدام `getOptions()`:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// تغيير التكوين
queue.setOptions({
  concurrency: 4, // معالجة مهام أكثر في نفس الوقت
  started: true // بدء المعالجة
})

// الحصول على التكوين الحالي
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### المهام النشطة والمنتظرة (Active and Pending Tasks)

يوفر `AsyncQueuer` طرقًا لمراقبة المهام النشطة والمنتظرة:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// إضافة بعض المهام
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// مراقبة حالات المهام
console.log(queue.getActiveItems().length) // المهام قيد المعالجة حاليًا
console.log(queue.getPendingItems().length) // المهام المنتظرة للمعالجة
```

### محولات الأطر (Framework Adapters)

يبني كل محول إطار وظائف وخطافات ملائمة حول فئات الطابور غير المتزامن. توفر خطافات مثل `useAsyncQueuer` أو `useAsyncQueuerState` أغلفة صغيرة يمكنها تقليل الكود المكرر المطلوب في الكود الخاص بك لبعض حالات الاستخدام الشائعة.
