---
source-updated-at: '2025-04-24T12:27:47.000Z'
translation-updated-at: '2025-05-02T04:33:54.548Z'
title: Guide sur le debouncing
id: debouncing
---
# Guide sur le Débouncement (Debouncing)

Le *Rate Limiting* (Limitation de débit), le *Throttling* (Lissage) et le *Débouncement* (Debouncing) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" (avec perte) - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour construire des applications performantes et fiables. Ce guide couvrira les concepts de Débouncement de TanStack Pacer.

## Concept de Débouncement

Le débouncement est une technique qui retarde l'exécution d'une fonction jusqu'à ce qu'une période d'inactivité spécifiée se soit écoulée. Contrairement à la limitation de débit qui autorise des rafales d'exécutions jusqu'à une limite, ou au lissage qui garantit des exécutions espacées régulièrement, le débouncement regroupe plusieurs appels de fonction rapides en une seule exécution qui ne se produit qu'après l'arrêt des appels. Cela rend le débouncement idéal pour gérer des rafales d'événements où seul l'état final après l'arrêt de l'activité compte.

### Visualisation du Débouncement

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Exécution ici après
                                                         3 ticks sans appels

             [Rafale d'appels]     [Plus d'appels]   [Attente]      [Nouvelle rafale]
             Aucune exécution      Réinitialise le timer  [Exécution différée]  [Attente] [Exécution différée]
```

### Quand Utiliser le Débouncement

Le débouncement est particulièrement efficace lorsque vous souhaitez attendre une "pause" dans l'activité avant d'agir. Cela le rend idéal pour gérer les saisies utilisateur ou d'autres événements déclenchés rapidement où seul l'état final compte.

Cas d'utilisation courants :
- Champs de recherche où vous voulez attendre que l'utilisateur finisse de taper
- Validation de formulaire qui ne devrait pas s'exécuter à chaque frappe
- Calculs de redimensionnement de fenêtre coûteux
- Sauvegarde automatique de brouillons pendant l'édition de contenu
- Appels API qui ne devraient se produire qu'après l'arrêt de l'activité utilisateur
- Tout scénario où seul le résultat final après des changements rapides compte

### Quand Ne Pas Utiliser le Débouncement

Le débouncement pourrait ne pas être le meilleur choix lorsque :
- Vous avez besoin d'une exécution garantie sur une période spécifique (utilisez plutôt le [lissage](../guides/throttling))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [mise en file d'attente](../guides/queueing))

## Débouncement dans TanStack Pacer

TanStack Pacer fournit du débouncement synchrone et asynchrone via les classes `Debouncer` et `AsyncDebouncer` respectivement (et leurs fonctions correspondantes `debounce` et `asyncDebounce`).

### Utilisation Basique avec `debounce`

La fonction `debounce` est le moyen le plus simple d'ajouter du débouncement à n'importe quelle fonction :

```ts
import { debounce } from '@tanstack/pacer'

// Débouncement de la saisie de recherche pour attendre que l'utilisateur arrête de taper
const debouncedSearch = debounce(
  (searchTerm: string) => performSearch(searchTerm),
  {
    wait: 500, // Attendre 500ms après la dernière frappe
  }
)

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value)
})
```

### Utilisation Avancée avec la Classe `Debouncer`

Pour plus de contrôle sur le comportement de débouncement, vous pouvez utiliser directement la classe `Debouncer` :

```ts
import { Debouncer } from '@tanstack/pacer'

const searchDebouncer = new Debouncer(
  (searchTerm: string) => performSearch(searchTerm),
  { wait: 500 }
)

// Obtenir des informations sur l'état actuel
console.log(searchDebouncer.getExecutionCount()) // Nombre d'exécutions réussies
console.log(searchDebouncer.getIsPending()) // Si un appel est en attente

// Mettre à jour les options dynamiquement
searchDebouncer.setOptions({ wait: 1000 }) // Augmenter le temps d'attente

