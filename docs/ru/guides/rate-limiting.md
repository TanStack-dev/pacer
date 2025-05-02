---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:37:05.445Z'
title: Руководство по ограничению частоты
id: rate-limiting
---
# Руководство по ограничению частоты запросов (Rate Limiting)

Ограничение частоты запросов (Rate Limiting), регулирование (Throttling) и устранение дребезга (Debouncing) — это три различных подхода к контролю частоты выполнения функций. Каждый метод блокирует выполнение по-своему, делая их "потерянными" (lossy) — это означает, что некоторые вызовы функций не будут выполнены, если они запрашиваются слишком часто. Понимание, когда использовать каждый подход, крайне важно для создания производительных и надежных приложений. В этом руководстве рассматриваются концепции ограничения частоты запросов в TanStack Pacer.

> [!NOTE]
> TanStack Pacer в настоящее время является только фронтенд-библиотекой. Это утилиты для ограничения частоты запросов на стороне клиента.

## Концепция ограничения частоты запросов (Rate Limiting)

Ограничение частоты запросов (Rate Limiting) — это метод, который ограничивает частоту выполнения функции в течение определенного временного окна. Он особенно полезен в сценариях, когда нужно предотвратить слишком частый вызов функции, например, при обработке API-запросов или других вызовов внешних сервисов. Это наиболее *наивный* подход, так как он позволяет выполнять вызовы пакетами до исчерпания квоты.

### Визуализация ограничения частоты запросов

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Когда использовать ограничение частоты запросов

Ограничение частоты запросов особенно важно при работе с фронтенд-операциями, которые могут случайно перегрузить бэкенд-сервисы или вызвать проблемы с производительностью в браузере.

Типичные сценарии использования:
- Предотвращение случайного спама API из-за быстрых действий пользователя (например, кликов по кнопке или отправки форм)
- Сценарии, где допустимо пакетное поведение, но требуется ограничить максимальную частоту
- Защита от случайных бесконечных циклов или рекурсивных операций

### Когда не следует использовать ограничение частоты запросов

Ограничение частоты запросов — это наиболее наивный подход к контролю частоты выполнения функций. Он наименее гибкий и наиболее ограничительный из трех методов. Вместо него рассмотрите использование [регулирования (throttling)](../guides/throttling) или [устранения дребезга (debouncing)](../guides/debouncing) для более равномерного выполнения.

> [!TIP]
> В большинстве случаев вам, скорее всего, не нужно использовать "ограничение частоты запросов". Вместо этого рассмотрите [регулирование (throttling)](../guides/throttling) или [устранение дребезга (debouncing)](../guides/debouncing).

"Потерянная" природа ограничения частоты запросов также означает, что некоторые вызовы будут отклонены и потеряны. Это может быть проблемой, если нужно гарантировать, что все вызовы всегда успешны. Рассмотрите использование [очереди (queueing)](../guides/queueing), если требуется обеспечить, чтобы все вызовы были поставлены в очередь для выполнения, но с задержкой для замедления скорости выполнения.

## Ограничение частоты запросов в TanStack Pacer

TanStack Pacer предоставляет синхронное и асинхронное ограничение частоты запросов через классы `RateLimiter` и `AsyncRateLimiter` соответственно (и соответствующие функции `rateLimit` и `asyncRateLimit`).

### Базовое использование с `rateLimit`

Функция `rateLimit` — это самый простой способ добавить ограничение частоты запросов к любой функции. Она идеально подходит для большинства случаев, когда нужно просто установить простое ограничение.

```ts
import { rateLimit } from '@tanstack/pacer'

// Ограничение API-запросов до 5 в минуту
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 минута в миллисекундах
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Первые 5 вызовов выполнятся немедленно
rateLimitedApi('user-1') // ✅ Выполняется
rateLimitedApi('user-2') // ✅ Выполняется
rateLimitedApi('user-3') // ✅ Выполняется
rateLimitedApi('user-4') // ✅ Выполняется
rateLimitedApi('user-5') // ✅ Выполняется
rateLimitedApi('user-6') // ❌ Отклоняется до сброса окна
```

### Продвинутое использование с классом `RateLimiter`

