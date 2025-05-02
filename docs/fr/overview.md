---
source-updated-at: '2025-04-07T12:06:53.000Z'
translation-updated-at: '2025-05-02T04:32:52.778Z'
title: Vue d'ensemble
id: overview
---
# Vue d'ensemble

TanStack Pacer est une bibliothèque axée sur la fourniture d'utilitaires de haute qualité pour contrôler le timing d'exécution des fonctions dans vos applications. Bien que des utilitaires similaires existent ailleurs, nous visons à respecter tous les détails importants - y compris la ***sécurité de type (type-safety)***, le ***tree-shaking*** et une ***API intuitive*** et cohérente. En nous concentrant sur ces fondamentaux et en les rendant disponibles de manière ***agnostique aux frameworks***, nous espérons rendre ces utilitaires et modèles plus courants dans vos applications. Le contrôle approprié de l'exécution est souvent une réflexion après coup dans le développement d'applications, entraînant des problèmes de performance, des conditions de course (race conditions) et des expériences utilisateur médiocres qui auraient pu être évitées. TanStack Pacer vous aide à implémenter correctement ces modèles critiques dès le départ !

> [!IMPORTANT]
> TanStack Pacer est actuellement en **alpha** et son API est susceptible de changer.
>
> La portée de cette bibliothèque peut s'étendre, mais nous espérons garder la taille du bundle de chaque utilitaire individuel léger et ciblé.

## Origine

Beaucoup des idées (et du code) de TanStack Pacer ne sont pas nouvelles. En fait, nombre de ces utilitaires existent depuis un certain temps dans d'autres bibliothèques TanStack. Nous avons extrait du code de TanStack Query, Router, Form et même de la bibliothèque originale [Swimmer](https://github.com/tannerlinsley/swimmer) de Tanner. Ensuite, nous avons nettoyé ces utilitaires, comblé certaines lacunes et les avons livrés sous forme de bibliothèque autonome.

## Fonctionnalités clés

- **Antirebond (Debouncing)**
  - Retarde l'exécution des fonctions jusqu'à une période d'inactivité
  - Utilitaires d'antirebond synchrones ou asynchrones avec prise en charge des promesses et gestion des erreurs
- **Étranglement (Throttling)**
  - Limite la fréquence à laquelle une fonction peut s'exécuter
  - Utilitaires d'étranglement synchrones ou asynchrones avec prise en charge des promesses et gestion des erreurs
- **Limitation de débit (Rate Limiting)**
  - Limite la fréquence à laquelle une fonction peut s'exécuter
  - Utilitaires de limitation de débit synchrones ou asynchrones avec prise en charge des promesses et gestion des erreurs
- **File d'attente (Queuing)**
  - Met en file d'attente les fonctions à exécuter dans un ordre spécifique
  - Choix parmi les implémentations FIFO, LIFO et file d'attente prioritaire
  - Contrôle le traitement avec des temps d'attente configurables ou des limites de concurrence
  - Gère l'exécution de la file d'attente avec des capacités de démarrage/arrêt
  - Utilitaires de file d'attente synchrones ou asynchrones avec prise en charge des promesses et gestion des succès, des résultats et des erreurs
- **Utilitaires de comparaison**
  - Effectue des vérifications d'égalité profonde entre les valeurs
  - Crée une logique de comparaison personnalisée pour des besoins spécifiques
- **Hooks pratiques**
  - Réduit le code passe-partout avec des hooks prédéfinis comme `useDebouncedCallback`, `useThrottledValue`, `useQueuerState`, et plus encore.
  - Plusieurs niveaux d'abstraction au choix selon votre cas d'utilisation.
- **Sécurité de type (Type Safety)**
  - Sécurité de type complète avec TypeScript qui garantit que vos fonctions seront toujours appelées avec les bons arguments
  - Génériques pour des utilitaires flexibles et réutilisables
- **Adaptateurs pour frameworks**
  - React, Solid, et plus
- **Tree Shaking**
  - Nous respectons bien sûr le tree-shaking par défaut pour vos applications, mais nous fournissons également des imports profonds supplémentaires pour chaque utilitaire, facilitant ainsi leur intégration dans vos bibliothèques sans augmenter les rapports de bundle-phobia.
