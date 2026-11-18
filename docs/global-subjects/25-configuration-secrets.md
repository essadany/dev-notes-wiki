# 25. Configuration et secrets

> 🎯 **Résumé.** Un mot de passe de base codé en dur dans `application.yml` finit dans Git et dans toutes les images déployées. La configuration vient de l'environnement (12-factor) et les secrets restent dans un coffre externe injecté au runtime, avec le même artefact partout. Le piège : appeler « secret » une variable d'environnement non chiffrée visible de tous, ou modifier un `.env` de prod sans régénérer le cache de configuration.

---

## 🧭 1. Contexte

**Principe (12-factor)** : la config vient de l'environnement, pas du code ; même artefact pour dev/staging/prod.

| Stack | Mécanisme |
|---|---|
| Spring | `application.yml` + profils (`dev`, `prod`), variables d'env |
| Django | `settings.py` + `django-environ` / variables d'env |
| Laravel | `.env` + `config/*.php` (`config:cache` en prod) |
| Node | `dotenv`, `@nestjs/config` |

Secrets : jamais dans Git. Utiliser variables CI/CD, Vault, AWS Secrets Manager, Kubernetes Secrets (chiffrés au repos si possible).

---

## 💻 2. Exemples techniques

Le document YAML Spring active le profil `prod` uniquement quand `SPRING_PROFILES_ACTIVE=prod` est fourni ; les valeurs sensibles sont des variables d'environnement sans valeur par défaut, donc une config incomplète échoue au démarrage.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/wiki
    username: wiki
    password: wiki
---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
```

Django lit l'environnement au démarrage via `django-environ` : `SECRET_KEY` et `DATABASE_URL` n'ont pas de valeur par défaut, ce qui évite un démarrage silencieux avec une clé connue.

```python
# config/settings.py
from pathlib import Path
import environ

BASE_DIR = Path(__file__).resolve().parent.parent
env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)

DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')
SECRET_KEY = env('DJANGO_SECRET_KEY')
DATABASES = {'default': env.db('DATABASE_URL')}
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

Le `.env.example` Laravel est versionné sans secret ; la CI/CD injecte les vraies valeurs, puis `php artisan config:cache` fige la configuration (à régénérer avec `config:clear` si l'environnement change).

```dotenv
APP_NAME=Wiki
APP_ENV=production
APP_DEBUG=false
APP_URL=https://wiki.example.com
LOG_CHANNEL=stderr
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=wiki
CACHE_DRIVER=redis
SESSION_DRIVER=redis
```

---

## 🔗 3. Voir aussi

- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Sujet 31 — Conteneurisation, CI/CD, health checks](31-conteneurisation-cicd-health-checks.md)
- [Configuration de production (first principles)](../backend-from-first-principles/17-production-grade-config-management.md)
- [Twelve-Factor App (first principles)](../backend-from-first-principles/27-twelve-factor-app.md)
