---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:22:38.032Z'
title: ديل الطابور (Queueing)
id: queueing
---
# دليل الطابور (Queueing Guide)

على عكس [الحد من المعدل (Rate Limiting)](../guides/rate-limiting)، [التحكم في التدفق (Throttling)](../guides/throttling)، و[إلغاء الاهتزاز (Debouncing)](../guides/debouncing) التي تتجاهل التنفيذات عند حدوثها بشكل متكرر، يمكن تكوين الطوابير (queuers) لضمان معالجة كل عملية. فهي توفر طريقة لإدارة والتحكم في تدفق العمليات دون فقدان أي طلبات. وهذا يجعلها مثالية للسيناريوهات التي يكون فيها فقدان البيانات غير مقبول. يمكن أيضًا ضبط الطابور ليكون له حجم أقصى، مما يمكن أن يكون مفيدًا لمنع تسرب الذاكرة أو مشاكل أخرى. سيغطي هذا الدليل مفاهيم الطابور في TanStack Pacer.

## مفهوم الطابور (Queueing Concept)

يضمن الطابور معالجة كل عملية في النهاية، حتى إذا جاءت بشكل أسرع مما يمكن معالجته. على عكس تقنيات التحكم في التنفيذ الأخرى التي تتجاهل العمليات الزائدة، يقوم الطابور بتخزين العمليات في قائمة مرتبة ويعالجها وفقًا لقواعد محددة. هذا يجعل الطابور تقنية التحكم في التنفيذ الوحيدة "الخالية من الفقد" في TanStack Pacer، ما لم يتم تحديد `maxSize` مما قد يتسبب في رفض العناصر عندما يكون المخزن المؤقت ممتلئًا.

### تصور الطابور (Queueing Visualization)

```text
Queueing (processing one item every 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
Queue:       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
Executed:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ Unlike rate limiting/throttling/debouncing,
               ALL calls are eventually processed in order

             [Items queue up]   [Process steadily]   [Empty]
              when busy          one by one           queue
```

### متى تستخدم الطابور (When to Use Queueing)

الطابور مهم بشكل خاص عندما تحتاج إلى ضمان معالجة كل عملية، حتى لو كان ذلك يعني إدخال بعض التأخير. هذا يجعله مثاليًا للسيناريوهات التي تكون فيها اتساق البيانات واكتمالها أكثر أهمية من التنفيذ الفوري. عند استخدام `maxSize`، يمكن أن يعمل أيضًا كمخزن مؤقت لمنع إرباك النظام بالعديد من العمليات المعلقة.

من حالات الاستخدام الشائعة:
- جلب البيانات مسبقًا قبل الحاجة إليها دون إرهاق النظام
- معالجة تفاعلات المستخدم في واجهة المستخدم حيث يجب تسجيل كل إجراء
- التعامل مع عمليات قاعدة البيانات التي تحتاج إلى الحفاظ على اتساق البيانات
- إدارة طلبات واجهة برمجة التطبيقات (API) التي يجب أن تكتمل جميعها بنجاح
- تنسيق المهام الخلفية التي لا يمكن إسقاطها
- تسلسلات الرسوم المتحركة حيث كل إطار مهم
- إرسال النماذج حيث يجب حفظ كل إدخال
- تخزين تدفقات البيانات بسعة ثابتة باستخدام `maxSize`

### متى لا تستخدم الطابور (When Not to Use Queueing)

قد لا يكون الطابور الخيار الأفضل عندما:
- يكون التغذية الراجعة الفورية أكثر أهمية من معالجة كل عملية
- تهتم فقط بالقيمة الأحدث (استخدم [إلغاء الاهتزاز (debouncing)](../guides/debouncing) بدلاً من ذلك)

> [!TIP]
> إذا كنت تستخدم حاليًا الحد من المعدل، التحكم في التدفق، أو إلغاء الاهتزاز ولكنك تجد أن العمليات المتجاهلة تسبب مشاكل، فمن المرجح أن الطابور هو الحل الذي تحتاجه.

