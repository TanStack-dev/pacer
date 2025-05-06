---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:17:19.969Z'
title: Guide sur la mise en file d'attente
id: queueing
---
# Guide de mise en file d'attente (Queueing Guide)

Contrairement à la [Limitation de débit (Rate Limiting)](../guides/rate-limiting), au [Lissage (Throttling)](../guides/throttling) et au [Rebond (Debouncing)](../guides/debouncing) qui abandonnent les exécutions lorsqu'elles se produisent trop fréquemment, les mécanismes de mise en file d'attente peuvent être configurés pour garantir que chaque opération est traitée. Ils offrent un moyen de gérer et de contrôler le flux d'opérations sans perdre aucune requête. Cela les rend idéaux pour les scénarios où la perte de données est inacceptable. La mise en file d'attente peut également être configurée avec une taille maximale, ce qui peut être utile pour prévenir les fuites mémoire ou d'autres problèmes. Ce guide couvrira les concepts de mise en file d'attente de TanStack Pacer.

## Concept de mise en file d'attente (Queueing Concept)

La mise en file d'attente garantit que chaque opération est finalement traitée, même si elles arrivent plus rapidement qu'elles ne peuvent être gérées. Contrairement aux autres techniques de contrôle d'exécution qui abandonnent les opérations excédentaires, la mise en file d'attente stocke les opérations dans une liste ordonnée et les traite selon des règles spécifiques. Cela fait de la mise en file d'attente la seule technique de contrôle d'exécution "sans perte" dans TanStack Pacer, sauf si une `maxSize` est spécifiée, ce qui peut entraîner le rejet d'éléments lorsque le tampon est plein.

### Visualisation de la mise en file d'attente

```text
Mise en file d'attente (traitement d'un élément toutes les 2 unités de temps)
Chronologie : [1 seconde par unité de temps]
Appels :        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
File d'attente :[ABC]   [BC]    [BCDE]    [DE]    [E]    []
Exécutés :      ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ Contrairement à la limitation de débit/lissage/rebond,
               TOUS les appels sont finalement traités dans l'ordre

             [Les éléments s'accumulent]   [Traitement régulier]   [File vide]
              lorsqu'occupé                un par un                vidée
```

### Quand utiliser la mise en file d'attente

La mise en file d'attente est particulièrement importante lorsque vous devez vous assurer que chaque opération est traitée, même si cela implique d'introduire un certain délai. Cela la rend idéale pour les scénarios où la cohérence et l'exhaustivité des données sont plus importantes qu'une exécution immédiate. Lorsqu'une `maxSize` est utilisée, elle peut également servir de tampon pour éviter de submerger un système avec trop d'opérations en attente.

Cas d'usage courants :
- Pré-récupération de données avant qu'elles ne soient nécessaires sans surcharger le système
- Traitement des interactions utilisateur dans une interface où chaque action doit être enregistrée
- Gestion des opérations de base de données qui doivent maintenir la cohérence des données
- Gestion des requêtes API qui doivent toutes se terminer avec succès
- Coordination des tâches en arrière-plan qui ne peuvent pas être abandonnées
- Séquences d'animation où chaque image compte
- Soumissions de formulaires où chaque entrée doit être sauvegardée
- Tamponnage de flux de données avec une capacité fixe en utilisant `maxSize`

### Quand ne pas utiliser la mise en file d'attente

La mise en file d'attente pourrait ne pas être le meilleur choix lorsque :
- Un retour immédiat est plus important que le traitement de chaque opération
- Vous ne vous intéressez qu'à la valeur la plus récente (utilisez plutôt le [rebond (debouncing)](../guides/debouncing)

> [!ASTUCE]
> Si vous utilisez actuellement la limitation de débit, le lissage ou le rebond mais que vous constatez que les opérations abandonnées causent des problèmes, la mise en file d'attente est probablement la solution dont vous avez besoin.

## Mise en file d'attente dans TanStack Pacer

TanStack Pacer fournit la mise en file d'attente via la simple fonction `queue` et la classe plus puissante `Queuer`. Alors que les autres techniques de contrôle d'exécution privilégient généralement leurs API basées sur des fonctions, la mise en file d'attente bénéficie souvent du contrôle supplémentaire offert par l'API basée sur des classes.

### Utilisation de base avec `queue`

La fonction `queue` offre un moyen simple de créer une file d'attente toujours active qui traite les éléments au fur et à mesure de leur ajout :

```ts
import { queue } from '@tanstack/pacer'

// Créer une file d'attente qui traite les éléments toutes les secondes
const processItems = queue<number>({
  wait: 1000,
  maxSize: 10, // Optionnel : limiter la taille de la file pour éviter des problèmes de mémoire ou de temps
  onItemsChange: (queuer) => {
    console.log('File actuelle :', queuer.getAllItems())
  }
})

// Ajouter des éléments à traiter
processItems(1) // Traité immédiatement
processItems(2) // Traité après 1 seconde
processItems(3) // Traité après 2 secondes
```

