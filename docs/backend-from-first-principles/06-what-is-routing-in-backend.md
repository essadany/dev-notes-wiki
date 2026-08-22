# 6. Qu'est-ce que le routing dans le backend ?

> *Objectif : Comprendre comment le serveur sait exactement quel bout de code exécuter quand il reçoit une requête. C'est l'aiguillage du backend !*

---

## 1. La Formule Magique du Routage

Pour faire simple, une requête reçue par le serveur contient deux informations indispensables :
*   **Le QUOI (L'Intention)** : Exprimé par la méthode HTTP (GET, POST, etc.).
*   **Le OÙ (La Destination)** : Exprimé par le chemin de la route (le path ou l'URL, ex: `/users`).

> 💡 **Définition à retenir** : Le routage, c'est l'action de mapper une combinaison unique de `[Méthode HTTP + Chemin URL]` vers une logique serveur spécifique (appelée un **Handler**).

```
   REQUÊTE ENTRANTE
 ┌──────────────────────┐
 │ • Méthode : GET      │ ───┐
 │ • Route : /books     │    │
 └──────────────────────┘    │     MAPPING UNIQUE (La Clé)
                             ├─> [ GET + /books ] ──> Exécute : handlerObtenirLivres()
 ┌──────────────────────┐    │
 │ • Méthode : POST     │ ───┘
 │ • Route : /books     │
 └──────────────────────┘ ─────> [ POST + /books ] ──> Exécute : handlerCreerLivre()
```

*Note perso* : Même si l'URL est identique (`/books`), le fait d'avoir deux méthodes différentes (GET vs POST) crée deux routes totalement uniques côté serveur. Elles ne se percuteront jamais !

---

## 2. Routes Statiques vs. Routes Dynamiques

Dans le code, on va croiser deux grandes familles de routes :

### A. Les Routes Statiques
Elles ne bougent pas, ce sont des chaînes de caractères constantes.
*   *Exemple* : `GET /api/books`
*   *Pourquoi ?* Elles ciblent une collection globale ou une action fixe. Elles renvoient toujours la même structure de données.

### B. Les Routes Dynamiques
Elles contiennent des variables. On utilise un **identifiant dynamique** pour cibler une ressource spécifique.
*   *Convention universelle* : On utilise le symbole `:` devant le nom de la variable dans le code (ex: `:id`). C'est le standard dans l'industrie (Node, Python, Go, Rust, etc.).
*   *Exemple dans le code* : `GET /api/users/:id`
*   *Dans la réalité (l'URL appelée)* : `GET /api/users/123`

```
   URL demandée : /api/users/123
                    │         │
   Correspondance : /api/users/:id   <── Le serveur extrait "123" et le convertit en chaîne
                                         de caractères pour l'injecter dans la variable "id".
```

---

## 3. Le Match : Path Parameters vs. Query Parameters

C'est souvent confus au début, mais chaque paramètre a un rôle sémantique ultra-précis :

| Type de Paramètre | Format dans l'URL | Exemple d'utilisation | Rôle Sémantique |
| :--- | :--- | :--- | :--- |
| **Path / Route Parameter** | Fait partie intégrante du chemin (`/api/users/123`) | Identifier une ressource unique précise | "Donne-moi les détails de l'utilisateur numéro 123." |
| **Query Parameter** | Après le point d'interrogation (`?key=value`) | `/api/books?page=2&limit=20` ou `/api/search?query=react` | Transmettre des métadonnées (Pagination, Filtres, Tri) |

### Pourquoi utiliser des Query Params plutôt que des Path Params ?
1.  **Limitation de HTTP GET** : Les requêtes GET n'ont pas de corps (body). Si on veut envoyer des données au serveur, on doit passer par l'URL.
2.  **Sémantique REST** : Mettre des filtres complexes ou de la recherche dans le chemin (ex: `/api/search/react`) rend l'API difficile à maintenir et détruit la sémantique. On préfère largement `/api/search?query=react`.

### Focus : Le cas d'école de la Pagination 📖
Quand on récupère une liste (ex: 100 livres), le serveur ne doit pas tout renvoyer d'un coup pour ne pas ramer. Il pagine :
1.  **Requête 1** (par défaut ou implicite) : `GET /api/books` (Le serveur renvoie les 20 premiers livres + des métadonnées : `currentPage: 1`, `totalPages: 5`).
2.  **Requête 2** (à l'initiative du client) : `GET /api/books?page=2` (Le serveur sait qu'il doit renvoyer la suite grâce au Query Parameter !).

---

## 4. Le Routage Imbriqué (Nested Routing)

Pour exprimer des relations logiques complexes entre différentes ressources, on "imbrique" les routes. C'est indispensable pour garder une API humaine et lisible !

```
┌───────────────────────────────────────┬────────────────────────────────────────────────────────┐
│ URL REST Imbriquée                    │ Ce que ça exprime sémantiquement                        │
├───────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ GET /api/users                        │ "Récupère la liste de tous les utilisateurs."          │
│ GET /api/users/123                    │ "Récupère les détails de l'utilisateur 123."           │
│ GET /api/users/123/posts              │ "Récupère TOUS les posts écrits par l'utilisateur 123." │
│ GET /api/users/123/posts/456          │ "Récupère le post précis 456 écrit par l'user 123."    │
└───────────────────────────────────────┴────────────────────────────────────────────────────────┘
```

Chaque niveau de barre oblique `/` nous fait descendre d'un niveau dans la hiérarchie des données.

---

## 5. Versioning et Dépréciation de l'API

En production, les besoins évoluent et on doit parfois modifier le format des données renvoyées par notre API. C'est là qu'interviennent les **Breaking Changes** (changements majeurs qui cassent l'application front-end si elle n'est pas mise à jour).

Pour éviter de tout casser, on utilise le **Versioning** :

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          STRATÉGIE DE VERSIONING                        │
├────────────────────────────────────┬────────────────────────────────────┤
│ Version 1 (V1)                     │ Version 2 (V2)                     │
│ /api/v1/products                   │ /api/v2/products                   │
├────────────────────────────────────┼────────────────────────────────────┤
│ Réponse JSON ancienne :            │ Réponse JSON nouvelle :            │
│ {                                  │ {                                  │
│   "id": 1,                         │   "id": 1,                         │
│   "name": "Clavier"                │   "title": "Clavier"               │
│ }                                  │ }                                  │
└────────────────────────────────────┴────────────────────────────────────┘
```

> 🛡️ **Pourquoi c'est vital ?**
> *   On ne change pas directement la route existante pour ne pas faire planter les applications mobiles ou front-ends déjà déployées.
> *   On crée une route `v2`.
> *   On laisse une **fenêtre de migration** aux développeurs front pour passer de `v1` à `v2`.
> *   Une fois la migration terminée, on peut **déprécier** (deprecate) puis supprimer définitivement la version 1.

---

## 6. Le Filet de Sécurité : La Route "Catch-All" (Fallback)

Qu'est-ce qui se passe si un client tente d'appeler une URL qui n'existe pas (ex: `/api/v3/products`) ?
Par défaut, sans configuration, le serveur peut renvoyer une réponse vide ou planter.

Pour éviter ça, on place toujours une **route Catch-All** tout à la fin de notre fichier de routage :

```
[ Algorithme de Matching du Serveur ]
 ├─ /api/v1/users (GET)     ──> Non adapté
 ├─ /api/v1/products (GET)  ──> Non adapté
 ├─ ...
 └─ /* (CATCH-ALL)          ──> MATCH !
                                 │
                                 ▼
                     Exécute le handler de secours
                     (Renvoie un message propre "Route non trouvée" avec un code 404)
```

*Règle d'or de l'ordre de déclaration* : La route Catch-All (souvent symbolisée par `*`) doit **impérativement être déclarée en dernier**. Si on la met au début, elle va intercepter absolument toutes les requêtes !