## الطابور في TanStack Pacer (Queueing in TanStack Pacer)

يوفر TanStack Pacer الطابور من خلال الدالة البسيطة `queue` والفئة الأكثر قوة `Queuer`. بينما تفضل تقنيات التحكم في التنفيذ الأخرى عادةً واجهات برمجة التطبيقات (APIs) القائمة على الدوال، فإن الطابور يستفيد غالبًا من التحكم الإضافي الذي توفره واجهة برمجة التطبيقات القائمة على الفئة.

### الاستخدام الأساسي مع `queue`

توفر الدالة `queue` طريقة بسيطة لإنشاء طابور يعمل دائمًا ويعالج العناصر عند إضافتها:

```ts
import { queue } from '@tanstack/pacer'

// إنشاء طابور يعالج العناصر كل ثانية
const processItems = queue<number>({
  wait: 1000,
  maxSize: 10, // اختياري: تحديد حجم الطابور لمنع مشاكل الذاكرة أو الوقت
  onItemsChange: (queuer) => {
    console.log('Current queue:', queuer.getAllItems())
  }
})

// إضافة العناصر للمعالجة
processItems(1) // يتم معالجته فورًا
processItems(2) // يتم معالجته بعد ثانية واحدة
processItems(3) // يتم معالجته بعد ثانيتين
```

بينما تكون الدالة `queue` سهلة الاستخدام، فإنها توفر فقط طابورًا أساسيًا يعمل دائمًا من خلال طريقة `addItem`. بالنسبة لمعظم حالات الاستخدام، سترغب في التحكم والميزات الإضافية التي توفرها فئة `Queuer`.

### الاستخدام المتقدم مع فئة `Queuer`

توفر فئة `Queuer` تحكمًا كاملاً في سلوك الطابور ومعالجته:

```ts
import { Queuer } from '@tanstack/pacer'

// إنشاء طابور يعالج العناصر كل ثانية
const queue = new Queuer<number>({
  wait: 1000, // الانتظار ثانية واحدة بين معالجة العناصر
  maxSize: 5, // اختياري: تحديد حجم الطابور لمنع مشاكل الذاكرة أو الوقت
  onItemsChange: (queuer) => {
    console.log('Current queue:', queuer.getAllItems())
  }
})

// بدء المعالجة
queue.start()

// إضافة العناصر للمعالجة
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// سيتم معالجة العناصر واحدًا تلو الآخر مع تأخير ثانية واحدة بين كل منها
// الإخراج:
// Processing: 1 (فورًا)
// Processing: 2 (بعد ثانية واحدة)
// Processing: 3 (بعد ثانيتين)
```

### أنواع الطوابير والترتيب (Queue Types and Ordering)

ما يجعل `Queuer` في TanStack Pacer فريدًا هو قدرته على التكيف مع حالات الاستخدام المختلفة من خلال واجهة برمجة التطبيقات القائمة على الموضع. يمكن لنفس `Queuer` أن يتصرف كطابور تقليدي، أو مكدس (stack)، أو طابور مزدوج النهاية، كل ذلك من خلال نفس الواجهة المتسقة.

#### طابور FIFO (أول ما يدخل أول ما يخرج) (FIFO Queue)

السلوك الافتراضي حيث يتم معالجة العناصر بالترتيب الذي تمت إضافتها به. هذا هو نوع الطابور الأكثر شيوعًا ويتبع مبدأ أن العنصر الأول الذي تمت إضافته يجب أن يكون أول عنصر يتم معالجته. عند استخدام `maxSize`، سيتم رفض العناصر الجديدة إذا كان الطابور ممتلئًا.

```text
تصور طابور FIFO (مع maxSize=3):

الدخول →  [A][B][C] → الخروج
         ⬇️     ⬆️
      يتم إضافة   يتم معالجة
      عناصر جديدة هنا العناصر هنا
      (يتم رفضها إذا كان ممتلئًا)

Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
Queue:       [ABC]   [BC]    [C]    []
Processed:    A       B       C
Rejected:     D      E
```

