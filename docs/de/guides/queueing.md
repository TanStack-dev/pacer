---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:14:40.272Z'
title: Warteschlangen-Anleitung
id: queueing
---
# Warteschlangen-Leitfaden

Im Gegensatz zu [Rate Limiting](../guides/rate-limiting), [Throttling](../guides/throttling) und [Debouncing](../guides/debouncing), die Ausführungen verwerfen, wenn sie zu häufig auftreten, können Warteschlangen so konfiguriert werden, dass jede Operation verarbeitet wird. Sie bieten eine Möglichkeit, den Fluss von Operationen zu verwalten und zu steuern, ohne Anfragen zu verlieren. Dies macht sie ideal für Szenarien, in denen Datenverlust nicht akzeptabel ist. Warteschlangen können auch mit einer maximalen Größe konfiguriert werden, was nützlich sein kann, um Speicherlecks oder andere Probleme zu verhindern. Dieser Leitfaden behandelt die Warteschlangen-Konzepte von TanStack Pacer.

## Warteschlangen-Konzept

Warteschlangen stellen sicher, dass jede Operation irgendwann verarbeitet wird, selbst wenn sie schneller eintreffen, als sie bearbeitet werden können. Im Gegensatz zu anderen Ausführungssteuerungstechniken, die überflüssige Operationen verwerfen, puffern Warteschlangen Operationen in einer geordneten Liste und verarbeiten sie nach bestimmten Regeln. Dies macht Warteschlangen zur einzigen "verlustfreien" Ausführungssteuerungstechnik in TanStack Pacer, es sei denn, es wird eine `maxSize` angegeben, die dazu führen kann, dass Elemente abgelehnt werden, wenn der Puffer voll ist.

### Visualisierung von Warteschlangen

```text
Warteschlange (Verarbeitung eines Elements alle 2 Ticks)
Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  �️  �️
Warteschlange: [ABC]   [BC]    [BCDE]    [DE]    [E]    []
Verarbeitet:     ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ Im Gegensatz zu Rate Limiting/Throttling/Debouncing
               werden ALLE Aufrufe irgendwann in Reihenfolge verarbeitet

             [Elemente sammeln sich]   [Stetige Verarbeitung]   [Leere Warteschlange]
              bei Überlastung           eins nach dem anderen
```

### Wann Warteschlangen verwendet werden sollten

Warteschlangen sind besonders wichtig, wenn sichergestellt werden muss, dass jede Operation verarbeitet wird, selbst wenn dies eine gewisse Verzögerung bedeutet. Dies macht sie ideal für Szenarien, in denen Datenkonsistenz und -vollständigkeit wichtiger sind als sofortige Ausführung. Bei Verwendung einer `maxSize` kann sie auch als Puffer dienen, um ein System mit zu vielen ausstehenden Operationen nicht zu überlasten.

Häufige Anwendungsfälle sind:
- Vorab-Laden von Daten, ohne das System zu überlasten
- Verarbeitung von Benutzerinteraktionen in einer UI, bei der jede Aktion aufgezeichnet werden muss
- Datenbankoperationen, die Datenkonsistenz aufrechterhalten müssen
- Verwaltung von API-Anfragen, die alle erfolgreich abgeschlossen werden müssen
- Koordination von Hintergrundaufgaben, die nicht verworfen werden können
- Animationssequenzen, bei denen jeder Frame wichtig ist
- Formularübermittlungen, bei denen jeder Eintrag gespeichert werden muss
- Puffern von Datenströmen mit fester Kapazität mittels `maxSize`

### Wann Warteschlangen nicht verwendet werden sollten

Warteschlangen sind möglicherweise nicht die beste Wahl, wenn:
- Sofortiges Feedback wichtiger ist als die Verarbeitung jeder Operation
- Nur der neueste Wert relevant ist (verwenden Sie stattdessen [Debouncing](../guides/debouncing))

> [!TIP]
> Wenn Sie derzeit Rate Limiting, Throttling oder Debouncing verwenden, aber feststellen, dass verworfenen Operationen Probleme verursachen, ist eine Warteschlange wahrscheinlich die Lösung, die Sie benötigen.

## Warteschlangen in TanStack Pacer

TanStack Pacer bietet Warteschlangen über die einfache `queue`-Funktion und die leistungsfähigere `Queuer`-Klasse. Während andere Ausführungssteuerungstechniken typischerweise ihre funktionsbasierten APIs bevorzugen, profitieren Warteschlangen oft von der zusätzlichen Kontrolle, die die klassenbasierte API bietet.

### Grundlegende Verwendung mit `queue`

Die `queue`-Funktion bietet eine einfache Möglichkeit, eine ständig laufende Warteschlange zu erstellen, die Elemente bei Hinzufügung verarbeitet:

```ts
import { queue } from '@tanstack/pacer'

// Erstelle eine Warteschlange, die Elemente jede Sekunde verarbeitet
const processItems = queue<number>({
  wait: 1000,
  maxSize: 10, // Optional: Begrenze die Warteschlangengröße, um Speicher- oder Zeitprobleme zu vermeiden
  onItemsChange: (queuer) => {
    console.log('Aktuelle Warteschlange:', queuer.getAllItems())
  }
})

// Füge Elemente zur Verarbeitung hinzu
processItems(1) // Sofort verarbeitet
processItems(2) // Nach 1 Sekunde verarbeitet
processItems(3) // Nach 2 Sekunden verarbeitet
```

Während die `queue`-Funktion einfach zu verwenden ist, bietet sie nur eine grundlegende ständig laufende Warteschlange über die `addItem`-Methode. Für die meisten Anwendungsfälle werden Sie die zusätzliche Kontrolle und Funktionen der `Queuer`-Klasse bevorzugen.

### Erweiterte Verwendung mit der `Queuer`-Klasse

Die `Queuer`-Klasse bietet vollständige Kontrolle über das Warteschlangenverhalten und die Verarbeitung:

```ts
import { Queuer } from '@tanstack/pacer'

// Erstelle eine Warteschlange, die Elemente jede Sekunde verarbeitet
const queue = new Queuer<number>({
  wait: 1000, // Warte 1 Sekunde zwischen der Verarbeitung von Elementen
  maxSize: 5, // Optional: Begrenze die Warteschlangengröße, um Speicher- oder Zeitprobleme zu vermeiden
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

### Warteschlangentypen und Reihenfolge

Was TanStack Pacers Queuer einzigartig macht, ist seine Fähigkeit, sich durch seine positionsbasierte API an verschiedene Anwendungsfälle anzupassen. Derselbe Queuer kann sich wie eine traditionelle Warteschlange, ein Stack oder eine doppelseitige Warteschlange verhalten, alles über dieselbe konsistente Schnittstelle.

#### FIFO-Warteschlange (First In, First Out)

Das Standardverhalten, bei dem Elemente in der Reihenfolge ihrer Hinzufügung verarbeitet werden. Dies ist der häufigste Warteschlangentyp und folgt dem Prinzip, dass das zuerst hinzugefügte Element als erstes verarbeitet werden sollte. Bei Verwendung von `maxSize` werden neue Elemente abgelehnt, wenn die Warteschlange voll ist.

```text
FIFO-Warteschlangen-Visualisierung (mit maxSize=3):

Eingang →  [A][B][C] → Ausgang
         ⬇️     ⬆️
      Neue Elemente   Elemente werden
      werden hier     hier verarbeitet
      hinzugefügt     (abgelehnt, wenn voll)

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
Warteschlange: [ABC]   [BC]    [C]    []
Verarbeitet:    A       B       C
Abgelehnt:     D      E
```

FIFO-Warteschlangen sind ideal für:
- Aufgabenverarbeitung, bei der die Reihenfolge wichtig ist
- Nachrichtenwarteschlangen, bei denen Nachrichten in Sequenz verarbeitet werden müssen
- Druckwarteschlangen, bei denen Dokumente in der Reihenfolge ihres Versands gedruckt werden sollen
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

#### LIFO-Stack (Last In, First Out)

Durch Angabe von 'back' als Position für das Hinzufügen und Abrufen von Elementen verhält sich der Queuer wie ein Stack. In einem Stack wird das zuletzt hinzugefügte Element als erstes verarbeitet. Bei Verwendung von `maxSize` werden neue Elemente abgelehnt, wenn der Stack voll ist.

```text
LIFO-Stack-Visualisierung (mit maxSize=3):

     ⬆️ Verarbeitung
    [C] ← Zuletzt hinzugefügt
    [B]
    [A] ← Zuerst hinzugefügt
     ⬇️ Eingang
     (abgelehnt, wenn voll)

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        �️  ⬇️  ⬇️     ⬇️  ⬇️
Warteschlange: [ABC]   [AB]    [A]    []
Verarbeitet:    C       B       A
Abgelehnt:     D      E
```

Stack-Verhalten ist besonders nützlich für:
- Rückgängig/Wiederholen-Systeme, bei denen die letzte Aktion zuerst rückgängig gemacht werden soll
- Browser-Historie-Navigation, bei der Sie zur zuletzt besuchten Seite zurückkehren möchten
- Funktionsaufrufstapel in Programmiersprachenimplementierungen
- Tiefensuche-Algorithmen

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // Standard
  getItemsFrom: 'back', // Standard für Stack-Verhalten überschreiben
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// Elemente werden in der Reihenfolge verarbeitet: 2, dann 1

stack.getNextItem('back') // Nächstes Element vom Ende der Warteschlange abrufen statt vom Anfang
```

