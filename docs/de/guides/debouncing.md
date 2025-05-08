---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:56:25.603Z'
title: Anleitung zu Debouncing
id: debouncing
---
# Debouncing Guide (Leitfaden zum Entprellen)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellen) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind - das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung leistungsfähiger und zuverlässiger Anwendungen. Dieser Leitfaden behandelt die Debouncing-Konzepte von TanStack Pacer.

## Debouncing-Konzept (Debouncing Concept)

Debouncing ist eine Technik, die die Ausführung einer Funktion verzögert, bis eine bestimmte Phase der Inaktivität eingetreten ist. Im Gegensatz zur Ratenbegrenzung, die Ausführungen bis zu einem bestimmten Limit erlaubt, oder zur Drosselung, die gleichmäßig verteilte Ausführungen sicherstellt, fasst Debouncing mehrere schnelle Funktionsaufrufe zu einer einzigen Ausführung zusammen, die erst nach dem Ende der Aufrufe erfolgt. Dies macht Debouncing ideal für die Handhabung von Ereignisbursts, bei denen nur der Endzustand nach Abschluss der Aktivität relevant ist.

### Visuelle Darstellung des Debouncing (Debouncing Visualization)

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Ausführung hier nach
                                                         3 Ticks ohne Aufrufe

             [Burst von Aufrufen]     [Weitere Aufrufe]   [Warten]      [Neuer Burst]
             Keine Ausführung         Setzt Timer zurück   [Verzögerte Ausführung]  [Warten] [Verzögerte Ausführung]
```

### Wann Debouncing verwendet werden sollte (When to Use Debouncing)

Debouncing ist besonders effektiv, wenn Sie auf eine "Pause" in der Aktivität warten möchten, bevor Sie eine Aktion ausführen. Dies macht es ideal für die Handhabung von Benutzereingaben oder anderen schnell ausgelösten Ereignissen, bei denen nur der Endzustand relevant ist.

Häufige Anwendungsfälle sind:
- Suchfelder, bei denen gewartet werden soll, bis der Benutzer mit der Eingabe fertig ist
- Formularvalidierung, die nicht bei jedem Tastendruck ausgeführt werden soll
- Berechnungen bei Fenstergrößenänderungen, die rechenintensiv sind
- Automatisches Speichern von Entwürfen während der Bearbeitung von Inhalten
- API-Aufrufe, die erst nach Abschluss der Benutzeraktivität erfolgen sollen
- Jedes Szenario, bei dem nur der Endwert nach schnellen Änderungen relevant ist

### Wann Debouncing nicht verwendet werden sollte (When Not to Use Debouncing)

Debouncing ist möglicherweise nicht die beste Wahl, wenn:
- Sie eine garantierte Ausführung über einen bestimmten Zeitraum benötigen (verwenden Sie stattdessen [Drosselung](../guides/throttling))
- Sie es sich nicht leisten können, Ausführungen zu verpassen (verwenden Sie stattdessen [Warteschlange](../guides/queueing))

## Debouncing in TanStack Pacer (Debouncing in TanStack Pacer)

TanStack Pacer bietet sowohl synchrones als auch asynchrones Debouncing über die Klassen `Debouncer` und `AsyncDebouncer` (und ihre entsprechenden Funktionen `debounce` und `asyncDebounce`).

### Grundlegende Verwendung mit `debounce` (Basic Usage with `debounce`)

Die Funktion `debounce` ist die einfachste Möglichkeit, Debouncing zu einer beliebigen Funktion hinzuzufügen:

```ts
import { debounce } from '@tanstack/pacer'

// Suchfeld entprellen, um auf das Ende der Benutzereingabe zu warten
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

### Erweiterte Verwendung mit der `Debouncer`-Klasse (Advanced Usage with `Debouncer` Class)

Für mehr Kontrolle über das Debouncing-Verhalten können Sie die `Debouncer`-Klasse direkt verwenden:

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// Informationen über den aktuellen Zustand abrufen
console.log(searchDebouncer.getExecutionCount()) // Anzahl der erfolgreichen Ausführungen
console.log(searchDebouncer.getIsPending()) // Ob ein Aufruf aussteht

// Optionen dynamisch aktualisieren
searchDebouncer.setOptions({ wait: 1000 }) // Wartezeit erhöhen

