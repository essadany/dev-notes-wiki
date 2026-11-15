# 23. Object Storage & Gestion des Fichiers en Backend (Partie 1)

Quand on construit un système backend, on finit toujours par tomber sur une fonctionnalité incontournable : **permettre aux utilisateurs d'uploader des fichiers** (photos de profil, pièces jointes, documents PDF, vidéos HD). Mais la manière dont on gère ces fichiers en production est à des années-lumière de ce qu'on imagine au début.

---

## 🚫 1. L'Approche Naïve du Développeur Junior & Ses 6 Catastrophes

Au premier abord, quand un besoin d'upload arrive (ex: mettre à jour sa photo de profil), l'intuition naïve d'un développeur junior ressemble à ceci :

```
┌─────────────────┐       1. POST Form-Data (File)      ┌───────────────────────────┐
│                 ├────────────────────────────────────►│                           │
│ Browser Client  │                                     │  Serveur Backend (Node/Go)│
│                 │◄────────────────────────────────────┤                           │
└─────────────────┘        6. GET /file (Render)        └─────────────┬─────────────┘
                                                                      │
                                                   2. Save file       │ 3. Save path
                                                      to disk         │    in DB
                                                                      ▼    ▼
                                                       ┌──────────────────┐ ┌───────────────┐
                                                       │  Système de      │ │ Base Postgres │
                                                       │  Fichiers Local  │ │ (chemins FS)  │
                                                       │  (ext4 / Linux)  │ └───────────────┘
                                                       └──────────────────┘
```

Cette approche fonctionne sur notre machine locale en développement avec un fichier de 50 Ko. Mais en production, elle **explose instantanément** pour 6 raisons fondamentales :

### ❌ Problème 1 : L'Épuisement Mémoire (Out of Memory - OOM)
Si un utilisateur uploade une vidéo de 4 Go et que notre framework web (Express, Go, Fastify) lit tout le fichier en RAM avant de nous le donner, un serveur avec 2 Go ou 4 Go de mémoire **crash sur le coup**. Le conteneur se fait tuer par l'OS (`OOMKilled`).

### ❌ Problème 2 : L'Éphémérité du Système de Fichiers (Ephemeral Storage)
En production moderne (Docker, Kubernetes, AWS ECS, Railway, Render, Vercel), les conteneurs sont **éphémères**. À chaque redéploiement de code ou crash d'instance, le conteneur est détruit et remplacé par un nouveau. **Tous les fichiers stockés sur le disque local sont définitivement effacés !**

