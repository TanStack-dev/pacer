---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:14:29.485Z'
title: Asynchrone Warteschlangen-Anleitung
id: async-queueing
---
# Asynchrones Queueing (Asynchronous Queueing) Guide

Während der [Queuer](../guides//queueing) synchrones Queueing mit Zeitsteuerung bietet, ist der `AsyncQueuer` speziell für die Handhabung gleichzeitiger asynchroner Operationen konzipiert. Er implementiert das traditionell als "Task Pool" oder "Worker Pool" bekannte Muster, das die gleichzeitige Verarbeitung mehrerer Operationen ermöglicht, während die Kontrolle über Parallelität und Timing erhalten bleibt. Die Implementierung ist größtenteils von [Swimmer](https://github.com/tannerlinsley/swimmer) übernommen, Tanners ursprüngliche Task-Pooling-Utility, die seit 2017 der JavaScript-Community dient.

## Konzept des Asynchronen Queueings

Asynchrones Queueing erweitert das grundlegende Queueing-Konzept durch die Hinzufügung von Fähigkeiten zur parallelen Verarbeitung. Anstatt ein Element nach dem anderen zu verarbeiten, kann ein asynchroner Queuer mehrere Elemente gleichzeitig verarbeiten und dabei die Reihenfolge und Kontrolle über die Ausführung beibehalten. Dies ist besonders nützlich bei I/O-Operationen, Netzwerkanfragen oder Aufgaben, die die meiste Zeit mit Warten verbringen, anstatt CPU-Ressourcen zu verbrauchen.

### Visualisierung des Asynchronen Queueings

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ Im Gegensatz zum regulären Queueing können mehrere Elemente
               gleichzeitig verarbeitet werden

             [Elemente reihen sich ein]   [2 gleichzeitig verarbeiten]   [Alle Elemente abschließen]
              wenn beschäftigt            mit Wartezeit dazwischen
```

### Wann Asynchrones Queueing verwendet werden sollte

Asynchrones Queueing ist besonders effektiv, wenn Sie:
- Mehrere asynchrone Operationen parallel verarbeiten müssen
- Die Anzahl der gleichzeitigen Operationen steuern müssen
- Promise-basierte Aufgaben mit ordnungsgemäßer Fehlerbehandlung handhaben
- Die Reihenfolge beibehalten und gleichzeitig den Durchsatz maximieren möchten
- Hintergrundaufgaben verarbeiten, die parallel ausgeführt werden können

Häufige Anwendungsfälle sind:
- Gleichzeitige API-Anfragen mit Ratenbegrenzung
- Gleichzeitige Verarbeitung mehrerer Datei-Uploads
- Parallele Datenbankoperationen
- Handhabung mehrerer WebSocket-Verbindungen
- Verarbeitung von Datenströmen mit Backpressure
- Verwaltung ressourcenintensiver Hintergrundaufgaben

### Wann Asynchrones Queueing nicht verwendet werden sollte

Der `AsyncQueuer` ist sehr vielseitig und kann in vielen Situationen eingesetzt werden. Er ist wirklich nur dann nicht geeignet, wenn Sie nicht planen, alle seine Funktionen zu nutzen. Wenn nicht alle in der Warteschlange eingereihten Ausführungen durchlaufen müssen, verwenden Sie stattdessen [Throttling][../guides/throttling]. Wenn Sie keine parallele Verarbeitung benötigen, verwenden Sie stattdessen [Queueing][../guides/queueing].

## Asynchrones Queueing in TanStack Pacer

TanStack Pacer bietet asynchrones Queueing über die einfache `asyncQueue`-Funktion und die leistungsfähigere `AsyncQueuer`-Klasse.

### Grundlegende Verwendung mit `asyncQueue`

Die `asyncQueue`-Funktion bietet eine einfache Möglichkeit, eine ständig laufende asynchrone Warteschlange zu erstellen:

```ts
import { asyncQueue } from '@tanstack/pacer'

// Erstelle eine Warteschlange, die bis zu 2 Elemente gleichzeitig verarbeitet
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('Aktive Aufgaben:', queuer.getActiveItems().length)
  }
})

// Füge asynchrone Aufgaben zur Verarbeitung hinzu
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

Die Verwendung der `asyncQueue`-Funktion ist etwas eingeschränkt, da sie nur ein Wrapper um die `AsyncQueuer`-Klasse ist, der nur die `addItem`-Methode verfügbar macht. Für mehr Kontrolle über die Warteschlange verwenden Sie die `AsyncQueuer`-Klasse direkt.

