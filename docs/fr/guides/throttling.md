---
source-updated-at: '2025-05-05T07:34:55.000Z'
translation-updated-at: '2025-05-06T23:17:09.779Z'
title: Guide sur le throttling
id: throttling
---
# Guide sur la limitation de débit (Throttling)

La limitation de débit (Rate Limiting), la régulation (Throttling) et l'anti-rebond (Debouncing) sont trois approches distinctes pour contrôler la fréquence d'exécution des fonctions. Chaque technique bloque les exécutions différemment, les rendant "avec perte" - ce qui signifie que certains appels de fonction ne seront pas exécutés lorsqu'ils sont demandés trop fréquemment. Comprendre quand utiliser chaque approche est crucial pour créer des applications performantes et fiables. Ce guide couvrira les concepts de régulation (Throttling) de TanStack Pacer.

## Concept de régulation (Throttling)

La régulation (Throttling) garantit que les exécutions de fonctions sont uniformément espacées dans le temps. Contrairement à la limitation de débit (Rate Limiting) qui permet des rafales d'exécutions jusqu'à une limite, ou à l'anti-rebond (Debouncing) qui attend que l'activité s'arrête, la régulation crée un modèle d'exécution plus fluide en imposant des délais constants entre les appels. Si vous définissez une régulation d'une exécution par seconde, les appels seront espacés uniformément quelle que soit la rapidité avec laquelle ils sont demandés.

### Visualisation de la régulation (Throttling)

```text
Régulation (une exécution toutes les 3 unités de temps)
Chronologie: [1 seconde par unité]
Appels:        ⬇️  ⬇️  ⬇️           ⬇️  ⬇️  ⬇️  ⬇️             ⬇️
Exécutés:     ✅  ❌  ⏳  ->   ✅  ❌  ❌  ❌  ✅             ✅ 
             [=================================================================]
             ^ Seule une exécution autorisée toutes les 3 unités,
               quel que soit le nombre d'appels effectués

             [Première rafale]    [Plus d'appels]              [Appels espacés]
             Exécute le premier   Exécute après                Exécute chaque fois
             puis régule          la période d'attente         que la période d'attente passe
```

### Quand utiliser la régulation (Throttling)

La régulation est particulièrement efficace lorsque vous avez besoin d'un timing d'exécution cohérent et prévisible. Cela la rend idéale pour gérer des événements ou des mises à jour fréquents où vous souhaitez un comportement fluide et contrôlé.

Cas d'utilisation courants :
- Mises à jour d'interface utilisateur nécessitant un timing cohérent (ex. : indicateurs de progression)
- Gestionnaires d'événements de défilement ou de redimensionnement qui ne doivent pas submerger le navigateur
- Polling de données en temps réel où des intervalles constants sont souhaités
- Opérations gourmandes en ressources nécessitant un rythme régulier
- Mises à jour de boucle de jeu ou gestion d'images d'animation
- Suggestions de recherche en direct pendant la saisie utilisateur

### Quand ne pas utiliser la régulation (Throttling)

