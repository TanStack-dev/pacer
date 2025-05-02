---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:30:49.844Z'
title: Throttling-Anleitung
id: throttling
---
# Throttling Guide (Drosselungs-Leitfaden)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" (lossy) sind – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung performanter und zuverlässiger Anwendungen. Dieser Leitfaden behandelt die Drosselungskonzepte (Throttling) von TanStack Pacer.

## Drosselungskonzept (Throttling Concept)

Drosselung stellt sicher, dass Funktionsausführungen gleichmäßig über die Zeit verteilt werden. Im Gegensatz zur Ratenbegrenzung (Rate Limiting), die Ausführungsbursts bis zu einem Limit erlaubt, oder zur Entprellung (Debouncing), die auf das Ende der Aktivität wartet, erzeugt Drosselung ein gleichmäßigeres Ausführungsmuster, indem sie konsistente Verzögerungen zwischen den Aufrufen erzwingt. Wenn Sie eine Drosselung von einer Ausführung pro Sekunde festlegen, werden die Aufrufe gleichmäßig verteilt, unabhängig davon, wie schnell sie angefordert werden.

### Drosselungsvisualisierung (Throttling Visualization)

```text
Throttling (one execution per 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Executed:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Nur eine Ausführung alle 3 Ticks erlaubt,
               unabhängig von der Anzahl der Aufrufe

             [Erster Burst]    [Weitere Aufrufe]         [Verteilte Aufrufe]
             Erste Ausführung  Ausführung nach           Ausführung jeweils nach
             dann drosseln    Wartezeit                 Ablauf der Wartezeit
```

### Wann Drosselung verwenden (When to Use Throttling)

Drosselung ist besonders effektiv, wenn Sie konsistente, vorhersehbare Ausführungszeiten benötigen. Dies macht sie ideal für die Handhabung häufiger Ereignisse oder Updates, bei denen Sie ein gleichmäßiges, kontrolliertes Verhalten wünschen.

Häufige Anwendungsfälle:
- UI-Updates mit konsistenter Timing-Anforderung (z.B. Fortschrittsanzeigen)
- Scroll- oder Resize-Event-Handler, die den Browser nicht überlasten sollen
- Echtzeit-Datenabfragen mit gewünschten konsistenten Intervallen
- Ressourcenintensive Operationen mit gleichmäßiger Ausführung
- Spielschleifen-Updates oder Animation Frame Handling
- Live-Suchvorschläge während der Benutzereingabe

### Wann Drosselung nicht verwenden (When Not to Use Throttling)

Drosselung ist möglicherweise nicht die beste Wahl, wenn:
- Sie auf das Ende der Aktivität warten möchten (verwenden Sie stattdessen [Debouncing](../guides/debouncing))
- Sie sich keine verpassten Ausführungen leisten können (verwenden Sie stattdessen [Queueing](../guides/queueing))

> [!TIP]
> Drosselung ist oft die beste Wahl, wenn Sie gleichmäßige, konsistente Ausführungszeiten benötigen. Sie bietet ein vorhersehbareres Ausführungsmuster als Ratenbegrenzung und unmittelbareres Feedback als Entprellung.

## Drosselung in TanStack Pacer (Throttling in TanStack Pacer)

TanStack Pacer bietet sowohl synchrone als auch asynchrone Drosselung über die Klassen `Throttler` und `AsyncThrottler` (sowie die entsprechenden Funktionen `throttle` und `asyncThrottle`).

### Grundlegende Verwendung mit `throttle` (Basic Usage with `throttle`)

Die Funktion `throttle` ist der einfachste Weg, Drosselung zu einer beliebigen Funktion hinzuzufügen:

```ts
import { throttle } from '@tanstack/pacer'

// UI-Updates auf alle 200ms drosseln
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// In einer schnellen Schleife wird nur alle 200ms ausgeführt
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // Viele Aufrufe werden gedrosselt
}
```

### Erweiterte Verwendung mit der `Throttler`-Klasse (Advanced Usage with `Throttler` Class)

Für mehr Kontrolle über das Drosselungsverhalten können Sie die `Throttler`-Klasse direkt verwenden:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Informationen zum Ausführungsstatus abrufen
console.log(updateThrottler.getExecutionCount()) // Anzahl erfolgreicher Ausführungen
console.log(updateThrottler.getLastExecutionTime()) // Zeitstempel der letzten Ausführung

// Ausstehende Ausführung abbrechen
updateThrottler.cancel()
```

### Führende und nachfolgende Ausführungen (Leading and Trailing Executions)

Der synchrone Throttler unterstützt sowohl führende (leading) als auch nachfolgende (trailing) Ausführungen:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Sofortige Ausführung beim ersten Aufruf (Standard)
  trailing: true,  // Ausführung nach Wartezeit (Standard)
})
```

