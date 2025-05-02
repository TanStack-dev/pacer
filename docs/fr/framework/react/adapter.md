---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:32:26.561Z'
title: Adaptateur React
id: adapter
---
Si vous utilisez TanStack Pacer dans une application React, nous vous recommandons d'utiliser l'adaptateur React. L'adaptateur React fournit un ensemble de hooks faciles à utiliser par-dessus les utilitaires de base de Pacer. Si vous souhaitez utiliser directement les classes/fonctions de base de Pacer, l'adaptateur React réexporte également tout depuis le package de base.

## Installation

```sh
npm install @tanstack/react-pacer
```

## Hooks React

Consultez la [Référence des fonctions React](./reference/index.md) pour voir la liste complète des hooks disponibles dans l'adaptateur React.

## Utilisation de base

Importez un hook spécifique à React depuis l'adaptateur React.

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

Ou importez une classe/fonction de base de Pacer qui est réexportée depuis l'adaptateur React.

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // pas besoin d'installer séparément le package de base
```
