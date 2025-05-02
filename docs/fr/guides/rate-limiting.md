---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:34:02.833Z'
title: Guide sur la limitation de débit
id: rate-limiting
---
# Guide de Limitation de Débit (Rate Limiting)

La limitation de débit (Rate Limiting), le throttling et le debouncing sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour construire des applications performantes et fiables. Ce guide couvrira les concepts de limitation de débit de TanStack Pacer.

> [!NOTE]
> TanStack Pacer est actuellement uniquement une bibliothèque front-end. Ce sont des utilitaires pour la limitation de débit côté client.

## Concept de Limitation de Débit

La limitation de débit est une technique qui limite la fréquence à laquelle une fonction peut s'exécuter sur une fenêtre de temps spécifique. Elle est particulièrement utile pour les scénarios où vous souhaitez empêcher une fonction d'être appelée trop fréquemment, comme lors de la gestion de requêtes API ou d'autres appels à des services externes. C'est l'approche la plus *naïve*, car elle permet aux exécutions de se produire en rafales jusqu'à ce que le quota soit atteint.

### Visualisation de la Limitation de Débit

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Quand Utiliser la Limitation de Débit

La limitation de débit est particulièrement importante lors de la gestion d'opérations front-end qui pourraient accidentellement submerger vos services back-end ou causer des problèmes de performance dans le navigateur.

Cas d'usage courants :
- Empêcher le spam accidentel d'API dû à des interactions utilisateur rapides (par exemple, clics sur un bouton ou soumissions de formulaire)
- Scénarios où un comportement en rafales est acceptable mais où vous souhaitez limiter le débit maximum
- Protection contre les boucles infinies accidentelles ou les opérations récursives

### Quand Ne Pas Utiliser la Limitation de Débit

La limitation de débit est l'approche la plus naïve pour contrôler la fréquence d'exécution des fonctions. C'est la moins flexible et la plus restrictive des trois techniques. Envisagez d'utiliser [le throttling](../guides/throttling) ou [le debouncing](../guides/debouncing) à la place pour des exécutions plus espacées.

> [!TIP]
> Vous ne voudrez probablement pas utiliser la "limitation de débit" pour la plupart des cas d'usage. Envisagez d'utiliser [le throttling](../guides/throttling) ou [le debouncing](../guides/debouncing) à la place.

La nature "lossy" de la limitation de débit signifie également que certaines exécutions seront rejetées et perdues. Cela peut poser problème si vous devez vous assurer que toutes les exécutions réussissent toujours. Envisagez d'utiliser [la mise en file d'attente (queueing)](../guides/queueing) si vous devez vous assurer que toutes les exécutions sont mises en file d'attente pour être exécutées, mais avec un délai throttlé pour ralentir le taux d'exécution.

## Limitation de Débit dans TanStack Pacer

TanStack Pacer fournit une limitation de débit synchrone et asynchrone via les classes `RateLimiter` et `AsyncRateLimiter` respectivement (et leurs fonctions correspondantes `rateLimit` et `asyncRateLimit`).

### Utilisation Basique avec `rateLimit`

La fonction `rateLimit` est le moyen le plus simple d'ajouter une limitation de débit à n'importe quelle fonction. Elle est parfaite pour la plupart des cas d'usage où vous avez juste besoin d'appliquer une limite simple.

```ts
import { rateLimit } from '@tanstack/pacer'

// Limiter les appels API à 5 par minute
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 minute in milliseconds
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// First 5 calls will execute immediately
rateLimitedApi('user-1') // ✅ Executes
rateLimitedApi('user-2') // ✅ Executes
rateLimitedApi('user-3') // ✅ Executes
rateLimitedApi('user-4') // ✅ Executes
rateLimitedApi('user-5') // ✅ Executes
rateLimitedApi('user-6') // ❌ Rejected until window resets
```

### Utilisation Avancée avec la Classe `RateLimiter`

