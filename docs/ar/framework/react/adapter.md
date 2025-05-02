---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:39:01.082Z'
title: أداة التكيف لـ React
id: adapter
---
إذا كنت تستخدم TanStack Pacer في تطبيق React، نوصي باستخدام محول React (React Adapter). يوفر محول React مجموعة من الوظائف المساعدة (hooks) سهلة الاستخدام فوق أدوات Pacer الأساسية. إذا وجدت نفسك ترغب في استخدام فئات/وظائف Pacer الأساسية مباشرة، فإن محول React سيعيد تصدير كل شيء من الحزمة الأساسية.

## التثبيت

```sh
npm install @tanstack/react-pacer
```

## وظائف React المساعدة (React Hooks)

راجع [مرجع وظائف React](./reference/index.md) لرؤية القائمة الكاملة للوظائف المساعدة المتاحة في محول React.

## الاستخدام الأساسي

استورد وظيفة مساعدة محددة لـ React من محول React.

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

أو استورد فئة/وظيفة أساسية من Pacer التي يتم إعادة تصديرها من محول React.

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // لا حاجة لتثبيت الحزمة الأساسية بشكل منفصل
```
