# 16. Gestion des Erreurs & Systèmes Tolérants aux Pannes (Fault Tolerance)

En développement backend, les erreurs ne sont pas de simples "bugs à corriger" : **elles sont inévitables**. Les bases de données échouent, les API externes tombent en panne, les utilisateurs envoient des données corrompues et la logique métier rencontre des cas limites. Le rôle d'un ingénieur backend n'est pas d'espérer que rien ne plante, mais d'avoir un **mindset tolérant aux pannes (Fault-Tolerant Mindset)** pour détecter, contenir et réparer les pannes avant qu'elles n'impactent la production.

Voici mon carnet de notes complet rédigé avec des métaphores, des schémas d'architecture et les règles de sécurité indispensables.

---

## 🧠 1. Le Mindset "Fault-Tolerant" : Préparer le Pire

```
                     ┌──────────────────────────────────────────────┐
                     │         LA RÉALITÉ DU BACKEND ENGINE         │
                     └──────────────────────┬───────────────────────┘
                                            │
        ┌───────────────────────────────────┼───────────────────────────────────┐
        ▼                                   ▼                                   ▼
┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
│     BASES DE DONNÉES    │     │      APIS EXTERNES      │     │  UTILISATEURS & DATA    │
│ • Pool saturé           │     │ • Timeouts réseau       │     │ • Payloads corrompus    │
│ • Deadlocks             │     │ • Rate limiting (429)   │     │ • Edge cases imprévus   │
│ • Vio. de contraintes   │     │ • Outages cloud / 503   │     │ • Injections / Fautes   │
└─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘
```

> 💡 **Le mot d'ordre** : La question n'est pas de savoir *si* une erreur va se produire, mais *comment* votre système va réagir quand elle surviendra.

---

## 🔍 2. Les 5 Catégories d'Erreurs en Backend

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                               CATÉGORIES D'ERREURS BACKEND                               │
├─────────────────┬─────────────────┬─────────────────┬──────────────────┬─────────────────┤
│    LOGIQUE      │ BASE DE DONNÉES │  APIS EXTERNES  │ VALIDATION INPUT │ CONFIGURATION   │
│ (Les plus sournois)│ (Crash système)│ (Hors contrôle) │  (1ère défense)  │(Fail at Startup)│
└─────────────────┴─────────────────┴─────────────────┴──────────────────┴─────────────────┘
```

### A. Les Erreurs de Logique (Logic Errors)
* **Pourquoi elles sont dangereuses** : Votre application ne plante pas (pas d'exception levée), mais elle produit des **résultats métiers complètement faux**.
* **Exemple concret** : Dans un e-commerce, un algorithme applique un code promo deux fois de suite ou calcule des frais de port négatifs. L'application fonctionne, mais l'entreprise perd de l'argent sur chaque commande pendant des semaines sans que personne ne s'en rende compte.
* **Causes principales** : Mauvaise compréhension des spécifications, algorithme trop complexe mal testé, oubli des cas limites (*edge cases*).

### B. Les Erreurs de Base de Données (Database Errors)
Elles peuvent mettre l'ensemble de la plateforme KO si elles sont mal gérées :
1. **Erreurs de Connexion** : Réseau coupé, base surchargée ou **épuisement du Connection Pool** (toutes les connexions TCP sont occupées).
2. **Violations de Contraintes** :
   * *Unique Constraint Violation* : Tenter d'inscrire un utilisateur avec un email qui existe déjà.
   * *Foreign Key Violation* : Tenter d'insérer une commande (`orders`) avec un `customer_id` qui n'existe pas dans la table `customers`.
3. **Erreurs de Requête & Deadlocks** :
   * *SQL Malformé* : Faute de frappe dans une requête (`SELECT * FROM custmers`).
   * *Deadlock (Verrouillage circulaire)* : Deux transactions concurrentes verrouillent des lignes en s'attendant mutuellement.

### C. Les Erreurs de Services Externes (External Dependencies)
Votre backend dépend souvent de services tiers (Stripe, Auth0, Clerk, Resend, S3, OpenAI). Chaque service externe est un point de défaillance hors de votre contrôle direct :
* **Coupures Réseau & DNS** : L'Internet n'est pas parfait, des paquets se perdent.
* **Erreurs d'Authentification/Tokens** : Clé API expirée ou révoquée.
* **Rate Limiting (`429 Too Many Requests`)** : Le fournisseur bloque vos requêtes si votre backend envoie un volume anormal d'appels.
* **Panne Générale (Outage Cloud)** : Une région AWS ou GCP rencontre un incident majeur.

### D. Les Erreurs de Validation d'Entrée (Input Validation)
* Elles surviennent lorsque les utilisateurs envoient des données non conformes aux règles de l'API.
* C'est votre **première ligne de défense**. On vérifie le format (email, date), la plage numérique (Range validation) et la présence des champs obligatoires.
* **Code HTTP renvoyé** : `400 Bad Request`. Ce sont les erreurs les plus simples à anticiper et à gérer.

### E. Les Erreurs de Configuration (Configuration Errors)
* Elles apparaissent souvent lors des déploiements entre Dev, Staging et Production (ex: l'oubli de la variable `OPENAI_API_KEY` dans le fichier `.env` de production).
* ⚠️ **La règle d'or du démarrage (Startup Validation)** : Validez TOUTES les variables d'environnement requises au démarrage du serveur. Si une clé manque, faites planter le serveur immédiatement (*Fail Fast*).
* **Pourquoi ?** Si vous ne validez pas au démarrage, votre déploiement Green/Blue réussira, mais l'application plantera avec des erreurs 500 en plein runtime lorsqu'un utilisateur cliquera sur la fonctionnalité !

---

## 🛡️ 3. Prévention & Détection Proactive : Agir Avant la Panne

La meilleure stratégie de gestion d'erreur est celle qui intervient **avant** que l'erreur ne survienne.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                               DÉTECTION PROACTIVE EN PROD                                │
├──────────────────────────────┬──────────────────────────────┬────────────────────────────┤
│   1. HEALTH CHECKS SIMPLES   │   2. DEEP HEALTH CHECKS      │   3. BUSINESS METRICS      │
│  • Endpoint /health          │  • Exécuter une requête SQL  │  • Chute soudaine des      │
│  • Vérifie si le serveur     │  • Transaction de test Stripe│    transactions réussies   │
│    répond 200 OK             │  • Tester un token Auth0     │  • Alerte même si 0 bug    │
└──────────────────────────────┴──────────────────────────────┴────────────────────────────┘
```

