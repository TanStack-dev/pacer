---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:17:19.300Z'
title: Guide sur la limitation de débit
id: rate-limiting
---
# Guide de limitation de débit (Rate Limiting)

La limitation de débit (Rate Limiting), le throttling et le debouncing sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" - ce qui signifie que certains appels de fonction ne s'exécuteront pas lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour construire des applications performantes et fiables. Ce guide couvrira les concepts de limitation de débit (Rate Limiting) de TanStack Pacer.

> [!NOTE]
> TanStack Pacer est actuellement uniquement une bibliothèque front-end. Ce sont des utilitaires pour la limitation de débit (Rate Limiting) côté client.

## Concept de limitation de débit (Rate Limiting)

La limitation de débit (Rate Limiting) est une technique qui limite la fréquence à laquelle une fonction peut s'exécuter sur une fenêtre de temps spécifique. Elle est particulièrement utile pour les scénarios où vous souhaitez empêcher une fonction d'être appelée trop fréquemment, comme lors de la gestion de requêtes API ou d'autres appels à des services externes. C'est l'approche la plus *naïve*, car elle permet aux exécutions de se produire par rafales jusqu'à ce que le quota soit atteint.

### Visualisation de la limitation de débit (Rate Limiting)

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Quand utiliser la limitation de débit (Rate Limiting)

La limitation de débit (Rate Limiting) est particulièrement importante lors de la gestion d'opérations front-end qui pourraient accidentellement submerger vos services back-end ou causer des problèmes de performance dans le navigateur.

Cas d'utilisation courants :
- Empêcher le spam accidentel d'API dû à des interactions utilisateur rapides (par exemple, clics sur un bouton ou soumissions de formulaire)
- Scénarios où un comportement par rafales est acceptable mais où vous souhaitez limiter le débit maximum
- Protection contre les boucles infinies accidentelles ou les opérations récursives

### Quand ne pas utiliser la limitation de débit (Rate Limiting)

La limitation de débit (Rate Limiting) est l'approche la plus naïve pour contrôler la fréquence d'exécution des fonctions. C'est la moins flexible et la plus restrictive des trois techniques. Envisagez d'utiliser le [throttling](../guides/throttling) ou le [debouncing](../guides/debouncing) pour des exécutions plus espacées.

> [!TIP]
> Vous ne voudrez probablement pas utiliser la "limitation de débit (Rate Limiting)" pour la plupart des cas d'utilisation. Envisagez plutôt d'utiliser le [throttling](../guides/throttling) ou le [debouncing](../guides/debouncing).

La nature "lossy" de la limitation de débit (Rate Limiting) signifie également que certaines exécutions seront rejetées et perdues. Cela peut poser problème si vous devez vous assurer que toutes les exécutions réussissent toujours. Envisagez d'utiliser la [mise en file d'attente (queueing)](../guides/queueing) si vous avez besoin de vous assurer que toutes les exécutions sont en file d'attente pour être exécutées, mais avec un délai throttlé pour ralentir le taux d'exécution.

## Limitation de débit (Rate Limiting) dans TanStack Pacer

TanStack Pacer fournit une limitation de débit (Rate Limiting) synchrone et asynchrone via les classes `RateLimiter` et `AsyncRateLimiter` respectivement (et leurs fonctions correspondantes `rateLimit` et `asyncRateLimit`).

### Utilisation de base avec `rateLimit`

La fonction `rateLimit` est le moyen le plus simple d'ajouter une limitation de débit (Rate Limiting) à n'importe quelle fonction. Elle est parfaite pour la plupart des cas d'utilisation où vous avez juste besoin d'appliquer une limite simple.

```ts
import { rateLimit } from '@tanstack/pacer'

// Limiter les appels API à 5 par minute
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 minute en millisecondes
    onReject: (rateLimiter) => {
      console.log(`Limite de débit (Rate Limit) dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Les 5 premiers appels s'exécuteront immédiatement
