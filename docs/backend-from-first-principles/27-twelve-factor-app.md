# 27. The Twelve-Factor App (12-Factor Methodology)

Après avoir vu 26 chapitres techniques, cette vidéo vient faire le lien et expliquer **pourquoi** nous avons pris toutes ces décisions d'architecture (variables d'environnement, logs sur stdout, conteneurs stateless, port binding, graceful shutdown, etc.). 

Ces 12 règles forment le contrat fondamental entre une application moderne et la plateforme cloud qui l'héberge.

---

## 📜 1. L'Origine et la Philosophie du 12-Factor App

Le manifeste **12-Factor App** a été rédigé en 2011 par **Adam Wiggins** et l'équipe d'**Heroku** (l’un des premiers PaaS - *Platform as a Service*).

```
   ┌─────────────────────────────────────────────────────────────┐
   │             L'ÈRE DE L'INFRASTRUCTURE "SNOWFLAKE"           │
   │  • Serveurs physiques ou VM louées configurés à la main     │
   │  • Transfert de fichiers par FTP / SSH                       │
   │  • Dépendances installées globalement sur le système        │
   │  • "Ça marche sur ma machine" (It works on my machine!)     │
   │  • Érosion logicielle (Software Erosion) et crashs prod     │
   └──────────────────────────────┬──────────────────────────────┘
                                  │
                                  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │            LE CONTRAT 12-FACTOR APP (Heroku / Cloud)        │
   │  • Contrat strict entre le Backend et la Plateforme d'Hôte  │
   │  • Portabilité totale (Docker, Kubernetes, AWS, Render...) │
   │  • Déploiement, Scaling et Maintenance automatisés sans bug │
   └─────────────────────────────────────────────────────────────┘
```

> 💡 **Mise à jour moderne (KubeCon 2024)** : Initialement figé depuis 2017, le projet a été rendu open-source sous gouvernance communautaire (Salesforce/AWS/Google Cloud). Les principes restent intemporels mais s'adaptent aujourd'hui aux conteneurs, au serverless et au cloud-native moderne.

---

## 🧱 2. Les 12 Facteurs Décortiqués

### 1️⃣ Codebase (Une seule base de code, plusieurs déploiements)
* **Le principe** : Une application = un seul dépôt Git (ou ensemble de dépôts partageant un commit racine).
* **Déploiements multiples** : La même base de code tourne en `local`, `staging`, `production`, `PR preview`.
* **Ce qui varie** : Ce n'est jamais le code source, mais la **configuration** injectée dans chaque environnement.
* **Point d'attention** : Ne pas confondre avec le copié-collé de code (anti-pattern). Pour les monorepos, chaque application possède son propre contrat 12-factor.

---

### 2️⃣ Dependencies (Déclarer et isoler explicitement les dépendances)
* **Déclaration** : Un fichier manifeste explicite liste toutes les bibliothèques nécessaires (`package.json`, `go.mod`, `requirements.txt`, `Gemfile`).
* **Verrouillage (Lockfile)** : Les fichiers de lock (`package-lock.json`, `go.sum`) fixent les versions exactes pour éviter tout changement furtif.
* **Isolation** : Aucune dépendance ne doit fuiter du système hôte (`virtualenv`, `bundle exec`).
* **L'évolution moderne** : **Docker** a poussé ce principe à son apogée en isolant non seulement les packages mais aussi l'OS, les binaires système et le runtime.

---

