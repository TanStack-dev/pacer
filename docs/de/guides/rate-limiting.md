---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:14:42.635Z'
title: Ratenbegrenzungs-Anleitung
id: rate-limiting
---
# Rate Limiting Guide (Anleitung zur Ratenbegrenzung)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz anzuwenden ist, ist entscheidend für die Entwicklung performanter und zuverlässiger Anwendungen. Diese Anleitung behandelt die Rate-Limiting-Konzepte von TanStack Pacer.

> [!NOTE]
> TanStack Pacer ist derzeit nur eine Frontend-Bibliothek. Dies sind Hilfsmittel für clientseitige Ratenbegrenzung.

## Konzept der Ratenbegrenzung

Rate Limiting (Ratenbegrenzung) ist eine Technik, die die Rate begrenzt, mit der eine Funktion innerhalb eines bestimmten Zeitfensters ausgeführt werden kann. Sie ist besonders nützlich für Szenarien, in denen verhindert werden soll, dass eine Funktion zu häufig aufgerufen wird, z. B. bei der Abwicklung von API-Anfragen oder anderen Aufrufen externer Dienste. Es ist der *einfachste* Ansatz, da er Ausführungen in Bursts erlaubt, bis das Kontingent erschöpft ist.

### Visualisierung der Ratenbegrenzung

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     �️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Wann Ratenbegrenzung verwendet werden sollte

Rate Limiting (Ratenbegrenzung) ist besonders wichtig, wenn es um Frontend-Operationen geht, die versehentlich Backend-Dienste überlasten oder Leistungsprobleme im Browser verursachen könnten.

Häufige Anwendungsfälle sind:
- Verhinderung versehentlicher API-Spam durch schnelle Benutzerinteraktionen (z. B. Button-Klicks oder Formularübermittlungen)
- Szenarien, in denen burstartiges Verhalten akzeptabel ist, aber die maximale Rate begrenzt werden soll
- Schutz vor versehentlichen Endlosschleifen oder rekursiven Operationen

### Wann Ratenbegrenzung nicht verwendet werden sollte

Rate Limiting (Ratenbegrenzung) ist der einfachste Ansatz zur Steuerung der Ausführungsfrequenz von Funktionen. Es ist die unflexibelste und restriktivste der drei Techniken. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing) für gleichmäßiger verteilte Ausführungen.

> [!TIP]
> In den meisten Fällen möchten Sie wahrscheinlich keine "Ratenbegrenzung" verwenden. Erwägen Sie stattdessen die Verwendung von [Throttling](../guides/throttling) oder [Debouncing](../guides/debouncing).

Die "verlustbehaftete" Natur der Ratenbegrenzung bedeutet auch, dass einige Ausführungen abgelehnt und verloren gehen. Dies kann ein Problem sein, wenn Sie sicherstellen müssen, dass alle Ausführungen immer erfolgreich sind. Erwägen Sie die Verwendung von [Queueing](../guides/queueing), wenn Sie sicherstellen müssen, dass alle Ausführungen in eine Warteschlange gestellt werden, um mit einer gedrosselten Verzögerung ausgeführt zu werden, um die Ausführungsrate zu verlangsamen.

## Ratenbegrenzung in TanStack Pacer

TanStack Pacer bietet sowohl synchrone als auch asynchrone Ratenbegrenzung über die Klassen `RateLimiter` und `AsyncRateLimiter` (und ihre entsprechenden Funktionen `rateLimit` und `asyncRateLimit`).

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

// Versuch, auszuführen (gibt einen Boolean zurück, der den Erfolg anzeigt)
limiter.maybeExecute('user-1')

// Optionen dynamisch aktualisieren
limiter.setOptions({ limit: 10 }) // Das Limit erhöhen

