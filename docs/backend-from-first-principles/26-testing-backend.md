# 26. Testing pour les Ingénieurs Backend (Mocks, TDD & Couverture)

Écrire un mauvais test ou un test bâclé est plus dangereux que ne pas avoir de test du tout, car il procure un **faux sentiment de sécurité**. Dans cette fiche, on décortique l'ensemble du sujet : de l'anatomie sous le capot d'un Test Runner jusqu'au Mutation Testing, en passant par le TDD et la gestion des Mocks.

---

## 🎯 1. Pourquoi Tester & Qu'est-ce qu'un Test ?

Un **test** est un *second programme* qui exécute notre application backend, lui fournit des entrées prédéfinies et vérifie qu'elle produit exactement les sorties et comportements attendus.

```
┌─────────────────────────────────────────────────────────────┐
│                    TEST RUNNER / SUITE                      │
│                                                             │
│   Inputs (Données) ──────►  [ NOTRE CODE BACKEND ]          │
│                                    │                        │
│                                    ▼ Output                 │
│   Expected Value  ──────►  [ ASSERTION (if !=) ]            │
│                                    │                        │
│                                    ▼                        │
│                           ✅ PASS  /  ❌ FAIL               │
└─────────────────────────────────────────────────────────────┘
```

### La Raison d'Être Ultime : Prévenir les Régressions
* **Régression** : Un bug réintroduit dans une fonctionnalité qui marchait auparavant suite à l'ajout de nouveau code ou à une refactorisation.
* **Le bénéfice du test** : Permettre à l'équipe de livrer du code à toute vitesse (*Rapid Shipping*) sans craindre de casser les fonctionnalités existantes.
* **Périmètre Dev Backend** : Ces notes se concentrent sur les **80% de tests écrits et exécutés directement par les développeurs backend** (hors QA manuelle entreprise, tests mobiles ou automation navigateur).

---

## ⚙️ 2. Sous le Capot : Comment Fonctionne un Test Runner ?

Au niveau le plus élémentaire, un test est simplement une fonction avec une condition `if (output != expected) return error`. Cependant, faire cela à la main n'est pas scalable. C'est le rôle des **Test Runners** (ex: `go test`, Jest, PyTest).

```
 1. Découverte   ──► Trouve tous les fichiers *_test.go / *.test.js
 2. Génération   ──► Crée automatiquement un fichier `main` éphémère (_testmain.go)
 3. Compilation  ──► Compile le binaire de test (ex: board.test)
 4. Exécution    ──► Lance le binaire et capture les sorties / logs
 5. Reporting    ──► Génère le rapport final (Nombre de tests OK / Échoués)
```

> 💡 **Assertions & Messages d'Erreur** : Dans la plupart des langages (`assert.equal`), veillez à écrire des messages d'erreur explicites (*"Wanted X, got Y"*). Un bon message d'erreur doit permettre de comprendre immédiatement ce qui a échoué sans avoir à dérouler tout le debugger.

---

## 📐 3. Pyramide des Tests vs Le Modèle Mental par Portée (Scope)

