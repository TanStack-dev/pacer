---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:56:39.060Z'
title: Anleitung zur Ratenbegrenzung
id: rate-limiting
---
# Rate Limiting Guide (Anleitung zur Ratenbegrenzung)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz anzuwenden ist, ist entscheidend für die Entwicklung leistungsfähiger und zuverlässiger Anwendungen. Diese Anleitung behandelt die Rate-Limiting-Konzepte von TanStack Pacer.

> [!NOTE]
> TanStack Pacer ist derzeit nur eine Front-End-Bibliothek. Dies sind Hilfsmittel für clientseitige Ratenbegrenzung.

## Konzept der Ratenbegrenzung

Rate Limiting ist eine Technik, die die Rate begrenzt, mit der eine Funktion innerhalb eines bestimmten Zeitfensters ausgeführt werden kann. Sie ist besonders nützlich für Szenarien, in denen verhindert werden soll, dass eine Funktion zu häufig aufgerufen wird, z. B. bei der Abwicklung von API-Anfragen oder anderen Aufrufen externer Dienste. Es handelt sich um den *naivsten* Ansatz, da er Ausführungen in Bursts erlaubt, bis das Kontingent erschöpft ist.

### Visualisierung der Ratenbegrenzung

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Fenstertypen

TanStack Pacer unterstützt zwei Arten von Ratenbegrenzungsfenstern:

1. **Festes Fenster (Fixed Window)** (Standard)
   - Ein strenges Fenster, das nach dem Fensterzeitraum zurückgesetzt wird
   - Alle Ausführungen innerhalb des Fensters zählen zur Begrenzung
   - Das Fenster wird nach dem Zeitraum vollständig zurückgesetzt
   - Kann zu burstartigem Verhalten an den Fenstergrenzen führen

2. **Gleitendes Fenster (Sliding Window)**
   - Ein rollierendes Fenster, das Ausführungen zulässt, sobald ältere ablaufen
   - Ermöglicht eine gleichmäßigere Ausführungsrate über die Zeit
   - Besser geeignet, um einen stetigen Ausführungsfluss aufrechtzuerhalten
   - Verhindert burstartiges Verhalten an den Fenstergrenzen

Hier eine Visualisierung der Ratenbegrenzung mit gleitendem Fenster:

```text
Sliding Window Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ✅                             ✅     ✅
             [=== 3 allowed ===][=== oldest expires, new allowed ===][=== continues sliding =======]
```

Der entscheidende Unterschied besteht darin, dass bei einem gleitenden Fenster eine neue Ausführung zugelassen wird, sobald die älteste Ausführung abläuft. Dies führt im Vergleich zum festen Fensteransatz zu einem gleichmäßigeren Ausführungsfluss.

### Wann Ratenbegrenzung verwendet werden sollte

Rate Limiting ist besonders wichtig bei Front-End-Operationen, die versehentlich Ihre Back-End-Dienste überlasten oder Leistungsprobleme im Browser verursachen könnten.

Häufige Anwendungsfälle sind:
- Verhindern von versehentlichem API-Spam durch schnelle Benutzerinteraktionen (z. B. Button-Klicks oder Formularübermittlungen)
- Szenarien, in denen burstartiges Verhalten akzeptabel ist, aber die maximale Rate begrenzt werden soll
- Schutz vor versehentlichen Endlosschleifen oder rekursiven Operationen

### Wann Ratenbegrenzung nicht verwendet werden sollte

Rate Limiting ist der naivste Ansatz zur Steuerung der Ausführungsfrequenz von Funktionen. Es ist die unflexibelste und restriktivste der drei Techniken. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing) für gleichmäßiger verteilte Ausführungen.

> [!TIP]
> In den meisten Fällen möchten Sie wahrscheinlich keine "Ratenbegrenzung" verwenden. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing).

Die "verlustbehaftete" Natur der Ratenbegrenzung bedeutet auch, dass einige Ausführungen abgelehnt und verloren gehen. Dies kann ein Problem sein, wenn Sie sicherstellen müssen, dass alle Ausführungen immer erfolgreich sind. Erwägen Sie die Verwendung von [Queueing](../guides/queueing), wenn Sie sicherstellen müssen, dass alle Ausführungen in die Warteschlange gestellt werden, um mit einer gedrosselten Verzögerung ausgeführt zu werden, die die Ausführungsrate verlangsamt.

## Ratenbegrenzung in TanStack Pacer

TanStack Pacer bietet sowohl synchrone als auch asynchrone Ratenbegrenzung über die Klassen `RateLimiter` bzw. `AsyncRateLimiter` (und die entsprechenden Funktionen `rateLimit` und `asyncRateLimit`).

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
    windowType: 'fixed', // Standard
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

Für komplexere Szenarien, in denen Sie zusätzliche Kontrolle über das Ratenbegrenzungsverhalten benötigen, können Sie die `RateLimiter`-Klasse direkt verwenden. Dies gibt Ihnen Zugriff auf zusätzliche Methoden und Statusinformationen.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Eine Rate-Limiter-Instanz erstellen
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
console.log(limiter.getExecutionCount()) // Gesamtzahl der erfolgreichen Ausführungen
console.log(limiter.getRejectionCount()) // Gesamtzahl der abgelehnten Ausführungen

// Versuch einer Ausführung (gibt einen booleschen Wert zurück, der den Erfolg anzeigt)
limiter.maybeExecute('user-1')

