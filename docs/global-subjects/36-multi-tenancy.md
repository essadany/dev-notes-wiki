# 36. Multi-tenancy

> 🎯 **Résumé.** Une seule application sert plusieurs clients, et un simple `WHERE` oublié suffit à exposer les factures d'un tenant à un autre. Le modèle le moins coûteux est une colonne `tenant_id` filtrée de façon centralisée, complétée par le Row-Level Security de PostgreSQL ; les schémas ou bases séparés montent l'isolation en même temps que le coût. Le piège : cloisonner la base mais oublier les clés de cache, les files de tâches et les migrations à appliquer N fois.

---

## 🧭 1. Contexte

**Contexte** : une même application sert plusieurs clients (tenants) dont les données doivent rester isolées.

| Modèle | Principe | Isolation | Coût |
|---|---|---|---|
| Schéma partagé | colonne `tenant_id` sur chaque table | faible (dépend du code) | le plus bas |
| Un schéma par tenant | schémas PostgreSQL séparés | moyenne | moyen |
| Une base par tenant | isolation maximale | forte | élevé |

| Stack | Solution |
|---|---|
| Django | `django-tenants` (schémas PostgreSQL), ou `tenant_id` + manager filtrant |
| Laravel | `stancl/tenancy`, `spatie/laravel-multitenancy` |
| Spring / Hibernate | support multi-tenant d'Hibernate (discriminant, schéma ou base) |
| PostgreSQL | **Row-Level Security** (RLS) pour forcer le filtre au niveau base |

> ⚠️ **Pièges.** oublier le filtre `tenant_id` dans une requête (fuite de données entre clients, faille grave) : centraliser le filtre (manager, filtre global, RLS) ; caches et files de tâches qui doivent aussi être cloisonnés par tenant ; migrations à appliquer sur N schémas/bases.

---

## 💻 2. Exemples techniques

PostgreSQL force l'isolation au niveau base : sans variable de session valide, la politique ne renvoie aucune ligne, même si le code ORM a oublié son filtre :

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices FORCE ROW LEVEL SECURITY;

CREATE POLICY invoices_tenant_isolation ON invoices
  USING (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::bigint)
  WITH CHECK (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::bigint);

CREATE INDEX invoices_tenant_created_idx ON invoices (tenant_id, created_at DESC);
```

Le middleware Django pose cette variable dans la transaction de la requête, après l'authentification, avec `set_config(..., true)` valable pour toute la transaction :

```python
from django.db import connection, transaction


class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        with transaction.atomic():
            with connection.cursor() as cursor:
                cursor.execute(
                    "SELECT set_config('app.tenant_id', %s, true)",
                    [str(request.user.tenant_id)],
                )
            return self.get_response(request)
```

Pour le modèle « un schéma par tenant », `django-tenants` sépare les apps partagées et les apps métier, et la migration doit être jouée sur le schéma public puis sur tous les schémas :

```python
SHARED_APPS = (
    "django_tenants",
    "tenants",
    "django.contrib.contenttypes",
    "django.contrib.auth",
    "django.contrib.sessions",
)

TENANT_APPS = ("invoices", "catalog")

TENANT_MODEL = "tenants.Client"

DATABASE_ROUTERS = ("django_tenants.routers.TenantSyncRouter",)

MIDDLEWARE = [
    "django_tenants.middleware.main.TenantMainMiddleware",
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
]
```

Les migrations se jouent d'abord sur le schéma partagé, puis sur chaque schéma tenant :

```bash
python manage.py migrate_schemas --shared
python manage.py migrate_schemas
```

Les clés de cache et de verrou portent le `tenant_id`, y compris dans les tâches Celery : un export d'un client ne peut pas écraser celui d'un autre :

```python
from celery import shared_task
from django.core.cache import cache
from django_redis import get_redis_connection


def tenant_key(tenant_id: int, key: str) -> str:
    return f"tenant:{tenant_id}:{key}"


@shared_task(bind=True, autoretry_for=(TimeoutError,), retry_backoff=True)
def export_invoices(self, tenant_id: int, period: str) -> str:
    result_key = tenant_key(tenant_id, f"export:{period}")
    lock_key = tenant_key(tenant_id, f"lock:export:{period}")
    lock = get_redis_connection("default").lock(lock_key, timeout=300, blocking_timeout=5)

    with lock:
        result = cache.get(result_key)
        if result is None:
            result = f"exports/{tenant_id}/{period}/invoices.csv"
            cache.set(result_key, result, timeout=3600)
        return result
```

---

## 🔗 3. Voir aussi

- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
- [Sujet 18 — Cache backend et HTTP](18-cache-backend-http.md)
- [Sujet 21 — Migrations base de données](21-migrations-base-de-donnees.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