Bien que la fonction `queue` soit simple à utiliser, elle ne fournit qu'une file d'attente toujours active de base via la méthode `addItem`. Pour la plupart des cas d'usage, vous voudrez le contrôle et les fonctionnalités supplémentaires offerts par la classe `Queuer`.

### Utilisation avancée avec la classe `Queuer`

La classe `Queuer` offre un contrôle complet sur le comportement et le traitement de la file d'attente :

```ts
import { Queuer } from '@tanstack/pacer'

// Créer une file d'attente qui traite les éléments toutes les secondes
const queue = new Queuer<number>({
  wait: 1000, // Attendre 1 seconde entre le traitement des éléments
  maxSize: 5, // Optionnel : limiter la taille de la file pour éviter des problèmes de mémoire ou de temps
  onItemsChange: (queuer) => {
    console.log('File actuelle :', queuer.getAllItems())
  }
})

// Démarrer le traitement
queue.start()

// Ajouter des éléments à traiter
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

// Les éléments seront traités un par un avec un délai de 1 seconde entre chacun
// Sortie :
// Traitement : 1 (immédiatement)
// Traitement : 2 (après 1 seconde)
// Traitement : 3 (après 2 secondes)
```

### Types de files d'attente et ordonnancement

Ce qui rend le Queuer de TanStack Pacer unique est sa capacité à s'adapter à différents cas d'usage grâce à son API basée sur la position. Le même Queuer peut se comporter comme une file d'attente traditionnelle, une pile ou une file double, le tout via la même interface cohérente.

#### File FIFO (Premier entré, premier sorti)

Le comportement par défaut où les éléments sont traités dans l'ordre où ils ont été ajoutés. C'est le type de file d'attente le plus courant et suit le principe selon lequel le premier élément ajouté doit être le premier traité. Lorsqu'on utilise `maxSize`, les nouveaux éléments seront rejetés si la file est pleine.

```text
Visualisation d'une file FIFO (avec maxSize=3) :

Entrée →  [A][B][C] → Sortie
         ⬇️     ⬆️
      Nouveaux éléments   Les éléments sont
      ajoutés ici        traités ici
      (rejetés si pleine)

Chronologie : [1 seconde par unité de temps]
Appels :        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
File d'attente :[ABC]   [BC]    [C]    []
Traités :       A       B       C
Rejetés :       D      E
```

Les files FIFO sont idéales pour :
- Le traitement de tâches où l'ordre compte
- Les files de messages où les messages doivent être traités dans l'ordre
- Les files d'impression où les documents doivent être imprimés dans l'ordre d'envoi
- Les systèmes de gestion d'événements où les événements doivent être traités chronologiquement

```ts
const queue = new Queuer<number>({
  addItemsTo: 'back', // par défaut
  getItemsFrom: 'front', // par défaut
})
queue.addItem(1) // [1]
queue.addItem(2) // [1, 2]
// Traite : 1, puis 2
```

#### Pile LIFO (Dernier entré, premier sorti)

En spécifiant 'back' comme position pour l'ajout et la récupération des éléments, le Queuer se comporte comme une pile. Dans une pile, l'élément ajouté le plus récemment est le premier à être traité. Lorsqu'on utilise `maxSize`, les nouveaux éléments seront rejetés si la pile est pleine.

```text
Visualisation d'une pile LIFO (avec maxSize=3) :

     ⬆️ Traitement
    [C] ← Ajouté le plus récemment
    [B]
    [A] ← Premier ajouté
     ⬇️ Entrée
     (rejeté si pleine)

Chronologie : [1 seconde par unité de temps]
Appels :        ⬇️  ⬇️  ⬇️     ⬇️  ⬇️
File d'attente :[ABC]   [AB]    [A]    []
Traités :       C       B       A
Rejetés :       D      E
```

Le comportement de pile est particulièrement utile pour :
- Les systèmes d'annulation/rétablissement où l'action la plus récente doit être annulée en premier
- La navigation dans l'historique du navigateur où vous voulez retourner à la page la plus récente
- Les piles d'appels de fonctions dans les implémentations de langages de programmation
- Les algorithmes de parcours en profondeur

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // par défaut
  getItemsFrom: 'back', // remplace le comportement par défaut pour un comportement de pile
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// Les éléments seront traités dans l'ordre : 2, puis 1

stack.getNextItem('back') // obtenir le prochain élément de l'arrière de la file au lieu de l'avant
```

#### File à priorité (Priority Queue)

Les files à priorité ajoutent une autre dimension à l'ordonnancement en permettant aux éléments d'être triés en fonction de leur priorité plutôt que de leur ordre d'insertion. Chaque élément se voit attribuer une valeur de priorité, et la file maintient automatiquement les éléments dans l'ordre de priorité. Lorsqu'on utilise `maxSize`, les éléments de priorité inférieure peuvent être rejetés si la file est pleine.

```text
Visualisation d'une file à priorité (avec maxSize=3) :

