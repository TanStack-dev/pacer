---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:04:57.624Z'
title: 概述
id: overview
---
TanStack Pacer 是一個專注於提供高品質工具的函式庫，用於控制應用程式中函式執行的時機。雖然其他地方也有類似的工具，但我們的目標是確保所有重要細節都正確無誤——包括 ***型別安全 (type-safety)***、***樹狀搖晃 (tree-shaking)***，以及一致且 ***直觀的 API (intuitive API)***。透過專注於這些基礎並以 ***框架無關 (framework agnostic)*** 的方式提供，我們希望這些工具和模式能在您的應用程式中更加普及。適當的執行控制經常在應用程式開發中被忽略，導致效能問題、競爭條件 (race conditions) 以及本可避免的不良使用者體驗。TanStack Pacer 幫助您從一開始就正確實作這些關鍵模式！

> [!IMPORTANT]
> TanStack Pacer 目前處於 **alpha** 階段，其 API 可能會變更。
>
> 此函式庫的範圍可能會擴大，但我們希望保持每個獨立工具的套件大小精簡且專注。

## 起源

TanStack Pacer 的許多概念（和程式碼）並非全新。事實上，這些工具中有許多已在其他 TanStack 函式庫中存在相當長的時間。我們從 TanStack Query、Router、Form，甚至是 Tanner 原始的 [Swimmer](https://github.com/tannerlinsley/swimmer) 函式庫中提取了程式碼。接著我們清理了這些工具，填補了一些缺口，並將其作為獨立的函式庫發布。

## 主要功能

- **防抖 (Debouncing)**
  - 延遲函式執行，直到一段時間沒有活動後才觸發
  - 支援同步或非同步的防抖工具，提供 Promise 支援與錯誤處理
- **節流 (Throttling)**
  - 限制函式觸發的速率
  - 支援同步或非同步的節流工具，提供 Promise 支援與錯誤處理
- **速率限制 (Rate Limiting)**
  - 限制函式觸發的速率
  - 支援同步或非同步的速率限制工具，提供 Promise 支援與錯誤處理
- **佇列 (Queuing)**
  - 將函式排入佇列以特定順序執行
  - 可選擇 FIFO、LIFO 或優先佇列實作
  - 透過可配置的等待時間或並發限制控制處理速度
  - 具備開始/停止功能以管理佇列執行
  - 可配置持續時間後使佇列項目過期
- **同步或非同步變體 (Async or Sync Variations)**
  - 可選擇同步或非同步版本的每種工具
  - 在非同步工具變體中，必要時可強制執行單次飛行 (single-flight) 的函式執行
  - 非同步變體提供可選的錯誤、成功及完成處理
- **比較工具 (Comparison Utilities)**
  - 對值進行深度相等檢查
  - 為特定需求建立自訂比較邏輯
- **便利的鉤子 (Convenient Hooks)**
  - 使用預建的鉤子如 `useDebouncedCallback`、`useThrottledValue` 和 `useQueuedState` 等減少樣板程式碼
  - 提供多層抽象供選擇，視使用情境而定
  - 可與各框架的預設狀態管理方案或您偏好的自訂狀態管理函式庫搭配使用
- **型別安全 (Type Safety)**
  - 完整的 TypeScript 型別安全，確保您的函式總是以正確的參數呼叫
  - 泛型支援，打造靈活且可重複使用的工具
- **框架適配器 (Framework Adapters)**
  - 支援 React、Solid 等框架
- **樹狀搖晃 (Tree Shaking)**
  - 我們當然預設為您的應用程式正確實作樹狀搖晃，但我們還為每個工具提供額外的深度導入 (deep imports)，讓這些工具更容易嵌入您的函式庫中，而不會增加您的函式庫的套件恐懼報告 (bundle-phobia reports)。
