# 12. Maîtriser les Bases de Données avec Postgres

Voici mes notes de cours rédigées chapitre par chapitre en suivant fidèlement la vidéo de Sriniously sur **Postgres et les bases de données backend**.

---

## 1. Pourquoi avons-nous besoin d'une Base de Données ?

*   **Le concept de Persistance** : Stocker des données de manière à ce qu'elles survivent même après l'arrêt ou le redémarrage du programme qui les a créées.
*   **Disque Dur (Stockage Secondaire) vs RAM (Mémoire Principale)** :
    *   *RAM* : Ultra-rapide mais très chère, limitée en capacité (8-32 Go) et volatile (s'efface à l'extinction). Utilisée pour les caches (ex: Redis).
    *   *Disque (HDD / SSD)* : Plus lent que la RAM, mais très peu cher avec une énorme capacité (512 Go - 2 To). C'est là que les bases de données traditionnelles enregistrent les données de manière permanente.

---

## 2. Qu'est-ce qu'un DBMS (Database Management System) ?

Un DBMS est un logiciel dont le rôle est d'organiser le stockage sur disque et de fournir un accès efficace aux opérations CRUD (Create, Read, Update, Delete).

### Les 4 Responsabilités d'un DBMS :
1.  **Organisation des données** : Structuration sur disque pour des recherches rapides.
2.  **Accès aux données** : Fournir une interface pour exécuter des requêtes.
3.  **Intégrité des données** : Garantir que la donnée est exacte, valide et non corrompue (ex: interdire d'insérer du texte dans un champ de prix numérique).
4.  **Sécurité** : Contrôle des accès et gestion des utilisateurs/rôles.

### Pourquoi ne pas utiliser de simples fichiers texte (.txt / .json) ?
*   **Parsing lourd & lent** : Lire un fichier texte ligne par ligne en code applicatif consomme énormément de CPU/RAM.
*   **Absence de structure et de contraintes** : Impossible de garantir le type ou la validité des données.
*   **Concurrence (Race Conditions)** : Si deux utilisateurs modifient le même fichier texte en même temps, les données sont écrasées ou corrompues.

---

## 3. DBMS Relationnel (SQL) vs Non-Relationnel (NoSQL)

```
┌───────────────────────────────────────────┬───────────────────────────────────────────┐
│        RELATIONNEL (ex: Postgres)         │       NON-RELATIONNEL (ex: MongoDB)       │
├───────────────────────────────────────────┼───────────────────────────────────────────┤
│ • Structure en Tables, Lignes et Colonnes │ • Structure en Collections et Documents   │
│ • Schéma STRICT défini à l'avance         │ • Schéma FLEXIBLE (Dynamic Schema)        │
│ • Forte intégrité des données (ACID)      │ • Évolution rapide / Prototypage          │
│ • Cas d'usage : CRM, E-commerce, Banque   │ • Cas d'usage : CMS, Blogs, Contenu varié │
└───────────────────────────────────────────┴───────────────────────────────────────────┘
```

---

## 4. Pourquoi choisir PostgreSQL ?

1.  **Open Source & Gratuit** : Pas de frais de licence, hébergeable partout.
2.  **Respect du Standard SQL** : Très facile de migrer vers ou depuis d'autres bases SQL.
3.  **Extensible & Fiable** : Plus de 1400 pages de documentation et des décennies de stabilité en production.
4.  **Support Natif d'Exception pour le JSON (JSONB)** : Permet de stocker des données dynamiques comme dans du NoSQL sans quitter Postgres !

---

## 5. Tour d'Horizon des Types de Données dans Postgres

*   **Identifiants & Entiers** :
    *   `SERIAL` / `BIGSERIAL` : Entier auto-incrémenté automatique (1, 2, 3...).
    *   `SMALLINT` < `INTEGER` < `BIGINT` : Selon la taille maximale requise.
    *   `UUID` : Identifiant unique universel sur 128 bits (recommandé en production).
*   **Nombres Décimaux & Flottants** :
    *   `DECIMAL(10, 2)` / `NUMERIC` : **Précision exacte**. Indispensable pour l'argent/prix (évite les erreurs d'arrondi).
    *   `REAL` / `DOUBLE PRECISION` / `FLOAT` : Nombres à virgule flottante. Ultra-rapides en calcul mais avec de légères imprécisions.
*   **Chaînes de Caractères (Strings)** :
    *   `CHAR(n)` : Rempli avec des espaces si le texte est plus court (obsolète).
    *   `VARCHAR(n)` : Longueur variable avec une limite max $n$.
    *   `TEXT` : **Recommandation officielle de Postgres** ! Aucune limite de taille arbitraire et aucune perte de performance par rapport à Varchar.
*   **Dates & Heures** :
    *   `DATE` (Jour/Mois/Année), `TIME` (Heure/Min/Sec).
    *   `TIMESTAMP WITH TIME ZONE` (`TIMESTAMPTZ`) : Stocke la date, l'heure ET le fuseau horaire.
