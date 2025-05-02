---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:23:34.805Z'
title: React 適配器
id: adapter
---
若您正在 React 應用程式中使用 TanStack Pacer，我們推薦使用 React 轉接器 (React Adapter)。此轉接器在核心 Pacer 工具之上提供了一系列易用的鉤子 (hooks)。如果您想直接使用核心 Pacer 的類別/函式，React 轉接器也會重新匯出核心套件的所有內容。

## 安裝

```sh
npm install @tanstack/react-pacer
```

## React 鉤子 (Hooks)

請參閱 [React 函式參考文件](./reference/index.md) 以查看 React 轉接器中提供的完整鉤子清單。

## 基本用法

從 React 轉接器匯入專為 React 設計的鉤子：

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

或是匯入由 React 轉接器重新匯出的核心 Pacer 類別/函式：

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // 無需另行安裝核心套件
```