Entrée →  [P:5][P:3][P:2] → Sortie
          ⬇️           ⬆️
     Éléments de haute priorité   Traités en dernier
     ici                         (rejetés si pleine)

Chronologie : [1 seconde par unité de temps]
Appels :        ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
File d'attente :[2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Traités :              5         -          3         2        1
Rejetés :                         4
```

Les files à priorité sont essentielles pour :
- Les planificateurs de tâches où certaines tâches sont plus urgentes que d'autres
- Le routage de paquets réseau où certains types de trafic nécessitent un traitement préférentiel
- Les systèmes d'événements où les événements de haute priorité doivent être gérés avant ceux de priorité inférieure
- L'allocation de ressources où certaines demandes sont plus importantes que d'autres

```ts
const priorityQueue = new Queuer<number>({
  getPriority: (n) => n // Les nombres plus élevés ont la priorité
})
priorityQueue.addItem(1) // [1]
priorityQueue.addItem(3) // [3, 1]
priorityQueue.addItem(2) // [3, 2, 1]
// Traite : 3, 2, puis 1
```

### Démarrage et arrêt

La classe `Queuer` prend en charge le démarrage et l'arrêt du traitement via les méthodes `start()` et `stop()`, et peut être configurée pour démarrer automatiquement avec l'option `started` :

```ts
const queue = new Queuer<number>({ 
  wait: 1000,
  started: false // Démarrer en pause
})

// Contrôle du traitement
queue.start() // Commencer à traiter les éléments
queue.stop()  // Mettre en pause le traitement

// Vérifier l'état du traitement
console.log(queue.getIsRunning()) // Si la file est actuellement en train de traiter
console.log(queue.getIsIdle())    // Si la file est en cours d'exécution mais vide
```

Si vous utilisez un adaptateur de framework où les options du Queuer sont réactives, vous pouvez définir l'option `started` sur une valeur conditionnelle :

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // Démarrer/arrêter en fonction du statut de connexion SI vous utilisez un adaptateur de framework qui prend en charge les options réactives
  }
)
```

### Fonctionnalités supplémentaires

Le Queuer fournit plusieurs méthodes utiles pour la gestion de la file d'attente :

```ts
// Inspection de la file
queue.getPeek()           // Voir le prochain élément sans le retirer
queue.getSize()          // Obtenir la taille actuelle de la file
queue.getIsEmpty()       // Vérifier si la file est vide
queue.getIsFull()        // Vérifier si la file a atteint maxSize
queue.getAllItems()   // Obtenir une copie de tous les éléments en file

// Manipulation de la file
queue.clear()         // Supprimer tous les éléments
queue.reset()         // Réinitialiser à l'état initial
queue.getExecutionCount() // Obtenir le nombre d'éléments traités

// Gestion des événements
queue.onItemsChange((item) => {
  console.log('Traité :', item)
})
```

### Expiration des éléments

Le Queuer prend en charge l'expiration automatique des éléments restés trop longtemps dans la file. Ceci est utile pour empêcher le traitement de données obsolètes ou pour implémenter des délais d'attente sur les opérations en file.

```ts
const queue = new Queuer<number>({
  expirationDuration: 5000, // Les éléments expirent après 5 secondes
  onExpire: (item, queuer) => {
    console.log('Élément expiré :', item)
  }
})

// Ou utiliser une vérification d'expiration personnalisée
const queue = new Queuer<number>({
  getIsExpired: (item, addedAt) => {
    // Logique d'expiration personnalisée
    return Date.now() - addedAt > 5000
  },
  onExpire: (item, queuer) => {
    console.log('Élément expiré :', item)
  }
})

// Vérifier les statistiques d'expiration
console.log(queue.getExpirationCount()) // Nombre d'éléments qui ont expiré
```

Les fonctionnalités d'expiration sont particulièrement utiles pour :
- Empêcher le traitement de données obsolètes
- Implémenter des délais d'attente sur les opérations en file
- Gérer l'utilisation de la mémoire en supprimant automatiquement les anciens éléments
- Gérer les données temporaires qui ne doivent être valides que pendant un temps limité

### Gestion des rejets

Lorsqu'une file atteint sa taille maximale (définie par l'option `maxSize`), les nouveaux éléments seront rejetés. Le Queuer offre des moyens de gérer et de surveiller ces rejets :

```ts
const queue = new Queuer<number>({
  maxSize: 2, // Autoriser seulement 2 éléments dans la file
  onReject: (item, queuer) => {
    console.log('La file est pleine. Élément rejeté :', item)
  }
})

queue.addItem(1) // Accepté
queue.addItem(2) // Accepté
queue.addItem(3) // Rejeté, déclenche le callback onReject

console.log(queue.getRejectionCount()) // 1
```

### Éléments initiaux

Vous pouvez pré-remplir une file d'attente avec des éléments initiaux lors de sa création :

```ts
