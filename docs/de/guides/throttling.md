---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:56:10.559Z'
title: Anleitung zu Throttling
id: throttling
---
# Throttling Guide (Drosselungsleitfaden)

Rate Limiting (Ratenbegrenzung), Throttling (Drosselung) und Debouncing (Entprellung) sind drei verschiedene Ansätze zur Steuerung der Ausführungsfrequenz von Funktionen. Jede Technik blockiert Ausführungen auf unterschiedliche Weise, wodurch sie "verlustbehaftet" sind – das bedeutet, dass einige Funktionsaufrufe nicht ausgeführt werden, wenn sie zu häufig angefordert werden. Das Verständnis, wann welcher Ansatz verwendet werden sollte, ist entscheidend für die Entwicklung leistungsfähiger und zuverlässiger Anwendungen. Dieser Leitfaden behandelt die Throttling-Konzepte von TanStack Pacer.

## Throttling-Konzept

Throttling stellt sicher, dass Funktionsausführungen gleichmäßig über die Zeit verteilt werden. Im Gegensatz zur Ratenbegrenzung, die Ausführungsbursts bis zu einem Limit erlaubt, oder zur Entprellung, die auf das Ende der Aktivität wartet, erzeugt Throttling ein gleichmäßigeres Ausführungsmuster durch konsequente Verzögerungen zwischen den Aufrufen. Wenn Sie eine Drosselung von einer Ausführung pro Sekunde festlegen, werden die Aufrufe gleichmäßig verteilt, unabhängig davon, wie schnell sie angefordert werden.

### Throttling-Visualisierung

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

### Wann Throttling verwendet werden sollte

Throttling ist besonders effektiv, wenn Sie konsistente, vorhersehbare Ausführungszeiten benötigen. Dies macht es ideal für die Handhabung häufiger Ereignisse oder Updates, bei denen Sie ein gleichmäßiges, kontrolliertes Verhalten wünschen.

Häufige Anwendungsfälle sind:
- UI-Updates, die konsistente Timing benötigen (z.B. Fortschrittsanzeigen)
- Scroll- oder Resize-Event-Handler, die den Browser nicht überlasten sollten
- Echtzeit-Datenabfragen mit gewünschten konsistenten Intervallen
- Ressourcenintensive Operationen, die eine gleichmäßige Geschwindigkeit benötigen
- Spielschleifen-Updates oder Animation Frame Handling
- Live-Suchvorschläge während der Eingabe

### Wann Throttling nicht verwendet werden sollte

Throttling ist möglicherweise nicht die beste Wahl, wenn:
- Sie auf das Ende der Aktivität warten möchten (verwenden Sie stattdessen [Debouncing](../guides/debouncing))
- Sie sich keine verpassten Ausführungen leisten können (verwenden Sie stattdessen [Queueing](../guides/queueing))

> [!TIP]
> Throttling ist oft die beste Wahl, wenn Sie ein gleichmäßiges, konsistentes Ausführungs-Timing benötigen. Es bietet ein vorhersehbareres Ausführungsmuster als Ratenbegrenzung und ein unmittelbareres Feedback als Entprellung.

## Throttling in TanStack Pacer

TanStack Pacer bietet sowohl synchrones als auch asynchrones Throttling durch die Klassen `Throttler` und `AsyncThrottler` (sowie deren entsprechende Funktionen `throttle` und `asyncThrottle`).

### Grundlegende Verwendung mit `throttle`

