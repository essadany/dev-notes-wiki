# 18. Cache backend et cache HTTP

> 🎯 **Résumé.** Le cache se déploie en couches, du CDN jusqu'au cache applicatif, chacune avec sa clé et son TTL : on gagne en latence mais on ajoute un mode de panne par couche. On choisit explicitement la stratégie de lecture (cache-aside par défaut), d'écriture (write-through, write-back, write-around) et d'expiration (TTL + jitter). Le piège principal est l'invalidation : une clé oubliée, partagée entre utilisateurs ou non versionnée transforme un gain de performances en fuite de données ou en données fausses silencieuses.

---

## 🧭 1. Contexte

**Couches, de l'extérieur vers l'intérieur** : navigateur → **CDN/edge** → cache applicatif (en mémoire) → cache distribué (Redis) → base. Chaque couche a son TTL et sa clé propres.

**Stratégies à choisir explicitement**
- **Lecture** : *cache-aside* (l'app consulte le cache, sur miss lit la base et remplit) = défaut, car une panne de cache rend seulement l'app plus lente ; *read-through* (la lib charge elle-même) si l'on veut centraliser la politique.
- **Écriture** : *write-through* (cache + base synchrones, lectures fraîches après écriture) ; *write-back* (cache d'abord, flush async — fenêtre de perte) ; *write-around* (base seulement, évite le churn des données jamais relues).
- **Expiration** : LRU (récence), LFU (popularité stable), TTL pour borner la fraîcheur. **Toujours un TTL + jitter** pour éviter une expiration synchronisée.

| Stack | Cache applicatif |
|---|---|
| Spring Boot | `@EnableCaching`, `@Cacheable`, `@CacheEvict`, Redis/Caffeine |
| Django | framework de cache (backend Redis natif), `cache_page`, `cache.get/set` |
| Laravel | `Cache::remember()`, tags (selon driver), Redis |
| NestJS | `@nestjs/cache-manager` (`CacheInterceptor`) |
| Express | `node-cache`, `ioredis` ; ETag géré par Express par défaut |

```java
@Cacheable(value = "users", key = "#id")
public User find(Long id) { ... }
```

**Cache HTTP / edge** : `Cache-Control` (`max-age`, `immutable`, `stale-while-revalidate`), `ETag` / `If-None-Match` → `304 Not Modified`, `Last-Modified`. Pour les assets statiques : **URLs fingerprintées** (`app.4f9a.js`) + `max-age` long, plutôt qu'une invalidation. Clé de cache = chemin + liste blanche de paramètres ; ne pas mettre de cookie dans la clé sur les chemins statiques.

## ⚠️ 2. Pièges
- Invalidation (« les deux choses difficiles en informatique ») : invalider à l'écriture, ou write-through, ou TTL court — choisir selon le budget de fraîcheur.
- Clé sans identifiant utilisateur/tenant → fuite de données dans un cache partagé. Préfixer et **versionner** les clés (`users:v2:123`) quand le schéma change.
- *Cache stampede* / thundering herd (hot key expirée, redémarrage) : **single-flight** (coalescing des requêtes), refresh probabiliste anticipé, TTL avec jitter, *origin shield* derrière le CDN.
- *Cache penetration* (clés inexistantes) : mettre en cache le résultat négatif avec un TTL court, ou filtre de Bloom.
- *Hot key* : répliquer la clé, ajouter un niveau L1 local, ou sharder la valeur.
- Cache froid après déploiement : préchauffer les clés critiques et monter en charge progressivement.
- Surveiller le **hit rate** (viser 90 %+), le taux d'éviction, la latence p99 et le QPS à l'origine (le signal qui grimpe quand le cache tombe).

---

## 💻 3. Exemples techniques

Cache applicatif Spring avec Caffeine : nom de cache versionné, clé incluant le tenant, TTL borné et éviction à l'écriture :

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager("users:v2");
        manager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(10))
            .maximumSize(10_000));
        return manager;
    }
}

@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }

    @Cacheable(cacheNames = "users:v2", key = "#tenantId + ':' + #id")
    public User find(Long tenantId, Long id) {
        return repo.findByTenantIdAndId(tenantId, id).orElseThrow();
    }

    @CacheEvict(cacheNames = "users:v2", key = "#tenantId + ':' + #user.id")
    public User update(Long tenantId, User user) {
        return repo.save(user);
    }
}
```

Laravel : `Cache::remember()` avec clé préfixée par tenant, versionnée, et TTL avec jitter ; le résultat négatif est lui aussi mis en cache brièvement pour éviter la *cache penetration* :

```php
use Illuminate\Support\Facades\Cache;

function findUser(int $tenantId, int $id): ?array
{
    $key = "users:v2:{$tenantId}:{$id}";
    $ttl = 600 + random_int(0, 60);

    return Cache::remember($key, $ttl, function () use ($tenantId, $id) {
        $user = User::where('tenant_id', $tenantId)->find($id);

        if ($user === null) {
            Cache::put("users:v2:{$tenantId}:missing:{$id}", true, 30);
            return null;
        }

        return $user->toArray();
    });
}
```

Express : assets fingerprintés servis avec `immutable`, réponse API avec `ETag` et renvoi d'un `304` via `req.fresh` quand le client renvoie `If-None-Match` :

```ts
import express from 'express';
import { PrismaClient } from '@prisma/client';

const app = express();
const db = new PrismaClient();

app.use(
  '/assets',
  express.static('public/assets', {
    immutable: true,
    maxAge: '1y',
    setHeaders(res) {
      res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
    },
  }),
);

app.get('/api/users/:id', async (req, res) => {
  const user = await db.user.findUniqueOrThrow({
    where: { id: Number(req.params.id) },
  });

  res.setHeader('Cache-Control', 'private, max-age=0, must-revalidate');
  res.setHeader('ETag', `"${user.updatedAt.getTime()}"`);

  if (req.fresh) {
    res.status(304).end();
    return;
  }

  res.json(user);
});

app.listen(3000);
```

---

## 🔗 4. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 6 — Performance (listes longues, debounce, annulation)](06-performance-front.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Caching: the secret behind it all](../backend-from-first-principles/13-caching-the-secret-behind-it-all.md)