*   **JSON** :
    *   `JSON` : Stocke le texte JSON brut.
    *   `JSONB` : Stocke le JSON sous format binaire sérialisé (beaucoup plus rapide à requêter et indexer).

---

## 6. Le Workflow des Migrations de Base de Données

On n'exécute jamais de requêtes de structure à la main en production. On utilise un outil de migration (ex: `dbmate`, `go-migrate`).

*   **Fichiers horodatés** : Ex: `20261031_create_users_table.sql`.
*   **Up Migration** : Contient les instructions SQL d'application (créer des tables, ajouter des colonnes).
*   **Down Migration** : Contient les instructions d'annulation / rollback (ex: `DROP TABLE`) si la mise en production échoue.
*   **Table `schema_migrations`** : Gérée automatiquement par l'outil pour suivre la version courante de la BDD.

---

## 7. Modélisation Pratique : SaaS de Gestion de Projet

### 7.1. Les Enums Custom (Types Énumérés)
Permettent de restreindre les valeurs autorisées directement au niveau BDD (doc & intégrité) :
*   `project_status` : `'active'`, `'completed'`, `'archived'`
*   `task_status` : `'pending'`, `'in_progress'`, `'completed'`, `'cancelled'`
*   `member_role` : `'owner'`, `'admin'`, `'member'`

### 7.2. Les Tables et leurs Relations

```
┌─────────────────┐       1:1       ┌─────────────────────┐
│      USERS      ├─────────────────►    USER_PROFILES    │
│  (id, email...) │ (user_id PK/FK) │ (bio, avatar_url...)│
└────────┬────────┘                 └─────────────────────┘
         │
         │ 1:N (Owner)
         ▼
┌─────────────────┐       1:N       ┌─────────────────────┐
│    PROJECTS     ├─────────────────►        TASKS        │
│  (id, name...)  │  (project_id)   │(priority CHECK 1..5)│
└────────┬────────┘                 └─────────────────────┘
         │
         │ N:M (Table de liaison)
         ▼
┌───────────────────────────────────┐
│          PROJECT_MEMBERS          │
│ (project_id, user_id) Composite PK│
└───────────────────────────────────┘
```

1.  **Relation 1:1 (Users $\rightarrow$ User Profiles)** : La clé primaire de `user_profiles` est aussi sa clé étrangère (`user_id`). Permet d'alléger la table principale `users`.
2.  **Relation 1:N (Projects $\rightarrow$ Tasks)** : `tasks` contient la clé étrangère `project_id`.
3.  **Contrainte `CHECK` (ex: Priority)** : `CHECK (priority >= 1 AND priority <= 5)` empêche d'insérer une priorité invalide.
4.  **Intégrité Référentielle (ON DELETE)** :
    *   `ON DELETE RESTRICT` : Empêche la suppression d'un utilisateur s'il possède encore des projets.
    *   `ON DELETE CASCADE` : Supprime automatiquement toutes les tâches liées quand un projet est supprimé.
    *   `ON DELETE SET NULL` : Met le champ `assigned_to` à `NULL` si l'utilisateur assigné est supprimé.
5.  **Relation N:M (Projects $\leftrightarrow$ Users)** : Utilisation d'une table de liaison `project_members` avec une **Clé Primaire Composée** `(project_id, user_id)` pour empêcher qu'un membre soit ajouté deux fois au même projet.

---

## 8. Requêtes Paramétrées (Parameterized Queries) & Sécurité

Pour exécuter une requête dynamic sans risque d'**Injection SQL** :
*   On laisse un slot de paramètre (ex: `$1` ou `?`).
*   Le driver BDD envoie la valeur sous forme de simple chaîne échappée, empêchant tout code SQL malveillant d'être exécuté.

---

## 9. Notions Avancées : Triggers & Indexation B-Tree

### 9.1. Les Triggers (Automatisations BDD)
Pour éviter de mettre à jour manuellement la colonne `updated_at` dans le code applicatif à chaque modification, on crée :
1.  Une fonction PL/pgSQL qui fixe `NEW.updated_at = CURRENT_TIMESTAMP`.
2.  Un **Trigger** configuré `BEFORE UPDATE` sur chaque table.

### 9.2. Les Index de Base de Données (B-Tree)

*   **Le problème du Sequential Scan** : Sans index, pour trouver une ligne parmi 1 million, Postgres doit lire tout le disque ligne par ligne.
*   **Le fonctionnement de l'Index** : C'est une table de correspondance séparée (ordonnée sous forme d'arbre B-Tree) qui associe la valeur d'une colonne à son emplacement physique précis sur le disque.
*   **Quand créer un Index ?**
    *   Sur les colonnes fréquemment utilisées dans un **`WHERE`**.
    *   Sur les clés étrangères utilisées dans un **`JOIN`**.
    *   Sur les colonnes utilisées dans un **`ORDER BY`** (ex: `created_at DESC`).
*   ⚠️ **Le compromis de l'Index** : Les index accélèrent la LECTURE (`SELECT`), mais ralentissent légèrement l'ÉCRITURE (`INSERT`, `UPDATE`, `DELETE`) car chaque index doit être réactualisé à chaque modification.