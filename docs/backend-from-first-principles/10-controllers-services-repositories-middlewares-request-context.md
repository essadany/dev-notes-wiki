# 10. Architecture en Couches, Middlewares & Request Context

Aujourd'hui, gros morceau sur la structuration interne d'un serveur backend : **comment organiser proprement son code quand l'application grandit** ! Sriniously explique pourquoi mettre tout le code dans une seule fonction géante est un piège et comment découper proprement une application avec la **Layered Architecture** (Controllers, Services, Repositories), utiliser les **Middlewares** et exploiter le **Request Context**.

---

## 🌐 1. Pourquoi Découper le Code ? (La Layered Architecture)

Au tout début, quand on apprend le backend, on a tendance à tout écrire dans le Handler de route : la lecture de la requête, la validation, la requête SQL, l'envoi d'email et la réponse HTTP. 

Sur une petite application, ça marche. Mais sur un projet de production avec des dizaines ou des centaines de routes, le code devient un cauchemar impossible à tester, maintenir ou faire évoluer.

Pour éviter le bazar, on utilise un modèle de conception classique : la **Séparation des Responsabilités (Separation of Concerns)** en découpant le code en 3 couches distinctes.

```
 [ CLIENT ] ──(Requête HTTP)──> [ ROUTEUR ]
                                   │
                                   ▼
                         ┌──────────────────┐
                         │ 1. CONTROLLER /  │ ──> Reçoit req/res, Désérialise, Valide & Transforme,
                         │    HANDLER       │     Gère les Status Codes HTTP (200, 400, 500)
                         └─────────┬────────┘
                                   │ (Données pures & Contexte)
                                   ▼
                         ┌──────────────────┐
                         │ 2. SERVICE LAYER │ ──> Logique Métier pure (Calculs, Orchestration,
                         │    (Business)    │     envoi d'emails, APIs externes). Zero HTTP !
                         └─────────┬────────┘
                                   │ (Requêtes BDD)
                                   ▼
                         ┌──────────────────┐
                         │ 3. REPOSITORY    │ ──> Accès direct à la Base de Données (SQL/ORM).
                         │    LAYER (Data)  │     Une méthode = Une tâche BDD précise.
                         └──────────────────┘
```

---

## 🧱 2. Le Rôle Détallé de Chaque Couche

### A. Le Controller / Handler (Le Gendarme du Trafic HTTP)
Le Controller est la seule couche qui parle le langage du protocole HTTP. Dans la plupart des langages (Go, Node.js avec Express, Python avec FastAPI), le Controller reçoit automatiquement deux objets : la requête (`req`) et la réponse (`res`).

* **Ses Responsabilités :**
  1. **Extraction** : Récupère les données envoyées par le client (`query params`, `request body`, `path params`).
  2. **Désérialisation / Binding** : Convertit le JSON reçu en objet/struct natif du langage. Si la désérialisation échoue, il renvoie immédiatement une erreur `400 Bad Request`.
  3. **Validation & Transformation** : S'assure que les données sont valides et applique les valeurs par défaut (ex: trier par date par défaut si le client n'a rien précisé).
  4. **Appel du Service** : Transmet les données validées et le contexte d'authentification à la couche Service.
  5. **Réponse HTTP** : Reçoit le résultat du Service et choisit le code d'état HTTP approprié (`200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `500 Server Error`).

> 💡 **Astuce de dev** : Le Controller est la frontière HTTP. Dès qu'on quitte le Controller pour entrer dans le Service, le protocole HTTP n'existe plus !

### B. Le Service (Le Cœur Métier)
La couche Service contient toute la **logique applicative / métier**. Elle est totalement isolée et agnostique du web.

* **Ses Responsabilités :**
  * Effectuer les calculs métier et appliquer les règles d'entreprise.
  * Orchestrer plusieurs requêtes vers la base de données (en appelant plusieurs méthodes de Repository).
  * Déclencher des actions externes : envoi d'emails, notifications Push, appels à des API tiers (Stripe, Twilio, etc.).
* **Règle d'or** : Si on regarde une fonction de Service, on ne doit **pas** pouvoir deviner qu'elle est appelée par une API HTTP. Elle prend des données pures en entrée, réalise un traitement et retourne un résultat pur.

### C. Le Repository (La Couche de Persistence BDD)
La couche Repository est la seule autorisée à exécuter des requêtes vers la base de données (SQL, MongoDB, Redis).

* **Ses Responsabilités :**
  * Construire et exécuter les requêtes de base de données (Insert, Select, Update, Delete).
  * Retourner les données brutes sous forme d'objets ou d'enregistrements à la couche Service.
* ⚠️ **Bonne pratique** : Une méthode de Repository doit avoir une **seule et unique responsabilité** (ex: `findBookById(id)`, `getAllBooks(sort)`). On évite de créer des méthodes géantes avec plein de paramètres optionnels pour gérer plusieurs cas.

### 📊 Tableau Comparatif des 3 Couches

