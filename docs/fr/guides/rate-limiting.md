---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:59:35.579Z'
title: Guide sur la limitation de débit
id: rate-limiting
---
# Guide sur la limitation de débit (Rate Limiting)

La limitation de débit (Rate Limiting), l'étranglement (Throttling) et l'anti-rebond (Debouncing) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "avec perte" (lossy) - ce qui signifie que certains appels de fonction ne s'exécuteront pas lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de limitation de débit de TanStack Pacer.

> [!NOTE]
> TanStack Pacer est actuellement uniquement une bibliothèque front-end. Ce sont des utilitaires pour la limitation de débit côté client.

## Concept de limitation de débit (Rate Limiting)

La limitation de débit est une technique qui limite la vitesse à laquelle une fonction peut s'exécuter sur une fenêtre de temps spécifique. Elle est particulièrement utile pour les scénarios où vous souhaitez empêcher une fonction d'être appelée trop fréquemment, comme lors de la gestion de requêtes API ou d'autres appels à des services externes. C'est l'approche la plus *naïve*, car elle permet des exécutions en rafale jusqu'à ce que le quota soit atteint.

### Visualisation de la limitation de débit

```text
Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ❌                             ✅     ✅
             [=== 3 allowed ===][=== blocked until window ends ===][=== new window =======]
```

### Types de fenêtres

TanStack Pacer prend en charge deux types de fenêtres de limitation de débit :

1. **Fenêtre fixe (Fixed Window)** (par défaut)
   - Une fenêtre stricte qui se réinitialise après la période de la fenêtre
   - Toutes les exécutions dans la fenêtre comptent dans la limite
   - La fenêtre se réinitialise complètement après la période
   - Peut entraîner un comportement en rafale aux limites des fenêtres

2. **Fenêtre glissante (Sliding Window)**
   - Une fenêtre roulante qui permet des exécutions lorsque les anciennes expirent
   - Fournit un taux d'exécution plus constant dans le temps
   - Mieux pour maintenir un flux constant d'exécutions
   - Empêche le comportement en rafale aux limites des fenêtres

Voici une visualisation de la limitation de débit avec fenêtre glissante :

```text
Sliding Window Rate Limiting (limit: 3 calls per window)
Timeline: [1 second per tick]
                                        Window 1                  |    Window 2            
Calls:        ⬇️     ⬇️     ⬇️     ⬇️     ⬇️                             ⬇️     ⬇️
Executed:     ✅     ✅     ✅     ❌     ✅                             ✅     ✅
             [=== 3 allowed ===][=== oldest expires, new allowed ===][=== continues sliding =======]
```

La différence clé est qu'avec une fenêtre glissante, dès que l'exécution la plus ancienne expire, une nouvelle exécution est autorisée. Cela crée un flux d'exécutions plus constant par rapport à l'approche de fenêtre fixe.

### Quand utiliser la limitation de débit

La limitation de débit est particulièrement importante lors de la gestion d'opérations front-end qui pourraient accidentellement submerger vos services back-end ou causer des problèmes de performance dans le navigateur.

Cas d'utilisation courants :
- Empêcher le spam accidentel d'API à partir d'interactions utilisateur rapides (par exemple, clics sur bouton ou soumissions de formulaire)
- Scénarios où un comportement en rafale est acceptable mais vous souhaitez limiter le taux maximum
- Protection contre les boucles infinies accidentelles ou les opérations récursives

### Quand ne pas utiliser la limitation de débit

La limitation de débit est l'approche la plus naïve pour contrôler la fréquence d'exécution des fonctions. C'est la moins flexible et la plus restrictive des trois techniques. Envisagez d'utiliser [l'étranglement (throttling)](../guides/throttling) ou [l'anti-rebond (debouncing)](../guides/debouncing) pour des exécutions plus espacées.

> [!TIP]
> Vous ne voulez probablement pas utiliser la "limitation de débit" pour la plupart des cas d'utilisation. Envisagez plutôt d'utiliser [l'étranglement (throttling)](../guides/throttling) ou [l'anti-rebond (debouncing)](../guides/debouncing).

La nature "avec perte" (lossy) de la limitation de débit signifie également que certaines exécutions seront rejetées et perdues. Cela peut poser problème si vous devez vous assurer que toutes les exécutions réussissent toujours. Envisagez d'utiliser [la mise en file d'attente (queueing)](../guides/queueing) si vous avez besoin de vous assurer que toutes les exécutions sont mises en file d'attente pour être exécutées, mais avec un délai limité pour ralentir le taux d'exécution.

## Limitation de débit dans TanStack Pacer

TanStack Pacer fournit une limitation de débit synchrone et asynchrone via les classes `RateLimiter` et `AsyncRateLimiter` respectivement (et leurs fonctions correspondantes `rateLimit` et `asyncRateLimit`).

### Utilisation de base avec `rateLimit`

La fonction `rateLimit` est le moyen le plus simple d'ajouter une limitation de débit à n'importe quelle fonction. Elle est parfaite pour la plupart des cas d'utilisation où vous avez juste besoin d'appliquer une limite simple.

```ts
import { rateLimit } from '@tanstack/pacer'

// Limiter les appels API à 5 par minute
const rateLimitedApi = rateLimit(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000, // 1 minute en millisecondes
    windowType: 'fixed', // par défaut
    onReject: (rateLimiter) => {
      console.log(`Limite de débit dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
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

Pour des scénarios plus complexes où vous avez besoin d'un contrôle supplémentaire sur le comportement de limitation de débit, vous pouvez utiliser directement la classe `RateLimiter`. Cela vous donne accès à des méthodes et des informations d'état supplémentaires.

