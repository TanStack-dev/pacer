---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:30:55.310Z'
title: Debouncing-Anleitung
id: debouncing
---
# Debouncing Guide (Übersetzung)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung leistungsfähiger und zuverlässiger Anwendungen. Dieser Leitfaden behandelt die Debouncing-Konzepte von TanStack Pacer.

## Debouncing-Konzept

Debouncing (Entprellung) ist eine Technik, die die Ausführung einer Funktion verzögert, bis eine bestimmte Phase der Inaktivität eingetreten ist. Im Gegensatz zur Ratenbegrenzung, die Ausführungen bis zu einem bestimmten Limit erlaubt, oder zur Drosselung, die gleichmäßig verteilte Ausführungen sicherstellt, fasst Debouncing mehrere schnelle Funktionsaufrufe zu einer einzigen Ausführung zusammen, die erst nach dem Ende der Aufrufe erfolgt. Dies macht Debouncing ideal für die Handhabung von Ereignisbursts, bei denen nur der Endzustand nach Beendigung der Aktivität relevant ist.

### Debouncing-Visualisierung

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Ausführung hier nach
                                                         3 Ticks ohne Aufrufe

             [Burst von Aufrufen]     [Weitere Aufrufe]   [Warten]      [Neuer Burst]
             Keine Ausführung         Timer zurücksetzen   [Verzögerte Ausführung]  [Warten] [Verzögerte Ausführung]
```

### Wann Debouncing verwendet werden sollte

Debouncing ist besonders effektiv, wenn Sie auf eine "Pause" in der Aktivität warten möchten, bevor eine Aktion ausgeführt wird. Dies macht es ideal für die Handhabung von Benutzereingaben oder anderen schnell aufeinanderfolgenden Ereignissen, bei denen nur der Endzustand relevant ist.

Häufige Anwendungsfälle sind:
- Suchfelder, bei denen gewartet werden soll, bis der Benutzer mit der Eingabe fertig ist
- Formularvalidierung, die nicht bei jedem Tastendruck ausgeführt werden soll
- Berechnungen bei Fenstergrößenänderungen, die rechenintensiv sind
- Automatisches Speichern von Entwürfen während der Bearbeitung von Inhalten
- API-Aufrufe, die erst nach Beendigung der Benutzeraktivität erfolgen sollen
- Jedes Szenario, in dem nur der Endwert nach schnellen Änderungen relevant ist

### Wann Debouncing nicht verwendet werden sollte

Debouncing ist möglicherweise nicht die beste Wahl, wenn:
- Sie eine garantierte Ausführung über einen bestimmten Zeitraum benötigen (verwenden Sie stattdessen [Throttling](../guides/throttling))
- Sie es sich nicht leisten können, Ausführungen zu verpassen (verwenden Sie stattdessen [Queueing](../guides/queueing))

## Debouncing in TanStack Pacer

TanStack Pacer bietet sowohl synchrone als auch asynchrone Debouncing-Funktionalität über die Klassen `Debouncer` und `AsyncDebouncer` (sowie deren entsprechende Funktionen `debounce` und `asyncDebounce`).

### Grundlegende Verwendung mit `debounce`

Die Funktion `debounce` ist die einfachste Möglichkeit, Debouncing zu einer beliebigen Funktion hinzuzufügen:

```ts
import { debounce } from '@tanstack/pacer'

// Suchfunktion mit Debouncing, um auf das Ende der Benutzereingabe zu warten
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // 500ms nach dem letzten Tastendruck warten
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### Erweiterte Verwendung mit der `Debouncer`-Klasse

Für eine detailliertere Steuerung des Debouncing-Verhaltens kann die `Debouncer`-Klasse direkt verwendet werden:

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// Informationen über den aktuellen Status abrufen
console.log(searchDebouncer.getExecutionCount()) // Anzahl der erfolgreichen Ausführungen
console.log(searchDebouncer.getIsPending()) // Ob ein Aufruf aussteht

// Optionen dynamisch aktualisieren
searchDebouncer.setOptions({ wait: 1000 }) // Wartezeit erhöhen

// Ausstehende Ausführung abbrechen
searchDebouncer.cancel()
```

### Führende und nachfolgende Ausführungen

Der synchrone Debouncer unterstützt sowohl führende als auch nachfolgende Ausführungen:

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // Bei erstem Aufruf ausführen
  trailing: true,  // Nach Wartezeit ausführen
})
```