- `leading: true` (Standard) - Sofortige Ausführung beim ersten Aufruf
- `leading: false` - Ersten Aufruf überspringen, auf nachfolgende Ausführung warten
- `trailing: true` (Standard) - Letzten Aufruf nach Wartezeit ausführen
- `trailing: false` - Letzten Aufruf überspringen, falls innerhalb der Wartezeit

Gängige Muster:
- `{ leading: true, trailing: true }` - Standard, reaktionsschnellste Variante
- `{ leading: false, trailing: true }` - Alle Ausführungen verzögern
- `{ leading: true, trailing: false }` - Gequeute Ausführungen überspringen

### Aktivieren/Deaktivieren (Enabling/Disabling)

Die `Throttler`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie den Throttler jederzeit aktivieren oder deaktivieren:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Standardmäßig deaktiviert
throttler.setOptions({ enabled: true }) // Jederzeit aktivierbar
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Throttler-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Throttler dynamisch zu aktivieren/deaktivieren. Bei direkter Verwendung der `throttle`-Funktion oder der `Throttler`-Klasse müssen Sie jedoch die Methode `setOptions` verwenden, da die übergebenen Optionen an den Konstruktor der `Throttler`-Klasse übergeben werden.

### Callback-Optionen (Callback Options)

Sowohl die synchronen als auch die asynchronen Throttler unterstützen Callback-Optionen zur Handhabung verschiedener Aspekte des Drosselungslebenszyklus:

#### Synchrone Throttler-Callbacks (Synchronous Throttler Callbacks)

Der synchrone `Throttler` unterstützt folgenden Callback:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Funktion ausgeführt', throttler.getExecutionCount())
  }
})
```

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der gedrosselten Funktion aufgerufen und eignet sich zur Verfolgung von Ausführungen, Aktualisierung des UI-Zustands oder Bereinigungsoperationen.

#### Asynchrone Throttler-Callbacks (Asynchronous Throttler Callbacks)

Der asynchrone `AsyncThrottler` unterstützt zusätzliche Callbacks zur Fehlerbehandlung:

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Async-Funktion ausgeführt', throttler.getExecutionCount())
  },
  onError: (error) => {
    // Wird aufgerufen, wenn die Async-Funktion einen Fehler wirft
    console.error('Async-Funktion fehlgeschlagen:', error)
  }
})
```

Der `onExecute`-Callback funktioniert wie beim synchronen Throttler, während `onError` eine ordnungsgemäße Fehlerbehandlung ohne Unterbrechung der Drosselungskette ermöglicht. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszahlen, UI-Zustandsaktualisierungen, Fehlerbehandlung, Bereinigungsoperationen und Protokollierung.

### Asynchrone Drosselung (Asynchronous Throttling)

Für asynchrone Funktionen oder bei Bedarf an Fehlerbehandlung verwenden Sie `AsyncThrottler` oder `asyncThrottle`:

```ts
import { asyncThrottle } from '@tanstack/pacer'

const throttledFetch = asyncThrottle(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    wait: 1000,
    onError: (error) => {
      console.error('API-Aufruf fehlgeschlagen:', error)
    }
  }
)

// Führt nur einen API-Aufruf pro Sekunde aus
await throttledFetch('123')
```

Die asynchrone Version bietet Promise-basierte Ausführungsverfolgung, Fehlerbehandlung über den `onError`-Callback, ordnungsgemäße Bereinigung ausstehender Async-Operationen und eine await-fähige `maybeExecute`-Methode.

### Framework-Adapter (Framework Adapters)

Jeder Framework-Adapter bietet Hooks, die auf der Kern-Drosselungsfunktionalität aufbauen und sich in das State-Management-System des Frameworks integrieren. Für jedes Framework stehen Hooks wie `createThrottler`, `useThrottledCallback`, `useThrottledState` oder `useThrottledValue` zur Verfügung.

Hier einige Beispiele:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Low-Level-Hook für volle Kontrolle
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Einfacher Callback-Hook für Basis-Anwendungsfälle
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// State-basierter Hook für reaktives State-Management
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // Zu drosselnder Wert
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// Low-Level-Hook für volle Kontrolle
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Signal-basierter Hook für State-Management
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('Gesamtausführungen:', throttler.getExecutionCount())
  }
})
```

Jeder Framework-Adapter bietet Hooks, die sich in das State-Management-System des Frameworks integrieren, während die Kern-Drosselungsfunktionalität erhalten bleibt.
