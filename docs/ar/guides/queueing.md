---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:41:00.569Z'
title: ديل الطابور (Queueing)
id: queueing
---
# دليل الطابور (Queueing Guide)

على عكس [الحد من المعدل (Rate Limiting)](../guides/rate-limiting)، [الخنق (Throttling)](../guides/throttling)، و[إلغاء الاهتزاز (Debouncing)](../guides/debouncing) التي تتجاهل التنفيذات عند حدوثها بكثرة، فإن أدوات الطابور تضمن معالجة كل عملية. فهي توفر طريقة لإدارة والتحكم في تدفق العمليات دون فقدان أي طلبات. هذا يجعلها مثالية للسيناريوهات التي يكون فيها فقدان البيانات غير مقبول. سيغطي هذا الدليل مفاهيم الطابور في TanStack Pacer.

## مفهوم الطابور (Queueing Concept)

يضمن الطابور معالجة كل عملية في النهاية، حتى لو جاءت أسرع من قدرة النظام على التعامل معها. على عكس تقنيات التحكم في التنفيذ الأخرى التي تتجاهل العمليات الزائدة، يقوم الطابور بتخزين العمليات في قائمة مرتبة ومعالجتها وفقًا لقواعد محددة. هذا يجعل الطابور تقنية التحكم في التنفيذ الوحيدة "الخالية من الفقد" في TanStack Pacer.

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

الطابور مهم بشكل خاص عندما تحتاج إلى ضمان معالجة كل عملية، حتى لو أدى ذلك إلى إدخال بعض التأخير. هذا يجعله مثاليًا للسيناريوهات التي تكون فيها اتساق البيانات واكتمالها أكثر أهمية من التنفيذ الفوري.

من حالات الاستخدام الشائعة:
- معالجة تفاعلات المستخدم في واجهة المستخدم حيث يجب تسجيل كل إجراء
- معالجة عمليات قاعدة البيانات التي تحتاج إلى الحفاظ على اتساق البيانات
- إدارة طلبات واجهة برمجة التطبيقات (API) التي يجب أن تكتمل بنجاح
- تنسيق المهام الخلفية التي لا يمكن تجاهلها
- تسلسلات الرسوم المتحركة حيث كل إطار مهم
- إرسال النماذج حيث يجب حفظ كل إدخال

### متى لا تستخدم الطابور (When Not to Use Queueing)

قد لا يكون الطابور الخيار الأفضل عندما:
- يكون التغذية الراجعة الفورية أكثر أهمية من معالجة كل عملية
- تهتم فقط بأحدث قيمة (استخدم [إلغاء الاهتزاز (Debouncing)](../guides/debouncing) بدلاً من ذلك)

> [!TIP]
> إذا كنت تستخدم حاليًا الحد من المعدل أو الخنق أو إلغاء الاهتزاز ولكنك تجد أن العمليات المتجاهلة تسبب مشاكل، فمن المحتمل أن الطابور هو الحل الذي تحتاجه.

## الطابور في TanStack Pacer (Queueing in TanStack Pacer)

يوفر TanStack Pacer الطابور من خلال الدالة البسيطة `queue` والفئة الأكثر قوة `Queuer`. بينما تفضل تقنيات التحكم في التنفيذ الأخرى عادةً واجهات برمجة التطبيقات (API) المستندة إلى الدوال، فإن الطابور يستفيد غالبًا من التحكم الإضافي الذي توفره واجهة برمجة التطبيقات (API) المستندة إلى الفئة.

### الاستخدام الأساسي مع `queue` (Basic Usage with `queue`)

توفر الدالة `queue` طريقة بسيطة لإنشاء طابور يعمل دائمًا ويعالج العناصر عند إضافتها:

```ts
import { queue } from '@tanstack/pacer'

// Create a queue that processes items every second
const processItems = queue<number>({
  wait: 1000,
  onItemsChange: (queuer) => {
    console.log('Current queue:', queuer.getAllItems())
  }
})

// Add items to be processed
processItems(1) // Processed immediately
processItems(2) // Processed after 1 second
processItems(3) // Processed after 2 seconds
```

بينما تكون الدالة `queue` سهلة الاستخدام، فإنها توفر فقط طابورًا أساسيًا يعمل دائمًا من خلال طريقة `addItem`. بالنسبة لمعظم حالات الاستخدام، سترغب في التحكم والميزات الإضافية التي توفرها فئة `Queuer`.