### 3️⃣ Config (Stocker la configuration dans l'environnement)
* **Définition de la config** : Tout ce qui varie entre deux déploiements (URLs de BDD, clés d'API, credentials S3, ports).
* **Le Test Ultime** : *"Pouvez-vous rendre votre code open-source tout de suite sans compromettre aucun secret ?"* Si la réponse est non, des secrets sont durs dans le code.
* **Variables d'environnement (`ENV`)** : Agnostiques par rapport au langage, faciles à injecter sans modifier le code source.
* **Nuance de Sécurité (Secrets Managers)** : Pour les mots de passe et clés très sensibles, utiliser un Secret Manager (AWS Parameter Store, HashiCorp Vault) avec chargement en mémoire au démarrage pour éviter de fuiter les variables d'environnement dans les processus enfants ou les crash logs.

---

### 4️⃣ Backing Services (Traiter les services tiers comme des ressources attachées)
* **Backing Service** : Tout service consommé à travers le réseau (Postgres, Redis, RabbitMQ, S3, SMTP/Resend, Stripe).
* **Resource Handle** : L'application communique avec le service via une URL d'attachement + credentials (ex: `postgres://user:pass@host:5432/dbname`).
* **Flexibilité** : Passer d'une BDD Postgres locale en conteneur à une BDD AWS RDS en prod ne doit nécessiter **aucun changement de code**, uniquement une modification de la variable d'environnement du handle.

---

### 5️⃣ Build, Release, Run (Séparer strictement les étapes d'assemblage et d'exécution)

```
   [ Code Git (Commit 100) ] + [ Manifeste ] ──► (1) BUILD ──► [ Image Docker / Binaire ]
                                                                       │
   [ Build Inmutable ] + [ Config Prod ]     ──► (2) RELEASE ──► [ Release v101 ]
                                                                       │
   [ Release v101 ]                          ──► (3) RUN     ──► [ Processus Exécuté ]
```

* **Build** : Transforme le code source et ses dépendances en un binaire ou une image Docker immuable.
* **Release** : Associe le Build immuable avec la configuration spécifique à l'environnement. Chaque release possède un ID unique et non modifiable (ledger d'historique).
* **Run** : Lance les processus à partir de la Release.
* **Bénéfice majeur** : **Rollbacks instantanés**. Si la release `v102` crash, on réinstancie immédiatement la release `v101` sans recompiler. Interdiction absolue d'éditer du code en direct sur le serveur de prod via SSH !

---

### 6️⃣ Processes (Exécuter l'application sous forme de processus stateless)
* **Stateless & Share-Nothing** : Les processus ne stockent aucune donnée persistante en mémoire RAM ou sur le disque local.
* **Gestion du stockage temporaire** : Le stockage temporaire sur disque/RAM est toléré uniquement pour la durée d'une seule transaction/requête HTTP.
* **Anti-pattern à bannir** : Les **Sticky Sessions** sur le Load Balancer.
* **Gestion de l'état** : Tout état (session utilisateur, jeton) doit résider dans un backing service stateful (Redis ou Postgres).
* **Cas des WebSockets** : La connexion WebSockets étant par nature stateful, on utilise un bus **Pub/Sub (Redis)** pour synchroniser les événements entre les instances.

---

### 7️⃣ Port Binding (Exporter les services via le binding de port)
* **Autonomie totale** : L'application embarque son propre serveur HTTP (`express`, Go `http.ListenAndServe`) et écoute directement sur un port réseau (ex: `8080`).
* **Rupture avec le passé** : Fini les applications hébergées en tant que "modules" à l'intérieur d'un serveur d'application lourd (type Apache PHP ou Tomcat Java).
* **Routage** : En production, un reverse proxy (Nginx, AWS ALB, Cloudflare) redirige le nom de domaine public vers le port écouté par le processus.
* **Adaptation Serverless** : En serverless (AWS Lambda), le principe s'élargit à l'exportation d'une interface/handler standardisée contrôlée par la plateforme.

---

### 8️⃣ Concurrency (Scaler via le modèle de processus)

```
                       ┌───────────────────────────────┐
                       │    LOAD BALANCER / INGRESS    │
                       └───────────────┬───────────────┘
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
┌───────────────────────┐                             ┌───────────────────────┐
│ WEB PROCESS (Copie 1) │                             │ WEB PROCESS (Copie 2) │
└───────────────────────┘                             └───────────────────────┘
            │                                                     │
            └──────────────────────────┬──────────────────────────┘
                                       ▼
                       ┌───────────────────────────────┐
                       │   WORKER PROCESS (Queue Job)  │
                       └───────────────────────────────┘
```

* **Découpage par type de charge** : Séparer les requêtes HTTP (`web`), les tâches lourdes en arrière-plan (`worker`), et les tâches planifiées (`clock`/`cron`).
* **Scale Out (Horizontal)** : Scaler chaque type de processus indépendamment. Si le trafic web augmente à midi, scaler uniquement les processus `web` sans gaspiller de la RAM pour les `workers`.
* **Rôle de la plateforme** : L'application tourne au premier plan (`foreground`) sans se dæmoniser elle-même. La gestion du cycle de vie des conteneurs est déléguée à l'orchestrateur (Kubernetes, Docker Swarm, Systemd).

---