### ❌ Problème 3 : L'Échec du Scaling Horizontal (Statelessness brisée)
Si nous avons 3 instances de serveurs derrière un Load Balancer (Round-Robin) :
* L'utilisateur uploade sa photo sur l'**Instance 1**.
* 2 minutes plus tard, il rafraîchit la page. Le Load Balancer l'envoie sur l'**Instance 3**.
* L'Instance 3 n'a pas le fichier sur son disque local $
ightarrow$ **Erreur 404 Not Found**. La probabilité de servir la photo tombe à $1/N$ (33% d'réussite pour 3 serveurs).

```
                        ┌──────────────────┐
                        │  Load Balancer   │
                        └────────┬─────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │ (Req 1: Upload)     │ (Req 2: Fetch)      │
           ▼                     ▼                     ▼
  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
  │   Instance 1    │   │   Instance 2    │   │   Instance 3    │
  │ 📁 photo.jpg    │   │ ❌ Pas de fichier│   │ ❌ 404 Not Found│
  └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### ❌ Problème 4 : Capacité Fixe du Disque & Risque de Durabilité
Grandir un disque dur est une opération **verticale** lourde (downtime, limites de taille du cloud provider). De plus, si l'unique disque dur physique de la machine flanche, **toutes les données clients sont perdues à jamais**.
* **Disponibilité (Availability)** : Capacité à joindre la donnée tout de suite (on peut attendre que le serveur redémarre).
* **Durabilité (Durability)** : La donnée existe-t-elle encore dans le monde ? Si le disque meurt sans réplication, la durabilité tombe à zéro.

### ❌ Problème 5 : Serveur Backend Bloqué par l'I/O (Pas un CDN !)
Si un client mobile avec un réseau 3G lent (2 Mbps) télécharge un fichier de 50 Mo, la connexion va durer **3,5 minutes**. Pendant 3,5 minutes, un worker backend (Goroutine ou boucle d'événements) est **bloqué à copier des octets**, au lieu d'exécuter de la logique métier. Notre serveur devient un très mauvais CDN très cher.

### ❌ Problème 6 : Absence de Transactions entre Fichiers et Base SQL
Il n'existe aucune transaction atomique entre le système de fichiers local et Postgres :
* Si on enregistre sur le disque d'abord et que l'insertion SQL échoue $
ightarrow$ **Fichier orphelin** qui coûte de l'argent.
* Si on insère en BDD d'abord et que l'écriture disque échoue $
ightarrow$ **Ligne BDD pointant vers un fichier inexistant** (Crash 500 / 404).

---

## 🗄️ 2. Les 3 Types de Stockage & Leurs Compromis

Pour comprendre où se situe l'Object Storage, comparons les trois grandes familles de stockage informatique :

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 COMPARAISON DES STOCKAGES                              │
├──────────────────────┬──────────────────────────┬──────────────────────────────────────┤
│ Type de Stockage     │ Interface & Abstraction  │ Latence & Cas d'usage idéal          │
├──────────────────────┼──────────────────────────┼──────────────────────────────────────┤
│ 1. Block Storage     │ Tableau brut de blocs    │ ⚡ Microsecondes (µs)                │
│    (ex: AWS EBS, SSD)│ fixes (512 B - 4 KB).    │ 🎯 Bases de données (Postgres, MySQL).│
│                      │ Pas de fichiers/dossiers.│ ⚠️ 1 seule machine attachée à la fois.│
├──────────────────────┼──────────────────────────┼──────────────────────────────────────┤
│ 2. File Storage      │ Hiérarchie POSIX riche   │ 🚀 Microsecondes à Millisecondes (ms)│
│    (ex: ext4, NFS)   │ (dossiers, atomicité,    │ 🎯 OS local, fichiers d'édition.      │
│                      │ verrous de fichiers).    │ ⚠️ Coordination réseau très lourde.  │
├──────────────────────┼──────────────────────────┼──────────────────────────────────────┤
│ 3. Object Storage    │ Interface minimale (4 ops│ 🌐 Millisecondes (ms) via HTTP        │
│    (ex: AWS S3, R2)  │ PUT, GET, DELETE, LIST). │ 🎯 Fichiers utilisateurs, assets,    │
│                      │ Immuable & Clé-Valeur.   │    Scalabilité à l'échelle de la terre│
└──────────────────────┴──────────────────────────┴──────────────────────────────────────┘
```

> 💡 **Le Compromis Fondamental du Distributed System** :
> Pour atteindre une **scalabilité infinie (Électronique / Pétabytes)** sans goulot d'étranglement de coordination, l'Object Storage supprime **l'immuabilité**. On ne peut pas modifier 1 octet au milieu d'un objet de 1 Go : il faut tout re-télécharger et tout re-uploader !

---

## 🔍 3. Anatomie d'un Objet & La Grande Illusion des Dossiers

### Qu'est-ce qu'un Objet ?
Un objet stocké dans l'Object Storage se compose de 4 éléments :
1. **Key (Clé)** : Chaîne de caractères unique servant d'identité complète.
2. **Value (Valeur)** : Le blob binaire d'octets de données.
3. **System Metadata** : Taille, date de modification, `Content-Type`, `ETag` (hash du contenu).
4. **User Metadata** : Clés/Valeurs personnalisées (ex: `uploaded_by: "user_123"`).

Tous les objets résident dans un **Bucket** (un conteneur globalement unique au niveau du Cloud Provider).

### 🚨 L'Illusion des Dossiers (Folders are a Lie !)
Dans AWS S3 ou Cloudflare R2, **les répertoires/dossiers n'existent pas** !

Quand on voit dans la console Web : `uploads/2026/cat.png`, il n'y a pas de sous-dossier `uploads` ni `2026`. Il y a juste **une seule chaîne de caractères plate** contenant des caractères `/`.

```
              ┌─────────────────────────────────────────────────────────┐
              │                LA VÉRITÉ SUR L'INDEX S3                 │
              ├─────────────────────────────────────────────────────────┤
              │ Clé (Key Flat String)           │ Blob Data             │
              ├─────────────────────────────────┼───────────────────────┤
              │ "uploads/2026/cat.png"          │ [Bytes binary...]     │
              │ "uploads/2026/dog.png"          │ [Bytes binary...]     │
              │ "avatar_user_99.png"            │ [Bytes binary...]     │
              └─────────────────────────────────┴───────────────────────┘
```

La console Web invente les "dossiers" grâce à un **algorithme de groupement par préfixe** (`delimiter='/'`).

* **Conséquence 1** : Déplacer un dossier contenant 50 000 images ne prend pas 1 ms. Cela déclenche **50 000 opérations de copie de clés + 50 000 suppressions** (très long et très cher !).
* **Conséquence 2** : Un dossier vide ne peut pas exister naturellement (sauf si un outil crée un objet factice de 0 octet finissant par `/`).

---

## 🔑 4. Les 3 Règles d'Or pour le Design des Clés (Keys)

Puisque la **Clé (Key)** détermine tout, sa conception est stratégique :

### 1️⃣ Ne JAMAIS utiliser le nom de fichier d'origine fourni par l'utilisateur
* **Collisions** : Deux utilisateurs uploadent `CV.pdf` $
ightarrow$ Le second écrase silencieusement le premier (car l'opération S3 est un `PUT` !).
* **Path Traversal** : Risques d'injections avec des `../` dans le nom.
* **Caractères spéciaux** : Emojis, null bytes ou caractères RTL qui cassent les scripts backend.
* **Solution** : Générer un UUID unique (ex: `550e8400-e29b...png`) et stocker le vrai nom d'origine dans Postgres ou en User Metadata.

### 2️⃣ Placer l'aléatoire (Haute Entropie) en début de Clé
Si toutes vos clés commencent par la date `2026-11-15/10:00/file1`, tous les uploads de la même heure frappent la **même partition de l'index S3** (Hot Partitioning $
ightarrow$ Erreurs `503 Slow Down`).
* En mettant un préfixe aléatoire ou l'ID utilisateur au début (`user_883/tenant_12/uuid`), les clés sont dispersées uniformément sur les partitions d'index.

### 3️⃣ Porter la propriété (Ownership) dans la Clé
Si la clé est structurée sous la forme `tenant_id/user_id/object_id`, votre middleware d'autorisation peut vérifier les droits avec une simple comparaison de chaînes de caractères, **sans interroger la BDD Postgres à chaque lecture !**

---

## ⚙️ 5. Sous le Capot : Les 2 Planes de l'Architecture S3

L'architecture S3 sépare deux mondes complètement distincts :

```
                                  ┌──────────────────────────┐
                                  │   Requête HTTP Upload    │
                                  └─────────────┬────────────┘
                                                │
                                                ▼
                                  ┌──────────────────────────┐
                                  │   Front-end Stateless    │
                                  └─────────────┬────────────┘
                                                │
              ┌─────────────────────────────────┴─────────────────────────────────┐
              ▼                                                                   ▼
┌───────────────────────────┐                                       ┌───────────────────────────┐
│       DATA PLANE          │                                       │      METADATA PLANE       │
├───────────────────────────┤                                       ├───────────────────────────┤
│ • Gère les volumes d'octets│                                       │ • Index B-Tree distribué. │
│ • Erasure Coding (Reed-   │                                       │ • Associe Clé ➔ Shards.   │
│   Solomon : 14 data + 6   │                                       │ • Strong Consistency      │
│   parity = 20 shards).    │                                       │   (Read-After-Write).     │
│ • Surcoût: 1.43x (vs 3x). │                                       │ • Règle : Ne JAMAIS faire │
│ • Durabilité : 11 Nines.  │                                       │   de LIST dans l'API !    │
└───────────────────────────┘                                       └───────────────────────────┘
```

### Erasure Coding & 11 Nines de Durabilité
Au lieu de tripler les fichiers (3 copies = 200% de surcoût), S3 utilise l'**Erasure Coding** (Algorithme Reed-Solomon) :
* Un objet est découpé en 14 shards de données + 6 shards de parité (total 20 shards répartis dans 20 racks/bâtiments différents).
* **N'importe quelles 14 parties sur 20** permettent de reconstruire l'objet intégralement !
* **Surcoût de stockage** : Seulement **1.43x** (gain financier massif tout en résistant à 6 pannes simultanées de disques).
* **99.999999999% (11 Nines)** : Statistiquement, perdre 1 objet sur 10 millions arrive 1 fois tous les 10 000 ans.

> ⚠️ **Durabilité $
eq$ Sauvegarde (Backup)** :
> Si votre code exécute un `DELETE` accidentel sur un préfixe, S3 va supprimer très durablement et très fidèlement vos fichiers ! Activez le **Versioning** et la **réplication inter-buckets** sur des comptes isolés pour parer aux erreurs humaines.

---

## 🔒 6. Nouveauté Critique : Écritures Conditionnelles (Conditional Writes)

Ajouté fin 2024 dans AWS S3, le support des écritures conditionnelles change la donne pour les systèmes distribués :
* `If-None-Match: *` : L'upload n'aboutit que si la clé **n'existe pas déjà**. Si elle existe, S3 renvoie une erreur `412 Precondition Failed`. Cela empêche les écrasements silencieux d'objets.
* `If-Match: "ETag"` : Remplacer l'objet uniquement si son ETag correspond à celui lu (implémentation d'un verrou **Compare-And-Swap** distribué).

---

## 🚀 7. L'Architecture d'Upload Production-Grade : Pre-signed URLs

### Pourquoi l'Upload via Serveur Relay est une mauvaise idée ?
Faire passer le fichier par notre serveur Node/Go (même en streaming) gâche la bande passante du serveur, risque des timeouts de proxy (Nginx `413 Request Entity Too Large`, ALB 60s timeout) et bloque le CPU.

### La Solution Élégante : Pre-signed POST avec Policy JSON

Au lieu de faire relai, notre serveur génère une **URL pré-signée** (avec HMAC temporaire validé par la clé secrète). Le navigateur uploade **DIRECTEMENT vers AWS S3** sans jamais toucher la mémoire de notre serveur backend !

```
┌──────────┐    1. POST /api/upload-intent (filename, size)   ┌──────────────┐
│          ├─────────────────────────────────────────────────►│              │
│          │    2. Retourne Pre-signed POST URL + Policy JSON │  Backend API │
│          │◄─────────────────────────────────────────────────┤  (Node / Go) │
│  Client  │                                                  └──────────────┘
│ Browser  │
│          │    3. PUT / POST Direct File Bytes                ┌──────────────┐
│          ├─────────────────────────────────────────────────►│   AWS S3     │
│          │◄─────────────────────────────────────────────────┤   Bucket     │
└──────────┘    4. HTTP 200 OK (Upload terminé)               └──────────────┘
```

