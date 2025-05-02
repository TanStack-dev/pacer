---
source-updated-at: '2025-04-07T12:06:53.000Z'
translation-updated-at: '2025-05-02T04:29:38.334Z'
title: Übersicht
id: overview
---
# Übersicht

TanStack Pacer ist eine Bibliothek, die sich darauf konzentriert, hochwertige Hilfsmittel zur Steuerung des Ausführungszeitpunkts von Funktionen in Ihren Anwendungen bereitzustellen. Obwohl ähnliche Hilfsmittel anderswo existieren, legen wir Wert darauf, alle wichtigen Details richtig umzusetzen – einschließlich ***Typsicherheit (type-safety)***, ***Tree-Shaking*** und einer konsistenten sowie ***intuitiven API***. Indem wir uns auf diese Grundlagen konzentrieren und sie auf eine ***framework-unabhängige (framework agnostic)*** Weise verfügbar machen, möchten wir diese Hilfsmittel und Muster in Ihren Anwendungen verbreiten. Die korrekte Steuerung der Ausführung wird bei der Anwendungsentwicklung oft vernachlässigt, was zu Leistungsproblemen, Race Conditions und schlechten Benutzererfahrungen führt, die hätten vermieden werden können. TanStack Pacer hilft Ihnen, diese kritischen Muster von Anfang an richtig umzusetzen!

> [!IMPORTANT]
> TanStack Pacer befindet sich derzeit in der **Alpha**-Phase, und seine API kann sich noch ändern.
>
> Der Umfang dieser Bibliothek könnte erweitert werden, aber wir möchten die Bundle-Größe jeder einzelnen Utility schlank und fokussiert halten.

## Ursprung

Viele der Ideen (und des Codes) für TanStack Pacer sind nicht neu. Tatsächlich existieren viele dieser Hilfsmittel bereits seit geraumer Zeit in anderen TanStack-Bibliotheken. Wir haben Code aus TanStack Query, Router, Form und sogar Tanners ursprünglicher [Swimmer](https://github.com/tannerlinsley/swimmer)-Bibliothek extrahiert. Dann haben wir diese Hilfsmittel bereinigt, Lücken geschlossen und sie als eigenständige Bibliothek veröffentlicht.

## Hauptmerkmale

- **Debouncing**
  - Verzögert die Ausführung von Funktionen bis nach einer Phase der Inaktivität
  - Synchrone oder asynchrone Debounce-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Throttling**
  - Begrenzt die Häufigkeit, mit der eine Funktion ausgelöst werden kann
  - Synchrone oder asynchrone Throttle-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Rate Limiting**
  - Begrenzt die Häufigkeit, mit der eine Funktion ausgelöst werden kann
  - Synchrone oder asynchrone Rate-Limiting-Hilfsmittel mit Promise-Unterstützung und Fehlerbehandlung
- **Queuing**
  - Reiht Funktionen zur Ausführung in einer bestimmten Reihenfolge ein
  - Auswahl zwischen FIFO-, LIFO- und Priority-Queue-Implementierungen
  - Steuerung der Verarbeitung mit konfigurierbaren Wartezeiten oder Parallelitätslimits
  - Verwaltung der Queue-Ausführung mit Start-/Stop-Funktionen
  - Synchrone oder asynchrone Queue-Hilfsmittel mit Promise-Unterstützung sowie Erfolgs-, Abschluss- und Fehlerbehandlung
- **Vergleichs-Hilfsmittel**
  - Führt tiefgehende Gleichheitsprüfungen zwischen Werten durch
  - Ermöglicht benutzerdefinierte Vergleichslogik für spezifische Anforderungen
- **Praktische Hooks**
  - Reduziert Boilerplate-Code mit vorgefertigten Hooks wie `useDebouncedCallback`, `useThrottledValue`, `useQueuerState` und mehr.
  - Mehrere Abstraktionsebenen zur Auswahl, abhängig von Ihrem Anwendungsfall.
- **Typsicherheit**
  - Volle Typsicherheit mit TypeScript, die sicherstellt, dass Ihre Funktionen immer mit den korrekten Argumenten aufgerufen werden
  - Generics für flexible und wiederverwendbare Hilfsmittel
- **Framework-Adapter**
  - React, Solid und mehr
- **Tree-Shaking**
  - Selbstverständlich unterstützen wir standardmäßig Tree-Shaking für Ihre Anwendungen, aber wir bieten zusätzlich tiefe Importe für jedes Hilfsmittel an, um die Integration in Ihre Bibliotheken zu erleichtern, ohne die Bundle-Größenberichte Ihrer Bibliothek zu erhöhen.