// Alle Zähler und den Status zurücksetzen
limiter.reset()
```

### Aktivieren/Deaktivieren

Die `RateLimiter`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie den Rate-Limiter jederzeit aktivieren/deaktivieren:

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Standardmäßig deaktiviert
})
limiter.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Rate-Limiter-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Rate-Limiter dynamisch zu aktivieren/deaktivieren. Wenn Sie jedoch die Funktion `rateLimit` oder die `RateLimiter`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die Option `enabled` zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `RateLimiter`-Klasse übergeben werden.

### Callback-Optionen

Sowohl die synchronen als auch die asynchronen Rate-Limiter unterstützen Callback-Optionen zur Behandlung verschiedener Aspekte des Ratenbegrenzungs-Lebenszyklus:

#### Callbacks des synchronen Rate-Limiters

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

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der rate-limitierten Funktion aufgerufen, während der `onReject`-Callback aufgerufen wird, wenn eine Ausführung aufgrund der Ratenbegrenzung abgelehnt wird. Diese Callbacks sind nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Status oder die Bereitstellung von Feedback für Benutzer.

#### Callbacks des asynchronen Rate-Limiters

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

Die Callbacks `onExecute` und `onReject` funktionieren auf die gleiche Weise wie beim synchronen Rate-Limiter, während der `onError`-Callback es Ihnen ermöglicht, Fehler elegant zu behandeln, ohne die Ratenbegrenzungskette zu unterbrechen. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszählern, die Aktualisierung des UI-Status, die Fehlerbehandlung und die Bereitstellung von Feedback für Benutzer.

### Asynchrone Ratenbegrenzung

Der asynchrone Rate-Limiter bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Ratenbegrenzung zu handhaben, und bietet mehrere entscheidende Vorteile gegenüber der synchronen Version. Während der synchrone Rate-Limiter ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für die Handhabung von API-Aufrufen, Datenbankoperationen und anderen asynchronen Aufgaben konzipiert.

#### Wichtige Unterschiede zur synchronen Ratenbegrenzung

1. **Handhabung von Rückgabewerten**
Im Gegensatz zum synchronen Rate-Limiter, der einen Boolean zurückgibt, der den Erfolg anzeigt, ermöglicht die asynchrone Version die Erfassung und Verwendung des Rückgabewerts Ihrer rate-limitierten Funktion. Dies ist besonders nützlich, wenn Sie mit den Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie auf das Ergebnis warten und es entsprechend verarbeiten können.

2. **Erweitertes Callback-System**
Der asynchrone Rate-Limiter bietet ein ausgefeilteres Callback-System im Vergleich zu den Callbacks der synchronen Version. Dieses System umfasst:
- `onExecute`: Wird nach jeder erfolgreichen Ausführung aufgerufen und stellt die Rate-Limiter-Instanz bereit
- `onReject`: Wird aufgerufen, wenn eine Ausführung aufgrund der Ratenbegrenzung abgelehnt wird, und stellt die Rate-Limiter-Instanz bereit
- `onError`: Wird aufgerufen, wenn die asynchrone Funktion einen Fehler wirft, und stellt sowohl den Fehler als auch die Rate-Limiter-Instanz bereit

3. **Ausführungsverfolgung**
Der asynchrone Rate-Limiter bietet eine umfassende Ausführungsverfolgung durch mehrere Methoden:
- `getExecutionCount()`: Anzahl der erfolgreichen Ausführungen
- `getRejectionCount()`: Anzahl der abgelehnten Ausführungen
- `getRemainingInWindow()`: Anzahl der verbleibenden Ausführungen im aktuellen Fenster
- `getMsUntilNextWindow()`: Millisekunden bis zum Start des nächsten Fensters

4. **Sequenzielle Ausführung**
Der asynchrone Rate-Limiter stellt sicher, dass nachfolgende Ausführungen warten, bis der vorherige Aufruf abgeschlossen ist, bevor sie beginnen. Dies verhindert eine nicht sequenzielle Ausführung und garantiert, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders wichtig, wenn es um Operationen geht, die von den Ergebnissen vorheriger Aufrufe abhängen oder wenn die Datenkonsistenz kritisch ist.

Wenn Sie beispielsweise das Profil eines Benutzers aktualisieren und dann sofort dessen aktualisierte Daten abrufen, stellt der asynchrone Rate-Limiter sicher, dass der Abrufvorgang auf die Fertigstellung der Aktualisierung wartet, wodurch Race Conditions verhindert werden, bei denen Sie möglicherweise veraltete Daten erhalten.

#### Grundlegendes Anwendungsbeispiel

Hier ist ein grundlegendes Beispiel, das zeigt, wie der asynchrone Rate-Limiter für eine API-Operation verwendet wird:

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

#### Erweiterte Muster

Der asynchrone Rate-Limiter kann mit verschiedenen Mustern kombiniert werden, um komplexe Probleme zu lösen:

1. **Integration der Zustandsverwaltung**
Wenn Sie den asynchronen Rate-Limiter mit Zustandsverwaltungssystemen (wie Reacts useState oder Solids createSignal) verwenden, können Sie leistungsstarke Muster für die Handhabung von Ladezuständen, Fehlerzuständen und Datenaktualisierungen erstellen. Die Callbacks des Rate-Limiters bieten perfekte Hooks für die Aktualisierung des UI-Status basierend auf dem Erfolg oder Misserfolg von Operationen.

2. **Verhinderung von Race Conditions**
Das Ratenbegrenzungsmuster verhindert auf natürliche Weise Race Conditions in vielen Szenarien. Wenn mehrere Teile Ihrer Anwendung versuchen, dieselbe Ressource gleichzeitig zu aktualisieren, stellt der Rate-Limiter sicher, dass die Aktualisierungen innerhalb der konfigurierten Grenzen erfolgen, während dennoch Ergebnisse an alle Aufrufer zurückgegeben werden.

3. **Fehlerbehebung**
Die Fehlerbehandlungsfähigkeiten des asynchronen Rate-Limiters machen ihn ideal für die Implementierung von Wiederholungslogik und Fehlerbehebungsmustern. Sie können den `onError`-Callback verwenden, um benutzerdefinierte Fehlerbehandlungsstrategien zu implementieren, wie z. B. exponentielles Backoff oder Fallback-Mechanismen.

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kernfunktionalität der Ratenbegrenzung aufbauen, um sich in das Zustandsverwaltungssystem des Frameworks zu integrieren. Hooks wie `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` oder `useRateLimitedValue` sind für jedes Framework verfügbar.

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

// Zustandsbasierter Hook für reaktive Zustandsverwaltung
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // Zu begrenzender Wert
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@