Die Funktion `throttle` ist der einfachste Weg, um Throttling zu einer Funktion hinzuzufügen:

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
  leading: true,   // Sofortige Ausführung beim ersten Aufruf (Standard)
  trailing: true,  // Ausführung nach Wartezeit (Standard)
})
```

- `leading: true` (Standard) - Sofortige Ausführung beim ersten Aufruf
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

Wenn Sie ein Framework-Adapter verwenden, bei dem die Throttler-Optionen reaktiv sind, können Sie die Option `enabled` auf einen bedingten Wert setzen, um den Throttler dynamisch zu aktivieren/deaktivieren. Wenn Sie jedoch die Funktion `throttle` oder die `Throttler`-Klasse direkt verwenden, müssen Sie die Methode `setOptions` verwenden, um die `enabled`-Option zu ändern, da die übergebenen Optionen tatsächlich an den Konstruktor der `Throttler`-Klasse übergeben werden.

### Callback-Optionen

Sowohl der synchrone als auch der asynchrone Throttler unterstützen Callback-Optionen zur Handhabung verschiedener Aspekte des Throttling-Lebenszyklus:

#### Synchroner Throttler-Callbacks

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

Der `onExecute`-Callback wird nach jeder erfolgreichen Ausführung der gedrosselten Funktion aufgerufen und eignet sich somit für die Verfolgung von Ausführungen, die Aktualisierung des UI-Zustands oder Bereinigungsoperationen.

#### Asynchrone Throttler-Callbacks

Der asynchrone `AsyncThrottler` unterstützt zusätzliche Callbacks für die Fehlerbehandlung:

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

Der `onExecute`-Callback funktioniert genauso wie beim synchronen Throttler, während der `onError`-Callback eine elegante Fehlerbehandlung ohne Unterbrechung der Throttling-Kette ermöglicht. Diese Callbacks sind besonders nützlich für die Verfolgung von Ausführungszahlen, die Aktualisierung des UI-Zustands, die Fehlerbehandlung, Bereinigungsoperationen und die Protokollierung von Ausführungsmetriken.

### Asynchrones Throttling

Der asynchrone Throttler bietet eine leistungsstarke Möglichkeit, asynchrone Operationen mit Throttling zu handhaben, mit mehreren Vorteilen gegenüber der synchronen Version. Während der synchrone Throttler ideal für UI-Ereignisse und sofortiges Feedback ist, ist die asynchrone Version speziell für API-Aufrufe, Datenbankoperationen und andere asynchrone Aufgaben konzipiert.

#### Wichtige Unterschiede zum synchronen Throttling

1. **Rückgabewertbehandlung**
Anders als der synchrone Throttler, der void zurückgibt, ermöglicht die asynchrone Version die Erfassung und Nutzung des Rückgabewerts Ihrer gedrosselten Funktion. Dies ist besonders nützlich, wenn Sie mit Ergebnissen von API-Aufrufen oder anderen asynchronen Operationen arbeiten müssen. Die Methode `maybeExecute` gibt ein Promise zurück, das mit dem Rückgabewert der Funktion aufgelöst wird, sodass Sie auf das Ergebnis warten und es entsprechend verarbeiten können.

2. **Unterschiedliche Callbacks**
Der `AsyncThrottler` unterstützt folgende Callbacks anstelle von nur `onExecute` in der synchronen Version:
- `onSuccess`: Wird nach jeder erfolgreichen Ausführung aufgerufen, mit der Throttler-Instanz
- `onSettled`: Wird nach jeder Ausführung aufgerufen, mit der Throttler-Instanz
- `onError`: Wird aufgerufen, wenn die Async-Funktion einen Fehler wirft, mit dem Fehler und der Throttler-Instanz

Sowohl der asynchrone als auch der synchrone Throttler unterstützen den `onExecute`-Callback für die Handhabung erfolgreicher Ausführungen.

3. **Sequenzielle Ausführung**
Da die Methode `maybeExecute` des Throttlers ein Promise zurückgibt, können Sie wählen, ob Sie jede Ausführung abwarten möchten, bevor Sie die nächste starten. Dies gibt Ihnen Kontrolle über die Ausführungsreihenfolge und stellt sicher, dass jeder Aufruf die aktuellsten Daten verarbeitet. Dies ist besonders nützlich bei Operationen, die von den Ergebnissen vorheriger Aufrufe abhängen oder bei denen die Datenkonsistenz kritisch ist.

Zum Beispiel, wenn Sie das Profil eines Benutzers aktualisieren und dann sofort die aktualisierten Daten abrufen, können Sie die Update-Operation abwarten, bevor Sie den Abruf starten:

#### Grundlegendes Anwendungsbeispiel

Hier ein grundlegendes Beispiel für die Verwendung des asynchronen Throttlers für eine Suchoperation:

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

### Framework-Adapter

Jeder Framework-Adapter bietet Hooks, die auf der grundlegenden Throttling-Funktionalität aufbauen und sich in das State-Management-System des Frameworks integrieren. Hooks wie `createThrottler`, `useThrottledCallback`, `useThrottledState` oder `useThrottledValue` sind für jedes Framework verfügbar.

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

Jeder Framework-Adapter bietet Hooks, die sich in das State-Management-System des Frameworks integrieren, während die grundlegende Throttling-Funktionalität erhalten bleibt.
