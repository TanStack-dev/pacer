---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:14:40.500Z'
title: Debouncing-Anleitung
id: debouncing
---
# Debouncing Guide (Anleitung zum Entprellen)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellen) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind - das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung performanter und zuverlässiger Anwendungen. Diese Anleitung behandelt die Debouncing-Konzepte von TanStack Pacer.

## Debouncing-Konzept

Debouncing (Entprellen) ist eine Technik, die die Ausführung einer Funktion verzögert, bis eine bestimmte Zeit der Inaktivität verstrichen ist. Im Gegensatz zur Ratenbegrenzung, die Ausführungen bis zu einem Limit erlaubt, oder zur Drosselung, die gleichmäßig verteilte Ausführungen gewährleistet, fasst Debouncing mehrere schnelle Funktionsaufrufe zu einer einzigen Ausführung zusammen, die erst nach dem Ende der Aufrufe erfolgt. Dies macht Debouncing ideal für die Handhabung von Ereignisausbrüchen, bei denen nur der Endzustand nach Abschluss der Aktivität relevant ist.

### Debouncing-Visualisierung

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Ausführung hier nach
                                                         3 Ticks ohne Aufrufe

             [Aufrufausbruch]     [Weitere Aufrufe]   [Warten]      [Neuer Ausbruch]
             Keine Ausführung     Setzt Timer zurück    [Verzögerte Ausführung]  [Warten] [Verzögerte Ausführung]
```

### Wann Debouncing verwendet werden sollte

Debouncing ist besonders effektiv, wenn Sie auf eine "Pause" in der Aktivität warten möchten, bevor Sie eine Aktion ausführen. Dies macht es ideal für die Handhabung von Benutzereingaben oder anderen schnell aufeinanderfolgenden Ereignissen, bei denen nur der Endzustand relevant ist.

Häufige Anwendungsfälle sind:
- Suchfelder, bei denen gewartet werden soll, bis der Benutzer mit der Eingabe fertig ist
- Formularvalidierung, die nicht bei jedem Tastendruck ausgeführt werden soll
- Fenstergrößenanpassungsberechnungen, die rechenintensiv sind
- Automatisches Speichern von Entwürfen während der Bearbeitung von Inhalten
- API-Aufrufe, die erst nach Abschluss der Benutzeraktivität erfolgen sollen
- Jedes Szenario, in dem nur der Endwert nach schnellen Änderungen relevant ist

### Wann Debouncing nicht verwendet werden sollte

Debouncing ist möglicherweise nicht die beste Wahl, wenn:
- Sie eine garantierte Ausführung über einen bestimmten Zeitraum benötigen (verwenden Sie stattdessen [Throttling](../guides/throttling))
- Sie es sich nicht leisten können, Ausführungen zu verpassen (verwenden Sie stattdessen [Queueing](../guides/queueing))

## Debouncing in TanStack Pacer

TanStack Pacer bietet sowohl synchrones als auch asynchrones Debouncing über die Klassen `Debouncer` und `AsyncDebouncer` (und ihre entsprechenden Funktionen `debounce` und `asyncDebounce`).

### Grundlegende Verwendung mit `debounce`

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

### Erweiterte Verwendung mit der `Debouncer`-Klasse

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
- `trailing: true` (Standard) - Funktion wird nach Wartezeit ausgeführt
- `trailing: false` - Keine Ausführung nach Wartezeit

Gängige Muster:
- `{ leading: false, trailing: true }` - Standard, Ausführung nach Wartezeit
- `{ leading: true, trailing: false }` - Sofortige Ausführung, nachfolgende Aufrufe ignorieren
- `{ leading: true, trailing: true }` - Ausführung bei erstem Aufruf und nach Wartezeit

### Maximale Wartezeit

Der TanStack Pacer Debouncer hat bewusst KEINE `maxWait`-Option wie andere Debouncing-Bibliotheken. Wenn Sie Ausführungen über einen längeren Zeitraum verteilen müssen, sollten Sie stattdessen die [Throttling](../guides/throttling)-Technik verwenden.

### Aktivieren/Deaktivieren

Die `Debouncer`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie den Debouncer jederzeit aktivieren/deaktivieren:

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
debouncer.setOptions({ enabled: true }) // Jederzeit aktivieren
```

Wenn Sie einen Framework-Adapter verwenden, bei dem die Debouncer-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Debouncer dynamisch zu aktivieren/deaktivieren:

```ts
// React-Beispiel
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Basierend auf Eingabelänge aktivieren/deaktivieren, WENN ein Framework-Adapter verwendet wird, der reaktive Optionen unterstützt
)
```

Wenn Sie jedoch die Funktion `debounce` oder die `Debouncer`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die Option `enabled` zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `Debouncer`-Klasse übergeben werden.

```ts
// Solid-Beispiel
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Standardmäßig deaktiviert
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Basierend auf Eingabelänge aktivieren/deaktivieren
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

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der entprellten Funktion aufgerufen und ist nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Zustands oder Bereinigungsoperationen.

#### Callbacks des asynchronen Debouncers

Der asynchrone `AsyncDebouncer` hat im Vergleich zur synchronen Version einen anderen Satz von Callbacks.

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // Wird nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Async-Funktion ausgeführt', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // Wird nach jedem Ausführungsversuch aufgerufen
    console.log('Async-Funktion abgeschlossen', debouncer.getSettledCount())
  },
  onError: (error) => {
    // Wird aufgerufen, wenn die Async-Funktion einen Fehler wirft
    console.error('Async-Funktion fehlgeschlagen:', error)
  }
})
```