Для более сложных сценариев, где требуется дополнительный контроль над поведением ограничения частоты запросов, можно использовать класс `RateLimiter` напрямую. Это дает доступ к дополнительным методам и информации о состоянии.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Создание экземпляра ограничителя частоты запросов
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Function executed', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Получение информации о текущем состоянии
console.log(limiter.getRemainingInWindow()) // Количество оставшихся вызовов в текущем окне
console.log(limiter.getExecutionCount()) // Общее количество успешных выполнений
console.log(limiter.getRejectionCount()) // Общее количество отклоненных выполнений

// Попытка выполнения (возвращает boolean, указывающий на успех)
limiter.maybeExecute('user-1')

// Динамическое обновление параметров
limiter.setOptions({ limit: 10 }) // Увеличение лимита

// Сброс всех счетчиков и состояния
limiter.reset()
```

### Включение/отключение

Класс `RateLimiter` поддерживает включение/отключение через параметр `enabled`. Используя метод `setOptions`, можно включать/отключать ограничитель частоты запросов в любое время:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Отключено по умолчанию
})
limiter.setOptions({ enabled: true }) // Включить в любое время
```

Если используется адаптер для фреймворка, где параметры ограничителя частоты запросов реактивны, можно установить параметр `enabled` в условное значение для динамического включения/отключения. Однако при использовании функции `rateLimit` или класса `RateLimiter` напрямую необходимо использовать метод `setOptions` для изменения параметра `enabled`, так как переданные параметры фактически передаются в конструктор класса `RateLimiter`.

### Параметры обратного вызова

Как синхронные, так и асинхронные ограничители частоты запросов поддерживают параметры обратного вызова для обработки различных аспектов жизненного цикла ограничения частоты запросов:

#### Обратные вызовы синхронного ограничителя частоты запросов

Синхронный `RateLimiter` поддерживает следующие обратные вызовы:

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Вызывается после каждого успешного выполнения
    console.log('Function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Вызывается при отклонении выполнения
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

Обратный вызов `onExecute` вызывается после каждого успешного выполнения функции с ограничением частоты запросов, а `onReject` — при отклонении выполнения из-за ограничения. Эти обратные вызовы полезны для отслеживания выполнений, обновления состояния UI или предоставления обратной связи пользователям.

#### Обратные вызовы асинхронного ограничителя частоты запросов

Асинхронный `AsyncRateLimiter` поддерживает дополнительные обратные вызовы для обработки ошибок:

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Вызывается после каждого успешного выполнения
    console.log('Async function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Вызывается при отклонении выполнения
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // Вызывается, если асинхронная функция выбрасывает ошибку
    console.error('Async function failed:', error)
  }
})
```

Обратные вызовы `onExecute` и `onReject` работают так же, как и в синхронном ограничителе, а `onError` позволяет обрабатывать ошибки без прерывания цепочки ограничения частоты запросов. Эти обратные вызовы особенно полезны для отслеживания количества выполнений, обновления состояния UI, обработки ошибок и предоставления обратной связи пользователям.

### Асинхронное ограничение частоты запросов

Используйте `AsyncRateLimiter`, когда:
- Ваша функция с ограничением частоты запросов возвращает Promise
- Нужно обрабатывать ошибки из асинхронной функции
- Требуется обеспечить правильное ограничение частоты запросов, даже если асинхронная функция выполняется долго

```ts
import { asyncRateLimit } from '@tanstack/pacer'

const rateLimited = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onError: (error) => {
      console.error('API call failed:', error)
    }
  }
)

// Возвращает Promise<boolean> — разрешается в true, если выполнено, false, если отклонено
const wasExecuted = await rateLimited('123')
```

Асинхронная версия предоставляет отслеживание выполнения на основе Promise, обработку ошибок через обратный вызов `onError`, правильную очистку ожидающих асинхронных операций и ожидаемый метод `maybeExecute`.

### Адаптеры для фреймворков

Каждый адаптер для фреймворка предоставляет хуки, которые расширяют базовую функциональность ограничения частоты запросов для интеграции с системой управления состоянием фреймворка. Для каждого фреймворка доступны хуки, такие как `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` или `useRateLimitedValue`.

Вот несколько примеров:

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// Низкоуровневый хук для полного контроля
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Простой хук обратного вызова для базовых сценариев
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Хук на основе состояния для реактивного управления состоянием
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // Значение для ограничения частоты запросов
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// Низкоуровневый хук для полного контроля
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Хук на основе сигналов для управления состоянием
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
