---
source-updated-at: '2025-05-08T02:24:20.000Z'
translation-updated-at: '2025-05-08T05:59:20.352Z'
title: Guide sur le débouncement
id: debouncing
---
# Guide sur le Débounce (Debouncing)

Le *Rate Limiting* (Limitation de débit), le *Throttling* (Limitation de fréquence) et le *Débounce* (Antirebond) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" (avec perte) - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de *Débounce* de TanStack Pacer.

## Concept de Débounce

Le *Débounce* est une technique qui retarde l'exécution d'une fonction jusqu'à ce qu'une période d'inactivité spécifiée se soit écoulée. Contrairement à la limitation de débit qui autorise des rafales d'exécutions jusqu'à une limite, ou à la limitation de fréquence qui garantit des exécutions régulièrement espacées, le *débounce* regroupe plusieurs appels rapides de fonction en une seule exécution qui se produit uniquement après l'arrêt des appels. Cela rend le *débounce* idéal pour gérer des rafales d'événements où seul l'état final après l'arrêt de l'activité compte.

### Visualisation du Débounce

```text
Débounce (attente : 3 ticks)
Chronologie : [1 seconde par tick]
Appels :        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Exécutés :     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Exécution ici après
                                                         3 ticks sans appels

             [Rafale d'appels]     [Plus d'appels]   [Attente]      [Nouvelle rafale]
             Aucune exécution      Réinitialise le timer  [Exécution différée]  [Attente] [Exécution différée]
```

### Quand utiliser le Débounce

Le *débounce* est particulièrement efficace lorsque vous souhaitez attendre une "pause" dans l'activité avant d'agir. Cela le rend idéal pour gérer les saisies utilisateur ou d'autres événements à déclenchement rapide où seul l'état final compte.

Cas d'utilisation courants :
- Champs de recherche où vous souhaitez attendre que l'utilisateur finisse de taper
- Validation de formulaire qui ne doit pas s'exécuter à chaque frappe
- Calculs de redimensionnement de fenêtre coûteux en performance
- Sauvegarde automatique de brouillons lors de l'édition de contenu
- Appels API qui ne doivent se produire qu'après l'arrêt de l'activité utilisateur
- Tout scénario où seul la valeur finale après des changements rapides compte

### Quand ne pas utiliser le Débounce

Le *débounce* n'est peut-être pas le meilleur choix lorsque :
- Vous avez besoin d'une exécution garantie sur une période spécifique (utilisez plutôt le [*throttling*](../guides/throttling))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [mise en file d'attente](../guides/queueing))

## Débounce dans TanStack Pacer

TanStack Pacer fournit des fonctionnalités de *débounce* synchrones et asynchrones via les classes `Debouncer` et `AsyncDebouncer` respectivement (et leurs fonctions correspondantes `debounce` et `asyncDebounce`).

### Utilisation basique avec `debounce`

La fonction `debounce` est le moyen le plus simple d'ajouter un *débounce* à n'importe quelle fonction :