// Ausstehende Ausführung abbrechen
searchDebouncer.cancel()
```

### Führende und nachfolgende Ausführungen (Leading and Trailing Executions)

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
- `trailing: true` (Standard) - Funktion wird nach Wartezeit ausgeführt
- `trailing: false` - Keine Ausführung nach Wartezeit

Gängige Muster:
- `{ leading: false, trailing: true }` - Standard, Ausführung nach Wartezeit
- `{ leading: true, trailing: false }` - Sofortige Ausführung, nachfolgende Aufrufe ignorieren
- `{ leading: true, trailing: true }` - Ausführung bei erstem Aufruf und nach Wartezeit

### Maximale Wartezeit (Max Wait Time)

Der TanStack Pacer Debouncer hat bewusst KEINE `maxWait`-Option wie andere Debouncing-Bibliotheken. Wenn Sie Ausführungen über einen längeren Zeitraum verteilen müssen, sollten Sie stattdessen die [Drosselung](../guides/throttling)-Technik verwenden.

### Aktivieren/Deaktivieren (Enabling/Disabling)

Die `Debouncer`-Klasse unterstützt das Aktivieren/Deaktivieren über die `enabled`-Option. Mit der Methode `setOptions` können Sie den Debouncer jederzeit aktivieren/deaktivieren:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
debouncer.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Debouncer-Optionen reaktiv sind, können Sie die `enabled`-Option auf einen bedingten Wert setzen, um den Debouncer dynamisch zu aktivieren/deaktivieren:

```ts
// React-Beispiel
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Basierend auf Eingabelänge aktivieren/deaktivieren, WENN ein Framework-Adapter verwendet wird, der reaktive Optionen unterstützt
)
```

Wenn Sie jedoch die Funktion `debounce` oder die `Debouncer`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die `enabled`-Option zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `Debouncer`-Klasse übergeben werden.

```ts
// Solid-Beispiel
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Basierend auf Eingabelänge aktivieren/deaktivieren
})
```

### Callback-Optionen (Callback Options)

Sowohl der synchrone als auch der asynchrone Debouncer unterstützen Callback-Optionen zur Handhabung verschiedener Aspekte des Debouncing-Lebenszyklus:

#### Callbacks des synchronen Debouncers (Synchronous Debouncer Callbacks)

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

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der entprellten Funktion aufgerufen und ist nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Zustands oder Bereinigungsoperationen.

#### Callbacks des asynchronen Debouncers (Asynchronous Debouncer Callbacks)

Der asynchrone `AsyncDebouncer` hat eine andere Reihe von Callbacks im Vergleich zur synchronen Version.

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Asynchrone Funktion ausgeführt', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // Wird nach jedem Ausführungsversuch aufgerufen
    console.log('Asynchrone Funktion abgeschlossen', debouncer.getSettledCount())
  },
  onError: (error) => {
    // Wird aufgerufen, wenn die asynchrone Funktion einen Fehler wirft
    console.error('Asynchrone Funktion fehlgeschlagen:', error)
  }
})
```

Der `onSuccess`-Callback wird nach jeder erfolgreichen Ausführung der entprellten Funktion aufgerufen, während der `onError`-Callback aufgerufen wird, wenn die asynchrone Funktion einen Fehler wirft. Der `onSettled`-Callback wird nach jedem Ausführungsversuch aufgerufen, unabhängig von Erfolg oder Fehler. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszählern, die Aktualisierung des UI-Zustands, die Fehlerbehandlung, Bereinigungsoperationen und die Protokollierung von Ausführungsmetriken.

### Asynchrones Debouncing (Asynchronous Debouncing)

Der asynchrone Debouncer bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Debouncing zu handhaben, und bietet mehrere wichtige Vorteile gegenüber der synchronen Version. Während der synchrone Debouncer ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für die Handhabung von API-Aufrufen, Datenbankoperationen und anderen asynchronen Aufgaben konzipiert.

#### Wichtige Unterschiede zum synchronen Debouncing (Key Differences from Synchronous Debouncing)

1. **Rückgabewertbehandlung**
Im Gegensatz zum synchronen Debouncer, der void zurückgibt, ermöglicht die asynchrone Version die Erfassung und Verwendung des Rückgabewerts Ihrer entprellten Funktion. Dies ist besonders nützlich, wenn Sie mit den Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie auf das Ergebnis warten und es entsprechend verarbeiten können.

2. **Unterschiedliche Callbacks**
Der `AsyncDebouncer` unterstützt folgende Callbacks anstelle von nur `onExecute` in der synchronen Version:
- `onSuccess`: Wird nach jeder erfolgreichen Ausführung aufgerufen, mit der Debouncer-Instanz
- `onSettled`: Wird nach jeder Ausführung aufgerufen, mit der Debouncer-Instanz
- `onError`: Wird aufgerufen, wenn die asynchrone Funktion einen Fehler wirft, mit dem Fehler und der Debouncer-Instanz

3. **Sequenzielle Ausführung**
Da die Methode `maybeExecute` des Debouncers ein Promise zurückgibt, können Sie wählen, ob Sie jede Ausführung abwarten möchten, bevor Sie die nächste starten. Dies gibt Ihnen Kontrolle über die Ausführungsreihenfolge und stellt sicher, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders nützlich bei Operationen, die von den Ergebnissen vorheriger Aufrufe abhängen oder bei denen die Datenkonsistenz kritisch ist.

Beispiel: Wenn Sie das Profil eines Benutzers aktualisieren und dann sofort seine aktualisierten Daten abrufen, können Sie auf den Abschluss der Aktualisierungsoperation warten, bevor Sie den Abruf starten:

#### Grundlegendes Anwendungsbeispiel (Basic Usage Example)

Hier ist ein grundlegendes Beispiel, das die Verwendung des asynchronen Debouncers für eine Suchoperation zeigt:

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('Suche erfolgreich:', results)
    },
    onError: (error, debouncer) => {
      console.error('Suche fehlgeschlagen:', error)
    }
  }
)

// Verwendung
const results = await debouncedSearch('query')
```

### Framework-Adapter (Framework Adapters)

Jeder Framework-Adapter bietet Hooks, die auf der Kern-Debouncing-Funktionalität aufbauen, um sich in das Zustandsmanagement-System des Frameworks zu integrieren. Hooks wie `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` oder `useDebouncedValue` sind für jedes Framework verfügbar.

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

// Zustandsbasierter Hook für reaktives Zustandsmanagement
const [instantState, setInstantState] = useState('')
const [debouncedValue] = useDebouncedValue(
  instantState, // Zu entprellender Wert
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

// Signalbasierter Hook für Zustandsmanagement
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('Gesamtausführungen:', debouncer.getExecutionCount())
  }
})
```