La régulation pourrait ne pas être le meilleur choix lorsque :
- Vous voulez attendre que l'activité s'arrête (utilisez plutôt l'[anti-rebond](../guides/debouncing))
- Vous ne pouvez pas vous permettre de manquer des exécutions (utilisez plutôt la [mise en file d'attente](../guides/queueing))

> [!TIP]
> La régulation est souvent le meilleur choix lorsque vous avez besoin d'un timing d'exécution fluide et cohérent. Elle fournit un modèle d'exécution plus prévisible que la limitation de débit et un retour plus immédiat que l'anti-rebond.

## Régulation dans TanStack Pacer

TanStack Pacer fournit à la fois une régulation synchrone et asynchrone via les classes `Throttler` et `AsyncThrottler` respectivement (et leurs fonctions correspondantes `throttle` et `asyncThrottle`).

### Utilisation basique avec `throttle`

La fonction `throttle` est le moyen le plus simple d'ajouter une régulation à n'importe quelle fonction :

```ts
import { throttle } from '@tanstack/pacer'

// Régule les mises à jour d'interface à une fois toutes les 200ms
const throttledUpdate = throttle(
  (value: number) => updateProgressBar(value),
  {
    wait: 200,
  }
)

// Dans une boucle rapide, n'exécute que toutes les 200ms
for (let i = 0; i < 100; i++) {
  throttledUpdate(i) // De nombreux appels sont régulés
}
```

### Utilisation avancée avec la classe `Throttler`

Pour plus de contrôle sur le comportement de régulation, vous pouvez utiliser directement la classe `Throttler` :

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

### Exécutions en début et fin

Le régulateur synchrone prend en charge les exécutions sur les fronts montant et descendant :

```ts
const throttledFn = throttle(fn, {
  wait: 200,
  leading: true,   // Exécute au premier appel (par défaut)
  trailing: true,  // Exécute après la période d'attente (par défaut)
})
```

- `leading: true` (par défaut) - Exécute immédiatement au premier appel
- `leading: false` - Ignore le premier appel, attend l'exécution en fin
- `trailing: true` (par défaut) - Exécute le dernier appel après la période d'attente
- `trailing: false` - Ignore le dernier appel s'il est dans la période d'attente

Modèles courants :
- `{ leading: true, trailing: true }` - Par défaut, le plus réactif
- `{ leading: false, trailing: true }` - Retarde toutes les exécutions
- `{ leading: true, trailing: false }` - Ignore les exécutions en file d'attente

### Activation/Désactivation

La classe `Throttler` prend en charge l'activation/désactivation via l'option `enabled`. En utilisant la méthode `setOptions`, vous pouvez activer/désactiver le régulateur à tout moment :

```ts
const throttler = new Throttler(fn, { wait: 200, enabled: false }) // Désactivé par défaut
throttler.setOptions({ enabled: true }) // Active à tout moment
```

Si vous utilisez un adaptateur de framework où les options du régulateur sont réactives, vous pouvez définir l'option `enabled` sur une valeur conditionnelle pour activer/désactiver le régulateur à la volée. Cependant, si vous utilisez la fonction `throttle` ou la classe `Throttler` directement, vous devez utiliser la méthode `setOptions` pour modifier l'option `enabled`, car les options passées sont en fait transmises au constructeur de la classe `Throttler`.

### Options de rappel

Les régulateurs synchrones et asynchrones prennent en charge des options de rappel pour gérer différents aspects du cycle de vie de la régulation :

#### Rappels du régulateur synchrone

Le `Throttler` synchrone prend en charge le rappel suivant :

```ts
const throttler = new Throttler(fn, {
  wait: 200,
  onExecute: (throttler) => {
    // Appelé après chaque exécution réussie
    console.log('Fonction exécutée', throttler.getExecutionCount())
  }
})
```

Le rappel `onExecute` est appelé après chaque exécution réussie de la fonction régulée, ce qui le rend utile pour suivre les exécutions, mettre à jour l'état de l'interface ou effectuer des opérations de nettoyage.

#### Rappels du régulateur asynchrone

Le `AsyncThrottler` asynchrone prend en charge des rappels supplémentaires pour la gestion des erreurs :

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

Le rappel `onExecute` fonctionne de la même manière que dans le régulateur synchrone, tandis que le rappel `onError` vous permet de gérer les erreurs gracieusement sans interrompre la chaîne de régulation. Ces rappels sont particulièrement utiles pour suivre le nombre d'exécutions, mettre à jour l'état de l'interface, gérer les erreurs, effectuer des opérations de nettoyage et enregistrer des métriques d'exécution.

### Régulation asynchrone

Le régulateur asynchrone offre un moyen puissant de gérer les opérations asynchrones avec régulation, offrant plusieurs avantages clés par rapport à la version synchrone. Alors que le régulateur synchrone est idéal pour les événements d'interface et les retours immédiats, la version asynchrone est spécialement conçue pour gérer les appels d'API, les opérations de base de données et autres tâches asynchrones.

#### Différences clés avec la régulation synchrone

1. **Gestion des valeurs de retour**
Contrairement au régulateur synchrone qui renvoie void, la version asynchrone vous permet de capturer et d'utiliser la valeur de retour de votre fonction régulée. Ceci est particulièrement utile lorsque vous devez travailler avec les résultats d'appels d'API ou d'autres opérations asynchrones. La méthode `maybeExecute` renvoie une Promise qui se résout avec la valeur de retour de la fonction, vous permettant d'attendre le résultat et de le gérer de manière appropriée.

2. **Système de rappel amélioré**
Le régulateur asynchrone fournit un système de rappel plus sophistiqué que le simple rappel `onExecute` de la version synchrone. Ce système comprend :
- `onSuccess` : Appelé lorsque la fonction asynchrone se termine avec succès, fournissant à la fois le résultat et l'instance du régulateur
- `onError` : Appelé lorsque la fonction asynchrone génère une erreur, fournissant à la fois l'erreur et l'instance du régulateur
- `onSettled` : Appelé après chaque tentative d'exécution, qu'elle réussisse ou échoue

3. **Suivi des exécutions**
Le régulateur asynchrone fournit un suivi complet des exécutions via plusieurs méthodes :
- `getSuccessCount()` : Nombre d'exécutions réussies
- `getErrorCount()` : Nombre d'exécutions ayant échoué
- `getSettledCount()` : Nombre total d'exécutions terminées (succès + échec)

4. **Exécution séquentielle**
Le régulateur asynchrone garantit que les exécutions suivantes attendent la fin de l'appel précédent avant de commencer. Cela empêche les exécutions désordonnées et garantit que chaque appel traite les données les plus à jour. Ceci est particulièrement important lors de la gestion d'opérations qui dépendent des résultats d'appels précédents ou lorsque la cohérence des données est critique.

Par exemple, si vous mettez à jour le profil d'un utilisateur puis récupérez immédiatement ses données mises à jour, le régulateur asynchrone garantira que l'opération de récupération attend la fin de la mise à jour, évitant ainsi les conditions de course où vous pourriez obtenir des données obsolètes.

#### Exemple d'utilisation basique

Voici un exemple basique montrant comment utiliser le régulateur asynchrone pour une opération de recherche :

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
      console.error('Recherche échouée:', error)
    }
  }
)

