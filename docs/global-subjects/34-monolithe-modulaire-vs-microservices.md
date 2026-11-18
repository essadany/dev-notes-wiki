# 34. Monolithe modulaire vs microservices

> 🎯 **Résumé.** Découper en microservices avant d'en avoir le besoin transforme des appels de fonctions en appels réseau, des transactions ACID en sagas et un débogage direct en chasse au trace distribué. La règle est de démarrer avec un monolithe modulaire aux frontières claires, et de ne découper que sous contrainte réelle d'équipes ou de charge. Le prérequis silencieux du scale-out reste le *statelessness* : sans lui, ajouter des instances ne fait que multiplier les bugs.

---

## 🧭 1. Contexte

> 💡 **Règle de base.** commencer par un **monolithe modulaire**, ne passer aux microservices que si un besoin réel l'impose.

| | Monolithe modulaire | Microservices |
|---|---|---|
| Déploiement | une unité | un par service |
| Complexité opérationnelle | faible | élevée (réseau, observabilité, versions) |
| Transactions | locales (ACID) | distribuées (sagas, cohérence éventuelle) |
| Scalabilité | globale | par service |
| Équipes | 1 à quelques équipes | plusieurs équipes autonomes |

```
        MODULITHE MODULAIRE                    MICROSERVICES
   ┌────────────┐  ┌────────────┐        ┌──────────┐  ┌──────────┐
   │ Facturation│  │  Clients   │        │ Factures │─▶│ Clients  │
   │  (module)  │  │  (module)  │        │ (service)│  │ (service)│
   └─────┬──────┘  └─────┬──────┘        └────┬─────┘  └────┬─────┘
         └────────┬──────┘                    └──────┬───────┘
                  ▼                                  ▼
        transaction ACID locale            saga + cohérence éventuelle
                  ▼                                  ▼
        une seule base Postgres           une base par service + outbox
                  ▲                                  ▲
                  └──────────────┬───────────────────┘
                                 │
                    sessions → Redis · uploads → S3
                    statelessness = prérequis du scale-out
```

**Quand découper ?** Équipes nombreuses qui se gênent, besoins de scalabilité très différents selon les parties, contraintes de déploiement ou de technologie distinctes.

**Monolithe modulaire** : modules aux frontières claires (chacun avec ses données), communication par interfaces ou événements internes, pas d'accès direct aux tables d'un autre module.
- Spring : **Spring Modulith**, ou modules Maven/Gradle.
- Django : apps découplées, imports contrôlés (import-linter).
- Laravel : modules/packages internes.
- NestJS : modules avec exports explicites.

> ⚠️ **Pièges des microservices.** « monolithe distribué » (services trop couplés qui se déploient ensemble), base de données partagée entre services, trop de services trop petits, absence d'observabilité (traces distribuées indispensables).

## 🧠 2. Logique first-principles

**Le statelessness est le prérequis absolu du scale-out.** Une instance qui garde une session dans sa RAM renverra un 401 dès que le load balancer routera la requête suivante vers une autre instance. Avant de parler de microservices, externaliser tout l'état : **sessions → Redis**, **uploads → S3**, **base → PostgreSQL/RDS centralisé**. Pas de *sticky sessions*, et une marge de sécurité de **20 à 40 % de headroom** (utilisation cible 60-80 %) pour absorber les rafales avant que la file d'attente n'explose.

**Les microservices scalent les équipes, pas les machines.** Ils ne sont pas un prérequis pour scaler la charge : ils règlent surtout la coordination humaine à partir de 100 à 500 développeurs répartis en équipes autonomes. Un monolithe bien structuré avec un index B-Tree PostgreSQL est 100 fois plus simple et performant qu'un malstrom Redis/Kafka mal maîtrisé. Mesurez avant de découper : tracer avant d'optimiser évite de payer le prix du réseau pour un goulot qui était un logger synchrone.

**Le serverless est une troisième option de déploiement.** Avec AWS Lambda, Cloudflare Workers ou Vercel, on ne loue plus de VM : l'API Gateway instancie la fonction à la demande et on paie les millisecondes d'exécution. Le coût caché est le **cold start** (boot du runtime), atténué par les MicroVM Firecracker, les V8 Isolates (0-5 ms) et les pings de chauffe. Cela ne dispense pas de la règle 12-factor : port binding, concurrence par type de processus et config par environnement restent le contrat avec la plateforme.

**Backing services et parité.** Traiter Postgres, Redis et S3 comme des ressources attachées via une URL + des credentials permet de passer du conteneur local à RDS sans changer une ligne de code. Le même build immuable (tag SHA) passe du staging à la prod : c'est le facteur *Build, Release, Run*. Spring Modulith ou des modules Gradle/Maven matérialisent les frontières sans payer le prix d'un réseau.

---

## 💻 3. Exemples techniques

Un upstream Nginx avec des seuils généreux, du *slow-start* et un `X-Request-ID` pour corréler les logs :

```nginx
upstream api {
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s slow_start=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s slow_start=30s;
    server 10.0.1.13:8080 max_fails=3 fail_timeout=30s slow_start=30s;
    keepalive 64;
}

server {
    listen 443 ssl;
    server_name api.internal;

    ssl_certificate     /etc/nginx/certs/api.crt;
    ssl_certificate_key /etc/nginx/certs/api.key;

    location / {
        proxy_pass http://api;
        proxy_next_upstream error timeout http_502 http_503;
        proxy_connect_timeout 2s;
        proxy_read_timeout 10s;
        proxy_set_header X-Request-ID $request_id;
    }

    location = /readyz {
        proxy_pass http://api;
        access_log off;
    }
}
```

Un PgBouncer externe garde un nombre fixe de connexions vers Postgres, ce qui évite qu'un autoscaling horizontal épuise le `max_connections` de la base :

```ini
[databases]
appdb = host=10.0.2.20 port=5432 dbname=appdb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 3000
default_pool_size = 40
reserve_pool_size = 10
reserve_pool_timeout = 3
server_idle_timeout = 120
server_lifetime = 3600
```

Cette configuration Django rend l'instance réellement *stateless* : base centralisée, sessions dans Redis et fichiers dans S3, le tout injecté par l'environnement :

```python
import os

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ["POSTGRES_DB"],
        "USER": os.environ["POSTGRES_USER"],
        "PASSWORD": os.environ["POSTGRES_PASSWORD"],
        "HOST": os.environ["POSTGRES_HOST"],
        "PORT": os.environ.get("POSTGRES_PORT", "5432"),
    }
}

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ["REDIS_URL"],
    }
}

SESSION_ENGINE = "django.contrib.sessions.backends.cache"

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
        "OPTIONS": {
            "bucket_name": os.environ["S3_BUCKET"],
            "region_name": os.environ["AWS_REGION"],
        },
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    },
}
```

---

## 🔗 4. Voir aussi

- [Sujet 02 — Gestion de l'état](02-gestion-etat.md)
- [Sujet 22 — Jobs asynchrones et files](22-jobs-asynchrones-files.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Scaling backend, partie 2 — statelessness, microservices, serverless](../backend-from-first-principles/21.2-backend-scaling-part2.md)
