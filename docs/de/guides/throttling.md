---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:14:29.456Z'
title: Throttling-Anleitung
id: throttling
---
# Throttling Guide (Drosselungsleitfaden)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, was sie "verlustbehaftet" macht - das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung leistungsfähiger und zuverlässiger Anwendungen. Dieser Leitfaden behandelt die Throttling-Konzepte von TanStack Pacer.

## Throttling-Konzept (Drosselungskonzept)

Throttling stellt sicher, dass Funktionsausführungen gleichmäßig über die Zeit verteilt sind. Im Gegensatz zur Ratenbegrenzung, die Ausführungsbursts bis zu einem Limit erlaubt, oder zur Entprellung, die auf das Ende der Aktivität wartet, erzeugt Throttling ein gleichmäßigeres Ausführungsmuster durch konsequente Verzögerungen zwischen den Aufrufen. Wenn Sie eine Drosselung von einer Ausführung pro Sekunde festlegen, werden die Aufrufe gleichmäßig verteilt, unabhängig davon, wie schnell sie angefordert werden.

### Throttling-Visualisierung (Drosselungsvisualisierung)

```text
Throttling (one execution per 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Executed:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Nur eine Ausführung alle 3 Ticks erlaubt,
               unabhängig von der Anzahl der Aufrufe

             [Erster Burst]    [Weitere Aufrufe]        [Gleichmäßige Aufrufe]
             Erste Ausführung  Ausführung nach          Ausführung jeweils nach
             dann drosseln     Wartezeit                Ablauf der Wartezeit
```

### Wann Throttling verwenden

Throttling ist besonders effektiv, wenn Sie konsistente, vorhersehbare Ausführungszeiten benötigen. Dies macht es ideal für die Handhabung häufiger Ereignisse oder Updates, bei denen Sie ein gleichmäßiges, kontrolliertes Verhalten wünschen.

Häufige Anwendungsfälle sind:
- UI-Updates, die konsistente Timing benötigen (z.B. Fortschrittsanzeigen)
- Scroll- oder Resize-Event-Handler, die den Browser nicht überlasten sollten
- Echtzeit-Datenabfragen mit konsistenten Intervallen
- Ressourcenintensive Operationen mit gleichmäßiger Geschwindigkeit
- Spielschleifen-Updates oder Animation Frame Handling
- Live-Suchvorschläge während der Eingabe

### Wann Throttling nicht verwenden

Throttling ist möglicherweise nicht die beste Wahl, wenn:
- Sie auf das Ende der Aktivität warten möchten (verwenden Sie stattdessen [Debouncing](../guides/debouncing))
- Sie es sich nicht leisten können, Ausführungen zu verpassen (verwenden Sie stattdessen [Queueing](../guides/queueing))

> [!TIP]
> Throttling ist oft die beste Wahl, wenn Sie gleichmäßige, konsistente Ausführungszeiten benötigen. Es bietet ein vorhersehbareres Ausführungsmuster als Ratenbegrenzung und unmittelbareres Feedback als Entprellung.

## Throttling in TanStack Pacer

TanStack Pacer bietet sowohl synchrones als auch asynchrones Throttling durch die Klassen `Throttler` und `AsyncThrottler` (und ihre entsprechenden Funktionen `throttle` und `asyncThrottle`).

### Grundlegende Verwendung mit `throttle`

Die `throttle`-Funktion ist der einfachste Weg, um Throttling zu einer Funktion hinzuzufügen:

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

### Erweiterte Verwendung mit der `Throttler`-Klasse

Für mehr Kontrolle über das Throttling-Verhalten können Sie die `Throttler`-Klasse direkt verwenden:

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Informationen über den Ausführungsstatus abrufen
console.log(updateThrottler.getExecutionCount()) // Anzahl erfolgreicher Ausführungen
console.log(updateThrottler.getLastExecutionTime()) // Zeitstempel der letzten Ausführung

