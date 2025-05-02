---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:31:31.269Z'
title: Warteschlangen-Anleitung
id: queueing
---
# Warteschlangen-Guide (Queueing Guide)

Im Gegensatz zu [Ratenbegrenzung (Rate Limiting)](../guides/rate-limiting), [Drosselung (Throttling)](../guides/throttling) und [Entprellung (Debouncing)](../guides/debouncing), die Ausführungen verwerfen, wenn sie zu häufig auftreten, stellen Warteschlangen (Queuers) sicher, dass jeder Vorgang verarbeitet wird. Sie bieten eine Möglichkeit, den Ablauf von Operationen zu verwalten und zu steuern, ohne Anfragen zu verlieren. Dies macht sie ideal für Szenarien, in denen Datenverlust nicht akzeptabel ist. Dieser Leitfaden behandelt die Warteschlangen-Konzepte (Queueing Concepts) von TanStack Pacer.

## Warteschlangen-Konzept (Queueing Concept)

Warteschlangen (Queueing) stellen sicher, dass jeder Vorgang letztendlich verarbeitet wird, selbst wenn sie schneller eintreffen, als sie bearbeitet werden können. Im Gegensatz zu anderen Techniken zur Ausführungskontrolle, die überflüssige Operationen verwerfen, puffern Warteschlangen Operationen in einer geordneten Liste und verarbeiten sie nach bestimmten Regeln. Dies macht Warteschlangen zur einzigen "verlustfreien" (lossless) Technik zur Ausführungskontrolle in TanStack Pacer.

### Visualisierung von Warteschlangen (Queueing Visualization)

```text
Queueing (processing one item every 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
Queue:       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
Executed:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ Im Gegensatz zu Ratenbegrenzung/Drosselung/Entprellung,
               werden ALLE Aufrufe letztendlich in Reihenfolge verarbeitet

             [Elemente sammeln sich]   [Stetige Verarbeitung]   [Leere]
              bei Überlastung          eins nach dem anderen     Warteschlange
```

### Wann Warteschlangen verwendet werden sollten (When to Use Queueing)

Warteschlangen sind besonders wichtig, wenn sichergestellt werden muss, dass jeder Vorgang verarbeitet wird, selbst wenn dies eine gewisse Verzögerung bedeutet. Dies macht sie ideal für Szenarien, in denen Datenkonsistenz und Vollständigkeit wichtiger sind als sofortige Ausführung.

Häufige Anwendungsfälle sind:
- Verarbeitung von Benutzerinteraktionen in einer UI, bei der jede Aktion aufgezeichnet werden muss
- Handhabung von Datenbankoperationen, die Datenkonsistenz aufrechterhalten müssen
- Verwaltung von API-Anfragen, die alle erfolgreich abgeschlossen werden müssen
- Koordination von Hintergrundaufgaben, die nicht verworfen werden können
- Animationssequenzen, bei denen jeder Frame wichtig ist
- Formularübermittlungen, bei denen jeder Eintrag gespeichert werden muss

### Wann Warteschlangen nicht verwendet werden sollten (When Not to Use Queueing)

Warteschlangen sind möglicherweise nicht die beste Wahl, wenn:
- Sofortiges Feedback wichtiger ist als die Verarbeitung jeder Operation
- Nur der neueste Wert von Interesse ist (verwenden Sie stattdessen [Entprellung (Debouncing)](../guides/debouncing))

> [!TIP]
> Falls Sie derzeit Ratenbegrenzung, Drosselung oder Entprellung verwenden, aber feststellen, dass verworfenen Operationen Probleme verursachen, ist eine Warteschlange wahrscheinlich die Lösung, die Sie benötigen.

## Warteschlangen in TanStack Pacer (Queueing in TanStack Pacer)

TanStack Pacer bietet Warteschlangen über die einfache `queue`-Funktion und die leistungsfähigere `Queuer`-Klasse. Während andere Techniken zur Ausführungskontrolle typischerweise ihre funktionsbasierten APIs bevorzugen, profitieren Warteschlangen oft von der zusätzlichen Kontrolle, die die klassenbasierte API bietet.

### Grundlegende Verwendung mit `queue`

Die `queue`-Funktion bietet eine einfache Möglichkeit, eine ständig laufende Warteschlange zu erstellen, die Elemente verarbeitet, sobald sie hinzugefügt werden:

```ts
import { queue } from '@tanstack/pacer'

// Erstelle eine Warteschlange, die Elemente jede Sekunde verarbeitet
const processItems = queue<number>({
  wait: 1000,
  onItemsChange: (queuer) => {
    console.log('Aktuelle Warteschlange:', queuer.getAllItems())
  }
})

// Füge Elemente zur Verarbeitung hinzu
processItems(1) // Sofort verarbeitet
processItems(2) // Nach 1 Sekunde verarbeitet
processItems(3) // Nach 2 Sekunden verarbeitet
```

Während die `queue`-Funktion einfach zu verwenden ist, bietet sie nur eine grundlegende, ständig laufende Warteschlange über die `addItem`-Methode. Für die meisten Anwendungsfälle werden Sie die zusätzliche Kontrolle und Funktionen der `Queuer`-Klasse bevorzugen.

