---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:37:01.809Z'
title: Руководство по троттлингу
id: throttling
---
# Руководство по троттлингу (Throttling Guide)

Rate Limiting (Ограничение частоты), Throttling (Троттлинг) и Debouncing (Дебаунсинг) — это три различных подхода к управлению частотой выполнения функций. Каждый метод по-разному блокирует выполнение, делая их "потерянными" (lossy) — это означает, что некоторые вызовы функций не будут выполнены, если они запрашиваются слишком часто. Понимание, когда использовать каждый подход, критически важно для создания производительных и надежных приложений. В этом руководстве рассматриваются концепции троттлинга в TanStack Pacer.

## Концепция троттлинга (Throttling Concept)

Троттлинг гарантирует, что выполнение функций равномерно распределено во времени. В отличие от ограничения частоты (rate limiting), которое допускает всплески вызовов до определенного лимита, или дебаунсинга (debouncing), который ждет прекращения активности, троттлинг создает более плавный шаблон выполнения, обеспечивая постоянные задержки между вызовами. Если вы установите троттлинг на одно выполнение в секунду, вызовы будут равномерно распределены, независимо от того, как часто они запрашиваются.

### Визуализация троттлинга (Throttling Visualization)

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

### Когда использовать троттлинг (When to Use Throttling)

Троттлинг особенно эффективен, когда вам нужно последовательное и предсказуемое время выполнения. Это делает его идеальным для обработки частых событий или обновлений, где требуется плавное и контролируемое поведение.

Распространенные сценарии использования:
- Обновления пользовательского интерфейса (UI updates), требующие постоянного времени (например, индикаторы прогресса)
- Обработчики событий прокрутки (scroll) или изменения размера (resize), которые не должны перегружать браузер
- Опрос данных в реальном времени (real-time data polling), где желательны постоянные интервалы
- Ресурсоемкие операции, требующие стабильного темпа
- Обновления игрового цикла (game loop) или обработка кадров анимации (animation frame handling)
- Живые подсказки поиска (live search suggestions) при вводе пользователя

### Когда не использовать троттлинг (When Not to Use Throttling)

Троттлинг может быть не лучшим выбором, когда:
- Вы хотите дождаться прекращения активности (используйте [дебаунсинг](../guides/debouncing))
- Вы не можете позволить себе пропустить ни одного выполнения (используйте [очередь (queueing)](../guides/queueing))

> [!TIP]
> Троттлинг часто является лучшим выбором, когда вам нужно плавное и последовательное время выполнения. Он обеспечивает более предсказуемый шаблон выполнения, чем ограничение частоты (rate limiting), и более немедленную обратную связь, чем дебаунсинг (debouncing).

## Троттлинг в TanStack Pacer (Throttling in TanStack Pacer)

TanStack Pacer предоставляет как синхронный, так и асинхронный троттлинг через классы `Throttler` и `AsyncThrottler` соответственно (и соответствующие функции `throttle` и `asyncThrottle`).

### Базовое использование с `throttle` (Basic Usage with `throttle`)

Функция `throttle` — это самый простой способ добавить троттлинг к любой функции:

```ts
import { throttle } from '@tanstack/pacer'

// Троттлинг обновлений UI до одного каждые 200 мс
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// В быстром цикле выполняется только каждые 200 мс
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // Многие вызовы будут троттлиться
}
```

### Расширенное использование с классом `Throttler` (Advanced Usage with `Throttler` Class)

Для большего контроля над поведением троттлинга можно использовать класс `Throttler` напрямую:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Получение информации о состоянии выполнения
console.log(updateThrottler.getExecutionCount()) // Количество успешных выполнений
console.log(updateThrottler.getLastExecutionTime()) // Время последнего выполнения

// Отмена любого ожидающего выполнения
updateThrottler.cancel()
```

### Первое и последнее выполнение (Leading and Trailing Executions)

Синхронный троттлер поддерживает выполнение как на переднем (leading), так и на заднем (trailing) крае:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Выполнить при первом вызове (по умолчанию)
  trailing: true,  // Выполнить после периода ожидания (по умолчанию)
})
```