// Ausstehende Ausführung abbrechen
updateThrottler.cancel()
```

### Führende und nachfolgende Ausführungen

Der synchrone Throttler unterstützt sowohl führende als auch nachfolgende Ausführungen:

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Sofort bei erstem Aufruf ausführen (Standard)
  trailing: true,  // Nach Wartezeit ausführen (Standard)
})
```

- `leading: true` (Standard) - Sofort bei erstem Aufruf ausführen
- `leading: false` - Ersten Aufruf überspringen, auf nachfolgende Ausführung warten
- `trailing: true` (Standard) - Letzten Aufruf nach Wartezeit ausführen
- `trailing: false` - Letzten Aufruf überspringen, wenn innerhalb der Wartezeit

Gängige Muster:
- `{ leading: true, trailing: true }` - Standard, am reaktionsschnellsten
- `{ leading: false, trailing: true }` - Alle Ausführungen verzögern
- `{ leading: true, trailing: false }` - Gequeue Ausführungen überspringen

### Aktivieren/Deaktivieren

Die `Throttler`-Klasse unterstützt das Aktivieren/Deaktivieren über die Option `enabled`. Mit der Methode `setOptions` können Sie den Throttler jederzeit aktivieren/deaktivieren:

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Standardmäßig deaktiviert
throttler.setOptions({ enabled: true }) // Jederzeit aktivierbar
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Throttler-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Throttler dynamisch zu aktivieren/deaktivieren. Wenn Sie jedoch die `throttle`-Funktion oder die `Throttler`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die `enabled`-Option zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `Throttler`-Klasse übergeben werden.

### Callback-Optionen

Sowohl die synchronen als auch die asynchronen Throttler unterstützen Callback-Optionen zur Handhabung verschiedener Aspekte des Throttling-Lebenszyklus:

#### Synchroner Throttler-Callbacks

Der synchrone `Throttler` unterstützt folgenden Callback:

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // Nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Funktion ausgeführt', throttler.getExecutionCount())
  }
})
```

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der gedrosselten Funktion aufgerufen, was ihn nützlich für die Verfolgung von Ausführungen, die Aktualisierung des UI-Zustands oder Bereinigungsoperationen macht.

#### Asynchrone Throttler-Callbacks

Der asynchrone `AsyncThrottler` unterstützt zusätzliche Callbacks für die Fehlerbehandlung:

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // Nach jeder erfolgreichen Ausführung aufgerufen
    console.log('Async-Funktion ausgeführt', throttler.getExecutionCount())
  },
  onError: (error) => {
    // Bei Fehlern der Async-Funktion aufgerufen
    console.error('Async-Funktion fehlgeschlagen:', error)
  }
})
```

Der `onExecute`-Callback funktioniert wie beim synchronen Throttler, während der `onError`-Callback eine elegante Fehlerbehandlung ohne Unterbrechung der Throttling-Kette ermöglicht. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszählern, UI-Zustandsaktualisierungen, Fehlerbehandlung, Bereinigungsoperationen und die Protokollierung von Ausführungsmetriken.

### Asynchrones Throttling

Der asynchrone Throttler bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Throttling zu handhaben, mit mehreren Vorteilen gegenüber der synchronen Version. Während der synchrone Throttler ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für API-Aufrufe, Datenbankoperationen und andere asynchrone Aufgaben konzipiert.

#### Wichtige Unterschiede zum synchronen Throttling

1. **Rückgabewertbehandlung**
Anders als der synchrone Throttler, der void zurückgibt, erlaubt die asynchrone Version die Erfassung und Verwendung des Rückgabewerts Ihrer gedrosselten Funktion. Dies ist besonders nützlich, wenn Sie mit Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie auf das Ergebnis warten und es entsprechend verarbeiten können.

2. **Erweitertes Callback-System**
Der asynchrone Throttler bietet ein ausgefeilteres Callback-System im Vergleich zum einzelnen `onExecute`-Callback der synchronen Version. Dieses System umfasst:
- `onSuccess`: Wird aufgerufen, wenn die Async-Funktion erfolgreich abgeschlossen wird, mit Ergebnis und Throttler-Instanz
- `onError`: Wird bei Fehlern der Async-Funktion aufgerufen, mit Fehler und Throttler-Instanz
- `onSettled`: Wird nach jedem Ausführungsversuch aufgerufen, unabhängig von Erfolg oder Misserfolg

