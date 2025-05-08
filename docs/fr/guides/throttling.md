---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:59:04.403Z'
title: Guide sur le throttling
id: throttling
---
# Guide sur le Throttling (Limitation de Fréquence)

La limitation de débit (Rate Limiting), le throttling (limitation de fréquence) et le debouncing (anti-rebond) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" (avec perte) - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de throttling de TanStack Pacer.

## Concept de Throttling

Le throttling garantit que les exécutions de fonction sont régulièrement espacées dans le temps. Contrairement à la limitation de débit qui permet des rafales d'exécutions jusqu'à une limite, ou au debouncing qui attend que l'activité s'arrête, le throttling crée un modèle d'exécution plus fluide en imposant des délais constants entre les appels. Si vous définissez un throttling d'une exécution par seconde, les appels seront espacés uniformément, quelle que soit la fréquence à laquelle ils sont demandés.

### Visualisation du Throttling

```text
Throttling (one execution per 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Executed:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Only one execution allowed per 3 ticks,
               regardless of how many calls are made

             [First burst]    [More calls]              [Spaced calls]
             Execute first    Execute after             Execute each time
             then throttle    wait period               wait period passes
```

### Quand Utiliser le Throttling

Le throttling est particulièrement efficace lorsque vous avez besoin d'une exécution cohérente et prévisible. Cela le rend idéal pour gérer des événements ou des mises à jour fréquents où vous souhaitez un comportement fluide et contrôlé.

Cas d'utilisation courants :
- Mises à jour d'interface utilisateur nécessitant un timing cohérent (par exemple, indicateurs de progression)
- Gestionnaires d'événements de défilement ou de redimensionnement qui ne doivent pas submerger le navigateur
- Interrogation de données en temps réel où des intervalles constants sont souhaités
- Opérations gourmandes en ressources nécessitant un rythme régulier
- Mises à jour de boucle de jeu ou gestion de trames d'animation
- Suggestions de recherche en direct pendant la saisie utilisateur

### Quand Ne Pas Utiliser le Throttling

Le throttling pourrait ne pas être le meilleur choix lorsque :
- Vous voulez attendre que l'activité s'arrête (utilisez plutôt le [debouncing](../guides/debouncing))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [queueing](../guides/queueing))

> [!TIP]
> Le throttling est souvent le meilleur choix lorsque vous avez besoin d'un timing d'exécution fluide et cohérent. Il offre un modèle d'exécution plus prévisible que la limitation de débit et un retour plus immédiat que le debouncing.

## Throttling dans TanStack Pacer

TanStack Pacer fournit à la fois du throttling synchrone et asynchrone via les classes `Throttler` et `AsyncThrottler` respectivement (et leurs fonctions correspondantes `throttle` et `asyncThrottle`).

### Utilisation Basique avec `throttle`

La fonction `throttle` est le moyen le plus simple d'ajouter du throttling à n'importe quelle fonction :

```ts
import { throttle } from '@tanstack/pacer'

// Limiter les mises à jour d'interface à une fois toutes les 200ms
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

### Utilisation Avancée avec la Classe `Throttler`

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

Le throttler synchrone prend en charge les exécutions sur les fronts montants (leading) et descendants (trailing) :

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Exécuter au premier appel (par défaut)
  trailing: true,  // Exécuter après la période d'attente (par défaut)
})
```

- `leading: true` (par défaut) - Exécuter immédiatement au premier appel
- `leading: false` - Ignorer le premier appel, attendre l'exécution trailing
- `trailing: true` (par défaut) - Exécuter le dernier appel après la période d'attente
- `trailing: false` - Ignorer le dernier appel s'il est dans la période d'attente

Modèles courants :
- `{ leading: true, trailing: true }` - Par défaut, le plus réactif
- `{ leading: false, trailing: true }` - Retarder toutes les exécutions
- `{ leading: true, trailing: false }` - Ignorer les exécutions en file d'attente

