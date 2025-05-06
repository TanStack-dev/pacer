---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:17:20.628Z'
title: Guide sur le debouncing
id: debouncing
---
# Guide sur le Débounçage (Debouncing)

La limitation de débit (Rate Limiting), le throttling et le débounçage (Debouncing) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "lossy" (avec perte) - ce qui signifie que certains appels de fonction ne s'exécuteront pas lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de débounçage de TanStack Pacer.

## Concept de Débounçage (Debouncing)

Le débounçage est une technique qui retarde l'exécution d'une fonction jusqu'à ce qu'une période d'inactivité spécifiée se soit écoulée. Contrairement à la limitation de débit qui autorise des rafales d'exécutions jusqu'à une limite, ou au throttling qui garantit des exécutions régulièrement espacées, le débounçage regroupe plusieurs appels rapides de fonction en une seule exécution qui ne se produit qu'après l'arrêt des appels. Cela rend le débounçage idéal pour gérer des rafales d'événements où seul l'état final après l'arrêt de l'activité vous intéresse.

### Visualisation du Débounçage

```text
Debouncing (wait: 3 ticks)
Timeline: [1 second per tick]
Calls:        ⬇️  ⬇️  ⬇️  ⬇️  ⬇️     ⬇️  ⬇️  ⬇️  ⬇️               ⬇️  ⬇️
Executed:     ❌  ❌  ❌  ❌  ❌     ❌  ❌  ❌  ⏳   ->   ✅     ❌  ⏳   ->    ✅
             [=================================================================]
                                                        ^ Exécution ici après
                                                         3 ticks sans appel

             [Rafale d'appels]     [Plus d'appels]   [Attente]      [Nouvelle rafale]
             Aucune exécution      Réinitialise le timer  [Exécution différée]  [Attente] [Exécution différée]
```

### Quand Utiliser le Débounçage

Le débounçage est particulièrement efficace lorsque vous souhaitez attendre une "pause" dans l'activité avant d'agir. Cela le rend idéal pour gérer les saisies utilisateur ou d'autres événements se déclenchant rapidement où seul l'état final vous intéresse.

Cas d'utilisation courants :
- Champs de recherche où vous voulez attendre que l'utilisateur finisse de taper
- Validation de formulaire qui ne devrait pas s'exécuter à chaque frappe
- Calculs de redimensionnement de fenêtre coûteux en performance
- Sauvegarde automatique de brouillons pendant l'édition de contenu
- Appels API qui ne devraient se produire qu'après l'arrêt de l'activité utilisateur
- Tout scénario où seul la valeur finale après des changements rapides vous intéresse

### Quand Ne Pas Utiliser le Débounçage

Le débounçage pourrait ne pas être le meilleur choix lorsque :
- Vous avez besoin d'une exécution garantie sur une période spécifique (utilisez plutôt le [throttling](../guides/throttling))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [mise en file d'attente (queueing)](../guides/queueing))

## Débounçage dans TanStack Pacer

TanStack Pacer fournit à la fois du débounçage synchrone et asynchrone via les classes `Debouncer` et `AsyncDebouncer` respectivement (et leurs fonctions correspondantes `debounce` et `asyncDebounce`).

### Utilisation Basique avec `debounce`

La fonction `debounce` est la manière la plus simple d'ajouter du débounçage à n'importe quelle fonction :

