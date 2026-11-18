# 35. Communication inter-services et résilience

> 🎯 **Résumé.** Dès qu'un service appelle une dépendance sur le réseau, la latence et les pannes s'additionnent : un retry sans budget transforme un micro-incident en *retry storm*. La réponse est un timeout calé sur le p99 mesuré, un circuit breaker, un bulkhead, un fallback explicite et un outbox pour ne jamais perdre un événement. Le piège : traiter la résilience comme une bibliothèque à ajouter à la fin, alors qu'elle conditionne le graceful shutdown, l'observabilité et le budget d'erreur dès le jour 1.

---

## 🧭 1. Contexte

**Synchrone** : REST (simple, universel), **gRPC** (binaire, typé, performant, adapté au service-à-service), GraphQL (agrégation pour le front).
**Asynchrone** : messages/événements via RabbitMQ, Kafka, SQS, Redis Streams. Découple les services dans le temps.

**Patterns de résilience**
- **Timeout** sur chaque appel externe (jamais d'attente infinie).
- **Retry** avec backoff exponentiel et jitter, seulement sur les opérations idempotentes.
- **Circuit breaker** : après N échecs, on arrête d'appeler un service en panne pendant un temps.
- **Bulkhead** : isoler les ressources pour qu'un service lent ne bloque pas tout.
- **Fallback** : réponse dégradée plutôt qu'une erreur.

```
 requête entrante
        │
        ▼
 ┌──────────────────────────────────────────────────────────┐
 │ timeout (p99)  ──▶  retry budget + jitter  ──▶  breaker  │
 └─────────────────────────────┬────────────────────────────┘
                               ▼
                  ┌────────────────────────┐
                  │      dépendance        │
                  └────────────┬───────────┘
                               ▼
        ┌──────────────────────┼────────────────────────┐
        ▼                      ▼                        ▼
      ✅ OK          ⚠️ dégradé {stale: true}      ❌ 5xx (fail-closed)
                        + cache + as_of              ou fail-open
```

**Précisions**
- **Timeout = p99 mesuré** de la dépendance (pas une valeur devinée) : c'est la défense de base, partout.
- **Retry budget** : plafonner le nombre total de tentatives par requête, avec jitter, sinon les retries se multiplient en cascade (retry storm). Ne réessayer une écriture qu'avec une `Idempotency-Key` (sujet 20).
- **Dégradation explicite** : renvoyer `{ "data": [...], "stale": true, "source": "cache", "as_of": "..." }` plutôt qu'une réponse normale qui ment. Décider délibérément fail-open ou fail-closed pour un rate limiter / circuit breaker.
- **Bulkhead** : pools séparés par dépendance, pour qu'un service lent n'épuise pas tous les threads.
- **Découverte des pairs** : registre (DNS, etcd/Consul/ZooKeeper) et, seulement pour un grand nombre de services, un service mesh (mTLS, retries, traffic shaping). Trop tôt, le mesh coûte plus qu'il n'apporte.
- **Reprise après panne** : half-open du circuit breaker, reconnexions avec jitter, slow-start — ne pas rendre toute la charge d'un coup à un service qui revient (thundering herd).

| Stack | Outils |
|---|---|
| Spring | Resilience4j (circuit breaker, retry, rate limiter, bulkhead), `RestClient`/`WebClient`, Spring Cloud |
| Python | `tenacity` (retry), `pybreaker`, httpx avec timeouts |
| Laravel | `Http::retry()`, `timeout()`, jobs avec `tries`/`backoff` |
| Node | `opossum` (circuit breaker), `axios-retry`, `p-retry` |

**Cohérence des données**
- **Outbox pattern** : écrire l'événement dans une table de la **même transaction** que la modification métier, puis un processus le publie sur le broker. Évite « base mise à jour mais événement perdu ».
- **Saga** : suite de transactions locales avec actions de compensation (orchestration ou chorégraphie).
- **Consommateurs idempotents** : un message peut arriver plusieurs fois (at-least-once).

> ⚠️ **Pièges.** appels synchrones en chaîne (la latence et les pannes s'additionnent) ; oublier la version/le schéma des événements (compatibilité ascendante) ; pas de *correlation id* pour tracer une requête à travers les services (sujet 24).

## 🧠 2. Logique first-principles

**Timeout = p99 mesuré de la dépendance.** On ne devine pas 2 secondes : on relève le p99 réel de l'appel (par exemple 320 ms) et on place le timeout juste au-dessus. C'est la première ligne de défense, partout, y compris sur les appels internes. Une moyenne de 50 ms peut masquer 1 % de requêtes à 5 s : ce sont souvent les opérations métier les plus précieuses.

**Retry budget + jitter.** Plafonner le total des tentatives par requête (par exemple 3 au lieu de 3 par service), espacer avec un backoff exponentiel randomisé, et ne retenter une écriture qu'avec une `Idempotency-Key`. Sans budget, N services en chaîne multiplient les appels et le service en difficulté reçoit le trafic de tout le monde au moment où il se rétablit.

**Graceful shutdown : le pendant local de la résilience.** Sur `SIGTERM` (orchestrateur) ou `SIGINT` (`Ctrl+C`), arrêter d'accepter du trafic, faire échouer `/health` (`/readyz`) pour que le load balancer retire l'instance, drainer les requêtes en vol avec un timeout dur de **30 à 60 s**, puis libérer les ressources **en ordre inverse de leur acquisition** (HTTP → BDD → Redis/files) et sortir avec `Exit 0`. C'est le facteur 12-factor *Disposability* : boot rapide, arrêt propre, et conception *crash-only* pour survivre à un `SIGKILL`.

**Dégradation explicite.** Un fallback qui renvoie des données périmées sans le dire est un mensonge : la réponse doit porter `stale: true`, `source` et `as_of`. Décider consciemment *fail-open* ou *fail-closed* pour chaque rate limiter ou circuit breaker, et masquer le bloc concerné plutôt que de renvoyer un 500 global.

**Outbox pattern et idempotence des consommateurs.** La modification métier et l'événement sont écrits dans la même transaction PostgreSQL, puis un publisher vacuum la table `FOR UPDATE SKIP LOCKED`. Un message étant livré au moins une fois, le consommateur doit dédupliquer par identifiant d'événement et incrémenter un `schema_version` pour garder la compatibilité ascendante.

**Observabilité dès le jour 1 et SLO/error budget.** Logs JSON structurés avec `correlation_id`, métriques et tracing distribué avant le premier incident : sans trace, un appel synchrone en chaîne est impossible à diagnostiquer. Un SLO à 99,9 % laisse 0,1 % d'erreurs, soit environ 43 minutes par mois ; budget épuisé = gel des déploiements. On alerte sur les **symptômes utilisateurs** (taux de succès, p95/p99) et non sur le CPU.

---

## 💻 3. Exemples techniques

Cette configuration Resilience4j traduit directement les règles : timeout calé sur un p99 mesuré à 320 ms, 3 tentatives maximum avec jitter, circuit breaker à half-open et bulkhead par dépendance :

```yaml
resilience4j:
  timelimiter:
    configs:
      default:
        timeoutDuration: 320ms
        cancelRunningFuture: true
  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 50ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 8
        maxWaitDuration: 0
  ratelimiter:
    configs:
      default:
        limitForPeriod: 50
        limitRefreshPeriod: 1s
        timeoutDuration: 0

management:
  endpoint:
    health:
      show-details: always
  endpoints:
    web:
      exposure:
        include: health,metrics
```

La table d'outbox et l'écriture métier partagent la même transaction ; le publisher récupère ensuite les événements non publiés sans verrouiller tout le monde :

```sql
CREATE TABLE outbox_events (
    id             UUID PRIMARY KEY,
    aggregate_id   UUID NOT NULL,
    event_type     TEXT NOT NULL,
    payload        JSONB NOT NULL,
    schema_version INTEGER NOT NULL DEFAULT 1,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at   TIMESTAMPTZ
);

CREATE INDEX outbox_events_unpublished_idx
    ON outbox_events (created_at)
    WHERE published_at IS NULL;

BEGIN;

UPDATE orders
SET status = 'paid', updated_at = now()
WHERE id = '3f9c1c0e-6b2a-4d41-9f18-2c7a5e0b31d4';

INSERT INTO outbox_events (id, aggregate_id, event_type, payload, schema_version)
VALUES (
    'b6f1d2c4-8e37-4a92-b15d-0c6f9a2e5741',
    '3f9c1c0e-6b2a-4d41-9f18-2c7a5e0b31d4',
    'order.paid',
    '{"order_id":"3f9c1c0e-6b2a-4d41-9f18-2c7a5e0b31d4","amount_cents":4990,"currency":"EUR"}'::jsonb,
    1
);

COMMIT;

SELECT id, event_type, payload, schema_version
FROM outbox_events
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

Ce serveur Express enregistre `SIGTERM`/`SIGINT`, fait échouer la readiness pour retirer le pod du load balancer, draine les requêtes puis ferme HTTP → Postgres → Redis avant `Exit 0` :

```js
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });
const redis = new Redis(process.env.REDIS_URL);
let ready = true;