### Activation/Désactivation

La classe `Throttler` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le throttler à tout moment :

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Désactivé par défaut
throttler.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options du throttler sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le throttler à la volée. Cependant, si vous utilisez la fonction `throttle` ou la classe `Throttler` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `Throttler`.

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

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction throttlée, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'interface ou effectuer des opérations de nettoyage.

#### Callbacks du Throttler Asynchrone

Le `AsyncThrottler` asynchrone prend en charge des callbacks supplémentaires pour la gestion des erreurs :

```ts
const asyncThrottler = new AsyncThrottler(async (value) => {
  await saveToAPI(value)
}, {
  wait: 200,
  onExecute: (throttler) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction asynchrone exécutée', throttler.getExecutionCount())
  },
  onError: (error) => {
    // Appelé si la fonction asynchrone génère une erreur
    console.error('Échec de la fonction asynchrone:', error)
  }
})
```

Le callback `onExecute` fonctionne de la même manière que dans le throttler synchrone, tandis que le callback `onError` vous permet de gérer les erreurs avec élégance sans interrompre la chaîne de throttling. Ces callbacks sont particulièrement utiles pour suivre les comptes d'exécution, mettre à jour l'état de l'interface, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Throttling Asynchrone

Le throttler asynchrone offre un moyen puissant de gérer les opérations asynchrones avec throttling, offrant plusieurs avantages clés par rapport à la version synchrone. Alors que le throttler synchrone est idéal pour les événements d'interface et les retours immédiats, la version asynchrone est spécialement conçue pour gérer les appels d'API, les opérations de base de données et autres tâches asynchrones.

#### Différences Clés par Rapport au Throttling Synchrone

1. **Gestion des Valeurs de Retour**
Contrairement au throttler synchrone qui renvoie void, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction throttlée. Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels d'API ou d'autres opérations asynchrones. La méthode `maybeExecute` renvoie une Promise qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le gérer de manière appropriée.

2. **Callbacks Différents**
Le `AsyncThrottler` prend en charge les callbacks suivants au lieu du seul `onExecute` dans la version synchrone :
- `onSuccess` : Appelé après chaque exécution réussie, fournissant l'instance du throttler
- `onSettled` : Appelé après chaque exécution, fournissant l'instance du throttler
- `onError` : Appelé si la fonction asynchrone génère une erreur, fournissant à la fois l'erreur et l'instance du throttler

Les throttlers asynchrones et synchrones prennent tous deux en charge le callback `onExecute` pour gérer les exécutions réussies.

3. **Exécution Séquentielle**
Étant donné que la méthode `maybeExecute` du throttler renvoie une Promise, vous pouvez choisir d'attendre chaque exécution avant de démarrer la suivante. Cela vous donne le contrôle sur l'ordre d'exécution et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement utile lorsque vous traitez des opérations qui dépendent des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur et que vous récupérez immédiatement ses données mises à jour, vous pouvez attendre l'opération de mise à jour avant de démarrer la récupération :

#### Exemple d'Utilisation Basique

Voici un exemple basique montrant comment utiliser le throttler asynchrone pour une opération de recherche :

```ts
const throttledSearch = asyncThrottle(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, throttler) => {
      console.log('Recherche réussie:', results)
    },
    onError: (error, throttler) => {
      console.error('Échec de la recherche:', error)
    }
  }
)

// Utilisation
const results = await throttledSearch('query')
```

### Adaptateurs de Framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur les fonctionnalités de base du throttling pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createThrottler`, `useThrottledCallback`, `useThrottledState` ou `useThrottledValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle total
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook de callback simple pour les cas d'utilisation basiques
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
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
    console.log('Exécutions totales:', throttler.getExecutionCount())
  }
})
```

Chaque adaptateur de framework fournit des hooks qui s'intègrent au système de gestion d'état du framework tout en conservant les fonctionnalités de base du throttling.
