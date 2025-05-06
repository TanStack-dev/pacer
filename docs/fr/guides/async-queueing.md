---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:17:06.053Z'
title: Guide sur la mise en file d'attente asynchrone
id: async-queueing
---
# Guide de mise en file d'attente asynchrone (Asynchronous Queueing Guide)

Alors que le [Queuer](../guides//queueing) fournit une mise en file d'attente synchrone avec des contrôles de temporisation, l'`AsyncQueuer` est conçu spécifiquement pour gérer des opérations asynchrones concurrentes. Il implémente ce qui est traditionnellement connu sous le nom de modèle "task pool" ou "worker pool", permettant à plusieurs opérations d'être traitées simultanément tout en conservant le contrôle sur la concurrence et la temporisation. L'implémentation est principalement copiée depuis [Swimmer](https://github.com/tannerlinsley/swimmer), l'utilitaire original de gestion de tâches de Tanner qui sert la communauté JavaScript depuis 2017.

## Concept de mise en file d'attente asynchrone (Async Queueing)

La mise en file d'attente asynchrone étend le concept de base de la file d'attente en ajoutant des capacités de traitement concurrent. Au lieu de traiter un élément à la fois, un gestionnaire de file d'attente asynchrone peut traiter plusieurs éléments simultanément tout en maintenant l'ordre et le contrôle sur l'exécution. Ceci est particulièrement utile pour les opérations d'E/S, les requêtes réseau ou toute tâche qui passe la plupart de son temps en attente plutôt qu'à consommer du CPU.

### Visualisation de la mise en file d'attente asynchrone

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ Contrairement à une file d'attente régulière, plusieurs éléments
               peuvent être traités de manière concurrente

             [Les éléments s'accumulent]   [Traitement de 2 à la fois]   [Terminaison]
              quand occupé                 avec attente entre les deux     de tous les éléments
```

### Quand utiliser la mise en file d'attente asynchrone

La mise en file d'attente asynchrone est particulièrement efficace lorsque vous avez besoin de :
- Traiter plusieurs opérations asynchrones de manière concurrente
- Contrôler le nombre d'opérations simultanées
- Gérer des tâches basées sur des Promises avec une bonne gestion des erreurs
- Maintenir l'ordre tout en maximisant le débit
- Traiter des tâches en arrière-plan pouvant s'exécuter en parallèle

Cas d'utilisation courants :
- Effectuer des requêtes API concurrentes avec limitation de débit
- Traiter plusieurs téléchargements de fichiers simultanément
- Exécuter des opérations de base de données en parallèle
- Gérer plusieurs connexions websocket
- Traiter des flux de données avec backpressure
- Gérer des tâches en arrière-plan intensives en ressources

### Quand ne pas utiliser la mise en file d'attente asynchrone

L'AsyncQueuer est très polyvalent et peut être utilisé dans de nombreuses situations. En réalité, il n'est pas adapté uniquement lorsque vous ne prévoyez pas de tirer parti de toutes ses fonctionnalités. Si vous n'avez pas besoin que toutes les exécutions en file d'attente soient traitées, utilisez plutôt [Throttling][../guides/throttling]. Si vous n'avez pas besoin de traitement concurrent, utilisez plutôt [Queueing][../guides/queueing].

## Mise en file d'attente asynchrone dans TanStack Pacer

TanStack Pacer fournit la mise en file d'attente asynchrone via la fonction simple `asyncQueue` et la classe plus puissante `AsyncQueuer`.

### Utilisation basique avec `asyncQueue`

La fonction `asyncQueue` offre un moyen simple de créer une file d'attente asynchrone toujours active :

```ts
import { asyncQueue } from '@tanstack/pacer'

// Créer une file d'attente traitant jusqu'à 2 éléments simultanément
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('Tâches actives:', queuer.getActiveItems().length)
  }
})

// Ajouter des tâches asynchrones à traiter
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

L'utilisation de la fonction `asyncQueue` est quelque peu limitée, car il s'agit simplement d'un wrapper autour de la classe `AsyncQueuer` qui n'expose que la méthode `addItem`. Pour un contrôle plus poussé sur la file d'attente, utilisez directement la classe `AsyncQueuer`.