// Optionen dynamisch aktualisieren
limiter.setOptions({ limit: 10 }) // Das Limit erhöhen

// Alle Zähler und den Status zurücksetzen
limiter.reset()
```

### Aktivieren/Deaktivieren

Die `RateLimiter`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie den Rate Limiter jederzeit aktivieren/deaktivieren:

> [!NOTE]
> Die Option `enabled` aktiviert/deaktiviert die tatsächliche Funktionsausführung. Das Deaktivieren des Rate Limiters schaltet die Ratenbegrenzung nicht aus, sondern verhindert lediglich die Ausführung der Funktion.

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Standardmäßig deaktiviert
})
limiter.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Rate-Limiter-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Rate Limiter dynamisch zu aktivieren/deaktivieren. Wenn Sie jedoch die Funktion `rateLimit` oder die Klasse `RateLimiter` direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die Option `enabled` zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `RateLimiter`-Klasse übergeben werden.

### Callback-Optionen

Sowohl die synchronen als auch die asynchronen Rate Limiter unterstützen Callback-Optionen zur Behandlung verschiedener Aspekte des Ratenbegrenzungs-Lebenszyklus:

#### Callbacks des synchronen Rate Limiters

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

Der Callback `onExecute` wird nach jeder erfolgreichen Ausführung der rate-limited-Funktion aufgerufen, während der Callback `onReject` aufgerufen wird, wenn eine Ausführung aufgrund der Ratenbegrenzung abgelehnt wird. Diese Callbacks sind nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Status oder die Bereitstellung von Feedback für Benutzer.

#### Callbacks des asynchronen Rate Limiters

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

Die Callbacks `onExecute` und `onReject` funktionieren auf die gleiche Weise wie beim synchronen Rate Limiter, während der Callback `onError` es Ihnen ermöglicht, Fehler elegant zu behandeln, ohne die Ratenbegrenzungskette zu unterbrechen. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszählern, die Aktualisierung des UI-Status, die Fehlerbehandlung und die Bereitstellung von Feedback für Benutzer.

### Asynchrone Ratenbegrenzung

Der asynchrone Rate Limiter bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Ratenbegrenzung zu handhaben, und bietet mehrere entscheidende Vorteile gegenüber der synchronen Version. Während der synchrone Rate Limiter ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für die Handhabung von API-Aufrufen, Datenbankoperationen und anderen asynchronen Aufgaben konzipiert.

#### Wichtige Unterschiede zur synchronen Ratenbegrenzung

1. **Behandlung von Rückgabewerten**
Im Gegensatz zum synchronen Rate Limiter, der einen booleschen Wert zurückgibt, der den Erfolg anzeigt, ermöglicht die asynchrone Version die Erfassung und Verwendung des Rückgabewerts Ihrer rate-limited-Funktion. Dies ist besonders nützlich, wenn Sie mit den Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie das Ergebnis erwarten und entsprechend behandeln können.

2. **Unterschiedliche Callbacks**
Der `AsyncRateLimiter` unterstützt die folgenden Callbacks anstelle von nur `onExecute` in der synchronen Version:
- `onSuccess`: Wird nach jeder erfolgreichen Ausführung aufgerufen und stellt die Rate-Limiter-Instanz bereit
- `onSettled`: Wird nach jeder Ausführung aufgerufen und stellt die Rate-Limiter-Instanz bereit
- `onError`: Wird aufgerufen, wenn die asynchrone Funktion einen Fehler wirft, und stellt sowohl den Fehler als auch die Rate-Limiter-Instanz bereit

Sowohl die asynchronen als auch die synchronen Rate Limiter unterstützen den Callback `onReject` für die Behandlung blockierter Ausführungen.

3. **Sequenzielle Ausführung**
Da die Methode `maybeExecute` des Rate Limiters ein Promise zurückgibt, können Sie wählen, ob Sie jede Ausführung abwarten möchten, bevor Sie die nächste starten. Dies gibt Ihnen Kontrolle über die Ausführungsreihenfolge und stellt sicher, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders nützlich, wenn Sie mit Operationen arbeiten, die von den Ergebnissen vorheriger Aufrufe abhängen, oder wenn die Aufrechterhaltung der Datenkonsistenz kritisch ist.

Beispielsweise, wenn Sie das Profil eines Benutzers aktualisieren und dann sofort dessen aktualisierte Daten abrufen, können Sie die Aktualisierungsoperation abwarten, bevor Sie den Abruf starten:

#### Grundlegendes Anwendungsbeispiel

Hier ein grundlegendes Beispiel, das zeigt, wie der asynchrone Rate Limiter für eine API-Operation verwendet wird:

```ts
const rateLimitedApi = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onExecute: (limiter) => {
      console.log('API call succeeded:', limiter.getExecutionCount())
    },
    onReject: (limiter) => {
      console.log(`Rate limit exceeded. Try again in ${limiter.getMsUntilNextWindow()}ms`)
    },
    onError: (error, limiter) => {
      console.error('API call failed:', error)
    }
  }
)

// Verwendung
const result = await rateLimitedApi('123')
```

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kernfunktionalität der Ratenbegrenzung aufbauen, um sich in das State-Management-System des Frameworks zu integrieren. Für jedes Framework sind Hooks wie `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` oder `useRateLimitedValue` verfügbar.

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
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // Zu begrenzender Wert
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// Low-Level-Hook
