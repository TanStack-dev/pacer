---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:34:34.775Z'
title: Guide sur la mise en file d'attente asynchrone
id: async-queueing
---
# Guide de mise en file d'attente asynchrone (Async Queueing Guide)

Alors que le [Queuer](../guides//queueing) fournit une mise en file d'attente synchrone avec des contrôles de temporisation, l'`AsyncQueuer` est conçu spécifiquement pour gérer des opérations asynchrones concurrentes. Il implémente ce qu'on appelle traditionnellement un modèle de "pool de tâches" (task pool) ou "pool de travailleurs" (worker pool), permettant à plusieurs opérations d'être traitées simultanément tout en conservant le contrôle sur la concurrence et la temporisation. L'implémentation est principalement copiée de [Swimmer](https://github.com/tannerlinsley/swimmer), l'utilitaire original de pooling de tâches de Tanner qui sert la communauté JavaScript depuis 2017.

## Concept de mise en file d'attente asynchrone (Async Queueing)

La mise en file d'attente asynchrone étend le concept de base de la file d'attente en ajoutant des capacités de traitement concurrent. Au lieu de traiter un élément à la fois, un gestionnaire de file d'attente asynchrone peut traiter plusieurs éléments simultanément tout en conservant l'ordre et le contrôle sur l'exécution. Ceci est particulièrement utile pour les opérations d'E/S (I/O), les requêtes réseau ou toute tâche qui passe la plupart de son temps en attente plutôt qu'à consommer du CPU.

### Visualisation de la mise en file d'attente asynchrone

```text
Async Queueing (concurrency: 2, wait: 2 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️     ⬇️
Queue:       [ABC]   [C]    [CDE]    [E]    []
Active:      [A,B]   [B,C]  [C,D]    [D,E]  [E]
Completed:    -       A      B        C      D,E
             [=================================================================]
             ^ Contrairement à une file d'attente classique, plusieurs éléments
               peuvent être traités simultanément

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
- Effectuer des requêtes API concurrentes avec limitation de débit (rate limiting)
- Traiter plusieurs téléchargements de fichiers simultanément
- Exécuter des opérations de base de données en parallèle
- Gérer plusieurs connexions websocket
- Traiter des flux de données avec contre-pression (backpressure)
- Gérer des tâches en arrière-plan intensives en ressources

### Quand ne pas utiliser la mise en file d'attente asynchrone

L'AsyncQueuer est très polyvalent et peut être utilisé dans de nombreuses situations. En réalité, il n'est pas adapté uniquement lorsque vous ne prévoyez pas de tirer parti de toutes ses fonctionnalités. Si vous n'avez pas besoin que toutes les exécutions en file d'attente soient traitées, utilisez plutôt [Throttling][../guides/throttling]. Si vous n'avez pas besoin de traitement concurrent, utilisez plutôt [Queueing][../guides/queueing].

## Mise en file d'attente asynchrone dans TanStack Pacer

TanStack Pacer fournit la mise en file d'attente asynchrone via la fonction simple `asyncQueue` et la classe plus puissante `AsyncQueuer`.

### Utilisation de base avec `asyncQueue`

La fonction `asyncQueue` offre un moyen simple de créer une file d'attente asynchrone toujours active :

```ts
import { asyncQueue } from '@tanstack/pacer'

// Crée une file d'attente traitant jusqu'à 2 éléments simultanément
const processItems = asyncQueue<string>({
  concurrency: 2,
  onItemsChange: (queuer) => {
    console.log('Tâches actives:', queuer.getActiveItems().length)
  }
})

// Ajoute des tâches asynchrones à traiter
processItems(async () => {
  const result = await fetchData(1)
  return result
})

processItems(async () => {
  const result = await fetchData(2)
  return result
})
```

L'utilisation de la fonction `asyncQueue` est quelque peu limitée, car il s'agit simplement d'un wrapper autour de la classe `AsyncQueuer` qui n'expose que la méthode `addItem`. Pour un meilleur contrôle sur la file d'attente, utilisez directement la classe `AsyncQueuer`.

### Utilisation avancée avec la classe `AsyncQueuer`

La classe `AsyncQueuer` offre un contrôle complet sur le comportement de la file d'attente asynchrone :

```ts
import { AsyncQueuer } from '@tanstack/pacer'

const queue = new AsyncQueuer<string>({
  concurrency: 2, // Traite 2 éléments à la fois
  wait: 1000,     // Attend 1 seconde entre le démarrage de nouveaux éléments
  started: true   // Commence le traitement immédiatement
})

// Ajoute des gestionnaires d'erreur et de succès
queue.onError((error) => {
  console.error('Échec de la tâche:', error)
})

queue.onSuccess((result) => {
  console.log('Tâche terminée:', result)
})

// Ajoute des tâches asynchrones
queue.addItem(async () => {
  const result = await fetchData(1)
  return result
})

queue.addItem(async () => {
  const result = await fetchData(2)
  return result
})
```

### Types de files d'attente et ordonnancement

L'AsyncQueuer prend en charge différentes stratégies de mise en file d'attente pour répondre à divers besoins de traitement. Chaque stratégie détermine comment les tâches sont ajoutées et traitées depuis la file d'attente.

#### File FIFO (Premier entré, premier sorti)

Les files FIFO traitent les tâches dans l'ordre exact où elles ont été ajoutées, ce qui les rend idéales pour maintenir une séquence :

```ts
const queue = new AsyncQueuer<string>({
  addItemsTo: 'back',  // par défaut
  getItemsFrom: 'front', // par défaut
  concurrency: 2
})

queue.addItem(async () => 'premier')  // [premier]
queue.addItem(async () => 'second') // [premier, second]
// Traitement : premier et second simultanément
```

#### Pile LIFO (Dernier entré, premier sorti)

Les piles LIFO traitent d'abord les tâches les plus récemment ajoutées, utiles pour prioriser les nouvelles tâches :

```ts
const stack = new AsyncQueuer<string>({
  addItemsTo: 'back',
  getItemsFrom: 'back', // Traite d'abord les éléments les plus récents
  concurrency: 2
})

stack.addItem(async () => 'premier')  // [premier]
stack.addItem(async () => 'second') // [premier, second]
// Traitement : second en premier, puis premier
```

#### File à priorité (Priority Queue)

Les files à priorité traitent les tâches en fonction de leurs valeurs de priorité attribuées, garantissant que les tâches importantes sont traitées en premier. Il existe deux façons de spécifier les priorités :

1. Valeurs de priorité statiques attachées aux tâches :
```ts
const priorityQueue = new AsyncQueuer<string>({
  concurrency: 2
})

// Crée des tâches avec des valeurs de priorité statiques
const lowPriorityTask = Object.assign(
  async () => 'résultat basse priorité',
  { priority: 1 }
)

const highPriorityTask = Object.assign(
  async () => 'résultat haute priorité',
  { priority: 3 }
)

const mediumPriorityTask = Object.assign(
  async () => 'résultat priorité moyenne',
  { priority: 2 }
)

// Ajoute des tâches dans n'importe quel ordre - elles seront traitées par priorité (nombres plus élevés en premier)
priorityQueue.addItem(lowPriorityTask)
priorityQueue.addItem(highPriorityTask)
priorityQueue.addItem(mediumPriorityTask)
// Traitement : haute et moyenne simultanément, puis basse
```

2. Calcul dynamique de priorité avec l'option `getPriority` :
```ts
const dynamicPriorityQueue = new AsyncQueuer<string>({
  concurrency: 2,
  getPriority: (task) => {
    // Calcule la priorité en fonction des propriétés de la tâche ou d'autres facteurs
    // Les nombres plus élevés ont la priorité
    return calculateTaskPriority(task)
  }
})

// Ajoute des tâches - la priorité sera calculée dynamiquement
dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('basse')
  return result
})

