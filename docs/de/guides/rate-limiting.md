---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:30:58.461Z'
title: Ratenbegrenzungs-Anleitung
id: rate-limiting
---
# Rate Limiting Guide (Anleitung zur Ratenbegrenzung)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei unterschiedliche Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, was sie "verlustbehaftet" macht – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung performanter und zuverlässiger Anwendungen. Diese Anleitung behandelt die Rate-Limiting-Konzepte von TanStack Pacer.

> [!NOTE]
> TanStack Pacer ist derzeit nur eine Frontend-Bibliothek. Dies sind Hilfsmittel für clientseitige Ratenbegrenzung.

## Konzept der Ratenbegrenzung

Rate Limiting (Ratenbegrenzung) ist eine Technik, die die Rate begrenzt, mit der eine Funktion innerhalb eines bestimmten Zeitfensters ausgeführt werden kann. Sie ist besonders nützlich für Szenarien, in denen verhindert werden soll, dass eine Funktion zu häufig aufgerufen wird, z. B. bei der Abwicklung von API-Anfragen oder anderen Aufrufen externer Dienste. Es ist der *naivste* Ansatz, da er Ausführungen in Bursts erlaubt, bis das Kontingent erschöpft ist.

### Visualisierung der Ratenbegrenzung

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Wann Ratenbegrenzung verwendet werden sollte

Rate Limiting (Ratenbegrenzung) ist besonders wichtig, wenn es um Frontend-Operationen geht, die versehentlich Ihre Backend-Dienste überlasten oder Leistungsprobleme im Browser verursachen könnten.

Häufige Anwendungsfälle sind:
- Verhindern von versehentlichem API-Spam durch schnelle Benutzerinteraktionen (z. B. Button-Klicks oder Formularübermittlungen)
- Szenarien, in denen burstartiges Verhalten akzeptabel ist, aber die maximale Rate begrenzt werden soll
- Schutz vor versehentlichen Endlosschleifen oder rekursiven Operationen

### Wann Ratenbegrenzung nicht verwendet werden sollte

Rate Limiting (Ratenbegrenzung) ist der naivste Ansatz zur Steuerung der Ausführungsfrequenz von Funktionen. Es ist die unflexibelste und restriktivste der drei Techniken. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing) für besser verteilte Ausführungen.

> [!TIP]
> In den meisten Fällen möchten Sie wahrscheinlich keine "Ratenbegrenzung" verwenden. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing).

Die "verlustbehaftete" Natur der Ratenbegrenzung bedeutet auch, dass einige Ausführungen abgelehnt und verloren gehen. Dies kann ein Problem sein, wenn Sie sicherstellen müssen, dass alle Ausführungen immer erfolgreich sind. Erwägen Sie die Verwendung von [Queueing](../guides/queueing), wenn Sie sicherstellen müssen, dass alle Ausführungen in einer Warteschlange für die Ausführung eingereiht werden, jedoch mit einer gedrosselten Verzögerung, um die Ausführungsrate zu verlangsamen.

## Ratenbegrenzung in TanStack Pacer

TanStack Pacer bietet sowohl synchrone als auch asynchrone Ratenbegrenzung über die Klassen `RateLimiter` bzw. `AsyncRateLimiter` (und ihre entsprechenden Funktionen `rateLimit` und `asyncRateLimit`).

### Grundlegende Verwendung mit `rateLimit`

Die Funktion `rateLimit` ist die einfachste Möglichkeit, einer beliebigen Funktion eine Ratenbegrenzung hinzuzufügen. Sie ist ideal für die meisten Anwendungsfälle, in denen Sie lediglich eine einfache Begrenzung durchsetzen müssen.

```ts
import { rateLimit } from '@tanstack/pacer'

// API-Aufrufe auf 5 pro Minute begrenzen
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 Minute in Millisekunden
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Die ersten 5 Aufrufe werden sofort ausgeführt
rateLimitedApi('user-1') // ✅ Wird ausgeführt
rateLimitedApi('user-2') // ✅ Wird ausgeführt
rateLimitedApi('user-3') // ✅ Wird ausgeführt
rateLimitedApi('user-4') // ✅ Wird ausgeführt
rateLimitedApi('user-5') // ✅ Wird ausgeführt
rateLimitedApi('user-6') // ❌ Wird abgelehnt, bis das Fenster zurückgesetzt wird
```

### Erweiterte Verwendung mit der `RateLimiter`-Klasse

