# 13. Le Caching (Le secret des systèmes haute performance)

Si on devait résumer l'impact du caching en une phrase : **c'est LE mécanisme ultime qui permet d'économiser du temps et de l'effort réseau/calcul pour servir des millions d'utilisateurs sans faire exploser les serveurs**. Voici mes notes personnelles rédigées avec mes schémas, mes tableaux et mon modèle mental de dev junior !

---

## 🧭 1. Qu'est-ce que le Caching ? (Définition & Exemples Géants)

### A. Définition Technique
> 💡 **Définition simple** : C'est un mécanisme qui diminue le temps et l'effort nécessaires pour exécuter une opération.
> 
> ⚙️ **Définition technique** : Cela consiste à conserver un **sous-ensemble (subset)** de la donnée primaire dans un emplacement mémoire beaucoup plus rapide à lire (ex: la RAM ou un serveur Edge) pour répondre aux requêtes fréquentes.

### B. 3 Exemples du Monde Réel

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                EXEMPLES DU MONDE RÉEL                                  │
├────────────────────────┬────────────────────────────────┬──────────────────────────────┤
│ Google Search          │ Netflix                        │ X (Twitter)                  │
├────────────────────────┼────────────────────────────────┼──────────────────────────────┤
│ Avoid re-running heavy │ CDN & Edge Locations           │ Trending Topics              │
│ ranking algorithms for │ Streams HD (1080p, 720p)       │ Real-time ML analysis of     │
│ millions of queries    │ served from local edge nodes   │ millions of tweets cached    │
│ like "weather today".  │ to prevent origin crash.       │ in Redis for instant load.   │
└────────────────────────┴────────────────────────────────┴──────────────────────────────┘
```

1. **Google Search** : Recalculer l'indexation et le classement de milliards de pages web pour une requête répétée des millions de fois ("météo aujourd'hui") détruirait les serveurs. Google utilise un **système de cache in-memory distribué** à travers le monde pour renvoyer instantanément le résultat.
2. **Netflix** : Pour livrer des terabytes de données vidéo HD sans Latence/Buffering, Netflix encode ses vidéos en plusieurs résolutions (1080p, 720p, 480p) et **cache ces fichiers sur des serveurs Edge (CDN)** géographiquement proches de l'utilisateur final.
3. **X (ex-Twitter)** : Le calcul des *Trending Topics* nécessite d'analyser en temps réel des milliards de tweets avec des modèles ML lourds. Pour éviter de relancer ce calcul à chaque rafraîchissement d'écran, X calcule les tendances toutes les X minutes et stocke le résultat dans une base in-memory comme **Redis**.

---

## 🗺️ 2. Les 3 Niveaux de Caching en Ingénierie Backend

```
                      ┌─────────────────────────────────────────┐
                      │    LES 3 NIVEAUX DE CACHING BACKEND     │
                      └────────────────────┬────────────────────┘
                                           │
      ┌────────────────────────────────────┼────────────────────────────────────┐
      ▼                                    ▼                                    ▼
┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
│   1. CACHE RÉSEAU       │   │   2. CACHE MATÉRIEL     │   │  3. CACHE LOGICIEL /    │
│  (Network Level Cache)  │   │   (Hardware Level)      │   │     IN-MEMORY (BDD)     │
├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
│ • CDN / Edge Locations  │   │ • Registres CPU & L1/L2/│   │ • Redis / Memcached     │
│ • PoP (Point of Pres.)  │   │   L3 Caches             │   │ • AWS ElastiCache       │
│ • Hiérarchie Cache DNS  │   │ • RAM (Main Memory) vs  │   │ • In-Memory Key-Value   │
│   (OS, Browser, ISP)    │   │   Stockage Disque/SSD   │   │   NoSQL Storage         │
└─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
```

---

### 🌐 A. Le Cache Niveau Réseau (Network Level)

#### 1. Les CDN (Content Delivery Networks)
* **Serveurs d'Origine (Origin Server)** : Le serveur central (ex: hébergé aux USA) qui possède la version originale des données.
* **Edge Locations / PoP (Point of Presence)** : Des grappes de serveurs distribués partout dans le monde.
* **Flux de travail** :
  1. Le client fait une requête HTTP.
  2. Le serveur Edge le plus proche vérifie son cache.
  3. **Cache Hit** : La ressource est renvoyée immédiatement.
  4. **Cache Miss** : Le serveur Edge va chercher la ressource sur le serveur d'origine, la stocke localement avec un **TTL (Time To Live)** et la sert au client.

#### 2. La Hiérarchie du Cache DNS
Résoudre un nom de domaine (ex: `example.com` en IP) coûte cher en requêtes réseau récursives (Root Server $
ightarrow$ TLD `.com` $
ightarrow$ Authoritative Name Server). Pour éviter cela, le DNS utilise **4 niveaux de cache successifs** :

```
 étape   niveau de cache DNS              action si trouvé (Cache Hit)
─────── ────────────────────             ────────────────────────────
  ①     [ Cache du Navigateur ]   ───>   Renvoie l'IP instantanément
                 │ (Cache Miss)
  ②     [ Cache de l'OS (Win/Mac) ] ───> Renvoie l'IP et évite le réseau
                 │ (Cache Miss)
  ③     [ Résolveur FAI / Cloudflare ] ─> Renvoie l'IP stockée en mémoire
                 │ (Cache Miss)
  ④     [ Requête Récursive Complète ] ──> Interroge Root $
ightarrow$ TLD $
ightarrow$ Authoritative
```

---

### 💻 B. Le Cache Niveau Matériel (Hardware Level) & La RAM

* **La hiérarchie mémoire** : CPU Registres $
ightarrow$ Caches L1 / L2 / L3 $
ightarrow$ **RAM (Main Memory)** $
ightarrow$ Disque SSD/HDD.
* **Pourquoi la RAM est-elle si rapide ?**
  * La RAM (Random Access Memory) accède aux données via des signaux électriques directs sur des adresses mémoire avec un temps d'accès quasi constant.
  * Les disques secondaires (SSD/HDD) effectuent des lectures électroniques ou mécaniques plus lentes mais **non-volatiles** (persistantes).
* **Le compromis de la RAM** : 
  * ⚡ **Vitesse extrême** mais ⚠️ **Capacité limitée** et 🔴 **Volatilité** (si le serveur redémarre, la RAM se vide).

---

### 🗄️ C. Le Cache Logicielles & Bases In-Memory (Redis / Memcached)

Pour nos applications backend, on utilise des bases de données **In-Memory Key-Value NoSQL** comme **Redis** ou **Memcached**.

* **In-Memory** : Les données résident en RAM pour des lectures en fractions de milliseconde.
* **Key-Value** : Pas de tables SQL complexes ni de contraintes jointures, juste des clés associées à des structures (strings, JSON, listes, sets).
* **Persistance** : Redis synchronise périodiquement la RAM sur le disque secondaire pour restaurer les données en cas de crash.

---

## ⚙️ 3. Stratégies d'Écriture & Politiques d'Éviction

### A. Stratégies d'Accès aux Données

```
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │                              STRATÉGIES DE LECTURE & ÉCRITURE                          │
 ├────────────────────────────────────────┬───────────────────────────────────────────────┤
 │ Lazy Caching (Cache-Aside)             │ Write-Through Caching                         │
 ├────────────────────────────────────────┼───────────────────────────────────────────────┤
 │ 1. Le serveur lit le cache.            │ 1. Lors d'un POST/PUT/PATCH, le serveur écrit │
 │ 2. Si Cache Miss, lit la BDD.          │    dans la BDD ET dans le cache simultanément.│
 │ 3. Écrit le résultat dans le cache.    │ 2. Avantage : Le cache est TOUJOURS frais.    │
 │ 4. Renvoie la réponse au client.       │ 3. Inconvénient : Surcoût de latence à        │
 │ • Avantage : Ne cache que le nécessaire.│    l'écriture.                                │
 └────────────────────────────────────────┴───────────────────────────────────────────────┘
```

---

### B. Les 4 Politiques d'Éviction (Eviction Policies)

Puisque la RAM a une capacité limitée, lorsque le cache est plein, il doit supprimer d'anciennes données pour faire de la place aux nouvelles :

| Politique | Nom Complet | Principe d'Éviction |
| :--- | :--- | :--- |
| **No Eviction** | Pas d'éviction | Le cache renvoie une erreur si la mémoire est pleine (à éviter en prod). |
| **LRU** | Least Recently Used | Supprime la clé dont la **dernière consultation est la plus ancienne**. |
| **LFU** | Least Frequently Used | Supprime la clé qui a le **nombre total de consultations le plus bas**. |
| **TTL** | Time-To-Live | Supprime automatiquement les clés dont le **délai d'expiration est dépassé**. |

---

## 🛠️ 4. Les 4 Cas d'Usage Incontournables en Backend (Redis)

```
             ┌─────────────────────────────────────────────────────────┐
             │         LES 4 CAS D'USAGE MAJEURS DE REDIS EN PROD      │
             └────────────────────────────┬────────────────────────────┘
                                          │
       ┌─────────────────────┬────────────┴────────────┬─────────────────────┐
       ▼                     ▼                         ▼                     ▼
┌──────────────┐      ┌──────────────┐          ┌──────────────┐      ┌──────────────┐
│  1. REQUÊTES │      │ 2. SESSIONS  │          │ 3. CACHING   │      │ 4. RATE      │
│  BDD LOURDES │      │ UTILISATEURS │          │ API EXTERNES │      │ LIMITING     │
└──────┬───────┘      └──────┬───────┘          └──────┬───────┘      └──────┬───────┘
       │                     │                         │                     │
• Cache le résultat    • Stocke les tokens de    • Stocke les réponses  • Compteur d'IP
  de requêtes SQL        session Auth en RAM.      d'API tierces (ex:     dans un middleware
  complexes (JOINs).                               météo) pour éviter     gérant le header
                       • Évite de taper dans     d'atteindre les        `X-Forwarded-For`.
