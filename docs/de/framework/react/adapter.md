---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:29:13.275Z'
title: React-Adapter
id: adapter
---
Wenn Sie TanStack Pacer in einer React-Anwendung verwenden, empfehlen wir die Verwendung des React Adapters. Der React Adapter bietet eine Reihe von einfach zu verwendenden Hooks auf Basis der Kern-Utilities von Pacer. Falls Sie die Kern-Klassen/Funktionen von Pacer direkt verwenden möchten, exportiert der React Adapter ebenfalls alles aus dem Core-Package erneut.

## Installation

```sh
npm install @tanstack/react-pacer
```

## React Hooks

Siehe die [React Functions Reference](./reference/index.md), um die vollständige Liste der im React Adapter verfügbaren Hooks einzusehen.

## Grundlegende Verwendung

Importieren Sie einen React-spezifischen Hook aus dem React Adapter.

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

Oder importieren Sie eine Kern-Klasse/Funktion von Pacer, die aus dem React Adapter erneut exportiert wird.

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // eine separate Installation des Core-Packages ist nicht erforderlich
```