### Erweiterte Verwendung mit der `Queuer`-Klasse (Advanced Usage with `Queuer` Class)

Die `Queuer`-Klasse bietet vollständige Kontrolle über das Verhalten und die Verarbeitung der Warteschlange:

```ts
import { Queuer } from '@tanstack/pacer'

// Erstelle eine Warteschlange, die Elemente jede Sekunde verarbeitet
const queue = new Queuer<number>({
  wait: 1000, // Warte 1 Sekunde zwischen der Verarbeitung von Elementen
  onItemsChange: (queuer) => {
    console.log('Aktuelle Warteschlange:', queuer.getAllItems())
  }
})

// Starte die Verarbeitung
queue.start()

// Füge Elemente zur Verarbeitung hinzu
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// Elemente werden nacheinander mit 1 Sekunde Verzögerung verarbeitet
// Ausgabe:
// Verarbeitung: 1 (sofort)
// Verarbeitung: 2 (nach 1 Sekunde)
// Verarbeitung: 3 (nach 2 Sekunden)
```

### Warteschlangentypen und Reihenfolge (Queue Types and Ordering)

Was den Queuer von TanStack Pacer einzigartig macht, ist seine Fähigkeit, sich durch seine positionsbasierte API an verschiedene Anwendungsfälle anzupassen. Derselbe Queuer kann sich wie eine traditionelle Warteschlange (Queue), ein Stapelspeicher (Stack) oder eine doppelseitige Warteschlange (Double-Ended Queue) verhalten, alles über dieselbe konsistente Schnittstelle.

#### FIFO-Warteschlange (First In, First Out)

Das Standardverhalten, bei dem Elemente in der Reihenfolge verarbeitet werden, in der sie hinzugefügt wurden. Dies ist der häufigste Warteschlangentyp und folgt dem Prinzip, dass das zuerst hinzugefügte Element als erstes verarbeitet werden sollte.

```text
Visualisierung einer FIFO-Warteschlange:

Eingang →  [A][B][C][D] → Ausgang
         ⬇️         ⬆️
      Neue Elemente   Elemente werden
      werden hier     hier verarbeitet
      hinzugefügt

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️  ⬇️  ⬇️     ⬇️
Warteschlange: [ABC]   [BC]    [C]    []
Verarbeitet:    A       B       C
```

FIFO-Warteschlangen sind ideal für:
- Aufgabenverarbeitung, bei der die Reihenfolge wichtig ist
- Nachrichtenwarteschlangen, bei denen Nachrichten in Sequenz verarbeitet werden müssen
- Druckwarteschlangen, bei denen Dokumente in der Reihenfolge ihres Versands gedruckt werden sollten
- Ereignisbehandlungssysteme, bei denen Ereignisse chronologisch verarbeitet werden müssen

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // Standard
  getItemsFrom: 'front', // Standard
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// Verarbeitet: 1, dann 2
```

#### LIFO-Stapelspeicher (Last In, First Out)

Durch Angabe von 'back' als Position sowohl für das Hinzufügen als auch das Abrufen von Elementen verhält sich der Queuer wie ein Stapelspeicher (Stack). In einem Stapelspeicher wird das zuletzt hinzugefügte Element als erstes verarbeitet.

```text
Visualisierung eines LIFO-Stapelspeichers:

     ⬆️ Verarbeitung
    [D] ← Zuletzt hinzugefügt
    [C]
    [B]
    [A] ← Zuerst hinzugefügt
     ⬇️ Eingang

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️  ⬇️  ⬇️     ⬇️
Warteschlange: [ABC]   [AB]    [A]    []
Verarbeitet:    C       B       A
```

Stapelverhalten ist besonders nützlich für:
- Rückgängig/Wiederherstellen-Systeme, bei denen die letzte Aktion zuerst rückgängig gemacht werden soll
- Browser-Verlaufsnavigation, bei der Sie zur zuletzt besuchten Seite zurückkehren möchten
- Funktionsaufrufstapel in Programmiersprachenimplementierungen
- Tiefensuche-Algorithmen (Depth-First Traversal)

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // Standard
  getItemsFrom: 'back', // Überschreibt Standard für Stapelverhalten
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// Elemente werden in der Reihenfolge verarbeitet: 2, dann 1

stack.getNextItem('back') // Nächstes Element vom Ende der Warteschlange abrufen statt vom Anfang
```

#### Prioritätswarteschlange (Priority Queue)

Prioritätswarteschlangen fügen der Reihenfolge von Warteschlangen eine weitere Dimension hinzu, indem sie Elemente basierend auf ihrer Priorität sortieren, anstatt nur auf ihrer Einfügereihenfolge. Jedes Element erhält einen Prioritätswert, und die Warteschlange behält die Elemente automatisch in Prioritätsreihenfolge bei.

