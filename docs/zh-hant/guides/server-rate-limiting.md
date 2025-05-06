---
source-updated-at: '2025-04-24T02:14:56.000Z'
translation-updated-at: '2025-05-06T20:30:23.590Z'
title: Server Rate Limiting Guide
id: server-rate-limiting
---
# 伺服器速率限制指南 (Server Rate Limiting Guide)

TanStack Pacer 是一個專注於提供高品質工具的函式庫，用於控制應用程式中函式的執行時機。雖然其他地方也有類似的工具，但我們的目標是確保所有重要細節都正確無誤 - 包括 ***型別安全 (type-safety)***、***樹搖優化 (tree-shaking)***，以及一致且 ***直觀的 API (intuitive API)***。透過專注於這些基礎要素並以 ***框架無關 (framework agnostic)*** 的方式提供，我們希望這些工具和模式能在您的應用程式中更普及。在應用程式開發中，適當的執行控制常常被忽視，導致可能被預防的效能問題、競爭條件 (race conditions) 和使用者體驗不佳等問題。TanStack Pacer 幫助您從一開始就正確地實作這些關鍵模式！
