---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:18:30.225Z'
title: React 适配器
id: adapter
---
如果您在 React 应用中使用 TanStack Pacer，我们推荐使用 React 适配器 (React Adapter)。该适配器在核心 Pacer 工具之上提供了一系列易用的钩子 (hooks)。如果您需要直接使用核心 Pacer 的类/函数，React 适配器也会重新导出核心包的所有内容。

## 安装

```sh
npm install @tanstack/react-pacer
```

## React 钩子 (React Hooks)

查看 [React 函数参考文档](./reference/index.md) 了解 React 适配器中可用的完整钩子列表。

## 基础用法

从 React 适配器中导入专为 React 设计的钩子：

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

或者导入由 React 适配器重新导出的核心 Pacer 类/函数：

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // 无需单独安装核心包
```
