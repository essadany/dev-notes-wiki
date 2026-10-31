# 11. Design Complet d'une API REST (Complete REST API Design)

Aujourd'hui, gros chapitre sur le **Design d'API REST**. Sriniously insiste : le design de l'interface d'une API se fait **avant d'écrire la moindre ligne de code**. Voici mes notes structurées chapitre par chapitre en suivant exactement le déroulé de la vidéo !

---

## 1. Introduction & Problématique du Design d'API REST

*   **Le problème fréquent** : En début de carrière, on se pose toujours les mêmes questions (Singulier ou pluriel pour les URL ? `PUT` ou `PATCH` pour mettre à jour ? Quel code de statut HTTP renvoyer ? Comment gérer une action personnalisée ?).
*   **L'objectif du cours** : Extraire et appliquer les directives et standards existants pour que l'interface soit **prévisible, cohérente et agréable** pour les développeurs front-end ou autres serveurs qui l'intègrent.
*   **Deux approches web** :
    *   *MPA (Multi-Page Applications)* : L'ancien modèle où le serveur renvoyait du HTML à chaque clic.
    *   *SPA (Single Page Applications)* : Le modèle moderne où le navigateur télécharge le JS au départ et fait des appels API pour échanger des données.

---

## 2. Contexte Historique : Pourquoi REST a été créé ?

*   **1990 (Tim Berners-Lee)** : Création du projet *World Wide Web* pour partager le savoir. En un an, il invente :
    1. `URI` (Uniform Resource Identifier)
    2. `HTTP` (Protocole de communication)
    3. `HTML` (Langage de balisage)
    4. Le premier serveur web
    5. Le premier navigateur web
    6. Le premier éditeur HTML
*   **La crise de scalabilité (1993)** : L'explosion exponentielle du nombre d'utilisateurs a failli faire s'effondrer le web, car l'architecture initiale n'était pas prévue pour cette échelle.
*   **L'intervention de Roy Fielding (1993-2000)** : Co-fondateur du projet Apache HTTP Server, il propose un ensemble de contraintes pour rendre le web extensible à très grande échelle. En 2000, dans sa thèse de doctorat, il nomme ce style d'architecture : **REST (Representational State Transfer)**.

---