### الاستخدام المتقدم مع فئة `Queuer` (Advanced Usage with `Queuer` Class)

توفر فئة `Queuer` تحكمًا كاملاً في سلوك الطابور ومعالجته:

```ts
import { Queuer } from '@tanstack/pacer'

// Create a queue that processes items every second
const queue = new Queuer<number>({
  wait: 1000, // Wait 1 second between processing items
  onItemsChange: (queuer) => {
    console.log('Current queue:', queuer.getAllItems())
  }
})

// Start processing
queue.start()

// Add items to be processed
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// Items will be processed one at a time with 1 second delay between each
// Output:
// Processing: 1 (immediately)
// Processing: 2 (after 1 second)
// Processing: 3 (after 2 seconds)
```

### أنواع الطوابير والترتيب (Queue Types and Ordering)

ما يجعل Queuer في TanStack Pacer فريدًا هو قدرته على التكيف مع حالات الاستخدام المختلفة من خلال واجهة برمجة التطبيقات (API) المستندة إلى الموضع. يمكن لنفس Queuer أن يتصرف كطابور تقليدي، أو كومة (stack)، أو طابور مزدوج النهاية، كل ذلك من خلال نفس الواجهة المتسقة.

#### طابور FIFO (أول ما يدخل أول ما يخرج) (FIFO Queue (First In, First Out))

السلوك الافتراضي حيث تتم معالجة العناصر بالترتيب الذي تمت إضافتها به. هذا هو نوع الطابور الأكثر شيوعًا ويتبع مبدأ أن العنصر الأول الذي تمت إضافته يجب أن يكون أول عنصر تتم معالجته.

```text
FIFO Queue Visualization:

Entry →  [A][B][C][D] → Exit
         ⬇️         ⬆️
      New items   Items are
      added here  processed here

Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [BC]    [C]    []
Processed:    A       B       C
```

طوابير FIFO مثالية لـ:
- معالجة المهام حيث يهم الترتيب
- طوابير الرسائل حيث يجب معالجة الرسائل بالتسلسل
- طوابير الطباعة حيث يجب طباعة المستندات بالترتيب الذي تم إرسالها به
- أنظمة معالجة الأحداث حيث يجب معالجة الأحداث بالترتيب الزمني

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // default
  getItemsFrom: 'front', // default
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// Processes: 1, then 2
```

#### كومة LIFO (آخر ما يدخل أول ما يخرج) (LIFO Stack (Last In, First Out))

من خلال تحديد 'back' كموضع لكل من إضافة واسترداد العناصر، يتصرف Queuer مثل كومة. في الكومة، يكون العنصر المضاف مؤخرًا هو أول عنصر تتم معالجته.

```text
LIFO Stack Visualization:

     ⬆️ Process
    [D] ← Most recently added
    [C]
    [B]
    [A] ← First added
     ⬇️ Entry

Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [AB]    [A]    []
Processed:    C       B       A
```

سلوك الكومة مفيد بشكل خاص لـ:
- أنظمة التراجع/الإعادة حيث يجب التراجع عن أحدث إجراء أولاً
- التنقل في سجل المتصفح حيث تريد العودة إلى أحدث صفحة
- مكالمات دالة مكدسة في تنفيذات لغات البرمجة
- خوارزميات اجتياز العمق أولاً (Depth-first)

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // default
  getItemsFrom: 'back', // override default for stack behavior
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// Items will process in order: 2, then 1

stack.getNextItem('back') // get next item from back of queue instead of front
```

#### طابور الأولوية (Priority Queue)

تضيف طوابير الأولوية بُعدًا آخر لترتيب الطابور من خلال السماح للعناصر بالترتيب بناءً على أولويتها بدلاً من مجرد ترتيب إدراجها. يتم تعيين قيمة أولوية لكل عنصر، ويحافظ الطابور تلقائيًا على العناصر بترتيب الأولوية.

```text
Priority Queue Visualization:

Entry →  [P:5][P:3][P:2][P:1] → Exit
          ⬇️           ⬆️
     High Priority   Low Priority
     items here      processed last

Timeline: [1 second per tick]
Calls:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
Queue:       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Processed:              5         -          3         2        1
```