// Utilisation
const results = await throttledSearch('query')
```

#### Modèles avancés

Le régulateur asynchrone peut être combiné avec divers modèles pour résoudre des problèmes complexes :

1. **Intégration avec la gestion d'état**
Lors de l'utilisation du régulateur asynchrone avec des systèmes de gestion d'état (comme useState de React ou createSignal de Solid), vous pouvez créer des modèles puissants pour gérer les états de chargement, les états d'erreur et les mises à jour de données. Les rappels du régulateur fournissent des crochets parfaits pour mettre à jour l'état de l'interface en fonction du succès ou de l'échec des opérations.

2. **Prévention des conditions de course**
Le modèle de régulation empêche naturellement les conditions de course dans de nombreux scénarios. Lorsque plusieurs parties de votre application tentent de mettre à jour la même ressource simultanément, le régulateur garantit que les mises à jour se produisent à un rythme contrôlé, tout en fournissant des résultats à tous les appelants.

3. **Récupération après erreur**
Les capacités de gestion des erreurs du régulateur asynchrone le rendent idéal pour implémenter une logique de réessai et des modèles de récupération après erreur. Vous pouvez utiliser le rappel `onError` pour implémenter des stratégies de gestion d'erreur personnalisées, comme un backoff exponentiel ou des mécanismes de repli.

### Adaptateurs de framework

Chaque adaptateur de framework fournit des hooks qui s'appuient sur la fonctionnalité de régulation de base pour s'intégrer au système de gestion d'état du framework. Des hooks comme `createThrottler`, `useThrottledCallback`, `useThrottledState` ou `useThrottledValue` sont disponibles pour chaque framework.

Voici quelques exemples :

#### React

```tsx
import { useThrottler, useThrottledCallback, useThrottledValue } from '@tanstack/react-pacer'

// Hook bas niveau pour un contrôle complet
const throttler = useThrottler(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook de rappel simple pour les cas d'utilisation basiques
const handleUpdate = useThrottledCallback(
  (value: number) => updateProgressBar(value),
  { wait: 200 }
)

// Hook basé sur l'état pour la gestion d'état réactive
const [instantState, setInstantState] = useState(0)
const [throttledValue] = useThrottledValue(
  instantState, // Valeur à réguler
  { wait: 200 }
)
```

#### Solid

```tsx
import { createThrottler, createThrottledSignal } from '@tanstack/solid-pacer'

// Hook bas niveau pour un contrôle complet
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

Chaque adaptateur de framework fournit des hooks qui s'intègrent au système de gestion d'état du framework tout en conservant la fonctionnalité de régulation de base.