طوابير FIFO مثالية لـ:
- معالجة المهام حيث يهم الترتيب
- طوابير الرسائل حيث يجب معالجة الرسائل بالتسلسل
- طوابير الطباعة حيث يجب طباعة المستندات بالترتيب الذي تم إرسالها به
- أنظمة معالجة الأحداث حيث يجب معالجة الأحداث بالترتيب الزمني

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // الافتراضي
  getItemsFrom: 'front', // الافتراضي
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// يتم المعالجة: 1، ثم 2
```

#### مكدس LIFO (آخر ما يدخل أول ما يخرج) (LIFO Stack)

عن طريق تحديد 'back' كموضع لكل من الإضافة واسترجاع العناصر، يتصرف `Queuer` مثل المكدس. في المكدس، يكون العنصر المضاف مؤخرًا هو أول عنصر يتم معالجته. عند استخدام `maxSize`، سيتم رفض العناصر الجديدة إذا كان المكدس ممتلئًا.

```text
تصور مكدس LIFO (مع maxSize=3):

     ⬆️ المعالجة
    [C] ← آخر عنصر تمت إضافته
    [B]
    [A] ← أول عنصر تمت إضافته
     ⬇️ الدخول
     (يتم رفضها إذا كان ممتلئًا)

Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
Queue:       [ABC]   [AB]    [A]    []
Processed:    C       B       A
Rejected:     D      E
```

سلوك المكدس مفيد بشكل خاص لـ:
- أنظمة التراجع/الإعادة حيث يجب التراجع عن أحدث إجراء أولاً
- التنقل في سجل المتصفح حيث تريد العودة إلى أحدث صفحة
- مكدسات استدعاء الدوال في تنفيذات لغات البرمجة
- خوارزميات اجتياز العمق أولاً (Depth-first)

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // الافتراضي
  getItemsFrom: 'back', // تجاوز الافتراضي لسلوك المكدس
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// سيتم معالجة العناصر بالترتيب: 2، ثم 1

stack.getNextItem('back') // الحصول على العنصر التالي من نهاية الطابور بدلاً من المقدمة
```

#### طابور الأولوية (Priority Queue)

تضيف طوابير الأولوية بُعدًا آخر لترتيب الطابور من خلال السماح بفرز العناصر بناءً على أولويتها بدلاً من مجرد ترتيب إدراجها. يتم تعيين قيمة أولوية لكل عنصر، ويحافظ الطابور تلقائيًا على العناصر بترتيب الأولوية. عند استخدام `maxSize`، قد يتم رفض العناصر ذات الأولوية المنخفضة إذا كان الطابور ممتلئًا.

```text
تصور طابور الأولوية (مع maxSize=3):

الدخول →  [P:5][P:3][P:2] → الخروج
          ⬇️           ⬆️
     عناصر ذات أولوية   معالجة
     عالية هنا         العناصر ذات الأولوية المنخفضة آخراً
     (يتم رفضها إذا كان ممتلئًا)

Timeline: [1 second per tick]
Calls:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
Queue:       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Processed:              5         -          3         2        1
Rejected:                         4
```