• Idéal pour e-commerce  la BDD SQL à chaque       quotas et réduire      
  (prix, catalogue) et   requête entrante.         la facturation.        • Renvoyer un code
  profils utilisateurs.                                                   `429 Too Many Req`.
```

### 1. Caching de Requêtes BDD Complexes
* **Le problème** : Une requête SQL avec plusieurs `JOIN` et agrégations sur 1 million de lignes prend du temps CPU.
* **La solution** : Stocker la réponse JSON dans Redis avec un TTL d'une heure. Les requêtes suivantes lisent directement dans Redis en 2 ms.

### 2. Stockage de Sessions Utilisateurs (Auth)
* **Le problème** : Vérifier la validité de la session d'un utilisateur à chaque appel d'API sur une BDD SQL génère un trafic monstre.
* **La solution** : Placer le jeton de session dans Redis. La vérification se fait à la vitesse de la RAM.

### 3. Caching d'API Externes
* **Le problème** : Appeler une API tierce payante ou limitée en quota (ex: API météo) à chaque requête client coûte cher.
* **La solution** : Mettre en cache la réponse de l'API externe pendant 1 heure.

### 4. Implémentation du Rate Limiting
* **Le problème** : Bloquer les attaques de bots et prévenir le déni de service.
* **La solution** : Un middleware extrait l'IP du header `X-Forwarded-For`, incrémente un compteur dans Redis pour la minute en cours, et si le compte dépasse 50 requêtes/min, renvoie immédiatement une erreur **`429 Too Many Requests`**.

---