### Utilisation avancée avec la classe `AsyncQueuer`

La classe `AsyncQueuer` offre un contrôle complet sur le comportement de la file d'attente asynchrone :

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // Traiter 2 éléments à la fois
  wait: 1000,     // Attendre 1 seconde entre le démarrage de nouveaux éléments
  started: true   // Commencer le traitement immédiatement
})

// Ajouter des gestionnaires d'erreur et de succès
queue.onError((error) => {
  console.error('Échec de la tâche:', error)
})

queue.onSuccess((result) => {
  console.log('Tâche terminée:', result)
})

// Ajouter des tâches asynchrones
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### Types de file d'attente et ordonnancement

L'AsyncQueuer prend en charge différentes stratégies de mise en file d'attente pour répondre à divers besoins de traitement. Chaque stratégie détermine comment les tâches sont ajoutées et traitées depuis la file d'attente.

#### File FIFO (First In, First Out)

Les files FIFO traitent les tâches dans l'ordre exact où elles ont été ajoutées, ce qui les rend idéales pour maintenir une séquence :

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // par défaut
  getItemsFrom: 'front', // par défaut
  concurrency: 2
})

queue.addItem(async () => 'first')  // [first]
queue.addItem(async () => 'second') // [first, second]
// Traitement : first et second de manière concurrente
```

#### Pile LIFO (Last In, First Out)

Les piles LIFO traitent les tâches les plus récemment ajoutées en premier, utile pour prioriser les nouvelles tâches :

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // Traiter les éléments les plus récents en premier
  concurrency: 2
})

stack.addItem(async () => 'first')  // [first]
stack.addItem(async () => 'second') // [first, second]
// Traitement : second d'abord, puis first
```

#### File à priorité (Priority Queue)

Les files à priorité traitent les tâches en fonction de leurs valeurs de priorité attribuées, garantissant que les tâches importantes sont traitées en premier. Il existe deux façons de spécifier les priorités :

1. Valeurs de priorité statiques attachées aux tâches :
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// Créer des tâches avec des valeurs de priorité statiques
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

// Ajouter des tâches dans n'importe quel ordre - elles seront traitées par priorité (nombres plus élevés en premier)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// Traitement : high et medium de manière concurrente, puis low
```

2. Calcul dynamique de la priorité avec l'option `getPriority` :
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // Calculer la priorité en fonction des propriétés de la tâche ou d'autres facteurs
    // Les nombres plus élevés ont la priorité
    return calculateTaskPriority(task)
  }
})

// Ajouter des tâches - la priorité sera calculée dynamiquement
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('low')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('high')
  return result
})
```

Les files à priorité sont essentielles lorsque :
- Les tâches ont différents niveaux d'importance
- Les opérations critiques doivent s'exécuter en premier
- Vous avez besoin d'un ordonnancement flexible des tâches basé sur la priorité
- L'allocation des ressources doit favoriser les tâches importantes
- La priorité doit être déterminée dynamiquement en fonction des propriétés de la tâche ou de facteurs externes

### Gestion des erreurs

L'AsyncQueuer fournit des capacités complètes de gestion des erreurs pour assurer un traitement robuste des tâches. Vous pouvez gérer les erreurs à la fois au niveau de la file d'attente et au niveau individuel des tâches :

```ts
const queue = new AsyncQueuer<string>()

// Gérer les erreurs globalement
const queue = new AsyncQueuer<string>({
  onError: (error) => {
    console.error('Échec de la tâche:', error)
  },
  onSuccess: (result) => {
    console.log('Tâche réussie:', result)
  },
  onSettled: (result) => {
    if (result instanceof Error) {
      console.log('Échec de la tâche:', result)
    } else {
      console.log('Tâche réussie:', result)
    }
  }
})

// Gérer les erreurs par tâche
queue.addItem(async () => {
  throw new Error('Échec de la tâche')
}).catch(error => {
  console.error('Erreur de tâche individuelle:', error)
})
```