rateLimitedApi('user-1') // ✅ Exécuté
rateLimitedApi('user-2') // ✅ Exécuté
rateLimitedApi('user-3') // ✅ Exécuté
rateLimitedApi('user-4') // ✅ Exécuté
rateLimitedApi('user-5') // ✅ Exécuté
rateLimitedApi('user-6') // ❌ Rejeté jusqu'à la réinitialisation de la fenêtre
```

### Utilisation avancée avec la classe `RateLimiter`

Pour des scénarios plus complexes où vous avez besoin d'un contrôle supplémentaire sur le comportement de limitation de débit (Rate Limiting), vous pouvez utiliser directement la classe `RateLimiter`. Cela vous donne accès à des méthodes et des informations d'état supplémentaires.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Créer une instance de limiteur de débit (Rate Limiter)
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Fonction exécutée', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Limite de débit (Rate Limit) dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
    }
  }
)

// Obtenir des informations sur l'état actuel
console.log(limiter.getRemainingInWindow()) // Nombre d'appels restants dans la fenêtre actuelle
console.log(limiter.getExecutionCount()) // Nombre total d'exécutions réussies
console.log(limiter.getRejectionCount()) // Nombre total d'exécutions rejetées

// Tenter d'exécuter (retourne un booléen indiquant le succès)
limiter.maybeExecute('user-1')

// Mettre à jour les options dynamiquement
limiter.setOptions({ limit: 10 }) // Augmenter la limite

// Réinitialiser tous les compteurs et l'état
limiter.reset()
```

### Activation/Désactivation

La classe `RateLimiter` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le limiteur de débit (Rate Limiter) à tout moment :

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Désactivé par défaut
})
limiter.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options du limiteur de débit (Rate Limiter) sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le limiteur de débit (Rate Limiter) à la volée. Cependant, si vous utilisez la fonction `rateLimit` ou la classe `RateLimiter` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `RateLimiter`.

### Options de callback

Les limiteurs de débit (Rate Limiters) synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie de la limitation de débit (Rate Limiting) :

#### Callbacks du limiteur de débit (Rate Limiter) synchrone

Le `RateLimiter` synchrone prend en charge les callbacks suivants :

```ts
const limiter = new RateLimiter(fn, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction exécutée', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Appelé lorsqu'une exécution est rejetée
    console.log(`Limite de débit (Rate Limit) dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction avec limitation de débit (Rate Limited), tandis que le callback `onReject` est appelé lorsqu'une exécution est rejetée en raison de la limitation de débit (Rate Limiting). Ces callbacks sont utiles pour suivre les exécutions, mettre à jour l'état de l'interface utilisateur ou fournir des retours aux utilisateurs.

#### Callbacks du limiteur de débit (Rate Limiter) asynchrone

Le `AsyncRateLimiter` asynchrone prend en charge des callbacks supplémentaires pour la gestion des erreurs :