```ts
import { RateLimiter } from '@tanstack/pacer'

// Créer une instance de limiteur de débit
const limiter = new RateLimiter(
  (id: string) => fetchUserData(id),
  {
    limit: 5,
    window: 60 * 1000,
    onExecute: (rateLimiter) => {
      console.log('Fonction exécutée', rateLimiter.getExecutionCount())
    },
    onReject: (rateLimiter) => {
      console.log(`Limite de débit dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
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

La classe `RateLimiter` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le limiteur de débit à tout moment :

> [!NOTE]
> L'option `enabled` active/désactive l'exécution réelle de la fonction. Désactiver le limiteur de débit ne désactive pas la limitation de débit, cela empêche simplement la fonction de s'exécuter.

```ts
const limiter = new RateLimiter(fn, { 
  limit: 5, 
  window: 1000,
  enabled: false // Désactivé par défaut
})
limiter.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options du limiteur de débit sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le limiteur de débit à la volée. Cependant, si vous utilisez directement la fonction `rateLimit` ou la classe `RateLimiter`, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `RateLimiter`.

### Options de rappel (Callbacks)

Les limiteurs de débit synchrones et asynchrones prennent en charge des options de rappel pour gérer différents aspects du cycle de vie de la limitation de débit :

#### Rappels du limiteur de débit synchrone

Le `RateLimiter` synchrone prend en charge les rappels suivants :

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
    console.log(`Limite de débit dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
  }
})
```

Le rappel `onExecute` est appelé après chaque exécution réussie de la fonction limitée, tandis que le rappel `onReject` est appelé lorsqu'une exécution est rejetée en raison de la limitation de débit. Ces rappels sont utiles pour suivre les exécutions, mettre à jour l'état de l'interface utilisateur ou fournir des retours aux utilisateurs.

#### Rappels du limiteur de débit asynchrone

Le `AsyncRateLimiter` asynchrone prend en charge des rappels supplémentaires pour la gestion des erreurs :

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
    console.log(`Limite de débit dépassée. Réessayez dans ${rateLimiter.getMsUntilNextWindow()}ms`)
  },
  onError: (error) => {
    // Appelé si la fonction asynchrone génère une erreur
    console.error('Échec de la fonction asynchrone :', error)
  }
})
```

Les rappels `onExecute` et `onReject` fonctionnent de la même manière que dans le limiteur de débit synchrone, tandis que le rappel `onError` vous permet de gérer les erreurs avec élégance sans interrompre la chaîne de limitation de débit. Ces rappels sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'interface utilisateur, gérer les erreurs et fournir des retours aux utilisateurs.

### Limitation de débit asynchrone

Le limiteur de débit asynchrone fournit un moyen puissant de gérer les opérations asynchrones avec limitation de débit, offrant plusieurs avantages clés par rapport à la version synchrone. Alors que le limiteur de débit synchrone est idéal pour les événements d'interface utilisateur et les retours immédiats, la version asynchrone est spécialement conçue pour gérer les appels API, les opérations de base de données et d'autres tâches asynchrones.

#### Différences clés par rapport à la limitation de débit synchrone

1. **Gestion des valeurs de retour**
Contrairement au limiteur de débit synchrone qui renvoie un booléen indiquant le succès, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction limitée. Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels API ou d'autres opérations asynchrones. La méthode `maybeExecute` renvoie une Promesse qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le gérer de manière appropriée.

2. **Rappels différents**
Le `AsyncRateLimiter` prend en charge les rappels suivants au lieu de simplement `onExecute` dans la version synchrone :
- `onSuccess` : Appelé après chaque exécution réussie, fournissant l'instance du limiteur de débit
- `onSettled` : Appelé après chaque exécution, fournissant l'instance du limiteur de débit
- `onError` : Appelé si la fonction asynchrone génère une erreur, fournissant à la fois l'erreur et l'instance du limiteur de débit

Les deux limiteurs de débit asynchrone et synchrone prennent en charge le rappel `onReject` pour gérer les exécutions bloquées.

3. **Exécution séquentielle**
Puisque la méthode `maybeExecute` du limiteur de débit renvoie une Promesse, vous pouvez choisir d'attendre chaque exécution avant de commencer la suivante. Cela vous donne le contrôle sur l'ordre d'exécution et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement utile lors de la gestion d'opérations qui dépendent des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur et que vous récupérez immédiatement ses données mises à jour, vous pouvez attendre l'opération de mise à jour avant de lancer la récupération :

#### Exemple d'utilisation de base

Voici un exemple de base montrant comment utiliser le limiteur de débit asynchrone pour une opération API :

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
      console.log(`Limite de débit dépassée. Réessayez dans ${limiter.getMsUntilNextWindow()}ms`)
    },
    onError: (error, limiter) => {
      console.error('Échec de l'appel API :', error)
    }
  }
)

// Utilisation
const result = await rateLimitedApi('123')
```

### Adaptateurs pour frameworks

Chaque adaptateur de framework fournit des hooks qui s'appuient sur la fonctionnalité de base de limitation de débit pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createRateLimiter`, `useRateLimitedCallback`, `useRateLimitedState` ou `useRateLimitedValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useRateLimiter, useRateLimitedCallback, useRateLimitedValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle complet
const limiter = useRateLimiter(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Hook de rappel simple pour les cas d'utilisation de base
const handleFetch = useRateLimitedCallback(
  (id: string) => fetchUserData(id),
  { limit: 5, window: 1000 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState('')
const [rateLimitedValue] = useRateLimitedValue(
  instantState, // Valeur à limiter
  { limit: 5, window: 1000 }
)
```

#### Solid

```tsx
import { createRateLimiter, createRateLimitedSignal } from