```ts
import { debounce } from '@tanstack/pacer'

// Appliquer un débounce à la recherche pour attendre que l'utilisateur arrête de taper
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

### Utilisation avancée avec la classe `Debouncer`

Pour plus de contrôle sur le comportement du *débounce*, vous pouvez utiliser directement la classe `Debouncer` :

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

Le *debouncer* synchrone prend en charge les exécutions sur les fronts montants (leading) et descendants (trailing) :

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
- `{ leading: true, trailing: true }` - Exécution au premier appel et après attente

### Temps d'attente maximum

Le *Debouncer* de TanStack Pacer ne propose intentionnellement PAS d'option `maxWait` comme d'autres bibliothèques de *débounce*. Si vous avez besoin d'exécutions réparties sur une période plus longue, envisagez plutôt d'utiliser la technique de [*throttling*](../guides/throttling).

### Activation/Désactivation

La classe `Debouncer` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le *debouncer* à tout moment :

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
debouncer.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options du *debouncer* sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le *debouncer* à la volée :

```ts
// Exemple React
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Activer/désactiver selon la longueur de l'entrée SI vous utilisez un adaptateur de framework prenant en charge les options réactives
)
```

Cependant, si vous utilisez la fonction `debounce` ou la classe `Debouncer` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `Debouncer`.

```ts
// Exemple Solid
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Activer/désactiver selon la longueur de l'entrée
})
```

### Options de callback

Les *debouncers* synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie du *débounce* :

#### Callbacks du Debouncer synchrone

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

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction avec *débounce*, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'interface ou effectuer des opérations de nettoyage.

#### Callbacks du Debouncer asynchrone

Le `AsyncDebouncer` asynchrone a un ensemble de callbacks différent par rapport à la version synchrone.

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction asynchrone exécutée', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // Appelé après chaque tentative d'exécution
    console.log('Fonction asynchrone terminée', debouncer.getSettledCount())
  },
  onError: (error) => {
    // Appelé si la fonction asynchrone génère une erreur
    console.error('Échec de la fonction asynchrone:', error)
  }
})
```

Le callback `onSuccess` est appelé après chaque exécution réussie de la fonction avec *débounce*, tandis que le callback `onError` est appelé si la fonction asynchrone génère une erreur. Le callback `onSettled` est appelé après chaque tentative d'exécution, qu'elle réussisse ou échoue. Ces callbacks sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'interface, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Débounce asynchrone

Le *debouncer* asynchrone offre un moyen puissant de gérer des opérations asynchrones avec *débounce*, présentant plusieurs avantages clés par rapport à la version synchrone. Alors que le *debouncer* synchrone est idéal pour les événements d'interface et les retours immédiats, la version asynchrone est spécialement conçue pour gérer les appels API, les opérations de base de données et d'autres tâches asynchrones.

#### Différences clés avec le Débounce synchrone

1. **Gestion des valeurs de retour**
Contrairement au *debouncer* synchrone qui renvoie void, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction avec *débounce*. Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels API ou d'autres opérations asynchrones. La méthode `maybeExecute` renvoie une Promise qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le traiter de manière appropriée.

2. **Callbacks différents**
L'`AsyncDebouncer` prend en charge les callbacks suivants au lieu du seul `onExecute` dans la version synchrone :
- `onSuccess` : Appelé après chaque exécution réussie, fournissant l'instance du *debouncer*
- `onSettled` : Appelé après chaque exécution, fournissant l'instance du *debouncer*
- `onError` : Appelé si la fonction asynchrone génère une erreur, fournissant à la fois l'erreur et l'instance du *debouncer*

3. **Exécution séquentielle**
Puisque la méthode `maybeExecute` du *debouncer* renvoie une Promise, vous pouvez choisir d'attendre chaque exécution avant de démarrer la suivante. Cela vous donne le contrôle sur l'ordre d'exécution et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement utile lorsque vous traitez des opérations qui dépendent des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur puis récupérez immédiatement ses données mises à jour, vous pouvez attendre l'opération de mise à jour avant de démarrer la récupération :

#### Exemple d'utilisation basique

Voici un exemple basique montrant comment utiliser le *debouncer* asynchrone pour une opération de recherche :

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('Recherche réussie:', results)
    },
    onError: (error, debouncer) => {
      console.error('Échec de la recherche:', error)
    }
  }
)

// Utilisation
const results = await debouncedSearch('query')
```

### Adaptateurs pour frameworks

Chaque adaptateur de framework fournit des hooks qui s'appuient sur les fonctionnalités de base du *débounce* pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` ou `useDebouncedValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useDebouncer, useDebouncedCallback, useDebouncedValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle complet
const debouncer = useDebouncer(
  (value: string) => saveToDatabase(value),
  { wait: 500 }
)

// Hook de callback simple pour des cas d'utilisation basiques
const handleSearch = useDebouncedCallback(
  (query: string) => fetchSearchResults(query),
  { wait: 500 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState('')
const [debouncedValue] = useDebouncedValue(
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