3. **Ausführungsverfolgung**
Der asynchrone Throttler bietet umfassende Ausführungsverfolgung durch mehrere Methoden:
- `getSuccessCount()`: Anzahl erfolgreicher Ausführungen
- `getErrorCount()`: Anzahl fehlgeschlagener Ausführungen
- `getSettledCount()`: Gesamtanzahl abgeschlossener Ausführungen (Erfolg + Fehler)

4. **Sequenzielle Ausführung**
Der asynchrone Throttler stellt sicher, dass nachfolgende Ausführungen auf den Abschluss des vorherigen Aufrufs warten. Dies verhindert Out-of-Order-Ausführungen und garantiert, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders wichtig bei Operationen, die von den Ergebnissen vorheriger Aufrufe abhängen oder bei denen die Datenkonsistenz kritisch ist.

Zum Beispiel, wenn Sie ein Benutzerprofil aktualisieren und dann sofort die aktualisierten Daten abrufen, sorgt der asynchrone Throttler dafür, dass der Abrufvorgang auf den Abschluss der Aktualisierung wartet, wodurch Race Conditions vermieden werden, bei denen Sie veraltete Daten erhalten könnten.

#### Grundlegendes Anwendungsbeispiel

Hier ein einfaches Beispiel für die Verwendung des asynchronen Throttlers für eine Suchoperation:

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('Suche erfolgreich:', results)
    },
    onError: (error, throttler) => {
      console.error('Suche fehlgeschlagen:', error)
    }
  }
)

// Verwendung
const results = await throttledSearch('query')
```

#### Erweiterte Muster

Der asynchrone Throttler kann mit verschiedenen Mustern kombiniert werden, um komplexe Probleme zu lösen:

1. **Integration mit State Management**
Bei der Verwendung des asynchronen Throttlers mit State-Management-Systemen (wie Reacts useState oder Solids createSignal) können Sie leistungsstarke Muster für die Handhabung von Ladezuständen, Fehlerzuständen und Datenaktualisierungen erstellen. Die Callbacks des Throttlers bieten ideale Hooks für die Aktualisierung des UI-Zustands basierend auf dem Erfolg oder Misserfolg von Operationen.

2. **Race Condition-Prävention**
Das Throttling-Muster verhindert natürlicherweise Race Conditions in vielen Szenarien. Wenn mehrere Teile Ihrer Anwendung versuchen, dieselbe Ressource gleichzeitig zu aktualisieren, stellt der Throttler sicher, dass Aktualisierungen mit kontrollierter Rate erfolgen, während Ergebnisse trotzdem an alle Aufrufer zurückgegeben werden.

3. **Fehlerbehebung**
Die Fehlerbehandlungsfähigkeiten des asynchronen Throttlers machen ihn ideal für die Implementierung von Wiederholungslogik und Fehlerbehebungsmustern. Sie können den `onError`-Callback verwenden, um benutzerdefinierte Fehlerbehandlungsstrategien wie exponentielles Backoff oder Fallback-Mechanismen zu implementieren.

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der Kern-Throttling-Funktionalität aufbauen und sich in das State-Management-System des Frameworks integrieren. Hooks wie `createThrottler`, `useThrottledCallback`, `useThrottledState` oder `useThrottledValue` sind für jedes Framework verfügbar.

Hier einige Beispiele:

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Low-Level-Hook für volle Kontrolle
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Einfacher Callback-Hook für grundlegende Anwendungsfälle
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// State-basierter Hook für reaktives State-Management
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
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

Jeder Framework-Adapter bietet Hooks, die sich in das State-Management-System des Frameworks integrieren, während die Kern-Throttling-Funktionalität erhalten bleibt.