### Gestion de la file d'attente

L'AsyncQueuer fournit plusieurs méthodes pour surveiller et contrôler l'état de la file d'attente :

```ts
// Inspection de la file d'attente
queue.getPeek()           // Voir le prochain élément sans le retirer
queue.getSize()          // Obtenir la taille actuelle de la file d'attente
queue.getIsEmpty()       // Vérifier si la file d'attente est vide
queue.getIsFull()        // Vérifier si la file d'attente a atteint maxSize
queue.getAllItems()   // Obtenir une copie de tous les éléments en file d'attente
queue.getActiveItems() // Obtenir les éléments en cours de traitement
queue.getPendingItems() // Obtenir les éléments en attente de traitement

// Manipulation de la file d'attente
queue.clear()         // Supprimer tous les éléments
queue.reset()         // Réinitialiser à l'état initial
queue.getExecutionCount() // Obtenir le nombre d'éléments traités

// Contrôle du traitement
queue.start()         // Commencer le traitement des éléments
queue.stop()          // Mettre en pause le traitement
queue.getIsRunning()     // Vérifier si la file d'attente est en cours de traitement
queue.getIsIdle()        // Vérifier si la file d'attente est vide et non en traitement
```

### Callbacks de tâche

L'AsyncQueuer fournit trois types de callbacks pour surveiller l'exécution des tâches :

```ts
const queue = new AsyncQueuer<string>()

// Gérer la réussite d'une tâche
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Tâche réussie:', result)
})

// Gérer les erreurs de tâche
const unsubError = queue.onError((error) => {
  console.error('Échec de la tâche:', error)
})

// Gérer l'achèvement d'une tâche quel que soit son succès/échec
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('Échec de la tâche:', result)
  } else {
    console.log('Tâche réussie:', result)
  }
})

// Se désabonner des callbacks lorsqu'ils ne sont plus nécessaires
unsubSuccess()
unsubError()
unsubSettled()
```

### Gestion des rejets

Lorsqu'une file d'attente atteint sa taille maximale (définie par l'option `maxSize`), les nouvelles tâches seront rejetées. L'AsyncQueuer fournit des moyens de gérer et surveiller ces rejets :

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // Autoriser seulement 2 tâches dans la file d'attente
  onReject: (task, queuer) => {
    console.log('File d'attente pleine. Tâche rejetée:', task)
  }
})

queue.addItem(async () => 'first') // Acceptée
queue.addItem(async () => 'second') // Acceptée
queue.addItem(async () => 'third') // Rejetée, déclenche le callback onReject

console.log(queue.getRejectionCount()) // 1
```

### Tâches initiales

Vous pouvez pré-remplir une file d'attente asynchrone avec des tâches initiales lors de sa création :

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'first',
    async () => 'second',
    async () => 'third'
  ],
  started: true // Commencer le traitement immédiatement
})

// La file d'attente commence avec trois tâches et commence à les traiter
```

### Configuration dynamique

Les options de l'AsyncQueuer peuvent être modifiées après création en utilisant `setOptions()` et récupérées en utilisant `getOptions()` :

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// Changer la configuration
queue.setOptions({
  concurrency: 4, // Traiter plus de tâches simultanément
  started: true // Commencer le traitement
})

// Obtenir la configuration actuelle
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### Tâches actives et en attente

L'AsyncQueuer fournit des méthodes pour surveiller à la fois les tâches actives et en attente :

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// Ajouter quelques tâches
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'first'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'third')

// Surveiller les états des tâches
console.log(queue.getActiveItems().length) // Tâches actuellement en cours de traitement
console.log(queue.getPendingItems().length) // Tâches en attente de traitement
```

### Adaptateurs pour frameworks

Chaque adaptateur de framework construit des hooks et fonctions pratiques autour des classes de gestionnaire de file d'attente asynchrone. Des hooks comme `useAsyncQueuer` ou `useAsyncQueuedState` sont de petits wrappers qui peuvent réduire le boilerplate nécessaire dans votre propre code pour certains cas d'utilisation courants.
