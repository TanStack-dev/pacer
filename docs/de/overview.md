---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:12:49.466Z'
title: Übersicht
id: overview
---
# Übersicht

TanStack Pacer ist eine Bibliothek, die sich darauf konzentriert, hochwertige Hilfsmittel zur Steuerung der Ausführungszeit von Funktionen in Ihren Anwendungen bereitzustellen. Obwohl ähnliche Hilfsmittel anderswo existieren, legen wir Wert darauf, alle wichtigen Details richtig zu machen – einschließlich ***Type-Safety***, ***Tree-Shaking*** und einer konsistenten sowie ***intuitiven API***. Indem wir uns auf diese Grundlagen konzentrieren und sie auf eine ***framework-agnostische*** Weise verfügbar machen, möchten wir diese Hilfsmittel und Muster in Ihren Anwendungen verbreiten. Die korrekte Steuerung der Ausführung wird bei der Anwendungsentwicklung oft vernachlässigt, was zu Leistungsproblemen, Race Conditions und schlechten Benutzererfahrungen führt, die hätten vermieden werden können. TanStack Pacer hilft Ihnen, diese kritischen Muster von Anfang an korrekt umzusetzen!

> [!IMPORTANT]
> TanStack Pacer befindet sich derzeit in der **Alpha**-Phase, und seine API kann sich noch ändern.
>
> Der Umfang dieser Bibliothek könnte erweitert werden, aber wir möchten die Bundle-Größe jeder einzelnen Utility schlank und fokussiert halten.

## Ursprung

Viele der Ideen (und des Codes) für TanStack Pacer sind nicht neu. Tatsächlich existieren viele dieser Hilfsmittel bereits seit einiger Zeit in anderen TanStack-Bibliotheken. Wir haben Code aus TanStack Query, Router, Form und sogar Tanners ursprünglicher [Swimmer](https://github.com/tannerlinsley/swimmer)-Bibliothek extrahiert. Dann haben wir diese Hilfsmittel bereinigt, Lücken geschlossen und sie als eigenständige Bibliothek veröffentlicht.

## Hauptmerkmale

- **Debouncing**
  - Verzögert die Ausführung von Funktionen bis nach einer Phase der Inaktivität
  - Synchrone oder asynchrone Debounce-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Throttling**
  - Begrenzt die Rate, mit der eine Funktion ausgelöst werden kann
  - Synchrone oder asynchrone Throttle-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Rate Limiting**
  - Begrenzt die Rate, mit der eine Funktion ausgelöst werden kann
  - Synchrone oder asynchrone Rate-Limiting-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Queuing**
  - Reiht Funktionen zur Ausführung in einer bestimmten Reihenfolge ein
  - Wählen Sie zwischen FIFO-, LIFO- und Priority-Queue-Implementierungen
  - Steuern Sie die Verarbeitungsgeschwindigkeit mit konfigurierbaren Wartezeiten oder Concurrency-Limits
  - Verwalten Sie die Queue-Ausführung mit Start-/Stop-Funktionen
  - Entfernen Sie Elemente nach einer konfigurierbaren Dauer aus der Queue
- **Async- oder Sync-Varianten**
  - Wählen Sie zwischen synchronen und asynchronen Versionen jeder Utility
  - Erzwingen Sie bei Bedarf Single-Flight-Ausführung von Funktionen in den asynchronen Varianten der Hilfsmittel
  - Optionale Fehler-, Erfolgs- und Abschlussbehandlung für asynchrone Varianten
- **Vergleichs-Hilfsmittel**
  - Führt Deep-Equality-Prüfungen zwischen Werten durch
  - Erstellt benutzerdefinierte Vergleichslogik für spezifische Anforderungen
- **Praktische Hooks**
  - Reduziert Boilerplate-Code mit vordefinierten Hooks wie `useDebouncedCallback`, `useThrottledValue`, `useQueuedState` und mehr.
  - Mehrere Abstraktionsebenen zur Auswahl, abhängig von Ihrem Anwendungsfall.
  - Funktioniert mit den Standard-State-Management-Lösungen jedes Frameworks oder mit jeder bevorzugten benutzerdefinierten State-Management-Bibliothek.
- **Type Safety**
  - Volle Type-Safety mit TypeScript, die sicherstellt, dass Ihre Funktionen immer mit den korrekten Argumenten aufgerufen werden
  - Generics für flexible und wiederverwendbare Hilfsmittel
- **Framework-Adapter**
  - React, Solid und mehr
- **Tree Shaking**
  - Selbstverständlich unterstützen wir standardmäßig korrektes Tree-Shaking für Ihre Anwendungen, aber wir bieten zusätzlich tiefe Imports für jede Utility an, um die Einbettung dieser Hilfsmittel in Ihre Bibliotheken zu erleichtern, ohne die Bundle-Phobia-Reports Ihrer Bibliothek zu erhöhen.
