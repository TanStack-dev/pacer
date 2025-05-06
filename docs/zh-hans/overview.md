---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:02:47.343Z'
title: 概述
id: overview
---
# 概述

TanStack Pacer 是一个专注于为应用程序提供高质量函数执行时序控制工具 (utility) 的库。虽然类似工具在其他地方也存在，但我们的目标是正确处理所有重要细节 —— 包括 ***类型安全 (type-safety)***、***tree-shaking*** 以及一致且 ***直观的 API (intuitive API)***。通过专注于这些基础功能并以 ***框架无关 (framework agnostic)*** 的方式提供它们，我们希望这些工具和模式能在您的应用程序中更加普及。恰当的执行控制在应用开发中常常被忽视，导致性能问题、竞态条件 (race conditions) 和本可避免的不良用户体验。TanStack Pacer 帮助您从一开始就正确实现这些关键模式！

> [!IMPORTANT]
> TanStack Pacer 目前处于 **alpha** 阶段，其 API 可能会发生变化。
>
> 该库的范围可能会扩大，但我们希望保持每个独立工具的包体积精简且专注。

## 起源

TanStack Pacer 的许多想法（和代码）并不新鲜。事实上，这些工具中有许多已经在其他 TanStack 库中存在了相当长的时间。我们从 TanStack Query、Router、Form 甚至 Tanner 最初的 [Swimmer](https://github.com/tannerlinsley/swimmer) 库中提取了代码。然后我们清理了这些工具，填补了一些空白，并将它们作为一个独立的库发布。

## 关键特性

- **防抖 (Debouncing)**
  - 延迟函数执行直到一段不活动期之后
  - 支持 Promise 和错误处理的同步或异步防抖工具
- **节流 (Throttling)**
  - 限制函数可以触发的速率
  - 支持 Promise 和错误处理的同步或异步节流工具
- **速率限制 (Rate Limiting)**
  - 限制函数可以触发的速率
  - 支持 Promise 和错误处理的同步或异步速率限制工具
- **队列 (Queuing)**
  - 将函数按特定顺序排队执行
  - 可选择 FIFO、LIFO 和优先级队列实现
  - 通过可配置的等待时间或并发限制控制处理速度
  - 通过启动/停止功能管理队列执行
  - 在可配置的持续时间后使队列中的项目过期
- **异步或同步变体 (Async or Sync Variations)**
  - 在每个工具的同步和异步版本之间选择
  - 在工具的异步变体中按需强制执行单次飞行 (single-flight) 执行
  - 异步变体的可选错误、成功和完成处理
- **比较工具 (Comparison Utilities)**
  - 在值之间执行深度相等性检查
  - 为特定需求创建自定义比较逻辑
- **便捷的钩子 (Convenient Hooks)**
  - 使用预构建的钩子如 `useDebouncedCallback`、`useThrottledValue` 和 `useQueuedState` 等减少样板代码
  - 根据您的用例选择多层抽象
  - 与每个框架的默认状态管理解决方案或您喜欢的任何自定义状态管理库一起工作
- **类型安全 (Type Safety)**
  - 使用 TypeScript 实现完整的类型安全，确保您的函数始终以正确的参数调用
  - 泛型支持灵活且可重用的工具
- **框架适配器 (Framework Adapters)**
  - React、Solid 等
- **Tree Shaking**
  - 我们当然会为您的应用程序默认正确地实现 tree-shaking，但我们也为每个工具提供了额外的深层导入 (deep imports)，使得将这些工具嵌入到您的库中而不增加 bundle-phobia 报告变得更加容易。
