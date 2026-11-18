# 23. Rate limiting

> 🎯 **Résumé.** Sans limite, un script de scraping ou un brute force épuise le pool de connexions avant même que la logique métier ne réponde. On compte les requêtes par clé (IP, utilisateur, endpoint) dans un store partagé avec un algorithme de *token bucket*, et on répond `429` avec les en-têtes de reprise. Le piège : faire du store de comptage un SPOF inconscient, ou croire que des compteurs locaux protègent une flotte de plusieurs instances.

---

## 🧭 1. Contexte

**But** : se protéger du brute force, du scraping, des abus.

| Stack | Solution |
|---|---|
| Spring | Bucket4j, Resilience4j `RateLimiter`, ou limitation au niveau gateway |
| DRF | `DEFAULT_THROTTLE_CLASSES` / `DEFAULT_THROTTLE_RATES` |
| Laravel | `RateLimiter::for()` + middleware `throttle` |
| NestJS | `@nestjs/throttler` |
| Express | `express-rate-limit` |

Idéalement aussi au niveau reverse proxy / API gateway (Nginx, Cloudflare). Stocker les compteurs dans Redis si plusieurs instances.

**Algorithmes** : *token bucket* (défaut, tolère un burst), *leaky bucket* (débit lisse), fenêtre fixe/glissante (simple, mais frontières marquées). Décider **fail-open** (disponibilité) ou **fail-closed** (protection) si le store de comptage tombe : ne pas en faire un SPOF inconscient.

Réponse : `429` avec `Retry-After` (secondes) et `X-RateLimit-Limit` / `X-RateLimit-Remaining`, pour qu'un client bien élevé ralentisse au lieu de marteler.

---

## 💻 2. Exemples techniques

Avec DRF, un scope dédié au login compte par adresse e-mail dans Redis, tandis que les anonymes sont plafonnés globalement.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'api.throttling.LoginRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '60/hour',
        'login': '5/minute',
    },
}

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://redis:6379/1',
    }
}

# api/throttling.py
from rest_framework.throttling import SimpleRateThrottle


class LoginRateThrottle(SimpleRateThrottle):
    scope = 'login'

    def get_cache_key(self, request, view):
        email = (request.data.get('email') or '').strip().lower()
        if not email:
            return None
        return f'throttle_login:{email}'
```

Laravel transmet les en-têtes de limitation au callback de réponse : le client sait quand réessayer sans marteler l'API.

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        RateLimiter::for('api', function (Request $request) {
            return Limit::perMinute(60)
                ->by($request->user()?->id ?: $request->ip())
                ->response(function (Request $request, array $headers) {
                    return response()->json([
                        'code' => 'RATE_LIMITED',
                        'message' => 'Trop de requêtes, réessayez plus tard.',
                    ], 429, $headers);
                });
        });
    }
}
```

En amont, Nginx porte une première limite par IP et renvoie lui aussi un `429` plutôt que son `503` par défaut.

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_status 429;

server {
    listen 443 ssl http2;
    server_name api.example.com;

    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend_upstream;
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
- [Sujet 17 — Gestion d'erreurs globale](17-erreurs-globales.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Sujet 31 — Conteneurisation, CI/CD, health checks](31-conteneurisation-cicd-health-checks.md)