### 9️⃣ Disposability (Maximiser la robustesse avec un démarrage rapide et un arrêt propre)
* **Démarrage rapide (Fast Startup)** : Le temps de boot doit prendre quelques secondes au maximum pour autoriser l'autoscaling réactif et les déploiements rolling sans interruption.
* **Graceful Shutdown (`SIGTERM`)** :
  * Arrêter d'accepter de nouvelles requêtes HTTP.
  * Laisser terminer les requêtes HTTP en cours (*connection draining*).
  * Replacer les tâches non finies des workers dans la file d'attente (**Idempotence & Re-entrancy**).
  * Libérer les pools de connexions et sortir proprement (`Exit 0`).
* **Crash-Only Design** : L'application doit être conçue pour résister à une coupure brutale du serveur (`SIGKILL`, panne matérielle) sans corrompre les données.

---

### 🔟 Dev/Prod Parity (Maintenir le développement, le staging et la production aussi similaires que possible)
* **Réduire les 3 écarts (Gaps)** :
  1. *Time Gap* : Déployer en quelques heures plutôt qu'en quelques semaines.
  2. *Personnel Gap* : Les développeurs qui écrivent le code participent au déploiement et au suivi des logs en prod.
  3. *Tools Gap* : Utiliser exactement les mêmes services en local qu'en production (ex: Postgres en conteneur Docker local au lieu de SQLite en dev vs Postgres en prod).
* **Attention aux divergences d'ORM** : SQLite et Postgres ne gèrent pas le `LIKE`/`ILIKE` de la même façon. Se fier aux conteneurs pour garantir une parité stricte des outils.

---

### 1️⃣1️⃣ Logs (Traiter les logs comme des flux d'événements)
* **Un flux temporel continu (*Event Stream*)** : L'application ne doit jamais gérer l'écriture dans des fichiers de logs, la rotation ou l'archivage.
* **Standard Output (`stdout` / `stderr`)** : L'application écrit simplement ses logs de manière non-bufferisée sur la sortie standard.
* **Rôle de l'infrastructure** : C'est la plateforme d'exécution (Docker, Kubernetes, AWS CloudWatch, Datadog, Grafana Loki) qui capture le flux `stdout`, l'horodate, l'anonymise et le redirige vers l'outil d'analyse centralisé.

---

### 1️⃣2️⃣ Admin Processes (Exécuter les tâches d'administration comme des processus ponctuels)
* **One-Off Processes** : Les tâches d'administration (migrations de BDD, scripts de nettoyage ou d'importation de données) doivent s'exécuter dans un processus ponctuel qui s'arrête une fois terminé.
* **Parité d'environnement** : Ces scripts doivent être lancés avec **le même code source, le même commit et la même configuration** que le processus web principal.
* **Exemple typique** : Les migrations de base de données (`db:migrate`) exécutées automatiquement lors du pipeline de déploiement avant le lancement du nouveau binaire web.
* **Sécurité** : Évite les interventions manuelles incontrôlées en SSH direct sur la BDD sans traçabilité ni revue de code.

---

## 📊 Tableau Récapitulatif des 12 Facteurs

| # | Facteur | Rôle Clé | Outil / Pratique Moderne |
|---|---|---|---|
| 1 | **Codebase** | 1 Repo = 1 App, déploiements multiples | Git, GitHub, GitLab |
| 2 | **Dependencies** | Déclaration et isolation explicites | `package.json`, `go.mod`, Docker |
| 3 | **Config** | Séparer la config et les secrets du code | Variables d'environnement (`ENV`), AWS Parameter Store / Vault |
| 4 | **Backing Services** | Traiter les BDD/Redis comme des ressources attachées | URLs de connexion (`postgres://...`), AWS RDS, Neon |
| 5 | **Build, Release, Run** | Séparer strictement les étapes de déploiement | CI/CD (GitHub Actions), Docker Registry, Release IDs |
| 6 | **Processes** | Exécuter sous forme de processus stateless | Architecture Share-Nothing, Redis pour sessions |
| 7 | **Port Binding** | Serveur HTTP autonome qui écoute sur un port | Express, Go HTTP, FastAPI, Nginx Reverse Proxy |
| 8 | **Concurrency** | Scaler horizontalement par type de processus | Process Formation (`web=3`, `worker=2`), Kubernetes |
| 9 | **Disposability** | Boot rapide & Graceful Shutdown (`SIGTERM`) | Connection Draining, Idempotence, Crash-Only |
| 10 | **Dev/Prod Parity** | Garder local, staging et prod identiques | Docker Compose, conteneurs de BDD locales |
| 11 | **Logs** | Streamer les logs sur `stdout` | Logs JSON, Grafana Loki, Datadog |
| 12 | **Admin Processes** | Executer les scripts admin en one-off | Migrations BDD automatisées en CI/CD |

---