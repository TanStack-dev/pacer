---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:19:51.589Z'
title: Руководство по троттлингу
id: throttling
---
# Руководство по троттлингу (Throttling)

Ограничение частоты (Rate Limiting), троттлинг (Throttling) и дебаунсинг (Debouncing) — это три различных подхода к управлению частотой выполнения функций. Каждый метод по-разному блокирует выполнение, делая их "потерянными" (lossy) — это означает, что некоторые вызовы функций не будут выполнены, если они запрашиваются слишком часто. Понимание, когда использовать каждый подход, крайне важно для создания производительных и надежных приложений. В этом руководстве рассматриваются концепции троттлинга в TanStack Pacer.

## Концепция троттлинга (Throttling)

Троттлинг гарантирует равномерное распределение выполнения функций во времени. В отличие от ограничения частоты (rate limiting), которое допускает всплески выполнения до определенного предела, или дебаунсинга (debouncing), который ждет прекращения активности, троттлинг создает более плавный паттерн выполнения, устанавливая постоянные задержки между вызовами. Если вы установите троттлинг на одно выполнение в секунду, вызовы будут равномерно распределены, независимо от того, как часто они запрашиваются.

### Визуализация троттлинга

```text
Троттлинг (одно выполнение за 3 тика)
Таймлайн: [1 секунда на тик]
Вызовы:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Выполнено:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Разрешено только одно выполнение за 3 тика,
               независимо от количества вызовов

             [Первый всплеск]    [Дополнительные вызовы]    [Равномерные вызовы]
             Выполнить первый    Выполнить после            Выполнять каждый раз,
             затем троттлинг     периода ожидания          когда проходит период ожидания
```

### Когда использовать троттлинг

Троттлинг особенно эффективен, когда вам нужно постоянное, предсказуемое время выполнения. Это делает его идеальным для обработки частых событий или обновлений, где требуется плавное, контролируемое поведение.

Типичные сценарии использования:
- Обновления пользовательского интерфейса (UI), требующие постоянного времени (например, индикаторы прогресса)
- Обработчики событий прокрутки (scroll) или изменения размера (resize), которые не должны перегружать браузер
- Опрос данных в реальном времени (real-time polling), где желательны постоянные интервалы
- Ресурсоемкие операции, требующие равномерного темпа
- Обновления игрового цикла (game loop) или обработка кадров анимации (animation frame)
- Живые подсказки поиска (live search) при вводе пользователем

### Когда не использовать троттлинг

Троттлинг может быть не лучшим выбором, когда:
- Нужно дождаться прекращения активности (используйте [дебаунсинг](../guides/debouncing))
- Нельзя пропускать ни одного выполнения (используйте [очередь (queueing)](../guides/queueing))

> [!TIP]
> Троттлинг часто является лучшим выбором, когда требуется плавное, постоянное время выполнения. Он обеспечивает более предсказуемый паттерн выполнения, чем ограничение частоты (rate limiting), и более быструю обратную связь, чем дебаунсинг (debouncing).

## Троттлинг в TanStack Pacer

TanStack Pacer предоставляет синхронный и асинхронный троттлинг через классы `Throttler` и `AsyncThrottler` соответственно (и соответствующие функции `throttle` и `asyncThrottle`).

### Базовое использование с `throttle`

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
  throttledUpdate(i) // Многие вызовы подвергаются троттлингу
}
```

### Продвинутое использование с классом `Throttler`

Для большего контроля над поведением троттлинга можно напрямую использовать класс `Throttler`:

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

### Первое и последнее выполнение

Синхронный троттлер поддерживает выполнение как по первому (leading), так и по последнему (trailing) вызову:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Выполнить при первом вызове (по умолчанию)
  trailing: true,  // Выполнить после периода ожидания (по умолчанию)
})
```

- `leading: true` (по умолчанию) — Выполнить сразу при первом вызове
- `leading: false` — Пропустить первый вызов, ждать последнего выполнения
- `trailing: true` (по умолчанию) — Выполнить последний вызов после периода ожидания
- `trailing: false` — Пропустить последний вызов, если он в пределах периода ожидания

Типичные паттерны:
- `{ leading: true, trailing: true }` — По умолчанию, наиболее отзывчивый
- `{ leading: false, trailing: true }` — Задержать все выполнения
- `{ leading: true, trailing: false }` — Пропускать вызовы в очереди

### Включение/отключение