Pour des scénarios plus complexes où vous avez besoin d'un contrôle supplémentaire sur le comportement de limitation de débit, vous pouvez utiliser la classe `RateLimiter` directement. Cela vous donne accès à des méthodes et des informations d'état supplémentaires.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Create a rate limiter instance
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Function executed', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Get information about current state
console.log(limiter.getRemainingInWindow()) // Number of calls remaining in current window
console.log(limiter.getExecutionCount()) // Total number of successful executions
console.log(limiter.getRejectionCount()) // Total number of rejected executions

// Attempt to execute (returns boolean indicating success)
limiter.maybeExecute('user-1')

// Update options dynamically
limiter.setOptions({ limit: 10 }) // Increase the limit

// Reset all counters and state
limiter.reset()
```

### Activation/Désactivation

La classe `RateLimiter` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le limiteur de débit à tout moment :

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Disable by default
})
limiter.setOptions({ enabled: true }) // Enable at any time
```

Si vous utilisez un adaptateur de framework où les options du limiteur de débit sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le limiteur de débit à la volée. Cependant, si vous utilisez la fonction `rateLimit` ou la classe `RateLimiter` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait passées au constructeur de la classe `RateLimiter`.

### Options de Callback

Les limiteurs de débit synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie de la limitation de débit :

#### Callbacks du Limiteur de Débit Synchrone

Le `RateLimiter` synchrone prend en charge les callbacks suivants :

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Called after each successful execution
    console.log('Function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Called when an execution is rejected
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction limitée en débit, tandis que le callback `onReject` est appelé lorsqu'une exécution est rejetée en raison de la limitation de débit. Ces callbacks sont utiles pour suivre les exécutions, mettre à jour l'état de l'interface utilisateur ou fournir des retours aux utilisateurs.

#### Callbacks du Limiteur de Débit Asynchrone

Le `AsyncRateLimiter` asynchrone prend en charge des callbacks supplémentaires pour la gestion des erreurs :

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Called after each successful execution
    console.log('Async function executed', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Called when an execution is rejected
    console.log(`Rate limit exceeded. Try again in ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // Called if the async function throws an error
    console.error('Async function failed:', error)
  }
})
```

Les callbacks `onExecute` et `onReject` fonctionnent de la même manière que dans le limiteur de débit synchrone, tandis que le callback `onError` vous permet de gérer les erreurs sans interrompre la chaîne de limitation de débit. Ces callbacks sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'interface utilisateur, gérer les erreurs et fournir des retours aux utilisateurs.

### Limitation de Débit Asynchrone

Utilisez `AsyncRateLimiter` lorsque :
- Votre fonction limitée en débit retourne une Promise
- Vous devez gérer les erreurs de la fonction asynchrone
- Vous voulez vous assurer d'une limitation de débit correcte même si la fonction asynchrone prend du temps à s'exécuter

```ts
import { asyncRateLimit } from '@tanstack/pacer'

const rateLimited = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onError: (error) => {
      console.error('API call failed:', error)
    }
  }
)

// Returns a Promise<boolean> - resolves to true if executed, false if rejected
const wasExecuted = await rateLimited('123')
```

La version asynchrone fournit un suivi d'exécution basé sur les Promises, une gestion des erreurs via le callback `onError`, un nettoyage approprié des opérations asynchrones en attente et une méthode `maybeExecute` awaitable.

### Adaptateurs de Framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur la fonctionnalité de base de limitation de débit pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` ou `useRateLimitedValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// Low-level hook for full control
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Simple callback hook for basic use cases
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// State-based hook for reactive state management
const [instantState, setInstantState] = useState('')
const [rateLimitedState, setRateLimitedState] = useRateLimitedValue(
  instantState, // Value to rate limit
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from '@tanstack/solid-pacer'

// Low-level hook for full control
const limiter = createRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Signal-based hook for state management
const [value, setValue, limiter] = createRateLimitedSignal('', {
  limit: 5,
  window: 1000,
  onExecute: (limiter) => {
    console.log('Total executions:', limiter.getExecutionCount())
  }
})
```