- `leading: true` (по умолчанию) — Выполнить немедленно при первом вызове
- `leading: false` — Пропустить первый вызов, ждать выполнения на заднем крае
- `trailing: true` (по умолчанию) — Выполнить последний вызов после периода ожидания
- `trailing: false` — Пропустить последний вызов, если он в пределах периода ожидания

Распространенные шаблоны:
- `{ leading: true, trailing: true }` — По умолчанию, наиболее отзывчивый
- `{ leading: false, trailing: true }` — Задержать все выполнения
- `{ leading: true, trailing: false }` — Пропустить запланированные выполнения

### Включение/отключение (Enabling/Disabling)

Класс `Throttler` поддерживает включение/отключение через опцию `enabled`. Используя метод `setOptions`, вы можете включать/отключать троттлер в любое время:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Отключено по умолчанию
throttler.setOptions({ enabled: true }) // Включить в любое время
```

Если вы используете адаптер для фреймворка, где параметры троттлера являются реактивными, вы можете установить опцию `enabled` в условное значение для динамического включения/отключения троттлера. Однако, если вы используете функцию `throttle` или класс `Throttler` напрямую, вы должны использовать метод `setOptions` для изменения опции `enabled`, так как передаваемые параметры фактически передаются в конструктор класса `Throttler`.

### Опции обратного вызова (Callback Options)

Как синхронный, так и асинхронный троттлеры поддерживают опции обратного вызова для обработки различных аспектов жизненного цикла троттлинга:

#### Обратные вызовы синхронного троттлера (Synchronous Throttler Callbacks)

Синхронный `Throttler` поддерживает следующий обратный вызов:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // Вызывается после каждого успешного выполнения
    console.log('Функция выполнена', throttler.getExecutionCount())
  }
})
```

Обратный вызов `onExecute` вызывается после каждого успешного выполнения троттлированной функции, что делает его полезным для отслеживания выполнений, обновления состояния UI или выполнения операций очистки.

#### Обратные вызовы асинхронного троттлера (Asynchronous Throttler Callbacks)

Асинхронный `AsyncThrottler` поддерживает дополнительные обратные вызовы для обработки ошибок:

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // Вызывается после каждого успешного выполнения
    console.log('Асинхронная функция выполнена', throttler.getExecutionCount())
  },
  onError: (error) => {
    // Вызывается, если асинхронная функция выбрасывает ошибку
    console.error('Асинхронная функция завершилась ошибкой:', error)
  }
})
```

Обратный вызов `onExecute` работает так же, как и в синхронном троттлере, в то время как `onError` позволяет обрабатывать ошибки без прерывания цепочки троттлинга. Эти обратные вызовы особенно полезны для отслеживания количества выполнений, обновления состояния UI, обработки ошибок, выполнения операций очистки и логирования метрик выполнения.

### Асинхронный троттлинг (Asynchronous Throttling)

Для асинхронных функций или когда требуется обработка ошибок используйте `AsyncThrottler` или `asyncThrottle`:

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
      console.error('Ошибка вызова API:', error)
    }
  }
)

// Будет выполнять только один вызов API в секунду
await throttledFetch('123')
```

Асинхронная версия предоставляет отслеживание выполнения на основе Promise, обработку ошибок через обратный вызов `onError`, правильную очистку ожидающих асинхронных операций и ожидаемый метод `maybeExecute`.

### Адаптеры для фреймворков (Framework Adapters)

Каждый адаптер для фреймворка предоставляет хуки, которые расширяют базовую функциональность троттлинга для интеграции с системой управления состоянием фреймворка. Для каждого фреймворка доступны хуки, такие как `createThrottler`, `useThrottledCallback`, `useThrottledState` или `useThrottledValue`.

Вот несколько примеров:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Низкоуровневый хук для полного контроля
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Простой хук обратного вызова для базовых случаев
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Хук на основе состояния для реактивного управления состоянием
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // Значение для троттлинга
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// Низкоуровневый хук для полного контроля
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Хук на основе сигналов для управления состоянием
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('Всего выполнений:', throttler.getExecutionCount())
  }
})
```

Каждый адаптер для фреймворка предоставляет хуки, которые интегрируются с системой управления состоянием фреймворка, сохраняя при этом базовую функциональность троттлинга.