### Erweiterte Verwendung mit der `AsyncQueuer`-Klasse

Die `AsyncQueuer`-Klasse bietet vollständige Kontrolle über das Verhalten der asynchronen Warteschlange:

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // Verarbeite 2 Elemente gleichzeitig
  wait: 1000,     // Warte 1 Sekunde zwischen dem Start neuer Elemente
  started: true   // Beginne sofort mit der Verarbeitung
})

// Füge Fehler- und Erfolgs-Handler hinzu
queue.onError((error) => {
  console.error('Aufgabe fehlgeschlagen:', error)
})

queue.onSuccess((result) => {
  console.log('Aufgabe abgeschlossen:', result)
})

// Füge asynchrone Aufgaben hinzu
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### Warteschlangentypen und Reihenfolge

Der `AsyncQueuer` unterstützt verschiedene Warteschlangenstrategien, um unterschiedliche Verarbeitungsanforderungen zu erfüllen. Jede Strategie bestimmt, wie Aufgaben zur Warteschlange hinzugefügt und aus dieser verarbeitet werden.

#### FIFO-Warteschlange (First In, First Out)

FIFO-Warteschlangen verarbeiten Aufgaben in der exakten Reihenfolge, in der sie hinzugefügt wurden, was sie ideal für die Beibehaltung der Sequenz macht:

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // Standard
  getItemsFrom: 'front', // Standard
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// Verarbeitet: first und second gleichzeitig
```

#### LIFO-Stack (Last In, First Out)

LIFO-Stacks verarbeiten die zuletzt hinzugefügten Aufgaben zuerst, was nützlich ist, um neuere Aufgaben zu priorisieren:

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // Verarbeite neueste Elemente zuerst
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// Verarbeitet: second zuerst, dann first
```

#### Prioritätswarteschlange

Prioritätswarteschlangen verarbeiten Aufgaben basierend auf ihren zugewiesenen Prioritätswerten, um sicherzustellen, dass wichtige Aufgaben zuerst behandelt werden. Es gibt zwei Möglichkeiten, Prioritäten anzugeben:

1. Statische Prioritätswerte, die Aufgaben zugeordnet sind:
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// Erstelle Aufgaben mit statischen Prioritätswerten
const lowPriorityTask = Object.assign(
  async () => 'low priority result',
  { priority: 1 }
)

const highPriorityTask = Object.assign(
  async () => 'high priority result',
  { priority: 3 }
)

const mediumPriorityTask = Object.assign(
  async () => 'medium priority result',
  { priority: 2 }
)

// Füge Aufgaben in beliebiger Reihenfolge hinzu - sie werden nach Priorität verarbeitet (höhere Zahlen zuerst)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// Verarbeitet: high und medium gleichzeitig, dann low
```

2. Dynamische Prioritätsberechnung mit der `getPriority`-Option:
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // Berechne Priorität basierend auf Aufgabeneigenschaften oder anderen Faktoren
    // Höhere Zahlen haben Priorität
    return calculateTaskPriority(task)
  }
})

// Füge Aufgaben hinzu - Priorität wird dynamisch berechnet
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

Prioritätswarteschlangen sind unerlässlich, wenn:
- Aufgaben unterschiedliche Wichtigkeitsstufen haben
- Kritische Operationen zuerst ausgeführt werden müssen
- Sie flexible Aufgabenreihenfolge basierend auf Priorität benötigen
- Die Ressourcenzuteilung wichtigen Aufgaben bevorzugen sollte
- Die Priorität dynamisch basierend auf Aufgabeneigenschaften oder externen Faktoren bestimmt werden muss

### Fehlerbehandlung

Der `AsyncQueuer` bietet umfassende Fehlerbehandlungsfähigkeiten, um eine robuste Aufgabenverarbeitung zu gewährleisten. Sie können Fehler sowohl auf Warteschlangenebene als auch auf individueller Aufgabenebene behandeln:

```ts
const queue = new AsyncQueuer<string>()

// Behandle Fehler global
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('Aufgabe fehlgeschlagen:', error)
  },
  onSuccess: (result) => {
    console.log('Aufgabe erfolgreich:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('Aufgabe fehlgeschlagen:', result)
    } else {
      console.log('Aufgabe erfolgreich:', result)
    }
  }
})