### La Pyramide Traditionnelle
Classiquement, les tests sont découpés en trois catégories :
1. **Unit Tests (Tests Unitaires)** : Testent une unité de code (fonction, struct, module) en isolement total.
2. **Integration Tests (Tests d'Intégration)** : Vérifient l'interaction entre deux ou plusieurs composants (ex: l'application + la BDD Postgres).
3. **End-to-End Tests (Tests E2E)** : Simulent le parcours complet d'un utilisateur réel (Auth $
ightarrow$ Session $
ightarrow$ Dashboard $
ightarrow$ Action BDD).

```
         /\          ▲ Plus Cher / Plus Lent
        /  \         │ (Nécessite Infra, Réseau, BDD)
       / E2E\        │
      /──────\       │
     /  Inte- \      │
    /  gration \     │
   /────────────\    │
  /  Unit Tests  \   │ Plus Rapide / Moins Cher
 /────────────────\  │ (In-process, CPU pur)
```

### Le Modèle Mental Pratique : La Portée (Scope) & Les Frontières
Au lieu de débattre indéfiniment de ce qu'est une "unité", Sriniously propose un modèle mental basé sur ce que le test **a le droit de toucher** :

| Niveau | Périmètre / Frontière | Dépendances Autorisées | Vitesse & Stabilité |
| :--- | :--- | :--- | :--- |
| **Small (Petit)** | **In-Process** | Mémoire RAM uniquement. Pas de réseau, pas de BDD, pas de système de fichiers. | Ultra-rapide (~0 ms), 100% Déterministe |
| **Medium (Moyen)** | **Single Machine** | Containers Docker locaux (Postgres local, Redis local sur `localhost`). Pas d'Internet. | Rapide (~quelques ms), Très Stable |
| **Large (Grand)** | **Multi-Machine** | Réseau externe, services Cloud, APIs tierces. | Plus lent, Risque de Flakiness |

---

## 🎭 4. Les Test Doubles : Mocks, Stubs, Dummies, Spies & Fakes

Lorsque notre code doit interagir avec des services externes (BDD, emails, Stripe), on utilise des **Test Doubles** (nom inspiré des cascades au cinéma qui remplacent l'acteur principal).

```
                        ┌──────────────────┐
                        │   TEST DOUBLE    │
                        └────────┬─────────┘
                                 │
     ┌─────────────┬─────────────┼─────────────┬─────────────┐
     ▼             ▼             ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  DUMMY   │  │   STUB   │  │   SPY    │  │   MOCK   │  │   FAKE   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

1. **Dummy** : Objet vide passé uniquement pour satisfaire la signature d'une fonction (ex: un logger inutilisé dans le test).
2. **Stub** : Renvoie une réponse pré-écrite fixe quelle que soit la demande (ex: renvoie toujours le User #7).
3. **Spy** : Comme un Stub, mais enregistre les appels reçus (ex: note la nouvelle valeur passée lors de l'appel).
4. **Mock** : Objet configuré avec des attentes strictes avant l'exécution (ex: *"Vérifie que la méthode `save()` est appelée exactement 1 fois"*).
5. **Fake** : Implémentation fonctionnelle simplifiée non destinée à la prod (ex: une BDD stockée dans une `Map` en mémoire).

### State Verification vs Interaction Verification
* **State Verification (Privilégiée)** : On vérifie l'état final du système (ex: *"Le statut de la tâche est-il 'Done' ?"*). Résistant au refactoring.
* **Interaction Verification** : On vérifie l'implémentation interne (ex: *"La fonction `publish()` a-t-elle été appelée ?"*). Fragile : casse dès qu'on modifie le code interne sans changer le comportement métier.

### La Règle d'Or : "Don't Mock What You Don't Own"
> ⚠️ **Attention** : Ne mockez jamais directement des SDK ou APIs tierces que vous ne contrôlez pas (ex: le SDK Stripe). Si l'API externe évolue, vos mocks continueront de passer alors que la production crashera.
> 
> **La Solution** : 
> 1. Enrobez le SDK tiers dans votre propre **Wrapper / Interface**.
> 2. Testez et mockez votre propre interface.
> 3. Validez l'interface avec les **APIs Sandbox** du fournisseur.

---

## 🏛️ 5. Architectural Patterns pour le Testing

### Injection de Dépendances (Dependency Injection - DI)
Si une fonction instancie elle-même ses dépendances (`const db = new PostgresClient()`), elle est impossible à tester en isolement. En passant les dépendances via les paramètres du constructeur ou de la fonction, on peut y injecter des Fakes ou des Stubs lors des tests.

### Functional Core, Imperative Shell
```
┌─────────────────────────────────────────────────────────────┐
│ 🧅 IMPERATIVE SHELL (Couche fine externe)                   │
│   - Gestion du réseau HTTP, requêtes Postgres, Redis, Emails │
│   - Difficile à tester (nécessite des Test Doubles / Infra) │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 💎 FUNCTIONAL CORE (Cœur Métier Pure)               │   │
│   │   - Fonctions pures : Entrées ──► Sorties           │   │
│   │   - Zéro I/O, Zéro dépendance externe               │   │
│   │   - Ultra-facile et rapide à tester à 100%          │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ 6. Tester la Base de Données : Vitesse vs Fidélité

Remplacer Postgres par une BDD en mémoire (H2, Maps JS) fait gagner en vitesse mais fait perdre la **Fidélité** (les fakes ne gèrent pas les contraintes `UNIQUE`, les index, les verrous SQL ou la syntaxe JSONB).

### Les 3 Solutions Production-Grade pour Tester avec Postgres :
1. **Transaction par Test** : Ouvrir `BEGIN` au début du test, exécuter la logique, puis lancer un `ROLLBACK` à la fin. La BDD reste totalement propre et c'est ultra-rapide.
2. **Template Database** : Exécuter les migrations sur une BDD modèle Postgres, puis cloner cette BDD en quelques millisecondes pour chaque test (`CREATE DATABASE test_1 TEMPLATE template_db`).
3. **Testcontainers** : Démarrer une vraie instance Docker Postgres directement depuis le code de test via la bibliothèque `Testcontainers` (ex: ~7s de boot unique pour toute la suite, puis des ms par test).

---

## ⏰ 7. Tester le Réseau, le Temps & Herméticité (Hermetic Testing)

* **Réseau** : Utilisez des serveurs HTTP de test locaux (`httptest.Server` en Go, `MSW` en Node.js) qui écoutent sur un vrai port local TCP au lieu de simuler le réseau.
* **Gestion du Temps & Flaky Tests** : Ne faites **jamais** de `sleep(100ms)` pour attendre une tâche asynchrone ! Si la machine CI est sous charge, le test échouera.
  * *Solution 1* : Attendre une **condition/signal** (Channels, Callbacks, Event Emitters).
  * *Solution 2* : **Injection d'Horloge (Clock Injection)** pour contrôler l'écoulement du temps dans vos tests.
* **Hermetic Testing (Tests Hermétiques)** : Un test est hermétique lorsqu'il initialise lui-même tout ce dont il a besoin et nettoie tout en fermant, fonctionnant de manière identique sur un ordinateur portable, un serveur SSH ou GitHub Actions.

---

## 🔄 8. Test-Driven Development (TDD) : Le Cycle Red-Green-Refactor

Le TDD n'est pas une sorte de test, mais une **méthodologie de travail** où l'on écrit le test *avant* le code de production.

```
          ┌──────────────────────────────────────────────┐
          │               1. RED (Échec)                 │
          │ Écrire le test d'une fonctionnalité          │
          │ inexistante. Il DOIT échouer !               │
          └──────────────────────┬───────────────────────┘
                                 │
                                 ▼
          ┌──────────────────────────────────────────────┐
          │              2. GREEN (Succès)               │
          │ Écrire le code minimal (même moche)          │
          │ pour faire passer le test.                   │
          └──────────────────────┬───────────────────────┘
                                 │
                                 ▼
          ┌──────────────────────────────────────────────┐
          │             3. REFACTOR (Propreté)           │
          │ Nettoyer le code et restructurer sans        │
          │ casser le test qui reste vert.               │
          └──────────────────────────────────────────────┘
```

### Quand Utiliser le TDD ?
* ✅ **Recommandé** : Quand le problème et les entrées/sorties sont clairement définis mais que la logique complexe reste à construire (ex: Parsers JSON/XML, moteurs de calcul de prix, machines à états).
* ❌ **À éviter** : En phase d'exploration système ou quand l'architecture globale est encore floue et sujette à de fréquents remaniements.

---

## 🚨 9. Flaky Tests : Diagnostics & Remèdes

Un **Flaky Test** est un test qui passe ou échoue aléatoirement sur le même code sans aucune modification. C'est un poison pour l'équipe car il détruit la confiance envers la suite de tests.

### Les 3 Causes Principales (Étude Luo et al. 2014) :
1. **Async / Await** : Attente basée sur du temps (`sleep`) au lieu de signaux d'achèvement.
2. **Concurrence** : Race conditions entre threads ou goroutines dans le test.
3. **Test Order Dependency** : Le test B ne passe que si le test A a été exécuté juste avant (fuite d'état en BDD ou variables globales).

### Comment Réagir face à un Flaky Test ?
1. **Quarantaine** : Isoler le test immédiatement dans une suite non bloquante avec le nom du dev, la date et la raison.
2. **Correction** : Remplacer les timers par des signaux et nettoyer l'état après chaque test.
3. **Suppression** : Supprimer le test s'il n'apporte pas de valeur réelle.

---

## 📊 10. Mesurer la Qualité : Coverage, Mutation Testing & Complexité

### 1. Code Coverage (Couverture de Code)
* **Principe** : Le compilateur injecte des compteurs dans les blocs de code pour mesurer le pourcentage de lignes (`Line Coverage`) ou de branches (`Branch Coverage`) exécutées.
* **Le Piège** : 100% de couverture ne garantit pas la qualité d'un test (un test sans aucune assertion peut atteindre 100% de couverture).
* **Consensus** : Cibler **~80% de couverture**. La vraie valeur du rapport réside dans la lecture des lignes *non couvertes* (les lignes rouges).

### 2. Mutation Testing (Ex: Stryker)
* **Principe** : "Tester les tests". L'outil altère volontairement votre code source (*Mutants* : remplace `>` par `>=`, `+` par `-`) et relance vos tests.
* **Score** : Si votre test échoue, le mutant est **tué** (bon test). S'il passe malgré l'altération, le mutant **survit** (test insuffisant).

```
  Source Originale:  if (amount > 100) -> Code altéré: if (amount >= 100)
  Résultat des Tests: 
   - Test Fails ──► 🟢 Mutant Tué (Test efficace)
   - Test Passes ──► 🔴 Mutant Survécu (Lacune dans les assertions)
```

### 3. Complexité du Code
* **Cyclomatic Complexity (McCabe, 1976)** : Compte le nombre de chemins décisionnels indépendants ($1 + 	ext{décisions}$). Seuil recommandé < 10. *(Inconvénient : Pénalise les grands `switch` plats très lisibles)*.
* **Cognitive Complexity (SonarSource)** : Mesure la charge mentale et la profondeur d'imbrication (*Nesting*). Un `switch` flat a un score faible, tandis que des `if` imbriqués dans des boucles ont un score élevé.

---

## 🛠️ 11. Boîte à Outils de Qualité de Code

* **Linters** (ex: ESLint, `golangci-lint`) : Analysent le code source pour repérer des *patterns* problématiques (variables inutilisées, erreurs non capturées).
* **Formatters** (ex: Prettier, `gofmt`) : Unifient le style visuel du code.
* **Type Checkers** (ex: TypeScript, Go, Rust) : Vérifient la cohérence des types à la compilation.
* **Static Analysis** : Analyse poussée du flux de données sans exécution pour prévenir les failles de sécurité.

---