#### Prioritätswarteschlange

Prioritätswarteschlangen fügen eine weitere Dimension zur Warteschlangenreihenfolge hinzu, indem Elemente basierend auf ihrer Priorität sortiert werden können, nicht nur nach ihrer Einfügereihenfolge. Jedes Element erhält einen Prioritätswert, und die Warteschlange behält die Elemente automatisch in Prioritätsreihenfolge bei. Bei Verwendung von `maxSize` können Elemente mit niedrigerer Priorität abgelehnt werden, wenn die Warteschlange voll ist.

```text
Prioritätswarteschlangen-Visualisierung (mit maxSize=3):

Eingang →  [P:5][P:3][P:2] → Ausgang
          ⬇️           ⬆️
     Hohe Priorität   Niedrige Priorität
     Elemente hier    werden zuletzt verarbeitet
     (abgelehnt, wenn voll)

Zeitachse: [1 Sekunde pro Tick]
Aufrufe:        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
Warteschlange: [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Verarbeitet:              5         -          3         2        1
Abgelehnt:                         4
```

Prioritätswarteschlangen sind essenziell für:
- Aufgabenplaner, bei denen einige Aufgaben dringender sind als andere
- Netzwerkpaket-Routing, bei dem bestimmte Verkehrstypen bevorzugt behandelt werden müssen
- Ereignissysteme, bei denen Ereignisse mit hoher Priorität vor solchen mit niedrigerer Priorität behandelt werden sollten
- Ressourcenzuweisung, bei der einige Anfragen wichtiger sind als andere

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // Höhere Zahlen haben Priorität
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// Verarbeitet: 3, 2, dann 1
```

### Starten und Stoppen

Die `Queuer`-Klasse unterstützt das Starten und Stoppen der Verarbeitung über die Methoden `start()` und `stop()` und kann mit der Option `started` automatisch gestartet werden:

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // Pausiert starten
})

// Verarbeitung steuern
queue.start() // Beginne mit der Verarbeitung von Elementen
queue.stop()  // Pausiere die Verarbeitung

// Verarbeitungsstatus prüfen
console.log(queue.getIsRunning()) // Ob die Warteschlange aktuell verarbeitet
console.log(queue.getIsIdle())    // Ob die Warteschlange läuft, aber leer ist
```

Wenn Sie ein Framework-Adapter verwenden, bei dem die Queuer-Optionen reaktiv sind, können Sie die `started`-Option auf einen bedingten Wert setzen:

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // Starte/Stoppe basierend auf Verbindungsstatus, WENN ein Framework-Adapter verwendet wird, der reaktive Optionen unterstützt
  }
)
```

### Zusätzliche Funktionen

Der Queuer bietet mehrere hilfreiche Methoden zur Warteschlangenverwaltung:

```ts
// Warteschlangeninspektion
queue.getPeek()           // Nächstes Element anzeigen, ohne es zu entfernen
queue.getSize()          // Aktuelle Warteschlangengröße abrufen
queue.getIsEmpty()       // Prüfen, ob die Warteschlange leer ist
queue.getIsFull()        // Prüfen, ob die Warteschlange maxSize erreicht hat
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

### Elementablauf

Der Queuer unterstützt den automatischen Ablauf von Elementen, die zu lange in der Warteschlange waren. Dies ist nützlich, um das Verarbeiten veralteter Daten zu verhindern oder Timeouts für wartende Operationen zu implementieren.

```ts
const queue = new Queuer<number>({
  expirationDuration: 5000, // Elemente laufen nach 5 Sekunden ab
  onExpire: (item, queuer) => {
    console.log('Element abgelaufen:', item)
  }
})

// Oder benutzerdefinierte Ablaufprüfung verwenden
const queue = new Queuer<number>({
  getIsExpired: (item, addedAt) => {
    // Benutzerdefinierte Ablauflogik
    return Date.now() - addedAt > 5000
  },
  onExpire: (item, queuer) => {
    console.log('Element abgelaufen:', item)
  }
})

// Ablaufstatistiken prüfen
console.log(queue.getExpirationCount()) // Anzahl der abgelaufenen Elemente
```

Ablauffunktionen sind besonders nützlich für:
- Verhindern, dass veraltete Daten verarbeitet werden
- Implementieren von Timeouts für wartende Operationen
- Speicherverwaltung durch automatisches Entfernen alter Elemente
- Handhabung temporärer Daten, die nur für begrenzte Zeit gültig sein sollten

### Ablehnungsbehandlung

Wenn eine Warteschlange ihre maximale Größe (durch die `maxSize`-Option festgelegt) erreicht, werden neue Elemente abgelehnt. Der Queuer bietet Möglichkeiten, diese Ablehnungen zu behandeln und zu überwachen:

```ts
const queue = new Queuer<number>({
  maxSize:
