# 🗄️ 12. Maîtriser les Bases de Données avec Postgres

Aujourd'hui, on entre dans le cœur dur du backend : **les bases de données**, et plus particulièrement **PostgreSQL (Postgres)** ! Une API REST sans base de données, c'est comme une voiture sans réservoir : ça ne conserve rien. Voici mes notes personnelles rédigées avec mes propres mots, des schémas visuels et des antisèches pour être au niveau en dev !

---

## 1. ⚔️ SQL (Relationnel) vs NoSQL (Non-Relationnel)

Avant de choisir un SGBD (Système de Gestion de Base de Données), il faut comprendre la grande fracture du stockage :

```
                        ┌─────────────────────────────────────────┐
                        │      CHOIX DE LA BASE DE DONNÉES        │
                        └────────────────────┬────────────────────┘
                                             │
               ┌─────────────────────────────┴─────────────────────────────┐
               ▼                                                           ▼
┌──────────────────────────────┐                         ┌──────────────────────────────┐
│       SQL (RELATIONNEL)      │                         │     NoSQL (DOCUMENT / KEY)   │
├──────────────────────────────┤                         ├──────────────────────────────┤
│ • Exemple : PostgreSQL, MySQL│                         │ • Exemple : MongoDB, Redis   │
│ • Structure : Tables strictes│                         │ • Structure : Documents JSON │
│   avec lignes et colonnes    │                         │   ou paires Clé-Valeur       │
│ • Schéma : Fixe et prévisib. │                         │ • Schéma : Flexible/Dynamique│
│ • Garantie : ACID stricte    │                         │ • Garantie : BASE/Éventuelle │
│ • Cas idéal : Données liées  │                         │ • Cas idéal : Données en vrac│
│   (Users, Orders, Payments)  │                         │   (Logs, Cache, Analytics)   │
└──────────────────────────────┘                         └──────────────────────────────┘
```

> 💡 **Mon aide-mémoire de Dev** : Pour 90% des projets web modernes (SaaS, E-commerce, Réseaux sociaux), **PostgreSQL est le choix par défaut**. Les relations entre utilisateurs, projets et paiements nécessitent la cohérence stricte du SQL.

---

## 2. 🏛️ Anatomie d'une Base Postgres : Schémas & Relations

Dans Postgres, les données sont organisées sous forme de **Tables** reliées entre elles par des **Clés**.

```
┌────────────────────────────────────────────────────────────────────────┐
│ TABLE: users                                                           │
├────────────┬───────────────────┬───────────────────┬───────────────────┤
│ id (PK)    │ email             │ password_hash     │ created_at        │
├────────────┼───────────────────┼───────────────────┼───────────────────┤
│ 101        │ alex@dev.io       │ $2b$12$...        │ 2026-10-31        │
└────────────┴─────────┬─────────┴───────────────────┴───────────────────┘
                       │
                       │ 🔗 Relation 1-to-N (Un utilisateur a plusieurs projets)
                       ▼
┌────────────────────────────────────────────────────────────────────────┐
│ TABLE: projects                                                        │
├────────────┬───────────────────┬───────────────────┬───────────────────┤
│ id (PK)    │ title             │ user_id (FK)      │ status            │
├────────────┼───────────────────┼───────────────────┼───────────────────┤
│ 1          │ API Redesign      │ 101               │ ACTIVE            │
│ 2          │ Mobile App        │ 101               │ PLANNED           │
└────────────┴───────────────────┴───────────────────┴───────────────────┘
```

### 🗝️ Les Clés et Contraintes Essentielles :
* **Primary Key (PK)** : Identifiant unique de la ligne (souvent un `BIGINT` auto-incrémenté ou un `UUID`).
* **Foreign Key (FK)** : Lien pointant vers la `PK` d'une autre table pour garantir l'intégrité référentielle.
* **Contraintes d'intégrité** :
  * `NOT NULL` : Empêche d'enregistrer une donnée vide.
  * `UNIQUE` : Garantit qu'aucun doublon n'existe (ex: adresse email).
  * `CHECK` : Valide une condition (ex: `age >= 18` ou `price > 0`).

---

## 3. 🛡️ Les Garanties ACID (Le Bouclier de Postgres)

Pour éviter que des transactions simultanées ne corrompent les données, Postgres respecte à la lettre les 4 principes **ACID** :

| Principe | Signification | Ce que ça garantit en pratique |
| :--- | :--- | :--- |
| **A** - Atomicity | Atomicité | Tout passe ou tout s'annule (Pas de demi-transaction). |
| **C** - Consistency | Cohérence | La base respecte toujours les règles et contraintes de schéma. |
| **I** - Isolation | Isolation | Deux transactions simultanées ne se marchent pas dessus. |
| **D** - Durability | Durabilité | Une fois la transaction validée (`COMMIT`), la donnée est sauvée sur disque. |

```
                    EXEMPLE DE TRANSACTION BANCAIRE (BEGIN -> COMMIT)

 [ Client A ] ───>  1. DÉBIT 100€ (Compte A)  ─────┐
                                                   ├─> [ TRANSACTION ACID ]
 [ Client B ] ───>  2. CRÉDIT 100€ (Compte B) ─────┘     (Si le serveur plante au milieu,
                                                         TOUT est annulé via ROLLBACK !)
```

---

## 4. ⚡ Indexation : Accélérer les Lectures sans Ruiner les Écritures

Un **Index** (souvent sous forme d'arbre B-Tree) fonctionne exactement comme l'index à la fin d'un livre d'apprentissage.

```
       RECHERCHE SANS INDEX (Sequential Scan)           RECHERCHE AVEC INDEX (Index Scan)
┌──────────────────────────────────────────────┐ ┌──────────────────────────────────────────────┐
│ Le serveur lit TOUTES les 1 000 000 lignes   │ │ Le serveur consulte l'arbre B-Tree en 3 ms  │
│ une par une pour trouver "email=alex@dev.io".│ │ et pointe direct sur le bloc mémoire exact ! │
│ ⏱️ Temps : 450 ms (Lent & Gourmand)          │ │ ⏱️ Temps : 2 ms (Ultra-Rapide)               │
└──────────────────────────────────────────────┘ └──────────────────────────────────────────────┘
```

> ⚠️ **Le Compromis de l'Index** :
> * ➕ **Lecture (`SELECT`)** : Divisée par 100 en vitesse.
> * ➖ **Écriture (`INSERT` / `UPDATE` / `DELETE`)** : Légèrement ralentie car Postgres doit aussi mettre à jour l'index à chaque modification.
> * **Règle d'or** : Mettre un Index (`CREATE INDEX`) uniquement sur les colonnes souvent utilisées dans les clauses `WHERE`, `JOIN` et `ORDER BY` (ex: `user_id`, `email`, `status`).

---

## 5. 🔌 Connection Pooling : Gérer les Milliers de Connexions

Ouvrir une nouvelle connexion réseau vers Postgres coûte cher en CPU et en mémoire (chaque connexion Postgres fork un processus lourd côté serveur).

```
┌──────────┐                     ┌───────────────────────┐                     ┌──────────┐
│  CLIENTS │ ─── Requêtes ───>  │  CONNECTION POOL      │ ─── Connexions ───> │ SERVEUR  │
│  (Node)  │                     │  (ex: pg-pool / PgBouncer)                  │ POSTGRES │
│          │                     │  [ 10 connexions réutilisables en boucle ]  │          │
└──────────┘                     └───────────────────────┘                     └──────────┘
```

* **Sans Pool** : 1 000 requêtes simultanées = 1 000 connexions Postgres = **Crash du serveur (`Too many connections`)**.
* **Avec Pool** : 1 000 requêtes se partagent un sous-ensemble fixe de connexions réutilisables (ex: 10 ou 20 connexions ouvertes en permanence).

---

## 6. 🛠️ Abstractions de Base de Données : Raw SQL, Query Builder & ORM

Comment interagir avec Postgres depuis notre code Backend ? Il existe 3 approches :

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. RAW SQL (Requêtes brutes)                                                           │
│    • ex: client.query('SELECT * FROM users WHERE id = $1', [userId])                   │
│    • Pros : Contrôle total, performances maximales.                                     │
│    • Cons : Pas de typage, risques d'injections SQL si mal écrit.                     │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. QUERY BUILDERS (Constructeurs de requêtes)                                          │
│    • ex: knex('users').where({ id: userId }).select()                                  │
│    • Pros : Syntaxe JS/TS fluide, requêtes dynamiques sécurisées.                     │
│    • Cons : Requêtes très complexes parfois difficiles à exprimer.                      │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. ORM (Object-Relational Mapping)                                                     │
│    • ex: prisma.user.findUnique({ where: { id: userId } })                             │
│    • Pros : Typage TypeScript automatique, migrations faciles, code très lisible.       │
│    • Cons : Légère surcouche d'abstraction, risque de requêtes SQL générées sous-optim.│
└────────────────────────────────────────────────────────────────────────────────────────┘
```

> 🎯 **Conseil pour dev junior** : Utiliser un **ORM moderne comme Prisma ou Drizzle** pour aller vite et sécuriser les types, mais **toujours inspecter le SQL généré** en arrière-plan pour éviter les pièges de requêtes N+1 !