1. **Health Checks d'Infrastructure** : Exposer un endpoint HTTP `/health` interrogé en permanence par vos outils de monitoring.
2. **Deep Health Checks (Vérification Métier)** : Un simple `/health` ne suffit pas (le serveur Node peut tourner alors que la BDD est morte). Il faut effectuer un *ping* réel sur la BDD, valider un token de test et envoyer un micro-message de test aux services d'emailing.
3. **Surveillance des Métriques de Performance & Business** :
   * Une dégradation de la latence (une requête qui passe de 200ms à 4s) est le signe annonciateur d'un crash imminent.
   * Une baisse soudaine du taux de commandes validées indique un problème technique, même si le taux d'erreur serveur reste bas.
   * **Logs structurés JSON** : Formatage universel des logs pour agrégation dans des outils comme Grafana Loki.

---

## ⚡ 4. Stratégies de Réponse & Récupération Implémentées

Quand une erreur survient en runtime, la réponse dépend de la nature de l'erreur :

```
                                  ┌───────────────────────────┐
                                  │   UN INCIDENT SURVIENT    │
                                  └─────────────┬─────────────┘
                                                │
                 ┌──────────────────────────────┴──────────────────────────────┐
                 ▼                                                             ▼
  ┌─────────────────────────────┐                               ┌─────────────────────────────┐
  │     ERREURS RÉCUPÉRABLES    │                               │   ERREURS NON RÉCUPÉRABLES  │
  ├─────────────────────────────┤                               ├─────────────────────────────┤
  │ • Timeout réseau temporaire │                               │ • Service tiers totalement  │
  │ • Pool BDD saturé 1 sec     │                               │   down / Panne AWS majeure  │
  ├─────────────────────────────┤                               ├─────────────────────────────┤
  │ STRATÉGIE :                 │                               │ STRATÉGIE :                 │
  │ 🔄 Exponential Backoff      │                               │ 🛡️ Containment & Graceful   │
  │    (Attendre 1s, 2s, 4s...) │                               │    Degradation (Basculer sur│
  │    + Retry avec répit.      │                               │    le cache, bloquer la feat│
  │                             │                               │    sans faire crasher l'app)│
  └─────────────────────────────┘                               └─────────────────────────────┘
```

* **Exponential Backoff** : En cas de rate limiting (`429`) ou de micro-coupure réseau, n'assommez pas le serveur cible avec des retries instantanés. Espacez progressivement les tentatives (ex: 1s, 2s, 4s, 8s).
* **Graceful Degradation** : Si le service de recommandation par IA tombe en panne, n'affichez pas une page d'erreur 500 à l'utilisateur. Masquez le bloc de recommandation ou servez une liste statique en cache !

---

## 🧱 5. Le "Filet de Sécurité Ultime" : Global Error Handling Middleware

Plutôt que d'éparpiller des blocs `try/catch` et des `if (err)` verbeux dans chaque fichier de votre projet, la meilleure architecture consiste à faire **remonter (bubble up)** toutes les erreurs vers un **Middleware Global de Gestion d'Erreurs**.

### Flux d'une Requête et Propagation des Erreurs :

