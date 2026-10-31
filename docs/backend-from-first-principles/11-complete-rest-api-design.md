# 11. Conception d'APIs REST (Complete REST API Design)

Aujourd'hui, chapitre capital dans la formation backend : **la conception d'APIs REST (Representational State Transfer)**. Après avoir compris HTTP, le routage et la sérialisation, il s'agit d'apprendre à concevoir des APIs propres, prévisibles et intuitives pour les équipes front-end et mobiles !

---

## 1. C'est quoi REST en gros ? (Le modèle mental)

> 💡 **REST (Representational State Transfer)** est un style architectural standardisé pour concevoir des services web. Au lieu d'inventer nos propres conventions bizarres, REST impose de structurer nos APIs autour de **Ressources** manipulables via les verbes HTTP standards.

### La différence entre la vue "Action" et la vue "Ressource" :

```
❌ ARCHITECTURE ORIENTÉE ACTIONS (Non REST / Style RPC antique)
┌─────────────────────────────────────────────────────────────┐
│ GET  /getUserDetails?id=42                                  │
│ POST /createNewUser                                         │
│ POST /updateUserEmail                                       │
│ POST /deleteUserPermanently                                 │
└─────────────────────────────────────────────────────────────┘

✅ ARCHITECTURE ORIENTÉE RESSOURCES (RESTful)
┌─────────────────────────────────────────────────────────────┐
│ GET    /api/v1/users/42   ──> Récupérer l'utilisateur 42    │
│ POST   /api/v1/users      ──> Créer un utilisateur          │
│ PATCH  /api/v1/users/42   ──> Modifier l'email / profil    │
│ DELETE /api/v1/users/42   ──> Supprimer l'utilisateur 42    │
└─────────────────────────────────────────────────────────────┘
```

**Règle d'or de dev** : L'URL désigne la **Ressource** (un nom au pluriel). La **Méthode HTTP** désigne l'**Action** à accomplir sur cette ressource.

---

## 2. Les Règles d'Or du Naming des Endpoints (URLs)

Pour qu'une API soit agréable à utiliser par les devs front-end, les URLs doivent être hyper prévisibles.

### A. Noms au pluriel & minuscules
* ❌ `/api/user` ou `/api/GetUser`
* ✅ `/api/v1/users`

### B. Hiérarchie et Ressources Imbriquées (Nested Resources)
Quand une ressource dépend directement d'une autre (ex: les commentaires d'un article spécifique) :

```
┌─────────────────────────────────────────────────────────────────────────┐
│ GET  /api/v1/posts/108/comments      ──> Tous les commentaires du post 108│
│ POST /api/v1/posts/108/comments      ──> Ajouter un commentaire au post │
│ GET  /api/v1/posts/108/comments/5    ──> Le commentaire spécifique #5   │
└─────────────────────────────────────────────────────────────────────────┘
```

*⚠️ Attention à la profondeur* : Ne jamais dépasser 2 niveaux d'imbrication (`/posts/108/comments/5`). Si c'est plus profond, ré-aplatir la route (ex: `/api/v1/comments/5`).

---

## 3. Matrice Générale des Opérations CRUD en REST

Voici l'antisèche complète des méthodes, routes et codes HTTP de retour :

| Opération CRUD | Méthode HTTP | Exemple d'Endpoint | Body de Requête | Code HTTP de Succès |
| :--- | :--- | :--- | :---: | :--- |
| **Lire la liste** | `GET` | `/api/v1/products` | Non | `200 OK` (avec tableau JSON) |
| **Lire un élément** | `GET` | `/api/v1/products/42` | Non | `200 OK` (avec objet JSON) |
| **Créer** | `POST` | `/api/v1/products` | **OUI** (JSON) | `201 Created` (avec la ressource créée) |
| **Remplacement total** | `PUT` | `/api/v1/products/42` | **OUI** (JSON) | `200 OK` |
| **Mise à jour partielle**| `PATCH` | `/api/v1/products/42` | **OUI** (JSON) | `200 OK` |
| **Supprimer** | `DELETE` | `/api/v1/products/42` | Non | `204 No Content` (body vide) |

---

## 4. Filtrage, Tri et Pagination (Query Parameters)

On ne renvoie JAMAIS 100 000 lignes de base de données d'un coup ! On passe par les **Query Parameters** dans l'URL.

```
                  REQUÊTE DE RECHERCHE AVANCÉE EN REST
┌─────────────────────────────────────────────────────────────────────────┐
│ GET /api/v1/products?category=electronics&sort=-price&page=2&limit=20  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
      ┌────────────────────────────┼────────────────────────────┐
      ▼                            ▼                            ▼
  FILTRAGE                      TRIER                      PAGINATION
category=electronics      sort=-price (- = décroissant)    page=2 & limit=20
```

### Format type d'une réponse paginée :

```json
{
  "status": "success",
  "results": 20,
  "pagination": {
    "current_page": 2,
    "per_page": 20,
    "total_items": 142,
    "total_pages": 8
  },
  "data": [
    { "id": 21, "name": "Clavier Mécanique", "price": 89.99 }
  ]
}
```

---

## 5. Gestion des Erreurs et Format Standardisé

Une API professionnelle doit renvoyer un format d'erreur prévisible et lisible par la machine comme par l'humain.

```
┌────────────────────────────────────────────────────────┐
│ ERREUR TYPIQUE D'UNE API REST (HTTP 400 BAD REQUEST)   │
├────────────────────────────────────────────────────────┤
│ {                                                      │
│   "error": {                                           │
│     "code": "INVALID_INPUT",                           │
│     "message": "La validation des données a échoué",   │
│     "details": [                                       │
│       {                                                │
│         "field": "email",                              │
│         "issue": "L'adresse email n'est pas valide"    │
│       }                                                │
│     ]                                                  │
│   }                                                    │
│ }                                                      │
└────────────────────────────────────────────────────────┘
```

* **Les Règles d'Erreurs REST** :
  * Toujours utiliser les vrais codes de statut HTTP (`400`, `401`, `403`, `404`, `422`, `500`).
  * Ne JAMAIS renvoyer un `200 OK` avec un corps `{ "error": "Problème" }` (anti-pattern absolu !).

---

## 6. Versioning de l'API (Éviter de casser le Front)

Quand l'API évolue, on doit pouvoir mettre à jour le backend sans faire planter l'application mobile ou le front-end existant.

```
┌─────────────────────────────────────────────────────────────────────────┐
│ METHODE RECOMMANDÉE : URI VERSIONING                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ • Ancienne version : GET /api/v1/users (Toujours maintenue)             │
│ • Nouvelle version : GET /api/v2/users (Nouvelle structure de données)  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Résumé des Bonnes Pratiques en Fiche Réflexe

```
  ✔ REST = Ressources (Noms au pluriel) + Verbes HTTP (Actions)
  ✔ Utiliser des codes HTTP appropriés (200, 201, 204, 400, 401, 403, 404, 500)
  ✔ Toujours paginer et filtrer via des Query Parameters
  ✔ Garantir un format JSON d'erreur clair et constant
  ✔ Versionner les routes d'API dès le départ (/api/v1/)
```