Für komplexere Szenarien, in denen Sie zusätzliche Kontrolle über das Ratenbegrenzungsverhalten benötigen, können Sie die Klasse `RateLimiter` direkt verwenden. Dies gibt Ihnen Zugriff auf zusätzliche Methoden und Statusinformationen.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Eine RateLimiter-Instanz erstellen
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Function executed', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Informationen über den aktuellen Status abrufen
console.log(limiter.getRemainingInWindow()) // Anzahl der verbleibenden Aufrufe im aktuellen Fenster
console.log(limiter.getExecutionCount()) // Gesamtanzahl der erfolgreichen Ausführungen
console.log(limiter.getRejectionCount()) // Gesamtanzahl der abgelehnten Ausführungen

// Versuch einer Ausführung (gibt einen Boolean zurück, der den Erfolg anzeigt)
limiter.maybeExecute('user-1')

// Optionen dynamisch aktualisieren
limiter.setOptions({ limit: 10 }) // Das Limit erhöhen

// Alle Zähler und den Status zurücksetzen
limiter.reset()
```

### Aktivieren/Deaktivieren

Die Klasse `RateLimiter` unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie die Ratenbegrenzung jederzeit aktivieren/deaktivieren:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Standardmäßig deaktiviert
})
limiter.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Rate-Limiter-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um die Ratenbegrenzung dynamisch zu aktivieren/deaktivieren. Wenn Sie jedoch die Funktion `rateLimit` oder die Klasse `RateLimiter` direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die Option `enabled` zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der Klasse `RateLimiter` übergeben werden.

### Callback-Optionen

Sowohl die synchronen als auch die asynchronen Ratenbegrenzer unterstützen Callback-Optionen zur Behandlung verschiedener Aspekte des Ratenbegrenzungs-Lebenszyklus:

#### Callbacks des synchronen Ratenbegrenzers

Der synchrone `RateLimiter` unterstützt die folgenden Callbacks:

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Wird aufgerufen, wenn eine Ausführung abgelehnt wird
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der rate-limited-Funktion aufgerufen, während der `onReject`-Callback aufgerufen wird, wenn eine Ausführung aufgrund der Ratenbegrenzung abgelehnt wird. Diese Callbacks sind nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Status oder die Bereitstellung von Feedback für Benutzer.

#### Callbacks des asynchronen Ratenbegrenzers

Der asynchrone `AsyncRateLimiter` unterstützt zusätzliche Callbacks für die Fehlerbehandlung:

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Async function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Wird aufgerufen, wenn eine Ausführung abgelehnt wird
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // Wird aufgerufen, wenn die asynchrone Funktion einen Fehler wirft
    console.error('Async function failed:', error)
  }
})
```

Die Callbacks `onExecute` und `onReject` funktionieren auf die gleiche Weise wie beim synchronen Ratenbegrenzer, während der `onError`-Callback es Ihnen ermöglicht, Fehler elegant zu behandeln, ohne die Ratenbegrenzungskette zu unterbrechen. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszählern, die Aktualisierung des UI-Status, die Fehlerbehandlung und die Bereitstellung von Feedback für Benutzer.

### Asynchrone Ratenbegrenzung

Verwenden Sie `AsyncRateLimiter`, wenn:
- Ihre rate-limited-Funktion ein Promise zurückgibt
- Sie Fehler aus der asynchronen Funktion behandeln müssen
- Sie eine ordnungsgemäße Ratenbegrenzung sicherstellen möchten, auch wenn die asynchrone Funktion Zeit zur Ausführung benötigt

```ts
import { asyncRateLimit } from '@tanstack/pacer'

const rateLimited = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onError: (error) => {
      console.error('API call failed:', error)
    }
  }
)

// Gibt ein Promise<boolean> zurück - wird mit true aufgelöst, wenn ausgeführt, mit false, wenn abgelehnt
const wasExecuted = await rateLimited('123')
```

Die asynchrone Version bietet Promise-basierte Ausführungsverfolgung, Fehlerbehandlung über den `onError`-Callback, ordnungsgemäße Bereinigung ausstehender asynchroner Operationen und eine awaitable `maybeExecute`-Methode.

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kernfunktionalität der Ratenbegrenzung aufbauen und sich in das State-Management-System des Frameworks integrieren. Für jedes Framework sind Hooks wie `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` oder `useRateLimitedValue` verfügbar.

Hier einige Beispiele:

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// Low-Level-Hook für volle Kontrolle
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Einfacher Callback-Hook für grundlegende Anwendungsfälle
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// State-basierter Hook für reaktives State-Management
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // Zu begrenzender Wert
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// Low-Level-Hook für volle Kontrolle
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Signal-basierter Hook für State-Management
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