dynamicPriorityQueue.addItem(async () => {
  const result = await processTask('haute')
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

// Gère les erreurs globalement
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

// Gère les erreurs par tâche
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
queue.getPeek()           // Affiche l'élément suivant sans le retirer
queue.getSize()          // Obtient la taille actuelle de la file d'attente
queue.getIsEmpty()       // Vérifie si la file d'attente est vide
queue.getIsFull()        // Vérifie si la file d'attente a atteint maxSize
queue.getAllItems()   // Obtient une copie de tous les éléments en file d'attente
queue.getActiveItems() // Obtient les éléments en cours de traitement
queue.getPendingItems() // Obtient les éléments en attente de traitement

// Manipulation de la file d'attente
queue.clear()         // Supprime tous les éléments
queue.reset()         // Réinitialise à l'état initial
queue.getExecutionCount() // Obtient le nombre d'éléments traités

// Contrôle du traitement
queue.start()         // Commence le traitement des éléments
queue.stop()          // Met en pause le traitement
queue.getIsRunning()     // Vérifie si la file d'attente est en cours de traitement
queue.getIsIdle()        // Vérifie si la file d'attente est vide et non en traitement
```

### Callbacks de tâche

L'AsyncQueuer fournit trois types de callbacks pour surveiller l'exécution des tâches :

```ts
const queue = new AsyncQueuer<string>()

// Gère la réussite d'une tâche
const unsubSuccess = queue.onSuccess((result) => {
  console.log('Tâche réussie:', result)
})

// Gère les erreurs de tâche
const unsubError = queue.onError((error) => {
  console.error('Échec de la tâche:', error)
})

// Gère la terminaison d'une tâche, qu'elle ait réussi ou échoué
const unsubSettled = queue.onSettled((result) => {
  if (result instanceof Error) {
    console.log('Échec de la tâche:', result)
  } else {
    console.log('Tâche réussie:', result)
  }
})

// Se désabonne des callbacks lorsqu'ils ne sont plus nécessaires
unsubSuccess()
unsubError()
unsubSettled()
```

### Gestion des rejets

Lorsqu'une file d'attente atteint sa taille maximale (définie par l'option `maxSize`), les nouvelles tâches seront rejetées. L'AsyncQueuer fournit des moyens de gérer et surveiller ces rejets :

```ts
const queue = new AsyncQueuer<string>({
  maxSize: 2, // Autorise seulement 2 tâches dans la file d'attente
  onReject: (task, queuer) => {
    console.log('File d\'attente pleine. Tâche rejetée:', task)
  }
})

queue.addItem(async () => 'premier') // Acceptée
queue.addItem(async () => 'second') // Acceptée
queue.addItem(async () => 'troisième') // Rejetée, déclenche le callback onReject

console.log(queue.getRejectionCount()) // 1
```

### Tâches initiales

Vous pouvez pré-remplir une file d'attente asynchrone avec des tâches initiales lors de sa création :

```ts
const queue = new AsyncQueuer<string>({
  initialItems: [
    async () => 'premier',
    async () => 'second',
    async () => 'troisième'
  ],
  started: true // Commence le traitement immédiatement
})

// La file d'attente commence avec trois tâches et commence à les traiter
```

### Configuration dynamique

Les options de l'AsyncQueuer peuvent être modifiées après la création en utilisant `setOptions()` et récupérées en utilisant `getOptions()` :

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2,
  started: false
})

// Change la configuration
queue.setOptions({
  concurrency: 4, // Traite plus de tâches simultanément
  started: true // Commence le traitement
})

// Obtient la configuration actuelle
const options = queue.getOptions()
console.log(options.concurrency) // 4
```

### Tâches actives et en attente

L'AsyncQueuer fournit des méthodes pour surveiller à la fois les tâches actives et en attente :

```ts
const queue = new AsyncQueuer<string>({
  concurrency: 2
})

// Ajoute quelques tâches
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'premier'
})
queue.addItem(async () => {
  await new Promise(resolve => setTimeout(resolve, 1000))
  return 'second'
})
queue.addItem(async () => 'troisième')

// Surveille l'état des tâches
console.log(queue.getActiveItems().length) // Tâches en cours de traitement
console.log(queue.getPendingItems().length) // Tâches en attente de traitement
```

### Adaptateurs pour frameworks

Chaque adaptateur de framework construit des hooks et fonctions pratiques autour des classes de gestionnaires de files d'attente asynchrones. Des hooks comme `useAsyncQueuer` ou `useAsyncQueuerState` sont de petits wrappers qui peuvent réduire le code passe-partout nécessaire dans votre propre code pour certains cas d'utilisation courants.