Класс `Throttler` поддерживает включение/отключение через опцию `enabled`. Используя метод `setOptions`, вы можете включать/отключать троттлер в любое время:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Отключить по умолчанию
throttler.setOptions({ enabled: true }) // Включить в любое время
```

Если вы используете адаптер для фреймворка, где параметры троттлера реактивны, вы можете установить опцию `enabled` в условное значение для динамического включения/отключения троттлера. Однако, если вы используете функцию `throttle` или класс `Throttler` напрямую, вы должны использовать метод `setOptions` для изменения опции `enabled`, так как переданные параметры фактически передаются в конструктор класса `Throttler`.

### Опции обратных вызовов (Callbacks)

Как синхронные, так и асинхронные троттлеры поддерживают опции обратных вызовов для обработки различных аспектов жизненного цикла троттлинга:

#### Обратные вызовы синхронного троттлера

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

Обратный вызов `onExecute` вызывается после каждого успешного выполнения троттлируемой функции, что делает его полезным для отслеживания выполнений, обновления состояния UI или выполнения операций очистки.

#### Обратные вызовы асинхронного троттлера

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

Обратный вызов `onExecute` работает так же, как и в синхронном троттлере, а `onError` позволяет обрабатывать ошибки без прерывания цепочки троттлинга. Эти обратные вызовы особенно полезны для отслеживания количества выполнений, обновления состояния UI, обработки ошибок, выполнения операций очистки и логирования метрик выполнения.

### Асинхронный троттлинг

Асинхронный троттлер предоставляет мощный способ обработки асинхронных операций с троттлингом, предлагая несколько ключевых преимуществ перед синхронной версией. В то время как синхронный троттлер отлично подходит для событий UI и мгновенной обратной связи, асинхронная версия специально разработана для обработки вызовов API, операций с базой данных и других асинхронных задач.

#### Ключевые отличия от синхронного троттлинга

1. **Обработка возвращаемого значения**
В отличие от синхронного троттлера, который возвращает void, асинхронная версия позволяет захватывать и использовать возвращаемое значение из вашей троттлируемой функции. Это особенно полезно, когда нужно работать с результатами вызовов API или других асинхронных операций. Метод `maybeExecute` возвращает Promise, который разрешается с возвращаемым значением функции, позволяя ожидать результат и обрабатывать его соответствующим образом.

2. **Улучшенная система обратных вызовов**
Асинхронный троттлер предоставляет более сложную систему обратных вызовов по сравнению с единственным `onExecute` в синхронной версии. Эта система включает:
- `onSuccess`: Вызывается при успешном завершении асинхронной функции, предоставляя как результат, так и экземпляр троттлера
- `onError`: Вызывается при ошибке в асинхронной функции, предоставляя как ошибку, так и экземпляр троттлера
- `onSettled`: Вызывается после каждой попытки выполнения, независимо от успеха или ошибки

3. **Отслеживание выполнения**
Асинхронный троттлер предоставляет комплексное отслеживание выполнения через несколько методов:
- `getSuccessCount()`: Количество успешных выполнений
- `getErrorCount()`: Количество неудачных выполнений
- `getSettledCount()`: Общее количество завершенных выполнений (успешные + ошибки)

4. **Последовательное выполнение**
Асинхронный троттлер гарантирует, что последующие выполнения ждут завершения предыдущего вызова перед началом. Это предотвращает выполнение в неправильном порядке и гарантирует, что каждый вызов обрабатывает самые актуальные данные. Это особенно важно при работе с операциями, которые зависят от результатов предыдущих вызовов, или когда критически важна согласованность данных.

Например, если вы обновляете профиль пользователя и сразу же запрашиваете обновленные данные, асинхронный троттлер гарантирует, что операция запроса дождется завершения обновления, предотвращая состояния гонки (race conditions), при которых можно получить устаревшие данные.

#### Пример базового использования

Вот простой пример использования асинхронного троттлера для операции поиска:

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('Поиск успешен:', results)
    },
    onError: (error, throttler) => {
      console.error('Поиск завершился ошибкой:', error)
    }
  }
)

// Использование
const results = await throttledSearch('query')
```

#### Продвинутые паттерны

Асинхронный троттлер можно комбинировать с различными паттернами для решения сложных задач:

1. **Интеграция с управлением состоянием**
При использовании асинхронного троттлера с системами управления состоянием (такими как useState в React или createSignal в Solid) можно создавать мощные паттерны для обработки состояний загрузки, ошибок и обновления данных. Обратные вызовы троттлера предоставляют идеальные хуки для обновления состояния UI на основе успеха или неудачи операций.

2. **Предотвращение состояний гонки**
Паттерн троттлинга естественным образом предотвращает состояния гонки во многих сценариях. Когда несколько частей вашего приложения пытаются одновременно обновить один и тот же ресурс, троттлер гарантирует, что обновления происходят с контролируемой скоростью, при этом предоставляя результаты всем вызывающим сторонам.

3. **Восстановление после ошибок**
Возможности обработки ошибок асинхронного троттлера делают его идеальным для реализации логики повторных попыток и стратегий восстановления. Вы можете использовать обратный вызов `onError` для реализации пользовательских стратегий обработки ошибок, таких как экспоненциальная задержка (exponential backoff) или механизмы отката.

### Адаптеры для фреймворков

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
const [throttledValue] = useThrottledValue(
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

// Хук на основе сигналов (signals) для управления состоянием
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('Всего выполнений:', throttler.getExecutionCount())
  }
})
```

Каждый адаптер для фреймворка предоставляет хуки, которые интегрируются с системой управления состоянием фреймворка, сохраняя при этом базовую функциональность троттлинга.