| Couche | Ce qu'elle reçoit | Ce qu'elle traite | Manipule HTTP (`req`/`res`) ? | Rôle Principal |
| :--- | :--- | :--- | :---: | :--- |
| **Controller** | Requête HTTP (`req`) | Parsing JSON, Validation, Code de réponse | **OUI** | Interface HTTP & Contrôle du flux |
| **Service** | Données pures / DTO | Règles métier, Orchestration, APIs externes | **NON** | Cœur applicatif & Logique métier |
| **Repository** | Paramètres de recherche | Requêtes BDD (SQL, ORM) | **NON** | Persistence & Lecture Base de données |

---

## 🚦 3. Les Middlewares (Les Douaniers du Flux)

### Qu'est-ce qu'un Middleware ?
Un middleware est une fonction qui s'exécute **dans l'intervalle** (au milieu) du cycle de vie d'une requête, entre l'entrée sur le serveur et l'arrivée au Controller final.

Chaque middleware reçoit trois arguments de la part du runtime : `(req, res, next)`.
* `req` : L'objet requête pour lire ou modifier les données.
* `res` : L'objet réponse pour pouvoir envoyer une réponse anticipée (ex: erreur 401).
* `next()` : Une fonction clé qui permet de passer la main au middleware suivant dans la chaîne.

### Pourquoi les utiliser ?
1. **Réduire la duplication de code (DRY)** : Éviter de répéter le même code de sécurité, de logs ou d'authentification sur des centaines de routes API.
2. **Couper court à la requête (Short-Circuit)** : Si une vérification échoue (ex: jeton d'authentification manquant ou expiré), le middleware renvoie immédiatement une erreur `401 Unauthorized` au client sans gaspiller les ressources du serveur.

### ⚠️ L'Ordre d'Exécution des Middlewares (CRUCIAL)
L'ordre dans lequel on déclare les middlewares est d'une importance capitale car la requête les traverse dans un sens séquentiel strict !

```
[ Entrée de la Requête HTTP ]
         │
         ▼
 ┌───────────────┐
 │  1. CORS      │ ──> Rejette immédiatement les origines non autorisées
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  2. LOGGING   │ ──> Enregistre la méthode, l'URL, l'IP et l'horodatage
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  3. SECURITY  │ ──> Applique le Rate Limiting (ex: max 30 req/sec → 429) & Headers CSP
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  4. BODY PARSE│ ──> Désérialise le JSON du Body en objet natif
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  5. AUTH      │ ──> Vérifie le Token JWT, extrait l'User ID & l'injecte dans le Context
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  6. ROUTEUR & │ ──> Exécute la chaîne : Controller ──> Service ──> Repository
 │     HANDLER   │
 └───────┬───────┘
         ▼
 ┌───────────────┐
 │  7. GLOBAL    │ ──> PLACÉ EN DERNIER ! Attrape toutes les erreurs non gérées
 │     ERROR     │     dans l'application et renvoie une réponse propre (400 / 500).
 └───────────────┘
```

> 🚨 **Piège classique en entretien** : Pourquoi le middleware de gestion globale des erreurs doit-il TOUJOURS être placé en dernier ? Parce que si une erreur survient dans un Controller ou un Service, le flux remonte vers le bas. S'il était placé en premier, il ne verrait jamais les erreurs survenues plus bas dans la chaîne !

---

## 🎒 4. Le Request Context (Le Sac à Dos de la Requête)

### Qu'est-ce que le Request Context ?
Le **Request Context** est un espace de stockage temporaire (sous forme de dictionnaire clé-valeur) créé au moment où une requête HTTP arrive sur le serveur, et détruit dès que la réponse est envoyée.

Il permet de partager des informations d'état **entre différents middlewares et controllers** sans coupler le code ni passer des dizaines d'arguments manuellement.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       REQUEST CONTEXT (Stockage Clé-Valeur)                  │
│                                                                             │
│  • user_id : "usr_987654"        <── Injecté par le Middleware d'Auth       │
│  • user_role : "admin"           <── Injecté par le Middleware d'Auth       │
│  • request_id : "uuid-1234-5678" <── Injecté par le Middleware de Logging │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Cas d'usages fondamentaux du Context :

1. **Sécurité & Isolation des données Utilisateur** :
   * **Le problème** : Si le client envoyait son `user_id` directement dans le body JSON de la requête, un utilisateur malveillant pourrait envoyer l'ID d'un autre utilisateur et pirater ses données.
   * **La solution avec Context** : Le middleware d'authentification valide le Token JWT, extrait le vrai `user_id` vérifié et l'enregistre dans le Request Context. Plus bas dans la chaîne, le Controller et le Service récupèrent cet `user_id` depuis le Context en toute sécurité.
2. **Traçabilité des Logs (Distributed Tracing)** :
   * Un middleware génère un identifiant unique `X-Request-ID` (ex: UUID) au tout début de la requête.
   * Toutes les fonctions de l'application et tous les appels vers d'autres microservices réutilisent cet ID depuis le Context pour suivre le cheminement complet de la requête dans les logs.
3. **Gestion des Timeouts & Annulations** :
   * Permet de propager des signaux d'annulation (*Abort signals*) si le client ferme son navigateur avant que le serveur n'ait fini d'exécuter la requête en base de données.

---

*Notes de cours rédigées avec soin sur l'architecture backend moderne.*