```ts
import { debounce } from '@tanstack/pacer'

// Débounçage d'une saisie de recherche pour attendre que l'utilisateur arrête de taper
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

Pour plus de contrôle sur le comportement de débounçage, vous pouvez utiliser directement la classe `Debouncer` :

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

Le débounçage synchrone prend en charge les exécutions sur les fronts montants (leading) et descendants (trailing) :

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
- `{ leading: false, trailing: true }` - Par défaut, exécute après l'attente
- `{ leading: true, trailing: false }` - Exécute immédiatement, ignore les appels suivants
- `{ leading: true, trailing: true }` - Exécute à la fois au premier appel et après l'attente

### Temps d'Attente Maximum (Max Wait Time)

Le Debouncer de TanStack Pacer n'a intentionnellement PAS d'option `maxWait` comme d'autres bibliothèques de débounçage. Si vous avez besoin de laisser les exécutions s'étaler sur une période plus longue, envisagez plutôt d'utiliser la technique de [throttling](../guides/throttling).

### Activation/Désactivation

La classe `Debouncer` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le débounçage à tout moment :

```ts
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
debouncer.setOptions({ enabled: true }) // Activer à tout moment
```

Si vous utilisez un adaptateur de framework où les options de débounçage sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le débounçage à la volée :

```ts
// Exemple React
const debouncer = useDebouncer(
  setSearch, 
  { wait: 500, enabled: searchInput.value.length > 3 } // Activer/désactiver selon la longueur de la saisie SI vous utilisez un adaptateur de framework prenant en charge les options réactives
)
```

Cependant, si vous utilisez la fonction `debounce` ou la classe `Debouncer` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `Debouncer`.

```ts
// Exemple Solid
const debouncer = new Debouncer(fn, { wait: 500, enabled: false }) // Désactivé par défaut
createEffect(() => {
  debouncer.setOptions({ enabled: search().length > 3 }) // Activer/désactiver selon la longueur de la saisie
})
```

### Options de Callback

Les débounçeurs synchrones et asynchrones prennent en charge des options de callback pour gérer différents aspects du cycle de vie du débounçage :

#### Callbacks du Débounçage Synchrone

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

Le callback `onExecute` est appelé après chaque exécution réussie de la fonction débouncée, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'UI ou effectuer des opérations de nettoyage.

#### Callbacks du Débounçage Asynchrone

Le `AsyncDebouncer` asynchrone a un ensemble de callbacks différent par rapport à la version synchrone.

```ts
const asyncDebouncer = new AsyncDebouncer(async (value) => {
  await saveToAPI(value)
}, {
  wait: 500,
  onSuccess: (result, debouncer) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction async exécutée', debouncer.getSuccessCount())
  },
  onSettled: (debouncer) => {
    // Appelé après chaque tentative d'exécution
    console.log('Fonction async terminée', debouncer.getSettledCount())
  },
  onError: (error) => {
    // Appelé si la fonction async génère une erreur
    console.error('Échec de la fonction async :', error)
  }
})
```

Le callback `onSuccess` est appelé après chaque exécution réussie de la fonction débouncée, tandis que le callback `onError` est appelé si la fonction asynchrone génère une erreur. Le callback `onSettled` est appelé après chaque tentative d'exécution, qu'elle réussisse ou échoue. Ces callbacks sont particulièrement utiles pour suivre les compteurs d'exécution, mettre à jour l'état de l'UI, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Débounçage Asynchrone

Le débounçage asynchrone offre un moyen puissant de gérer des opérations asynchrones avec débounçage, présentant plusieurs avantages clés par rapport à la version synchrone. Alors que le débounçage synchrone est idéal pour les événements UI et le retour immédiat, la version asynchrone est spécialement conçue pour gérer les appels API, les opérations de base de données et autres tâches asynchrones.

#### Différences Clés par Rapport au Débounçage Synchrone

1. **Gestion des Valeurs de Retour**
Contrairement au débounçage synchrone qui retourne void, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction débouncée. Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels API ou d'autres opérations asynchrones. La méthode `maybeExecute` retourne une Promise qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le gérer de manière appropriée.

2. **Système de Callback Amélioré**
Le débounçage asynchrone fournit un système de callback plus sophistiqué que le simple callback `onExecute` de la version synchrone. Ce système inclut :
- `onSuccess` : Appelé lorsque la fonction async se termine avec succès, fournissant à la fois le résultat et l'instance du débounçage
- `onError` : Appelé lorsque la fonction async génère une erreur, fournissant à la fois l'erreur et l'instance du débounçage
- `onSettled` : Appelé après chaque tentative d'exécution, qu'elle réussisse ou échoue

3. **Suivi des Exécutions**
Le débounçage asynchrone fournit un suivi complet des exécutions via plusieurs méthodes :
- `getSuccessCount()` : Nombre d'exécutions réussies
- `getErrorCount()` : Nombre d'exécutions échouées
- `getSettledCount()` : Nombre total d'exécutions terminées (succès + échec)

4. **Exécution Séquentielle**
Le débounçage asynchrone garantit que les exécutions suivantes attendent la fin de l'appel précédent avant de démarrer. Cela empêche les exécutions désordonnées et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement important lors de la gestion d'opérations dépendant des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur puis récupérez immédiatement ses données mises à jour, le débounçage asynchrone garantira que l'opération de récupération attend la fin de la mise à jour, évitant les conditions de course où vous pourriez obtenir des données obsolètes.

#### Exemple d'Utilisation Basique

Voici un exemple basique montrant comment utiliser le débounçage asynchrone pour une opération de recherche :

```ts
const debouncedSearch = asyncDebounce(
  async (searchTerm: string) => {
    const results = await fetchSearchResults(searchTerm)
    return results
  },
  {
    wait: 500,
    onSuccess: (results, debouncer) => {
      console.log('Recherche réussie :', results)
    },
    onError: (error, debouncer) => {
      console.error('Recherche échouée :', error)
    }
  }
)

// Utilisation
const results = await debouncedSearch('query')
```

#### Modèles Avancés

Le débounçage asynchrone peut être combiné avec divers modèles pour résoudre des problèmes complexes :

1. **Intégration avec la Gestion d'État**
Lors de l'utilisation du débounçage asynchrone avec des systèmes de gestion d'état (comme useState de React ou createSignal de Solid), vous pouvez créer des modèles puissants pour gérer les états de chargement, les états d'erreur et les mises à jour de données. Les callbacks du débounçage fournissent des hooks parfaits pour mettre à jour l'état de l'UI en fonction du succès ou de l'échec des opérations.

2. **Prévention des Conditions de Course**
Le modèle de mutation à vol unique (single-flight mutation) prévient naturellement les conditions de course dans de nombreux scénarios. Lorsque plusieurs parties de votre application tentent de mettre à jour la même ressource simultanément, le débounçage garantit que seule la mise à jour la plus récente se produit réellement, tout en fournissant des résultats à tous les appelants.

3. **Récupération après Erreur**
Les capacités de gestion d'erreur du débounçage asynchrone le rendent idéal pour implémenter des logiques de réessai et des modèles de récupération après erreur. Vous pouvez utiliser le callback `onError` pour implémenter des stratégies personnalisées de gestion d'erreur, comme un backoff exponentiel ou des mécanismes de repli.

### Adaptateurs pour Frameworks

Chaque adaptateur de framework fournit des hooks qui s'appuient sur la fonctionnalité de base du débounçage pour s'intégrer avec le système de gestion d'état du framework. Des hooks comme `createDebouncer`, `useDebouncedCallback`, `useDebouncedState` ou `useDebouncedValue` sont disponibles pour chaque framework.

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

// Hook basé sur l'état pour une gestion d'état réactive
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
  onExecute: (deb
