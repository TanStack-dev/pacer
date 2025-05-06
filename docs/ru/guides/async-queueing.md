---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:19:37.127Z'
title: Руководство по асинхронной очереди
id: async-queueing
---
# Руководство по асинхронной очереди (Asynchronous Queueing Guide)

В то время как [Queuer](../guides//queueing) предоставляет синхронную очередь с контролем времени выполнения, `AsyncQueuer` разработан специально для обработки параллельных асинхронных операций. Он реализует традиционно известный шаблон "пула задач" (task pool) или "пула воркеров" (worker pool), позволяя обрабатывать несколько операций одновременно, сохраняя контроль над параллелизмом и временем выполнения. Реализация в основном скопирована из [Swimmer](https://github.com/tannerlinsley/swimmer) — оригинальной утилиты Таннера для пулинга задач, которая служит сообществу JavaScript с 2017 года.

## Концепция асинхронной очереди (Async Queueing Concept)

Асинхронная очередь расширяет базовую концепцию очереди, добавляя возможности параллельной обработки. Вместо обработки одного элемента за раз, асинхронная очередь может обрабатывать несколько элементов одновременно, сохраняя порядок и контроль над выполнением. Это особенно полезно при работе с операциями ввода-вывода, сетевыми запросами или любыми задачами, которые большую часть времени ожидают, а не используют процессор.

### Визуализация асинхронной очереди (Async Queueing Visualization)

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ В отличие от обычной очереди, несколько элементов
               могут обрабатываться параллельно

             [Элементы становятся в очередь]   [Обрабатываются 2 одновременно]   [Завершаются]
              когда занято                     с ожиданием между ними            все элементы
```

### Когда использовать асинхронную очередь (When to Use Async Queueing)

Асинхронная очередь особенно эффективна, когда вам нужно:
- Обрабатывать несколько асинхронных операций параллельно
- Контролировать количество одновременных операций
- Работать с Promise-задачами с правильной обработкой ошибок
- Сохранять порядок при максимальной пропускной способности
- Обрабатывать фоновые задачи, которые могут выполняться параллельно

Распространенные сценарии использования:
- Параллельные API-запросы с ограничением скорости (rate limiting)
- Одновременная обработка нескольких загрузок файлов
- Параллельные операции с базой данных
- Управление несколькими websocket-соединениями
- Обработка потоков данных с противодавлением (backpressure)
- Управление ресурсоемкими фоновыми задачами

### Когда не использовать асинхронную очередь (When Not to Use Async Queueing)

`AsyncQueuer` очень универсален и может использоваться во многих ситуациях. По сути, он не подходит только тогда, когда вы не планируете использовать все его возможности. Если вам не нужно, чтобы все поставленные в очередь выполнения проходили, используйте [Throttling][../guides/throttling]. Если вам не нужна параллельная обработка, используйте [Queueing][../guides/queueing].

## Асинхронная очередь в TanStack Pacer (Async Queueing in TanStack Pacer)

TanStack Pacer предоставляет асинхронную очередь через простую функцию `asyncQueue` и более мощный класс `AsyncQueuer`.

### Базовое использование с `asyncQueue` (Basic Usage with `asyncQueue`)

Функция `asyncQueue` предоставляет простой способ создания постоянно работающей асинхронной очереди:

```ts
import { asyncQueue } from '@tanstack/pacer'

// Создаем очередь, обрабатывающую до 2 элементов параллельно
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('Active tasks:', queuer.getActiveItems().length)
  }
})

// Добавляем асинхронные задачи для обработки
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

Использование функции `asyncQueue` несколько ограничено, так как это всего лишь обертка вокруг класса `AsyncQueuer`, предоставляющая только метод `addItem`. Для большего контроля над очередью используйте класс `AsyncQueuer` напрямую.

### Продвинутое использование с классом `AsyncQueuer` (Advanced Usage with `AsyncQueuer` Class)

Класс `AsyncQueuer` предоставляет полный контроль над поведением асинхронной очереди:

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // Обрабатывать 2 элемента одновременно
  wait: 1000,     // Ждать 1 секунду между запуском новых элементов
  started: true   // Начать обработку немедленно
})

// Добавляем обработчики ошибок и успешного выполнения
queue.onError((error) => {
  console.error('Task failed:', error)
})

queue.onSuccess((result) => {
  console.log('Task completed:', result)
})

// Добавляем асинхронные задачи
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### Типы очередей и порядок обработки (Queue Types and Ordering)

`AsyncQueuer` поддерживает различные стратегии обработки очереди для удовлетворения различных требований. Каждая стратегия определяет, как задачи добавляются и обрабатываются из очереди.

#### Очередь FIFO (First In, First Out)

Очереди FIFO обрабатывают задачи в точном порядке их добавления, что идеально подходит для сохранения последовательности:

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // по умолчанию
  getItemsFrom: 'front', // по умолчанию
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// Обрабатывает: first и second параллельно
```

#### Стек LIFO (Last In, First Out)

Стеки LIFO обрабатывают самые последние добавленные задачи первыми, что полезно для приоритизации новых задач:

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // Обрабатывать новые элементы первыми
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// Обрабатывает: second сначала, затем first
```

#### Приоритетная очередь (Priority Queue)

Приоритетные очереди обрабатывают задачи на основе их приоритетов, гарантируя, что важные задачи выполняются первыми. Есть два способа указать приоритеты:

1. Статические значения приоритета, прикрепленные к задачам:
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// Создаем задачи со статическими приоритетами
const lowPriorityTask = Object.assign(
  async () => 'low priority result',
  { priority: 1 }
)

const highPriorityTask = Object.assign(
  async () => 'high priority result',
  { priority: 3 }
)

const mediumPriorityTask = Object.assign(
  async () => 'medium priority result',
  { priority: 2 }
)

// Добавляем задачи в любом порядке - они будут обработаны по приоритету (большие числа первыми)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// Обрабатывает: high и medium параллельно, затем low
```

2. Динамический расчет приоритета с использованием опции `getPriority`:
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // Рассчитываем приоритет на основе свойств задачи или других факторов
    // Большие числа имеют приоритет
    return calculateTaskPriority(task)
  }
})

// Добавляем задачи - приоритет будет рассчитан динамически
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

Приоритетные очереди необходимы, когда:
- Задачи имеют разный уровень важности
- Критические операции должны выполняться первыми
- Нужна гибкая очередность задач на основе приоритета
- Распределение ресурсов должно учитывать важные задачи
- Приоритет должен определяться динамически на основе свойств задачи или внешних факторов

### Обработка ошибок (Error Handling)

`AsyncQueuer` предоставляет комплексные возможности обработки ошибок для надежной обработки задач. Вы можете обрабатывать ошибки как на уровне очереди, так и на уровне отдельных задач:

```ts
const queue = new AsyncQueuer<string>()

// Глобальная обработка ошибок
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('Task failed:', error)
  },
  onSuccess: (result) => {
    console.log('Task succeeded:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('Task failed:', result)
    } else {
      console.log('Task succeeded:', result)
    }
  }
})

// Обработка ошибок для отдельных задач
queue.addItem(async () => {
  throw new Error('Task failed')
}).catch(error => {
  console.error('Individual task error:', error)
})
```

### Управление очередью (Queue Management)

`AsyncQueuer` предоставляет несколько методов для мониторинга и управления состоянием очереди:

```ts
// Инспекция очереди
queue.getPeek()           // Просмотр следующего элемента без удаления
queue.getSize()          // Получение текущего размера очереди
queue.getIsEmpty()       // Проверка, пуста ли очередь
queue.getIsFull()        // Проверка, достигла ли очередь maxSize
queue.getAllItems()   // Получение копии всех элементов в очереди
queue.getActiveItems() // Получение текущих обрабатываемых элементов
queue.getPendingItems() // Получение элементов, ожидающих обработки

// Манипуляции с очередью
queue.clear()         // Удаление всех элементов
queue.reset()         // Сброс в начальное состояние
queue.getExecutionCount() // Получение количества обработанных элементов

// Управление обработкой
queue.start()         // Начать обработку элементов
queue.stop()          // Приостановить обработку
queue.getIsRunning()     // Проверка, обрабатывается ли очередь
queue.getIsIdle()        // Проверка, пуста ли очередь и не обрабатывается
```

### Колбэки задач (Task Callbacks)

`AsyncQueuer` предоставляет три типа колбэков для мониторинга выполнения задач:

```ts
const queue = new AsyncQueuer<string>()

// Обработка успешного завершения задачи
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Task succeeded:', result)
})

// Обработка ошибок задач
const unsubError = queue.onError((error) => {
  console.error('Task failed:', error)
})

// Обработка завершения задачи независимо от успеха/ошибки
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('Task failed:', result)
  } else {
    console.log('Task succeeded:', result)
  }
})

// Отписка от колбэков, когда они больше не нужны
unsubSuccess()
unsubError()
unsubSettled()
```

### Обработка отклонений (Rejection Handling)

Когда очередь достигает максимального размера (установленного опцией `maxSize`), новые задачи будут отклоняться. `AsyncQueuer` предоставляет способы обработки и мониторинга этих отклонений:

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // Разрешить только 2 задачи в очереди
  onReject: (task, queuer) => {
    console.log('Queue is full. Task rejected:', task)
  }
})

queue.addItem(async () => 'first') // Принято
queue.addItem(async () => 'second') // Принято
queue.addItem(async () => 'third') // Отклонено, вызывает колбэк onReject

console.log(queue.getRejectionCount()) // 1
```

### Начальные задачи (Initial Tasks)

Вы можете предварительно заполнить асинхронную очередь задачами при создании:

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // Начать обработку немедленно
})

// Очередь начинается с трех задач и сразу начинает их обработку
```

### Динамическая конфигурация (Dynamic Configuration)

Параметры `AsyncQueuer` можно изменять после создания с помощью `setOptions()` и получать с помощью `getOptions()`:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// Изменяем конфигурацию
queue.setOptions({
  concurrency: 4, // Обрабатывать больше задач одновременно
  started: true // Начать обработку
})

// Получаем текущую конфигурацию
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### Активные и ожидающие задачи (Active and Pending Tasks)

`AsyncQueuer` предоставляет методы для мониторинга как активных, так и ожидающих задач:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// Добавляем несколько задач
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// Мониторим состояния задач
console.log(queue.getActiveItems().length) // Текущие обрабатываемые задачи
console.log(queue.getPendingItems().length) // Задачи, ожидающие обработки
```

### Адаптеры для фреймворков (Framework Adapters)

Каждый адаптер для фреймворка создает удобные хуки и функции вокруг классов асинхронной очереди. Хуки типа `useAsyncQueuer` или `useAsyncQueuedState` — это небольшие обертки, которые могут сократить стандартный код, необходимый в вашем коде для некоторых распространенных случаев использования.