// Annuler l'exécution en attente
searchDebouncer.cancel()
```

### Exécutions Leading et Trailing

Le débounceur synchrone prend en charge les exécutions sur les fronts montants (leading) et descendants (trailing) :

```ts
const debouncedFn = debounce(fn, {
  wait: 500,
  leading: true,   // Exécuter au premier appel
  trailing: true,  // Exécuter après la période d'attente
})
```

- `leading: true` - La fonction s'exécute immédiatement au premier appel
- `leading: false` (par défaut) - Le premier appel démarre le timer d'attente
- `trailing: true` (par défaut) - La fonction s'exécute après la période d'attente
- `trailing: false` - Aucune exécution après la période d'attente

Modèles courants :
- `{ leading: false, trailing: true }` - Par défaut, exécution après attente
- `{ leading: true, trailing: false }` - Exécution immédiate, ignore les appels suivants
- `{ leading: true, trailing: true }` - Exécution à la fois au premier appel et après attente

### Temps d'Attente Maximum

Le Débounceur TanStack Pacer ne possède PAS d'option `maxWait` comme d'autres bibliothèques de débouncement. Si vous avez besoin de laisser les exécutions s'étaler sur une période plus longue, envisagez plutôt d'utiliser la technique de [lissage](../guides/throttling).

### Activation/Désactivation

La classe `Debouncer` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le débounceur à tout moment :

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
debouncer.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options du débounceur sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le débounceur à la volée :

```ts
// Exemple React
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Activer/désactiver selon la longueur de la saisie SI vous utilisez un adaptateur de framework qui prend en charge les options réactives
)
```

Cependant, si vous utilisez la fonction `debounce` ou la classe `Debouncer` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait passées au constructeur de la classe `Debouncer`.

```ts
// Exemple Solid
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Activer/désactiver selon la longueur de la saisie
})
```

### Options de Callback

Les débounceurs synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie du débouncement :

#### Callbacks du Débounceur Synchrone

Le `Debouncer` synchrone prend en charge le callback suivant :

```ts
const debouncer = new Debouncer(fn, {
  wait: 500,
  onExecute: (debouncer) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction exécutée', debouncer.getExecutionCount())
  }
})
```

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction débouncée, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'interface utilisateur ou effectuer des opérations de nettoyage.

#### Callbacks du Débounceur Asynchrone

Le `AsyncDebouncer` asynchrone prend en charge des callbacks supplémentaires pour la gestion des erreurs :

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onExecute: (debouncer) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction asynchrone exécutée', debouncer.getExecutionCount())
  },
  onError: (error) => {
    // Appelé si la fonction asynchrone lance une erreur
    console.error('Échec de la fonction asynchrone:', error)
  }
})
```

Le callback `onExecute` fonctionne de la même manière que dans le débounceur synchrone, tandis que le callback `onError` vous permet de gérer les erreurs sans interrompre la chaîne de débouncement. Ces callbacks sont particulièrement utiles pour suivre le nombre d'exécutions, mettre à jour l'état de l'interface utilisateur, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Débouncement Asynchrone

Pour les fonctions asynchrones ou lorsque vous avez besoin de gestion d'erreurs, utilisez `AsyncDebouncer` ou `asyncDebounce` :

```ts
import { asyncDebounce } from '@tanstack/pacer'

const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    updateUI(results)
  },
  {
    wait: 500,
    onError: (error) => {
      console.error('Échec de la recherche:', error)
    }
  }
)

// Ne fera qu'un seul appel API après l'arrêt de la saisie
searchInput.addEventListener('input', async (e) => {
  await debouncedSearch(e.target.value)
})
```

La version asynchrone fournit un suivi d'exécution basé sur les Promises, une gestion des erreurs via le callback `onError`, un nettoyage approprié des opérations asynchrones en attente et une méthode `maybeExecute` awaitable.

### Adaptateurs de Framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur les fonctionnalités de base du débouncement pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` ou `useDebouncedValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle complet
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// Hook de callback simple pour les cas d'utilisation basiques
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState('')
const [debouncedState, setDebouncedState] = useDebouncedValue(
  instantState, // Valeur à débouncer
  { wait: 500 }
)
```

#### Solid

```tsx
import { createDebouncer, createDebouncedSignal } from '@tanstack/solid-pacer'

// Hook bas niveau pour un contrôle complet
const debouncer = createDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// Hook basé sur les signaux pour la gestion d'état
const [searchTerm, setSearchTerm, debouncer] = createDebouncedSignal('', {
  wait: 500,
  onExecute: (debouncer) => {
    console.log('Exécutions totales:', debouncer.getExecutionCount())
  }
})
```