- `leading: true` - Funktion wird sofort beim ersten Aufruf ausgeführt
- `leading: false` (Standard) - Erster Aufruf startet den Warte-Timer
- `trailing: true` (Standard) - Funktion wird nach der Wartezeit ausgeführt
- `trailing: false` - Keine Ausführung nach der Wartezeit

Gängige Muster:
- `{ leading: false, trailing: true }` - Standard, Ausführung nach Wartezeit
- `{ leading: true, trailing: false }` - Sofortige Ausführung, nachfolgende Aufrufe ignorieren
- `{ leading: true, trailing: true }` - Ausführung bei erstem Aufruf und nach Wartezeit

### Maximale Wartezeit

Der TanStack Pacer Debouncer hat KEINE `maxWait`-Option wie andere Debouncing-Bibliotheken. Wenn Sie Ausführungen über einen längeren Zeitraum verteilen müssen, sollten Sie stattdessen die [Throttling](../guides/throttling)-Technik verwenden.

### Aktivieren/Deaktivieren

Die `Debouncer`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` kann der Debouncer jederzeit aktiviert oder deaktiviert werden:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
debouncer.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Debouncer-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Debouncer dynamisch zu aktivieren/deaktivieren:

```ts
// React-Beispiel
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Aktivieren/Deaktivieren basierend auf der Eingabelänge, WENN ein Framework-Adapter verwendet wird, der reaktive Optionen unterstützt
)
```

Wenn Sie jedoch die Funktion `debounce` oder die `Debouncer`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die Option `enabled` zu ändern, da die übergebenen Optionen an den Konstruktor der `Debouncer`-Klasse übergeben werden.

```ts
// Solid-Beispiel
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Aktivieren/Deaktivieren basierend auf der Eingabelänge
})
```

### Callback-Optionen

Sowohl der synchrone als auch der asynchrone Debouncer unterstützen Callback-Optionen zur Handhabung verschiedener Aspekte des Debouncing-Lebenszyklus:

#### Callbacks des synchronen Debouncers

Der synchrone `Debouncer` unterstützt folgenden Callback:

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Funktion ausgeführt', debouncer.getExecutionCount())
  }
})
```

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der debounced-Funktion aufgerufen und eignet sich daher für die Verfolgung von Ausführungen, die Aktualisierung des UI-Status oder Bereinigungsoperationen.

#### Callbacks des asynchronen Debouncers

Der asynchrone `AsyncDebouncer` unterstützt zusätzliche Callbacks für die Fehlerbehandlung:

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Async-Funktion ausgeführt', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // Wird aufgerufen, wenn die Async-Funktion einen Fehler wirft
    console.error('Async-Funktion fehlgeschlagen:', error)
  }
})
```

Der `onExecute`-Callback funktioniert wie beim synchronen Debouncer, während der `onError`-Callback eine elegante Fehlerbehandlung ohne Unterbrechung der Debouncing-Kette ermöglicht. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszahlen, die Aktualisierung des UI-Status, die Fehlerbehandlung, Bereinigungsoperationen und die Protokollierung von Ausführungsmetriken.

### Asynchrones Debouncing

Für asynchrone Funktionen oder wenn Fehlerbehandlung benötigt wird, verwenden Sie `AsyncDebouncer` oder `asyncDebounce`:

```ts
import { asyncDebounce } from '@tanstack/pacer'

const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    updateUI(results)
  },
  {
    wait: 500,
    onError: (error) => {
      console.error('Suche fehlgeschlagen:', error)
    }
  }
)

// Führt nur einen API-Aufruf aus, nachdem die Eingabe beendet wurde
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

Die asynchrone Version bietet Promise-basierte Ausführungsverfolgung, Fehlerbehandlung über den `onError`-Callback, ordnungsgemäße Bereinigung ausstehender asynchroner Operationen und eine awaitable `maybeExecute`-Methode.

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kern-Debouncing-Funktionalität aufbauen und sich in das State-Management-System des Frameworks integrieren. Hooks wie `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` oder `useDebouncedValue` sind für jedes Framework verfügbar.

Hier einige Beispiele:

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// Low-Level-Hook für volle Kontrolle
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// Einfacher Callback-Hook für grundlegende Anwendungsfälle
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// State-basierter Hook für reaktives State-Management
const [instantState, setInstantState] = useState('')
const [debouncedState, setDebouncedState] = useDebouncedValue(
  instantState, // Zu debouncender Wert
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// Low-Level-Hook für volle Kontrolle
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// Signal-basierter Hook für State-Management
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('Gesamte Ausführungen:', debouncer.getExecutionCount())
  }
})
```