## 3. Les 6 Contraintes d'une Architecture RESTful (Roy Fielding)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LES 6 CONTRAINTES DE ROY FIELDING                    │
├───────────────────┬───────────────────┬─────────────────────────────────┤
│ 1. Client-Serveur │ Separation des    │ Le client gère l'UI/UX,         │
│                   │ responsabilités   │ le serveur gère la BDD/métier.  │
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ 2. Uniform        │ Interface         │ Standardisation absolue de la   │
│    Interface      │ unifiée           │ communication entre composants. │
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ 3. Layered System │ Système en        │ Architecture hiérarchique       │
│                   │ couches           │ (Proxies, Load Balancers).      │
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ 4. Cache          │ Gestion du        │ Les réponses doivent se déclarer│
│                   │ cache             │ cachables ou non-cachables.     │
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ 5. Stateless      │ Sans état         │ Chaque requête contient TOUTES  │
│                   │                   │ les infos nécessaires au serveur│
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ 6. Code on Demand │ Code à la demande │ (Optionnel) Le serveur envoie   │
│                   │                   │ du JS exécutable au client.     │
└───────────────────┴───────────────────┴─────────────────────────────────┘
```

---

## 4. Démystification : Que signifie "R.E.S.T" ?

REST veut dire **Representational State Transfer** :

1.  **Representational (Représentation)** : Une même ressource (ex: un utilisateur) peut être représentée dans différents formats selon le client : `JSON` (pour une API / communication inter-serveurs), `HTML` (pour un navigateur), ou `XML`.
2.  **State (État)** : L'état actuel d'une ressource à un instant T (ex: les articles présentés dans un panier d'achat avec leurs quantités et le montant total).
3.  **Transfer (Transfert)** : Le déplacement des représentations d'état entre le client et le serveur via le protocole HTTP (`GET`, `POST`, `PUT`, `DELETE`, etc.).

---

## 5. Anatomie d'une URL & Conventions de Nommage REST

### A. Structure d'une URL classique vs URL d'API
`https://api.example.com/v1/books`
*   **Scheme** : `https://` (Sécurisé/chiffré).
*   **Subdomain / Authority** : `api.example.com` (Pratique courante en industrie d'isoler l'API sur un sous-domaine `api.`).
*   **Versioning** : `/v1/` ou `/v2/` (Pour éviter de casser le front-end lors d'évolutions).
*   **Path Segment / Resource** : `/books` (**Règle d'or : Toujours utiliser des noms au PLURIEL**, même pour récupérer un seul élément comme `/books/123`).

### B. Bonnes pratiques pour les Slugs dans l'URL
*   Pas de majuscules (ex: `/books/harry-potter` et non `/books/HarryPotter`).
*   Remplacer les espaces par des **tirets du haut `-`** (kebab-case) et bannir les tirets bas `_` ou espaces.
*   Le caractère slash `/` exprime une **relation hiérarchique** entre ressources (ex: `/organizations/5/projects` = la collection de projets qui appartiennent à l'organisation 5).

---

## 6. L'Idempotence des Méthodes HTTP

> 💡 **Définition** : Une opération est **idempotente** si le fait de l'exécuter une seule fois ou plusieurs fois produit exactement le même effet et le même état sur le serveur.

| Méthode HTTP | Usage principal | Idempotente ? | Explication concrète |
| :--- | :--- | :---: | :--- |
| `GET` | Récupérer des données | **OUI** | Ne modifie rien sur le serveur. 1000 appels renvoient le même état. |
| `PUT` | Remplacer entièrement une ressource | **OUI** | Remplace tout l'objet. Répéter le même payload laisse l'état identique. |
| `PATCH` | Modifier partiellement une ressource | **OUI** | Met à jour certains champs. Le résultat reste identique après le 1er appel. |
| `DELETE` | Supprimer une ressource | **OUI** | Le 1er appel supprime la ressource. Les appels suivants renvoient `404` mais aucun autre effet de bord n'a lieu. |
| `POST` | Créer une ressource / Action | **NON** | Chaque appel crée une nouvelle ressource en BDD (nouveau ID généré). |

*   **POST comme méthode "Open-Ended" (Actions Personnalisées)** : Quand une opération ne rentre pas dans le CRUD classique (ex: `send-email`, `archive`, `clone`), la spécification REST stipule d'utiliser `POST`.

---

## 7. Démo Pratique sur Insomnia : Modélisation d'un SaaS de Gestion de Projet

Dans la démo, Sriniously modélise les endpoints pour une plateforme style Jira/Linear autour de 3 ressources : `organizations`, `projects`, `tasks`.

### 7.1. Gestion des Organisations (`/organizations`)

1.  **Lister les organisations** (`GET /organizations`) :
    *   Renvoie une réponse paginée avec le code `200 OK`.
    *   **Structure d'une réponse paginée** :
        ```json
        {
          "data": [ { "id": "1", "name": "Org 1", "status": "active" } ],
          "total": 5,
          "page": 1,
          "totalPages": 3
        }
        ```
    *   **Paramètres de pagination** : `?limit=10&page=1`.
    *   **Tri (Sorting)** : `?sortBy=created_at&sortOrder=desc` (Tri par défaut : date de création décroissante).
    *   **Filtrage (Filtering)** : `?status=archived` ou `?name=Org`.
    *   ⚠️ **Règle pour les listes vides** : Si aucun résultat ne correspond aux filtres, renvoyer un tableau vide `[]` avec `200 OK` (et **jamais de 404** ! Le 404 est réservé à la recherche d'un identifiant spécifique inexistant).

2.  **Créer une organisation** (`POST /organizations`) :
    *   Payload JSON en dromadaire (`camelCase`).
    *   Renvoie la ressource créée avec le code **`201 Created`**.

3.  **Récupérer une organisation spécifique** (`GET /organizations/:id`) :
    *   Renvoie `200 OK` avec l'objet, ou **`404 Not Found`** si l'ID n'existe pas.

4.  **Mettre à jour une organisation** (`PATCH /organizations/:id`) :
    *   Renvoie `200 OK` avec l'objet mis à jour.

5.  **Supprimer une organisation** (`DELETE /organizations/:id`) :
    *   Renvoie le code **`204 No Content`** avec un corps de réponse vide.

6.  **Action personnalisée : Archiver** (`POST /organizations/:id/archive`) :
    *   Puisque l'archivage déclenche un effet cascade (notification, suppression des projets sous-jacents, etc.), ce n'est pas un simple PATCH. On utilise `POST` avec le nom de l'action à la fin du chemin hiérarchique, et cela renvoie `200 OK`.

### 7.2. Gestion des Projets (`/projects`)

*   `POST /projects` $\rightarrow$ Création (`201 Created`).
*   `GET /projects` $\rightarrow$ Liste paginée (`200 OK`).
*   `GET /projects/:id` $\rightarrow$ Détail (`200 OK`).
*   `PATCH /projects/:id` $\rightarrow$ Mise à jour partielle (`200 OK`).
*   `DELETE /projects/:id` $\rightarrow$ Suppression (`204 No Content`).
*   `POST /projects/:id/clone` $\rightarrow$ **Action personnalisée de clonage** : Génère un nouveau projet dupliqué sur le serveur et renvoie `201 Created`.

---

## 8. Les 5 Règles d'Or pour devenir un Excellent Ingénieur Backend

1.  **Conception de l'Interface avant le Code** : Utiliser Figma/Wireframes pour identifier les noms (ressources) et actions, puis prototyper les endpoints dans Postman/Insomnia ou Swagger avant d'écrire le code métier.
2.  **Documentation Interactive (OpenAPI / Swagger)** : Maintenir une documentation d'API à jour et testable pour éviter les devinettes.
3.  **Consistance Absolue (Styling & Naming)** : Ne jamais mélanger le camelCase et le snake_case dans les payloads JSON. Utiliser la même convention sur l'ensemble des ressources.
4.  **Sane Defaults (Valeurs par défaut intelligentes)** : Le serveur doit fonctionner même si le client ne fournit pas les paramètres de pagination (ex: par défaut `page=1`, `limit=10`, `status=active`).
5.  **Éviter les Abbréviations** : Écrire `description` et non `desc`, `organization` et non `org` dans les clés JSON pour garantir la lisibilité.