#### Sécuriser avec une Policy JSON (Taille & Type)
Pour éviter qu'un utilisateur n'uploade un fichier de 10 Go via un lien prévu pour une photo de profil, on signe une **Policy JSON** imposant des contraintes strictes vérifiées directement par S3 :

```json
{
  "expiration": "2026-11-15T12:00:00Z",
  "conditions": [
    ["content-length-range", 1024, 5242880],
    ["starts-with", "$Content-Type", "image/"],
    ["starts-with", "$key", "tenants/t1/users/u42/"]
  ]
}
```
Si le navigateur tente d'uploader 1 Go ou un fichier `.exe`, **AWS S3 rejette la requête immédiatement avec une erreur 400 Bad Request !**

---

## 🔄 8. Le Workflow à 2 Phases pour la Synchronisation BDD / S3

Pour garantir que la BDD Postgres et S3 restent synchronisés sans état incohérent :

```
                  PHASE 1 : INTENTION D'UPLOAD
┌────────┐  1. Demande upload  ┌─────────┐  2. Insert status='pending'  ┌──────────┐
│ Client ├────────────────────►│ Backend ├─────────────────────────────►│ Postgres │
│        │◄────────────────────┤         │                              └──────────┘
└────────┘  3. Pre-signed POST └─────────┘

                  PHASE 2 : UPLOAD DIRECT & CONFIRMATION
┌────────┐  4. Upload direct octets  ┌────────┐
│ Client ├──────────────────────────►│ AWS S3 │
│        │                           └───┬────┘
│        │  5. POST /complete (id)   ┌───┴─────┐  6. HEAD Object (check) ┌────────┐
│        ├──────────────────────────►│ Backend ├────────────────────────►│ AWS S3 │
│        │                           │         │                         └────────┘
│        │                           │         │  7. Update status='ready'┌──────────┐
│        │◄──────────────────────────┤         ├────────────────────────►│ Postgres │
└────────┘  8. 200 OK (Profil à jour)└─────────┘                          └──────────┘
```

### Nettoyage Automatique des Uploads Abandonnés
Si l'utilisateur ferme l'onglet pendant l'upload :
1. **Règle de Lifecycle S3** : Supprime automatiquement tout objet resté sous le préfixe `pending/` après 24 heures.
2. **Background Job (Cron)** : Passe toutes les lignes BDD restées à `pending` depuis plus de 24h au statut `expired`.

---

🎓 **En résumé** : L'Object Storage est un pilier de l'ingénierie backend moderne. En déléguant le stockage à S3/R2 et en utilisant les **Pre-signed POST URLs avec Policy**, notre backend reste **100% Stateless, ultra-rapide, léger en RAM** et immunisé contre les crashs de stockage !