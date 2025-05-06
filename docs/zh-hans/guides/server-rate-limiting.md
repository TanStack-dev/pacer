---
source-updated-at: '2025-04-24T02:14:56.000Z'
translation-updated-at: '2025-05-06T20:27:49.069Z'
title: Server Rate Limiting Guide
id: server-rate-limiting
---
# 服务端速率限制指南 (Server Rate Limiting Guide)

TanStack Pacer 是一个专注于为应用程序提供高质量函数执行时序控制工具 (utilities) 的库。虽然其他地方也有类似的工具，但我们的目标是处理好所有重要细节 —— 包括 ***类型安全 (type-safety)***、***摇树优化 (tree-shaking)*** 以及一致且 ***直观的 API (intuitive API)***。通过专注于这些基础功能并以 ***框架无关 (framework agnostic)*** 的方式提供它们，我们希望这些工具和模式能在您的应用程序中更加普及。在应用开发中，正确的执行控制常常被忽视，导致性能问题、竞态条件 (race conditions) 和本可避免的不良用户体验。TanStack Pacer 帮助您从一开始就正确实现这些关键模式！
