---
source-updated-at: '2025-04-24T02:14:56.000Z'
translation-updated-at: '2025-05-06T20:43:58.729Z'
id: asyncDebounce
title: asyncDebounce
---

<!-- DO NOT EDIT: this page is autogenerated from the type comments -->

# Function: asyncDebounce()

```ts
function asyncDebounce<TFn, TArgs>(fn, initialOptions): (...args) => Promise<void>
```

Defined in: [async-debouncer.ts:219](https://github.com/TanStack/pacer/blob/main/packages/pacer/src/async-debouncer.ts#L219)

Creates an async debounced function that delays execution until after a specified wait time.
The debounced function will only execute once the wait period has elapsed without any new calls.
If called again during the wait period, the timer resets and a new wait period begins.

## Type Parameters

• **TFn** *extends* [`AnyAsyncFunction`](../type-aliases/anyasyncfunction.md)

• **TArgs** *extends* `any`[]

## Parameters

### fn

`TFn`

### initialOptions

`Omit`\<[`AsyncDebouncerOptions`](../interfaces/asyncdebounceroptions.md)\<`TFn`, `TArgs`\>, `"enabled"`\>

## Returns

`Function`

Attempts to execute the debounced function
If a call is already in progress, it will be queued

### Parameters

#### args

...`TArgs`

### Returns

`Promise`\<`void`\>

## Example

```ts
const debounced = asyncDebounce(async (value: string) => {
  await saveToAPI(value);
}, { wait: 1000 });

// Will only execute once, 1 second after the last call
await debounced("first");  // Cancelled
await debounced("second"); // Cancelled
await debounced("third");  // Executes after 1s
```
