---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:34:35.073Z'
title: Guide sur la mise en file d'attente
id: queueing
---
Contrairement à la [Limitation de débit (Rate Limiting)](../guides/rate-limiting), à la [Restriction (Throttling)](../guides/throttling) et au [Rebond (Debouncing)](../guides/debouncing) qui abandonnent les exécutions lorsqu'elles se produisent trop fréquemment, les mécanismes de mise en file d'attente (Queueing) garantissent que chaque opération est traitée. Ils offrent un moyen de gérer et de contrôler le flux d'opérations sans perdre aucune requête. Cela les rend idéaux pour les scénarios où la perte de données est inacceptable. Ce guide couvrira les concepts de mise en file d'attente de TanStack Pacer.

## Concept de mise en file d'attente (Queueing)

La mise en file d'attente garantit que chaque opération est finalement traitée, même si elles arrivent plus rapidement qu'elles ne peuvent être gérées. Contrairement aux autres techniques de contrôle d'exécution qui abandonnent les opérations excédentaires, la mise en file d'attente stocke les opérations dans une liste ordonnée et les traite selon des règles spécifiques. Cela fait de la mise en file d'attente la seule technique de contrôle d'exécution "sans perte" dans TanStack Pacer.

### Visualisation de la mise en file d'attente

```text
Mise en file d'attente (traitement d'un élément toutes les 2 unités de temps)
Chronologie : [1 seconde par unité]
Appels :      ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️  ⬇️  ⬇️
File :       [ABC]   [BC]    [BCDE]    [DE]    [E]    []
Exécutés :    ✅     ✅       ✅        ✅      ✅     ✅
             [=================================================================]
             ^ Contrairement à la limitation de débit/restriction/rebond,
               TOUS les appels sont finalement traités dans l'ordre

             [Les éléments s'accumulent]   [Traitement régulier]   [File vide]
              quand occupé                  un par un                terminée
```

### Quand utiliser la mise en file d'attente

La mise en file d'attente est particulièrement importante lorsque vous devez vous assurer que chaque opération est traitée, même si cela implique un certain délai. Cela la rend idéale pour les scénarios où la cohérence et l'exhaustivité des données sont plus importantes qu'une exécution immédiate.

Cas d'utilisation courants :
- Traitement des interactions utilisateur dans une interface où chaque action doit être enregistrée
- Gestion des opérations de base de données nécessitant une cohérence des données
- Gestion des requêtes API qui doivent toutes se terminer avec succès
- Coordination des tâches en arrière-plan qui ne peuvent pas être abandonnées
- Séquences d'animation où chaque image compte
- Soumissions de formulaires où chaque entrée doit être sauvegardée

### Quand ne pas utiliser la mise en file d'attente

La mise en file d'attente pourrait ne pas être le meilleur choix lorsque :
- Un retour immédiat est plus important que le traitement de chaque opération
- Vous ne vous intéressez qu'à la valeur la plus récente (utilisez plutôt le [rebond (debouncing)](../guides/debouncing))

> [!TIP]
> Si vous utilisez actuellement la limitation de débit, la restriction ou le rebond mais que les opérations abandonnées causent des problèmes, la mise en file d'attente est probablement la solution dont vous avez besoin.

## Mise en file d'attente dans TanStack Pacer

TanStack Pacer fournit la mise en file d'attente via la fonction simple `queue` et la classe plus puissante `Queuer`. Alors que les autres techniques de contrôle d'exécution privilégient généralement leurs API basées sur des fonctions, la mise en file d'attente bénéficie souvent du contrôle supplémentaire offert par l'API basée sur des classes.

### Utilisation basique avec `queue`

La fonction `queue` offre un moyen simple de créer une file d'attente toujours active qui traite les éléments au fur et à mesure de leur ajout :

```ts
import { queue } from '@tanstack/pacer'

// Créer une file d'attente qui traite les éléments toutes les secondes
const processItems = queue<number>({
  wait: 1000,
  onItemsChange: (queuer) => {
    console.log('File actuelle :', queuer.getAllItems())
  }
})

// Ajouter des éléments à traiter
processItems(1) // Traité immédiatement
processItems(2) // Traité après 1 seconde
processItems(3) // Traité après 2 secondes
```

Bien que la fonction `queue` soit simple à utiliser, elle ne fournit qu'une file d'attente toujours active de base via la méthode `addItem`. Pour la plupart des cas d'utilisation, vous voudrez le contrôle et les fonctionnalités supplémentaires offerts par la classe `Queuer`.

### Utilisation avancée avec la classe `Queuer`

La classe `Queuer` offre un contrôle complet sur le comportement et le traitement de la file d'attente :

```ts
import { Queuer } from '@tanstack/pacer'

// Créer une file d'attente qui traite les éléments toutes les secondes
const queue = new Queuer<number>({
  wait: 1000, // Attendre 1 seconde entre le traitement des éléments
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

// Les éléments seront traités un par un avec un délai d'une seconde entre chacun
// Sortie :
// Traitement : 1 (immédiatement)
// Traitement : 2 (après 1 seconde)
// Traitement : 3 (après 2 secondes)
```

### Types de files d'attente et ordonnancement

Ce qui rend le Queuer de TanStack Pacer unique est sa capacité à s'adapter à différents cas d'utilisation grâce à son API basée sur la position. Le même Queuer peut se comporter comme une file d'attente traditionnelle, une pile ou une file double, le tout via la même interface cohérente.

#### File FIFO (Premier entré, premier sorti)

Comportement par défaut où les éléments sont traités dans l'ordre où ils ont été ajoutés. C'est le type de file d'attente le plus courant et suit le principe que le premier élément ajouté doit être le premier traité.

