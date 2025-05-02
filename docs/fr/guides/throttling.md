---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:33:56.489Z'
title: Guide sur le throttling
id: throttling
---
# Guide du Throttling (Limitation de fréquence)

La limitation de débit (Rate Limiting), le throttling et le debouncing sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" (avec perte) - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de Throttling de TanStack Pacer.

## Concept du Throttling

Le throttling garantit que les exécutions de fonction sont espacées uniformément dans le temps. Contrairement à la limitation de débit qui autorise des rafales d'exécutions jusqu'à une limite, ou au debouncing qui attend que l'activité s'arrête, le throttling crée un modèle d'exécution plus fluide en imposant des délais constants entre les appels. Si vous définissez un throttling d'une exécution par seconde, les appels seront espacés uniformément, quelle que soit la rapidité avec laquelle ils sont demandés.

### Visualisation du Throttling

```text
Throttling (one execution per 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Executed:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Seule une exécution autorisée toutes les 3 ticks,
               peu importe le nombre d'appels effectués

             [Première rafale]    [Appels supplémentaires]    [Appels espacés]
             Exécute le premier   Exécute après la période    Exécute à chaque fois
             puis throttle       d'attente                    que la période passe
```

### Quand utiliser le Throttling

Le throttling est particulièrement efficace lorsque vous avez besoin d'une temporisation d'exécution cohérente et prévisible. Cela le rend idéal pour gérer des événements ou des mises à jour fréquents où vous souhaitez un comportement fluide et contrôlé.

Cas d'usage courants :
- Mises à jour d'interface utilisateur nécessitant une temporisation cohérente (ex. : indicateurs de progression)
- Gestionnaires d'événements de défilement ou de redimensionnement qui ne doivent pas submerger le navigateur
- Interrogation de données en temps réel où des intervalles constants sont souhaités
- Opérations gourmandes en ressources nécessitant un rythme régulier
- Mises à jour de boucle de jeu ou gestion d'images d'animation
- Suggestions de recherche en direct pendant la saisie utilisateur

### Quand ne pas utiliser le Throttling

Le throttling pourrait ne pas être le meilleur choix lorsque :
- Vous voulez attendre que l'activité s'arrête (utilisez plutôt le [debouncing](../guides/debouncing))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [mise en file d'attente (queueing)](../guides/queueing))

> [!ASTUCE]
> Le throttling est souvent le meilleur choix lorsque vous avez besoin d'une temporisation d'exécution fluide et cohérente. Il offre un modèle d'exécution plus prévisible que la limitation de débit et un retour plus immédiat que le debouncing.

## Throttling dans TanStack Pacer

TanStack Pacer fournit à la fois du throttling synchrone et asynchrone via les classes `Throttler` et `AsyncThrottler` respectivement (et leurs fonctions correspondantes `throttle` et `asyncThrottle`).

### Utilisation basique avec `throttle`

La fonction `throttle` est le moyen le plus simple d'ajouter du throttling à n'importe quelle fonction :

```ts
import { throttle } from '@tanstack/pacer'

// Limite les mises à jour UI à une fois toutes les 200ms
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// Dans une boucle rapide, n'exécute que toutes les 200ms
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // De nombreux appels sont throttlés
}
```

### Utilisation avancée avec la classe `Throttler`

Pour plus de contrôle sur le comportement du throttling, vous pouvez utiliser directement la classe `Throttler` :

```ts
import { Throttler } from '@tanstack/pacer'

const updateThrottler = new Throttler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Obtenir des informations sur l'état d'exécution
console.log(updateThrottler.getExecutionCount()) // Nombre d'exécutions réussies
console.log(updateThrottler.getLastExecutionTime()) // Horodatage de la dernière exécution

// Annuler toute exécution en attente
updateThrottler.cancel()
```

### Exécutions Leading et Trailing

Le throttler synchrone prend en charge les exécutions sur les fronts montant et descendant :

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Exécute au premier appel (par défaut)
  trailing: true,  // Exécute après la période d'attente (par défaut)
})
```

- `leading: true` (par défaut) - Exécute immédiatement au premier appel
- `leading: false` - Ignore le premier appel, attend l'exécution trailing
- `trailing: true` (par défaut) - Exécute le dernier appel après la période d'attente
- `trailing: false` - Ignore le dernier appel s'il est dans la période d'attente

Modèles courants :
- `{ leading: true, trailing: true }` - Par défaut, le plus réactif
- `{ leading: false, trailing: true }` - Retarde toutes les exécutions
- `{ leading: true, trailing: false }` - Ignore les exécutions en file d'attente

### Activation/Désactivation

La classe `Throttler` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le throttler à tout moment :

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Désactivé par défaut
throttler.setOptions({ enabled: true }) // Active à tout moment
```

Si vous utilisez un adaptateur de framework où les options du throttler sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le throttler à la volée. Cependant, si vous utilisez la fonction `throttle` ou la classe `Throttler` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en réalité transmises au constructeur de la classe `Throttler`.

### Options de Callback

Les throttlers synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie du throttling :

#### Callbacks du Throttler Synchrone

Le `Throttler` synchrone prend en charge le callback suivant :

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction exécutée', throttler.getExecutionCount())
  }
})
```

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction throttlée, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'interface utilisateur ou effectuer des opérations de nettoyage.

#### Callbacks du Throttler Asynchrone

Le `AsyncThrottler` asynchrone prend en charge des callbacks supplémentaires pour la gestion des erreurs :

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction async exécutée', throttler.getExecutionCount())
  },
  onError: (error) => {
    // Appelé si la fonction async génère une erreur
    console.error('Échec de la fonction async :', error)
  }
})
```

Le callback `onExecute` fonctionne de la même manière que dans le throttler synchrone, tandis que le callback `onError` vous permet de gérer les erreurs sans interrompre la chaîne de throttling. Ces callbacks sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'interface utilisateur, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Throttling Asynchrone

Pour les fonctions asynchrones ou lorsque vous avez besoin de gestion d'erreurs, utilisez `AsyncThrottler` ou `asyncThrottle` :

```ts
import { asyncThrottle } from '@tanstack/pacer'

const throttledFetch = asyncThrottle(
  async (id: string) => {
    const response = await fetch(`/api/data/${id}`)
    return response.json()
  },
  {
    wait: 1000,
    onError: (error) => {
      console.error('Échec de l\'appel API :', error)
    }
  }
)

// Ne fera qu'un seul appel API par seconde
await throttledFetch('123')
```

La version asynchrone fournit un suivi d'exécution basé sur les Promises, une gestion des erreurs via le callback `onError`, un nettoyage approprié des opérations asynchrones en attente et une méthode `maybeExecute` awaitable.

### Adaptateurs de Framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur la fonctionnalité de throttling de base pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createThrottler`, `useThrottledCallback`, `useThrottledState` ou `useThrottledValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle total
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook de callback simple pour les cas d'usage basiques
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState(0)
const [throttledState, setThrottledState] = useThrottledValue(
  instantState, // Valeur à throttler
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// Hook bas niveau pour un contrôle total
const throttler = createThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook basé sur les signaux pour la gestion d'état
const [value, setValue, throttler] = createThrottledSignal(0, {
  wait: 200,
  onExecute: (throttler) => {
    console.log('Exécutions totales :', throttler.getExecutionCount())
  }
})
```

Chaque adaptateur de framework fournit des hooks qui s'intègrent au système de gestion d'état du framework tout en conservant la fonctionnalité de throttling de base.