```
[ ROUTER ] ───> [ HANDLER / CONTROLLER ] ───> [ SERVICE MÉTIER ] ───> [ REPOSITORY (SQL) ]
                        │                             │                        │
                        │ (Validation Failed)         │ (Logic Error)          │ (DB Error: Unique/NoRows)
                        └─────────────────────────────┴────────────────────────┘
                                                      │
                                                      ▼  (Bubble Up / Throw Exception)
                                        ┌───────────────────────────┐
                                        │   GLOBAL ERROR HANDLER    │
                                        │        MIDDLEWARE         │
                                        └─────────────┬─────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        ▼                             ▼                             ▼
         ┌───────────────────────────┐ ┌───────────────────────────┐ ┌───────────────────────────┐
         │     400 BAD REQUEST       │ │       404 NOT FOUND       │ │ 500 INTERNAL SERVER ERR.  │
         │ • Champ > 500 caractères  │ │ • Ressource introuvable   │ │ • Erreur non gérée /      │
         │ • Unique key déjà en BDD  │ │   (No rows returned / FK) │ │   Crash inattendu (Masqué)│
         └───────────────────────────┘ └───────────────────────────┘ └───────────────────────────┘
```

### Conversion des Erreurs dans le Middleware Global :
1. **Erreur de Validation (Handler)** $
ightarrow$ Renvoie HTTP `400 Bad Request` avec la liste exacte des champs invalides.
2. **Erreur BDD `Unique Constraint` (ex: Livre déjà existant)** $
ightarrow$ Renvoie HTTP `400 Bad Request` avec un message propre : `"Ce livre existe déjà dans la base."`
3. **Erreur BDD `No Rows Returned` (ex: `GET /books/123` inconnu)** $
ightarrow$ Renvoie HTTP `404 Not Found` : `"Le livre avec l'ID 123 n'existe pas."`
4. **Erreur BDD `Foreign Key Violation` (ex: `author_id` inexistant)** $
ightarrow$ Renvoie HTTP `404 Not Found` : `"L'auteur spécifié n'existe pas."`
5. **Erreur Inconnue / Non Gérée** $
ightarrow$ Renvoie HTTP `500 Internal Server Error` avec un message générique pour préserver la sécurité.

> 💎 **Les 2 Avantages Majeurs** :
> 1. **Robustesse** : Zéro risque d'oublier de gérer une erreur dans une route. Tout est rattrapé au même endroit.
> 2. **Réduction de la Redondance** : Évite de répéter la même logique de traitement d'erreurs SQL dans des dizaines de fichiers Repositories.

---

## 🔒 6. Sécurité & Pratiques Avancées d'Error Handling

La gestion des erreurs est l'une des cibles principales des attaquants. Voici les 3 règles de sécurité OWASP indispensables :

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                 RÈGLES DE SÉCURITÉ ERROR                                 │
├──────────────────────────────┬──────────────────────────────┬────────────────────────────┤
│  1. MASQUER LES DÉTAILS BDD  │   2. ANTI-ENUMÉRATION D'AUTH │    3. NETTOYAGE DES LOGS   │
│ • Ne JAMAIS leaker les noms  │ • Ne pas dire "Password faux"│ • Ne JAMAIS logger tokens, │
│   de tables, colonnes ou     │ • Renvoyer un message uni-   │   mots de passe ou CB.     │
│   exceptions SQL brutes.     │   forme: "Identifiants faux" │ • Logger uniquement des IDs│
└──────────────────────────────┴──────────────────────────────┴────────────────────────────┘
```

1. **Zéro Fuite d'Informations Internes (Information Disclosure)** :
   * Ne renvoyez jamais la stack trace brute ou l'erreur SQL du driver à l'utilisateur. Des hackers expérimentés utilisent ces noms de tables et d'index pour construire des attaques par injection SQL complexes.
   * Pour les erreurs 500, le message doit TOUJOURS être générique : `"Internal Server Error"` ou `"Une erreur est survenue"`.
2. **Protection contre l'Énumération d'Utilisateurs (Auth Module)** :
   * Sur un endpoint de connexion (`POST /login`), ne dites pas *"Cet email n'existe pas"* puis *"Mot de passe incorrect"*. Un attaquant pourrait tester des listes d'emails pour trouver lesquels ont un compte sur votre site.
   * **Solution** : Renvoyez toujours un message uniforme : `"Identifiants ou mot de passe invalides"`.
3. **Sécurisation des Logs (Log Sanitization)** :
   * Lors des fuites de données d'entreprises, ce sont souvent les serveurs de logs qui sont compromis.
   * Ne loggez **jamais** de données sensibles : mots de passe, clés d'API, tokens JWT, numéros de carte bancaire ou emails en clair.
   * Loggez toujours un **User ID anonymisé** et un **`Correlation-ID` / `Request-ID`** pour pouvoir tracer l'erreur sans compromettre la vie privée de vos utilisateurs.