```text
Visualisation d'une file FIFO :

Entrée →  [A][B][C][D] → Sortie
         ⬇️         ⬆️
      Nouveaux    Les éléments
      éléments    sont traités
      ajoutés ici ici

Chronologie : [1 seconde par unité]
Appels :      ⬇️  ⬇️  ⬇️     ⬇️
File :       [ABC]   [BC]    [C]    []
Traités :    A       B       C
```

Les files FIFO sont idéales pour :
- Le traitement de tâches où l'ordre compte
- Les files de messages où les messages doivent être traités en séquence
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

En spécifiant 'back' comme position pour l'ajout et la récupération des éléments, le Queuer se comporte comme une pile. Dans une pile, l'élément ajouté le plus récemment est le premier à être traité.

```text
Visualisation d'une pile LIFO :

     ⬆️ Traitement
    [D] ← Ajouté le plus récemment
    [C]
    [B]
    [A] ← Premier ajouté
     ⬇️ Entrée

Chronologie : [1 seconde par unité]
Appels :      ⬇️  ⬇️  ⬇️     ⬇️
File :       [ABC]   [AB]    [A]    []
Traités :    C       B       A
```

Le comportement de pile est particulièrement utile pour :
- Les systèmes d'annulation/rétablissement où l'action la plus récente doit être annulée en premier
- La navigation dans l'historique du navigateur où vous souhaitez revenir à la page la plus récente
- Les piles d'appels de fonctions dans les implémentations de langages de programmation
- Les algorithmes de parcours en profondeur

```ts
const stack = new Queuer<number>({
  addItemsTo: 'back', // par défaut
  getItemsFrom: 'back', // remplace le défaut pour un comportement de pile
})
stack.addItem(1) // [1]
stack.addItem(2) // [1, 2]
// Les éléments seront traités dans l'ordre : 2, puis 1

stack.getNextItem('back') // obtenir le prochain élément de l'arrière de la file au lieu de l'avant
```

#### File à priorité

Les files à priorité ajoutent une autre dimension à l'ordonnancement en permettant aux éléments d'être triés en fonction de leur priorité plutôt que de leur ordre d'insertion. Chaque élément se voit attribuer une valeur de priorité, et la file maintient automatiquement les éléments dans l'ordre de priorité.

```text
Visualisation d'une file à priorité :

Entrée →  [P:5][P:3][P:2][P:1] → Sortie
          ⬇️           ⬆️
     Éléments de      Priorité
     haute priorité   faible traités
     ici              en dernier

Chronologie : [1 seconde par unité]
Appels :      ⬇️(P:2)  ⬇️(P:5)  ⬇️(P:1)     ⬇️(P:3)
File :       [2]      [5,2]    [5,2,1]    [3,2,1]    [2,1]    [1]    []
Traités :              5         -          3         2        1
```

Les files à priorité sont essentielles pour :
- Les planificateurs de tâches où certaines tâches sont plus urgentes que d'autres
- Le routage de paquets réseau où certains types de trafic nécessitent un traitement préférentiel
- Les systèmes d'événements où les événements de haute priorité doivent être gérés avant ceux de faible priorité
- L'allocation de ressources où certaines requêtes sont plus importantes que d'autres

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
console.log(queue.getIsRunning()) // Si la file est en cours de traitement
console.log(queue.getIsIdle())    // Si la file est active mais vide
```

Si vous utilisez un adaptateur de framework où les options du Queuer sont réactives, vous pouvez définir l'option `started` sur une valeur conditionnelle :

```ts
const queue = useQueuer(
  processItem, 
  { 
    wait: 1000,
    started: isOnline // Démarrer/arrêter en fonction du statut de connexion SI vous utilisez un adaptateur de framework prenant en charge les options réactives
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

### Gestion des rejets

Lorsqu'une file atteint sa taille maximale (définie par l'option `maxSize`), les nouveaux éléments seront rejetés. Le Queuer offre des moyens de gérer et surveiller ces rejets :

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

Vous pouvez préremplir une file d'attente avec des éléments initiaux lors de sa création :

```ts
const queue = new Queuer<number>({
  initialItems: [1, 2, 3],
  started: true // Commencer le traitement immédiatement
})

// La file commence avec [1, 2, 3] et commence le traitement
```

### Configuration dynamique

Les options du Queuer peuvent être modifiées après création en utilisant `setOptions()` et récupérées en utilisant `getOptions()` :

```ts
const queue = new Queuer<number>({
  wait: 1000,
  started: false
})

// Modifier la configuration
queue.setOptions({
  wait: 500, // Traiter les éléments deux fois plus vite
  started: true // Commencer le traitement
})

// Obtenir la configuration actuelle
const options = queue.getOptions()
console.log(options.wait) // 500
```

### Surveillance des performances

Le Queuer fournit des méthodes pour surveiller ses performances :

```ts
const queue = new Queuer<number>()

// Ajouter et traiter quelques éléments
queue.addItem(1)
queue.addItem(2)
queue.addItem(3)

console.log(queue.getExecutionCount()) // Nombre d'éléments traités
console.log(queue.getRejectionCount()) // Nombre d'éléments rejetés
```

### Mise en file d'attente asynchrone

Pour gérer les opérations asynchrones avec plusieurs travailleurs, consultez le [Guide de mise en file d'attente asynchrone (Async Queueing Guide)](../guides/async-queueing) qui couvre la classe `AsyncQueuer`.

### Adaptateurs de framework

Chaque adaptateur de framework construit des hooks et fonctions pratiques autour des classes de mise en file d'attente. Les hooks comme `useQueuer` ou `useQueueState` sont de petites enveloppes qui peuvent réduire le code passe-partout nécessaire dans votre propre code pour certains cas d'utilisation courants.