طوابير الأولوية ضرورية لـ:
- مجدولات المهام حيث تكون بعض المهام أكثر إلحاحًا من غيرها
- توجيه حزم الشبكة حيث تحتاج أنواع معينة من حركة المرور إلى معاملة تفضيلية
- أنظمة الأحداث حيث يجب معالجة الأحداث ذات الأولوية العالية قبل الأحداث ذات الأولوية المنخفضة
- تخصيص الموارد حيث تكون بعض الطلبات أكثر أهمية من غيرها

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // Higher numbers have priority
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// Processes: 3, 2, then 1
```

### البدء والإيقاف (Starting and Stopping)

تدعم فئة `Queuer` بدء وإيقاف المعالجة من خلال طرق `start()` و`stop()`، ويمكن تكوينها للبدء تلقائيًا باستخدام خيار `started`:

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // Start paused
})

// Control processing
queue.start() // Begin processing items
queue.stop()  // Pause processing

// Check processing state
console.log(queue.getIsRunning()) // Whether the queue is currently processing
console.log(queue.getIsIdle())    // Whether the queue is running but empty
```

إذا كنت تستخدم أداة تكييف إطار عمل حيث تكون خيارات الطابور تفاعلية، يمكنك تعيين خيار `started` إلى قيمة شرطية:

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // Start/stop based on connection status IF using a framework adapter that supports reactive options
  }
)
```

### ميزات إضافية (Additional Features)

يوفر Queuer عدة طرق مفيدة لإدارة الطابور:

```ts
// Queue inspection
queue.getPeek()           // View next item without removing it
queue.getSize()          // Get current queue size
queue.getIsEmpty()       // Check if queue is empty
queue.getIsFull()        // Check if queue has reached maxSize
queue.getAllItems()   // Get copy of all queued items

// Queue manipulation
queue.clear()         // Remove all items
queue.reset()         // Reset to initial state
queue.getExecutionCount() // Get number of processed items

// Event handling
queue.onItemsChange((item) => {
  console.log('Processed:', item)
})
```

### معالجة الرفض (Rejection Handling)

عندما يصل الطابور إلى أقصى حجم له (المحدد بخيار `maxSize`)، سيتم رفض العناصر الجديدة. يوفر Queuer طرقًا للتعامل مع هذه الرفضات ومراقبتها:

```ts
const queue = new Queuer<number>({
  maxSize: 2, // Only allow 2 items in queue
  onReject: (item, queuer) => {
    console.log('Queue is full. Item rejected:', item)
  }
})

queue.addItem(1) // Accepted
queue.addItem(2) // Accepted
queue.addItem(3) // Rejected, triggers onReject callback

console.log(queue.getRejectionCount()) // 1
```

### العناصر الأولية (Initial Items)

يمكنك تعبئة الطابور مسبقًا بعناصر أولية عند إنشائه:

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // Start processing immediately
})

// Queue starts with [1, 2, 3] and begins processing
```

### التكوين الديناميكي (Dynamic Configuration)

يمكن تعديل خيارات Queuer بعد الإنشاء باستخدام `setOptions()` واستردادها باستخدام `getOptions()`:

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// Change configuration
queue.setOptions({
  wait: 500, // Process items twice as fast
  started: true // Start processing
})

// Get current configuration
const options = queue.getOptions()
console.log(options.wait) // 500
```

### مراقبة الأداء (Performance Monitoring)

يوفر Queuer طرقًا لمراقبة أدائه:

```ts
const queue = new Queuer<number>()

// Add and process some items
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // Number of items processed
console.log(queue.getRejectionCount()) // Number of items rejected
```

### الطابور غير المتزامن (Asynchronous Queueing)

للتعامل مع العمليات غير المتزامنة مع عمال متعددين، راجع [دليل الطابور غير المتزامن (Async Queueing Guide)](../guides/async-queueing) الذي يغطي فئة `AsyncQueuer`.

### أدوات تكييف الإطار (Framework Adapters)

تقوم كل أداة تكييف إطار ببناء خطافات ووظائف ملائمة حول فئات الطابور. الخطافات مثل `useQueuer` أو `useQueueState` هي أغلفة صغيرة يمكن أن تقلل من الكود المكرر المطلوب في الكود الخاص بك لبعض حالات الاستخدام الشائعة.