```text
Visualisierung einer Prioritätswarteschlange:

Eingang →  [P:5][P:3][P:2][P:1] → Ausgang
          ⬇️           ⬆️
     Hohe Priorität   Niedrige Priorität
     Elemente hier    werden zuletzt verarbeitet

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
Warteschlange: [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Verarbeitet:              5         -          3         2        1
```

Prioritätswarteschlangen sind essenziell für:
- Aufgabenplaner, bei denen einige Aufgaben dringender sind als andere
- Netzwerkpaketrouting, bei dem bestimmte Verkehrstypen bevorzugt behandelt werden müssen
- Ereignissysteme, bei denen hochpriorisierte Ereignisse vor niedrigpriorisierten behandelt werden sollen
- Ressourcenzuteilung, bei der einige Anfragen wichtiger sind als andere

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // Höhere Zahlen haben Priorität
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// Verarbeitet: 3, 2, dann 1
```

### Starten und Stoppen (Starting and Stopping)

Die `Queuer`-Klasse unterstützt das Starten und Stoppen der Verarbeitung über die Methoden `start()` und `stop()` und kann so konfiguriert werden, dass sie automatisch mit der Option `started` startet:

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // Pausiert starten
})

// Verarbeitung steuern
queue.start() // Beginne mit der Verarbeitung von Elementen
queue.stop()  // Pausiere die Verarbeitung

// Verarbeitungsstatus überprüfen
console.log(queue.getIsRunning()) // Ob die Warteschlange aktuell verarbeitet
console.log(queue.getIsIdle())    // Ob die Warteschlange läuft, aber leer ist
```

Falls Sie ein Framework-Adapter verwenden, bei dem die Queuer-Optionen reaktiv sind, können Sie die `started`-Option auf einen bedingten Wert setzen:

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // Starte/Stoppe basierend auf Verbindungsstatus, WENN ein Framework-Adapter verwendet wird, der reaktive Optionen unterstützt
  }
)
```

### Zusätzliche Funktionen (Additional Features)

Der Queuer bietet mehrere hilfreiche Methoden zur Verwaltung der Warteschlange:

```ts
// Warteschlangeninspektion
queue.getPeek()           // Nächstes Element anzeigen, ohne es zu entfernen
queue.getSize()          // Aktuelle Größe der Warteschlange abrufen
queue.getIsEmpty()       // Überprüfen, ob die Warteschlange leer ist
queue.getIsFull()        // Überprüfen, ob die Warteschlange maxSize erreicht hat
queue.getAllItems()   // Kopie aller Elemente in der Warteschlange abrufen

// Warteschlangenmanipulation
queue.clear()         // Alle Elemente entfernen
queue.reset()         // Auf Anfangszustand zurücksetzen
queue.getExecutionCount() // Anzahl der verarbeiteten Elemente abrufen

// Ereignisbehandlung
queue.onItemsChange((item) => {
  console.log('Verarbeitet:', item)
})
```

### Ablehnungsbehandlung (Rejection Handling)

Wenn eine Warteschlange ihre maximale Größe (durch die `maxSize`-Option festgelegt) erreicht, werden neue Elemente abgelehnt. Der Queuer bietet Möglichkeiten, diese Ablehnungen zu behandeln und zu überwachen:

```ts
const queue = new Queuer<number>({
  maxSize: 2, // Nur 2 Elemente in der Warteschlange erlauben
  onReject: (item, queuer) => {
    console.log('Warteschlange ist voll. Element abgelehnt:', item)
  }
})

queue.addItem(1) // Akzeptiert
queue.addItem(2) // Akzeptiert
queue.addItem(3) // Abgelehnt, löst onReject-Callback aus

console.log(queue.getRejectionCount()) // 1
```

### Anfangselemente (Initial Items)

Sie können eine Warteschlange beim Erstellen mit Anfangselementen vorbelegen:

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // Beginne sofort mit der Verarbeitung
})

// Warteschlange startet mit [1, 2, 3] und beginnt mit der Verarbeitung
```

### Dynamische Konfiguration (Dynamic Configuration)

Die Optionen des Queuers können nach der Erstellung mit `setOptions()` geändert und mit `getOptions()` abgerufen werden:

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// Konfiguration ändern
queue.setOptions({
  wait: 500, // Verarbeite Elemente doppelt so schnell
  started: true // Beginne mit der Verarbeitung
})

// Aktuelle Konfiguration abrufen
const options = queue.getOptions()
console.log(options.wait) // 500
```

### Leistungsüberwachung (Performance Monitoring)

Der Queuer bietet Methoden zur Überwachung seiner Leistung:

```ts
const queue = new Queuer<number>()

// Füge einige Elemente hinzu und verarbeite sie
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // Anzahl der verarbeiteten Elemente
console.log(queue.getRejectionCount()) // Anzahl der abgelehnten Elemente
```

### Asynchrone Warteschlangen (Asynchronous Queueing)

Für die Handhabung asynchroner Operationen mit mehreren Workern siehe den [Leitfaden zu asynchronen Warteschlangen (Async Queueing Guide)](../guides/async-queueing), der die `AsyncQueuer`-Klasse behandelt.

### Framework-Adapter
