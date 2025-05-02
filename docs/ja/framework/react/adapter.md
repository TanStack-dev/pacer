---
source-updated-at: '2025-03-30T22:07:29.000Z'
translation-updated-at: '2025-05-02T04:13:12.255Z'
title: Reactアダプター
id: adapter
---
ReactアプリケーションでTanStack Pacerを使用する場合、Reactアダプターの使用を推奨します。Reactアダプターは、コアのPacerユーティリティの上に、使いやすいフックのセットを提供します。コアのPacerクラス/関数を直接使用したい場合でも、Reactアダプターはコアパッケージのすべてを再エクスポートします。

## インストール

```sh
npm install @tanstack/react-pacer
```

## Reactフック

Reactアダプターで利用可能なフックの完全なリストについては、[React Functions Reference](./reference/index.md)を参照してください。

## 基本的な使い方

ReactアダプターからReact専用のフックをインポートします。

```tsx
import { useDebouncedValue } from '@tanstack/react-pacer'

const [instantValue, instantValueRef] = useState(0)
const [debouncedValue, debouncer] = useDebouncedValue(instantValue, {
  wait: 1000,
})
```

または、Reactアダプターから再エクスポートされたコアのPacerクラス/関数をインポートします。

```tsx
import { debounce, Debouncer } from '@tanstack/react-pacer' // コアパッケージを別途インストールする必要はありません
```
