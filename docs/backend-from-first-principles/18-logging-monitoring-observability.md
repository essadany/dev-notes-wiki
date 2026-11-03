# 18. Logging, Monitoring et Observabilité (Le Trio Réseau & Prod)

Dans le monde moderne des architectures distribuées (microservices, serveurs multirégions, cloud), déployer du code ne suffit pas. Si une erreur survient à 3h du matin pour un utilisateur à l'autre bout du monde, comment savoir **ce qui s'est passé**, **où ça a cassé** et **pourquoi** ? C'est tout le rôle de ce trio indispensable !

---

## 🧭 1. Le Concept de "Spectre" & Pourquoi on en a besoin

Contrairement à un algo ou une règle de syntaxe, le Logging, le Monitoring et l'Observabilité ne sont pas des règles binarisées (0 ou 1). 

> 💡 **Le Spectre de l'Observabilité** : Aucune entreprise ne suit 100% des meilleures pratiques d'un coup. C'est une démarche progressive (un *spectre*) mise en place selon la maturité de l'équipe et la complexité des systèmes.

### Pourquoi est-ce vital en backend ?
* **Applications distribuées** : Nos serveurs tournent sur plusieurs instances, conteneurs ou régions cloud.
* **Complexité des requêtes** : Une seule action utilisateur (ex: créer une commande) traverse plusieurs couches (Controller -> Middleware -> Service -> Repository -> Base de données -> Service d'emailing).
* **Débogage impossible à l'aveugle** : Sans traçabilité, débugger un bug intermittent en production relève de la devinette.

---

## 🏛️ 2. Les 3 Piliers de l'Observabilité

Une application est dite **observable** si on peut comprendre son état interne uniquement en observant ses sorties externes. L'observabilité repose sur **3 piliers fondamentaux** :

```
                        ┌─────────────────────────────────────────┐
                        │      LES 3 PILIERS DE L'OBSERVABILITÉ   │
                        └────────────────────┬────────────────────┘
                                             │
      ┌──────────────────────────────────────┼──────────────────────────────────────┐
      ▼                                      ▼                                      ▼
┌──────────────────────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐
│        1. LOGS           │   │       2. METRICS         │   │        3. TRACES         │
│   "Qu'est-ce qui s'est   │   │  "Comment se comporte    │   │   "Où et comment la      │
│     passé et quand ?"    │   │     le système ?"        │   │  requête a-t-elle voyagé"│
└──────────────────────────┘   └──────────────────────────┘   └──────────────────────────┘
```

---

## 📜 3. Pilier 1 : Le Logging (Le Journal d'Événements)

Le **Logging** consiste à enregistrer tous les événements importants du cycle de vie de l'application avec leurs **métadonnées** (Timestamp, User ID, Request ID, IP, Latence).

### A. Les Niveaux de Log (Log Levels)
Chaque log doit être catégorisé par niveau de sévérité pour pouvoir filtrer le bruit :

| Niveau | Usage / Contexte | Environnement | Exemple typique |
| :--- | :--- | :---: | :--- |
| **`DEBUG`** | Informations ultra-détaillées pour le dépannage local. | **Dev uniquement** | `Connexion BDD établie sur le port 5432` |
| **`INFO`** | Événements business normaux et succès d'opérations. | Dev & Prod | `Utilisateur 42 a créé la tâche #108` |
| **`WARN`** | Comportement anormal mais non critique (pas une erreur système). | Dev & Prod | `Échec de connexion : mot de passe erroné pour user X` |
| **`ERROR`** | Dysfonctionnement d'une opération (requête SQL échouée, API tierce down). | Dev & Prod | `Erreur SQL 500 : violation de contrainte UNIQUE` |
| **`FATAL`** | Erreur critique entraînant l'arrêt/crash immédiat de l'application. | Dev & Prod | `Impossible de charger la clé privée JWT : arrêt du serveur` |

### B. Logs Structurés vs Non-Structurés

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  EN DÉVELOPPEMENT : Unstructured Logs (Plain Text / Console avec couleurs)              │
│  [2026-11-03 09:12:00] INFO: User 42 created todo "Acheter du pain" (Latency: 12ms)     │
│  => Lisible par un humain en un coup d'œil dans son terminal VS Code.                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│  EN PRODUCTION : Structured Logs (JSON)                                                 │
│  {"timestamp":"2026-11-03T09:12:00Z","level":"info","user_id":42,"action":"create_todo",│
│   "request_id":"req-9912","latency_ms":12}                                             │
│  => Facilement analysable et indexable par des outils automatisés (ELK, Loki, New Relic) │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📈 4. Pilier 2 : Le Monitoring & Les Métriques

Le **Monitoring** est le suivi en temps réel (ou quasi réel avec un léger délai de 10 à 15 secondes pour éviter de surcharger les serveurs) de la santé globale de l'infrastructure et de l'application.

### Que mesure-t-on avec les Métriques ?
Les métriques sont des **données chiffrées agrégées** dans le temps :
* **Métriques Système** : Utilisation du CPU, consommation de mémoire RAM (ex: 3 MB), nombre de connexions ouvertes dans le pool Postgres.
* **Métriques Applicatives / HTTP** : Débit de requêtes (Throughput - req/sec), temps de réponse moyen (Latency), taux d'erreurs HTTP (pourcentage de réponses > 200 ou 5xx).
* **Métriques Business** : Nombre de to-dos créés par heure, nombre d'abonnements souscrits.

---

## 🔍 5. Pilier 3 : L'Observabilité & Les Traces (Tracing)

### Monitoring vs Observabilité
* **Monitoring** : Te dit **qu'il y a un problème** (ex : *"Alerte Slack : Le taux d'erreur 500 dépasse 80%"*).
* **Observabilité** : Te permet de comprendre **exactement pourquoi et où le problème se situe** sans avoir à redéployer de code.

### Qu'est-ce qu'une Trace (Distributed Tracing) ?
Une **Trace** représente le voyage complet d'une requête à travers toutes les couches et tous les microservices de votre backend.

```
[ Client / Router ] ───(Request ID: req-881)───┐
                                               ▼
                                 ┌──────────────────────────┐
                                 │ 1. Middleware Tracing    │ (Span 1 : Init Transaction)
                                 └─────────────┬────────────┘
                                               │
                                               ▼
                                 ┌──────────────────────────┐
                                 │ 2. Controller Validation │ (Span 2 : Parsing JSON)
                                 └─────────────┬────────────┘
                                               │
                                               ▼
                                 ┌──────────────────────────┐
                                 │ 3. Service Business      │ (Span 3 : Logic & Checks)
                                 └─────────────┬────────────┘
                                               │
                                               ▼
                                 ┌──────────────────────────┐
                                 │ 4. Repository / BDD      │ (Span 4 : Exec SQL Query) -> CRASH !
                                 └──────────────────────────┘
```

* **Contexte de Requête & Context Propagation** : En injectant le `Request-ID` et la transaction dans le contexte de l'application (Request Context), chaque couche ajoute ses propres attributs (Span ID, User ID, SQL Query) au fil de l'exécution.

### Concepts clés de l'industrie :
* **Instrumentation** : L'action de mesurer et d'extraire les attributs des fonctions et routes du code.
* **OpenTelemetry (OTel)** : Le standard open-source universel (APIs & SDKs pour Node, Go, Python, Rust) permettant d'instrumenter son code de façon neutre, indépendante des fournisseurs d'outils.

---

## 🔄 6. Le Workflow d'Investigation en Production

Voici le déroulé exact d'une résolution d'incident quand le trio est correctement configuré :

```
 ┌────────────────┐         ┌────────────────┐         ┌────────────────┐         ┌────────────────┐
 │ 1. ALERTE      │         │ 2. MÉTRIQUES   │         │ 3. LOGS        │         │ 4. TRACES      │
 │ Notification   │  ────>  │ Graphique      │  ────>  │ Filtrage JSON  │  ────>  │ Span exact de  │
 │ Slack / Pager  │         │ d'erreur > 80% │         │ sur Status 500 │         │ la panne (BDD) │
 └────────────────┘         └────────────────┘         └────────────────┘         └────────────────┘
```

1. **Alerte** : Un webhook Slack signale un pic anormal d'erreurs HTTP 500.
2. **Métriques** : Sur le dashboard (Grafana/New Relic), on confirme que le taux d'erreur a grimpé à 80% sur la route `/api/todos`.
3. **Logs** : On filtre les logs structurés associés à cette métrique et on isole le log JSON d'erreur `500 Internal Server Error`.
4. **Traces** : On clique sur le `Span ID` du log pour afficher la trace complète. On voit immédiatement que la requête a traversé le middleware et le service, mais a échoué dans la couche Repository sur la fonction `db.Exec()`. Le problème est corrigé en quelques minutes !

---

## 🛠️ 7. Écosystème des Outils : Open Source vs SaaS Propriétaire

Le choix des outils dépend de la taille de l'équipe et des ressources d'infrastructure :

| Catégorie | Outils Open-Source (Self-Hosted) | Solutions SaaS Propriétaires (All-in-One) |
| :--- | :--- | :--- |
| **Composants** | • **Prometheus** (Métriques)<br>• **Grafana** (Dashboards & Alerts)<br>• **Loki / Promtail** (Logs)<br>• **Jaeger** (Traces) | • **New Relic**<br>• **Datadog**<br>• **Sentry** (Erreurs & APM) |
| **Avantages** | Contrôle total des données, aucun coût de licence logicielle. | Configuration rapide en 5 min, tableau de bord unifié, zéro maintenance infra. |
| **Inconvénients** | Nécessite une équipe DevOps dédiée pour maintenir l'infrastructure d'observabilité. | Coût financier pouvant grimper vite selon le volume d'ingestion de données. |

---

## 🤝 8. Collaboration Dev & DevOps

L'observabilité est un effort d'équipe :
* **Le Développeur Backend** : Intègre les logs structurés au niveau du code (`log.info()`, `log.error()`), enrichit le contexte de requête (`user_id`, `tenant_id`) et instrumente les fonctions clés.
* **L'Ingénieur DevOps / Infra** : Met en place les collecteurs de logs/traces (Collectors OpenTelemetry), configure les agents de monitoring (Prometheus/Agent Datadog) et définit les alertes d'infrastructures.