// Behandle Fehler pro Aufgabe
queue.addItem(async () => {
  throw new Error('Aufgabe fehlgeschlagen')
}).catch(error => {
  console.error('Individueller Aufgabenfehler:', error)
})
```

### Warteschlangenverwaltung

Der `AsyncQueuer` bietet mehrere Methoden zur Überwachung und Steuerung des Warteschlangenzustands:

```ts
// Warteschlangeninspektion
queue.getPeek()           // Zeige nächstes Element an, ohne es zu entfernen
queue.getSize()          // Aktuelle Warteschlangengröße abrufen
queue.getIsEmpty()       // Prüfen, ob die Warteschlange leer ist
queue.getIsFull()        // Prüfen, ob die Warteschlange maxSize erreicht hat
queue.getAllItems()   // Kopie aller eingereihten Elemente abrufen
queue.getActiveItems() // Aktuell verarbeitete Elemente abrufen
queue.getPendingItems() // Auf Verarbeitung wartende Elemente abrufen

// Warteschlangenmanipulation
queue.clear()         // Alle Elemente entfernen
queue.reset()         // Auf Anfangszustand zurücksetzen
queue.getExecutionCount() // Anzahl der verarbeiteten Elemente abrufen

// Verarbeitungssteuerung
queue.start()         // Mit der Verarbeitung von Elementen beginnen
queue.stop()          // Verarbeitung pausieren
queue.getIsRunning()     // Prüfen, ob die Warteschlange verarbeitet
queue.getIsIdle()        // Prüfen, ob die Warteschlange leer und nicht in Verarbeitung ist
```

### Aufgaben-Callbacks

Der `AsyncQueuer` bietet drei Arten von Callbacks zur Überwachung der Aufgabenausführung:

```ts
const queue = new AsyncQueuer<string>()

// Behandle erfolgreichen Aufgabenabschluss
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Aufgabe erfolgreich:', result)
})

// Behandle Aufgabenfehler
const unsubError = queue.onError((error) => {
  console.error('Aufgabe fehlgeschlagen:', error)
})

// Behandle Aufgabenabschluss unabhängig von Erfolg/Misserfolg
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('Aufgabe fehlgeschlagen:', result)
  } else {
    console.log('Aufgabe erfolgreich:', result)
  }
})

// Melde Callbacks ab, wenn sie nicht mehr benötigt werden
unsubSuccess()
unsubError()
unsubSettled()
```

### Ablehnungsbehandlung

Wenn eine Warteschlange ihre maximale Größe (festgelegt durch die `maxSize`-Option) erreicht, werden neue Aufgaben abgelehnt. Der `AsyncQueuer` bietet Möglichkeiten, diese Ablehnungen zu behandeln und zu überwachen:

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // Nur 2 Aufgaben in der Warteschlange erlauben
  onReject: (task, queuer) => {
    console.log('Warteschlange ist voll. Aufgabe abgelehnt:', task)
  }
})

queue.addItem(async () => 'first') // Akzeptiert
queue.addItem(async () => 'second') // Akzeptiert
queue.addItem(async () => 'third') // Abgelehnt, löst onReject-Callback aus

console.log(queue.getRejectionCount()) // 1
```

### Initiale Aufgaben

Sie können eine asynchrone Warteschlange beim Erstellen mit initialen Aufgaben vorbelegen:

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // Beginne sofort mit der Verarbeitung
})

// Warteschlange startet mit drei Aufgaben und beginnt mit deren Verarbeitung
```

### Dynamische Konfiguration

Die Optionen des `AsyncQueuer` können nach der Erstellung mit `setOptions()` geändert und mit `getOptions()` abgerufen werden:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// Konfiguration ändern
queue.setOptions({
  concurrency: 4, // Verarbeite mehr Aufgaben gleichzeitig
  started: true // Beginne mit der Verarbeitung
})

// Aktuelle Konfiguration abrufen
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### Aktive und ausstehende Aufgaben

Der `AsyncQueuer` bietet Methoden zur Überwachung sowohl aktiver als auch ausstehender Aufgaben:

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// Füge einige Aufgaben hinzu
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// Überwache Aufgabenstatus
console.log(queue.getActiveItems().length) // Aktuell verarbeitete Aufgaben
console.log(queue.getPendingItems().length) // Auf Verarbeitung wartende Aufgaben
```

### Framework-Adapter

Jeder Framework-Adapter baut bequeme Hooks und Funktionen um die asynchronen Queuer-Klassen. Hooks wie `useAsyncQueuer` oder `useAsyncQueuedState` sind kleine Wrapper, die den Boilerplate-Code in Ihrem eigenen Code für einige gängige Anwendungsfälle reduzieren können.