app.get('/livez', (req, res) => res.status(200).json({ status: 'alive' }));
app.get('/readyz', (req, res) => {
  if (ready) return res.status(200).json({ status: 'ready' });
  return res.status(503).json({ status: 'draining' });
});
app.get('/orders/:id', async (req, res) => {
  const { rows } = await pool.query('SELECT id, status FROM orders WHERE id = $1', [req.params.id]);
  if (rows.length === 0) return res.status(404).json({ error: 'not found' });
  return res.json(rows[0]);
});

const server = app.listen(8080, () => console.log('listening on 8080'));

function shutdown(signal) {
  if (!ready) return;
  ready = false;

  const hardTimeout = setTimeout(() => {
    console.error(`timeout de grâce dépassé après ${signal}`);
    process.exit(1);
  }, 60000);
  hardTimeout.unref();

  server.close(async () => {
    try {
      await pool.end();
      await redis.quit();
      clearTimeout(hardTimeout);
      console.log(`graceful shutdown terminé (${signal})`);
      process.exit(0);
    } catch (error) {
      console.error('échec du nettoyage', error);
      process.exit(1);
    }
  });
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

Kubernetes doit laisser au moins ce délai au processus avant de passer en `SIGKILL`, avec une readiness probe courte pour le retrait immédiat du pool :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      image: api:9f3c1ab2e4f6
      ports:
        - containerPort: 8080
      readinessProbe:
        httpGet:
          path: /readyz
          port: 8080
        periodSeconds: 2
        failureThreshold: 2
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

---

## 🔗 4. Voir aussi

- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
- [Sujet 22 — Jobs asynchrones et files](22-jobs-asynchrones-files.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Gestion des erreurs et tolérance aux pannes](../backend-from-first-principles/16-error-handling-and-fault-tolerance.md)