Der `onSuccess`-Callback wird nach jeder erfolgreichen Ausführung der entprellten Funktion aufgerufen, während der `onError`-Callback aufgerufen wird, wenn die Async-Funktion einen Fehler wirft. Der `onSettled`-Callback wird nach jedem Ausführungsversuch aufgerufen, unabhängig von Erfolg oder Fehler. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszahlen, die Aktualisierung des UI-Zustands, die Fehlerbehandlung, Bereinigungsoperationen und die Protokollierung von Ausführungsmetriken.

### Asynchrones Debouncing

Der asynchrone Debouncer bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Debouncing zu handhaben und bietet mehrere wichtige Vorteile gegenüber der synchronen Version. Während der synchrone Debouncer ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für die Handhabung von API-Aufrufen, Datenbankoperationen und anderen asynchronen Aufgaben konzipiert.

#### Wichtige Unterschiede zum synchronen Debouncing

1. **Rückgabewertbehandlung**
Im Gegensatz zum synchronen Debouncer, der void zurückgibt, ermöglicht die asynchrone Version die Erfassung und Verwendung des Rückgabewerts Ihrer entprellten Funktion. Dies ist besonders nützlich, wenn Sie mit den Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie auf das Ergebnis warten und es entsprechend verarbeiten können.

2. **Erweitertes Callback-System**
Der asynchrone Debouncer bietet ein ausgefeilteres Callback-System im Vergleich zum einzelnen `onExecute`-Callback der synchronen Version. Dieses System umfasst:
- `onSuccess`: Wird aufgerufen, wenn die Async-Funktion erfolgreich abgeschlossen wird, mit Ergebnis und Debouncer-Instanz
- `onError`: Wird aufgerufen, wenn die Async-Funktion einen Fehler wirft, mit Fehler und Debouncer-Instanz
- `onSettled`: Wird nach jedem Ausführungsversuch aufgerufen, unabhängig von Erfolg oder Fehler

3. **Ausführungsverfolgung**
Der asynchrone Debouncer bietet eine umfassende Ausführungsverfolgung durch mehrere Methoden:
- `getSuccessCount()`: Anzahl der erfolgreichen Ausführungen
- `getErrorCount()`: Anzahl der fehlgeschlagenen Ausführungen
- `getSettledCount()`: Gesamtzahl der abgeschlossenen Ausführungen (Erfolg + Fehler)

4. **Sequenzielle Ausführung**
Der asynchrone Debouncer stellt sicher, dass nachfolgende Ausführungen warten, bis der vorherige Aufruf abgeschlossen ist. Dies verhindert eine nicht sequenzielle Ausführung und garantiert, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders wichtig, wenn Operationen von den Ergebnissen vorheriger Aufrufe abhängen oder die Datenkonsistenz kritisch ist.

Beispiel: Wenn Sie das Profil eines Benutzers aktualisieren und dann sofort dessen aktualisierte Daten abrufen, stellt der asynchrone Debouncer sicher, dass der Abrufvorgang auf die Aktualisierung wartet, wodurch Race Conditions verhindert werden, bei denen Sie möglicherweise veraltete Daten erhalten.

#### Grundlegendes Anwendungsbeispiel

Hier ein einfaches Beispiel für die Verwendung des asynchronen Debouncers für eine Suchoperation:

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

#### Erweiterte Muster

Der asynchrone Debouncer kann mit verschiedenen Mustern kombiniert werden, um komplexe Probleme zu lösen:

1. **Integration der Zustandsverwaltung**
Bei der Verwendung des asynchronen Debouncers mit Zustandsverwaltungssystemen (wie Reacts useState oder Solids createSignal) können Sie leistungsstarke Muster für die Handhabung von Ladezuständen, Fehlerzuständen und Datenaktualisierungen erstellen. Die Callbacks des Debouncers bieten perfekte Hooks für die Aktualisierung des UI-Zustands basierend auf dem Erfolg oder Misserfolg von Operationen.

2. **Verhinderung von Race Conditions**
Das Single-Flight-Mutation-Muster verhindert natürlicherweise Race Conditions in vielen Szenarien. Wenn mehrere Teile Ihrer Anwendung versuchen, dieselbe Ressource gleichzeitig zu aktualisieren, stellt der Debouncer sicher, dass nur die neueste Aktualisierung tatsächlich erfolgt, während dennoch Ergebnisse an alle Aufrufer zurückgegeben werden.

3. **Fehlerbehebung**
Die Fehlerbehandlungsfähigkeiten des asynchronen Debouncers machen ihn ideal für die Implementierung von Wiederholungslogik und Fehlerbehebungsmustern. Sie können den `onError`-Callback verwenden, um benutzerdefinierte Fehlerbehandlungsstrategien wie exponentielles Backoff oder Fallback-Mechanismen zu implementieren.

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kern-Debouncing-Funktionalität aufbauen, um sich in das Zustandsverwaltungssystem des Frameworks zu integrieren. Für jedes Framework sind Hooks wie `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` oder `useDebouncedValue` verfügbar.

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

// Zustandsbasierter Hook für reaktive Zustandsverwaltung
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

// Signalbasierter Hook für Zustandsverwaltung
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('Gesamtausführungen:', debouncer.getExecutionCount())
  }
})
```
