---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:35:19.100Z'
title: Адаптер для React
id: adapter
---
Если вы используете TanStack Pacer в React-приложении, мы рекомендуем использовать React Adapter. React Adapter предоставляет набор удобных хуков поверх основных утилит Pacer. Если вам потребуется использовать основные классы/функции Pacer напрямую, React Adapter также реэкспортирует всё из основного пакета.

## Установка

```sh
npm install @tanstack/react-pacer
```

## React-хуки

Ознакомьтесь с [React Functions Reference](./reference/index.md), чтобы увидеть полный список хуков, доступных в React Adapter.

## Базовое использование

Импортируйте React-специфичный хук из React Adapter.

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

Или импортируйте основной класс/функцию Pacer, которые реэкспортируются из React Adapter.

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // нет необходимости устанавливать основной пакет отдельно
```