```ts
const asyncLimiter = new AsyncRateLimiter(async (id) => {
  await saveToAPI(id)
}, {
  limit: 5,
  window: 1000,
  onExecute: (rateLimiter) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction asynchrone exécutée', rateLimiter.getExecutionCount())
  },
  onReject: (rateLimiter) => {
    // Appelé lorsqu'une exécution est rejetée
    console.log(`Limite de débit (Rate Limit) dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // Appelé si la fonction asynchrone génère une erreur
    console.error('Échec de la fonction asynchrone :', error)
  }
})
```

Les callbacks `onExecute` et `onReject` fonctionnent de la même manière que dans le limiteur de débit (Rate Limiter) synchrone, tandis que le callback `onError` vous permet de gérer les erreurs avec élégance sans interrompre la chaîne de limitation de débit (Rate Limiting). Ces callbacks sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'interface utilisateur, gérer les erreurs et fournir des retours aux utilisateurs.

### Limitation de débit (Rate Limiting) asynchrone

Le limiteur de débit (Rate Limiter) asynchrone offre un moyen puissant de gérer les opérations asynchrones avec limitation de débit (Rate Limiting), offrant plusieurs avantages clés par rapport à la version synchrone. Alors que le limiteur de débit (Rate Limiter) synchrone est idéal pour les événements d'interface utilisateur et les retours immédiats, la version asynchrone est spécialement conçue pour gérer les appels API, les opérations de base de données et d'autres tâches asynchrones.

#### Différences clés par rapport à la limitation de débit (Rate Limiting) synchrone

1. **Gestion des valeurs de retour**
Contrairement au limiteur de débit (Rate Limiter) synchrone qui retourne un booléen indiquant le succès, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction avec limitation de débit (Rate Limited). Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels API ou d'autres opérations asynchrones. La méthode `maybeExecute` retourne une Promise qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le gérer de manière appropriée.

2. **Système de callback amélioré**
Le limiteur de débit (Rate Limiter) asynchrone fournit un système de callback plus sophistiqué que les callbacks de la version synchrone. Ce système comprend :
- `onExecute` : Appelé après chaque exécution réussie, fournissant l'instance du limiteur de débit (Rate Limiter)
- `onReject` : Appelé lorsqu'une exécution est rejetée en raison de la limitation de débit (Rate Limiting), fournissant l'instance du limiteur de débit (Rate Limiter)
- `onError` : Appelé si la fonction asynchrone génère une erreur, fournissant à la fois l'erreur et l'instance du limiteur de débit (Rate Limiter)

3. **Suivi des exécutions**
Le limiteur de débit (Rate Limiter) asynchrone fournit un suivi complet des exécutions via plusieurs méthodes :
- `getExecutionCount()` : Nombre d'exécutions réussies
- `getRejectionCount()` : Nombre d'exécutions rejetées
- `getRemainingInWindow()` : Nombre d'exécutions restantes dans la fenêtre actuelle
- `getMsUntilNextWindow()` : Millisecondes jusqu'au début de la prochaine fenêtre

4. **Exécution séquentielle**
Le limiteur de débit (Rate Limiter) asynchrone garantit que les exécutions suivantes attendent la fin de l'appel précédent avant de commencer. Cela empêche les exécutions désordonnées et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement important lors de la gestion d'opérations qui dépendent des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur et que vous récupérez immédiatement ses données mises à jour, le limiteur de débit (Rate Limiter) asynchrone garantira que l'opération de récupération attend la fin de la mise à jour, évitant ainsi les conditions de course où vous pourriez obtenir des données obsolètes.

#### Exemple d'utilisation de base

Voici un exemple de base montrant comment utiliser le limiteur de débit (Rate Limiter) asynchrone pour une opération API :

```ts
const rateLimitedApi = asyncRateLimit(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    limit: 5,
    window: 1000,
    onExecute: (limiter) => {
      console.log('Appel API réussi :', limiter.getExecutionCount())
    },
    onReject: (limiter) => {
      console.log(`Limite de débit (Rate Limit) dépassée. Réessayez dans ${limiter.getMsUntilNextWindow()}ms`)
    },
    onError: (error, limiter) => {
      console.error('Échec de l'appel API :', error)
    }
  }
)

// Utilisation
const result = await rateLimitedApi('123')
```

#### Modèles avancés

Le limiteur de débit (Rate Limiter) asynchrone peut être combiné avec divers modèles pour résoudre des problèmes complexes :

1. **Intégration avec la gestion d'état**
Lorsque vous utilisez le limiteur de débit (Rate Limiter) asynchrone avec des systèmes de gestion d'état (comme useState de React ou createSignal de Solid), vous pouvez créer des modèles puissants pour gérer les états de chargement, les états d'erreur et les mises à jour de données. Les callbacks du limiteur de débit (Rate Limiter) fournissent des hooks parfaits pour mettre à jour l'état de l'interface utilisateur en fonction du succès ou de l'échec des opérations.

2. **Prévention des conditions de course**
Le modèle de limitation de débit (Rate Limiting) prévient naturellement les conditions de course dans de nombreux scénarios. Lorsque plusieurs parties de votre application tentent de mettre à jour la même ressource simultanément, le limiteur de débit (Rate Limiter) garantit que les mises à jour se produisent dans les limites configurées, tout en fournissant des résultats à tous les appelants.

3. **Récupération après erreur**
Les capacités de gestion des erreurs du limiteur de débit (Rate Limiter) asynchrone en font un outil idéal pour implémenter une logique de réessai et des modèles de récupération après erreur. Vous pouvez utiliser le callback `onError` pour implémenter des stratégies de gestion d'erreur personnalisées, comme un backoff exponentiel ou des mécanismes de repli.

### Adaptateurs de framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur les fonctionnalités de base de limitation de débit (Rate Limiting) pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` ou `useRateLimitedValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback,