طوابير الأولوية ضرورية لـ:
- مجدولات المهام حيث تكون بعض المهام أكثر إلحاحًا من غيرها
- توجيه حزم الشبكة حيث تحتاج أنواع معينة من حركة المرور إلى معاملة تفضيلية
- أنظمة الأحداث حيث يجب معالجة الأحداث ذات الأولوية العالية قبل الأحداث ذات الأولوية المنخفضة
- تخصيص الموارد حيث تكون بعض الطلبات أكثر أهمية من غيرها

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // الأرقام الأعلى لها أولوية أعلى
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// يتم المعالجة: 3، 2، ثم 1
```

### البدء والإيقاف (Starting and Stopping)

تدعم فئة `Queuer` بدء وإيقاف المعالجة من خلال طرق `start()` و `stop()`، ويمكن تكوينها للبدء تلقائيًا باستخدام خيار `started`:

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // البدء متوقفًا
})

// التحكم في المعالجة
queue.start() // بدء معالجة العناصر
queue.stop()  // إيقاف المعالجة مؤقتًا

// التحقق من حالة المعالجة
console.log(queue.getIsRunning()) // ما إذا كان الطابور يعالج حاليًا
console.log(queue.getIsIdle())    // ما إذا كان الطابور يعمل ولكنه فارغ
```

إذا كنت تستخدم أداة تكييف إطار عمل حيث تكون خيارات `Queuer` تفاعلية، يمكنك تعيين خيار `started` إلى قيمة شرطية:

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // البدء/الإيقاف بناءً على حالة الاتصال إذا كنت تستخدم أداة تكييف إطار عمل تدعم الخيارات التفاعلية
  }
)
```

### ميزات إضافية (Additional Features)

يوفر `Queuer` عدة طرق مفيدة لإدارة الطابور:

```ts
// فحص الطابور
queue.getPeek()           // عرض العنصر التالي دون إزالته
queue.getSize()          // الحصول على حجم الطابور الحالي
queue.getIsEmpty()       // التحقق مما إذا كان الطابور فارغًا
queue.getIsFull()        // التحقق مما إذا كان الطابور قد وصل إلى maxSize
queue.getAllItems()   // الحصول على نسخة من جميع العناصر في الطابور

// معالجة الطابور
queue.clear()         // إزالة جميع العناصر
queue.reset()         // إعادة التعيين إلى الحالة الأولية
queue.getExecutionCount() // الحصول على عدد العناصر المعالجة

// معالجة الأحداث
queue.onItemsChange((item) => {
  console.log('Processed:', item)
})
```

### انتهاء صلاحية العنصر (Item Expiration)

يدعم `Queuer` انتهاء صلاحية العناصر التي بقيت في الطابور لفترة طويلة جدًا. هذا مفيد لمنع معالجة البيانات القديمة أو لتنفيذ مهلات زمنية على العمليات في الطابور.

```ts
const queue = new Queuer<number>({
  expirationDuration: 5000, // تنتهي صلاحية العناصر بعد 5 ثوانٍ
  onExpire: (item, queuer) => {
    console.log('Item expired:', item)
  }
})

// أو استخدام فحص انتهاء صلاحية مخصص
const queue = new Queuer<number>({
  getIsExpired: (item, addedAt) => {
    // منطق انتهاء الصلاحية المخصص
    return Date.now() - addedAt > 5000
  },
  onExpire: (item, queuer) => {
    console.log('Item expired:', item)
  }
})

// التحقق من إحصائيات انتهاء الصلاحية
console.log(queue.getExpirationCount()) // عدد العناصر التي انتهت صلاحيتها
```

تعد ميزات انتهاء الصلاحية مفيدة بشكل خاص لـ:
- منع معالجة البيانات القديمة
- تنفيذ مهلات زمنية على العمليات في الطابور
- إدارة استخدام الذاكرة عن طريق إزالة العناصر القديمة تلقائيًا
- التعامل مع البيانات المؤقتة التي يجب أن تكون صالحة فقط لفترة محدودة

### معالجة الرفض (Rejection Handling)

عندما يصل الطابور إلى حجمه الأقصى (المحدد بخيار `maxSize`)، سيتم رفض العناصر الجديدة. يوفر `Queuer` طرقًا للتعامل مع هذه الرفضات ومراقبتها:

```ts
const queue = new Queuer<number>({
  maxSize: 2, // السماح بعنصرين فقط في الطابور
  onReject: (item, queuer) => {
    console.log('Queue is full. Item rejected:', item)
  }
})

queue.addItem(1) //
