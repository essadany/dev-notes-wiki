# 24. Logging et observabilité

> 🎯 **Résumé.** Une alerte « CPU > 80 % » arrive toujours après que les utilisateurs se sont plaints, et sans correlation id on ne retrouve pas la requête fautive. On collecte métriques, logs structurés et traces, avec des endpoints de santé distincts et des alertes sur des symptômes utilisateurs (SLO). Le piège : logger de façon synchrone sur le chemin de requête ou vérifier la base dans la liveness — dans les deux cas, l'observabilité devient la panne.

---

## 🧭 1. Contexte

**Trois piliers complémentaires** : **métriques** (séries numériques, pour dashboards et alertes), **logs** (événements contextualisés, pour déboguer *cette* requête après l'alerte), **traces** (parcours d'une requête à travers les services, pour trouver le saut lent/faillant).

**Cadres de mesure** : **RED** (Rate, Errors, Duration) pour les services qui servent des requêtes ; **USE** (Utilization, Saturation, Errors) pour les ressources (CPU, disque, pool de connexions, files). Quatre signaux d'or : latence, trafic, erreurs, saturation.

| Stack | Logs | Métriques / traces |
|---|---|---|
| Spring | SLF4J + Logback | Actuator + Micrometer, OpenTelemetry |
| Django | `logging`, structlog | OpenTelemetry, Sentry |
| Laravel | Monolog | Telescope (dev), Sentry, OpenTelemetry |
| Node | pino, winston | prom-client, OpenTelemetry |

**Bonnes pratiques**
- Logs **structurés** (JSON) avec `ts`, `level`, `service`, `trace_id`/`span_id` (W3C `traceparent`), `msg`. Le **correlation id** est posé à l'entrée et propagé à chaque appel sortant : une requête = une seule recherche. Niveaux cohérents (INFO/WARN/ERROR).
- **Endpoints de santé distincts** : `/livez` = processus vivant, **sans vérifier les dépendances** (échec ⇒ redémarrage) ; `/readyz` = dépendances et warmup OK (échec ⇒ retirer du pool sans redémarrer). Inclure la version/build dans la réponse.
- **Alerter sur les symptômes utilisateurs** (SLO, ex. 99,9 % des requêtes < 300 ms sur 30 jours ≈ 43 min de budget d'erreur), pas sur « CPU > 80 % » (ça, c'est un ticket). Alertes actionnables avec runbook ; regrouper les alertes corrélées pour éviter l'orage d'alertes.
- Mesurer **p95/p99**, pas seulement la moyenne. Échantillonner les traces (1–10 % à fort QPS). Bannir les labels à forte cardinalité (`user_id`, URL brute) des métriques.
- Télémétrie **asynchrone et échantillonnée** : jamais de logging synchrone ni d'export de traces non samplé sur le chemin de requête — sinon l'observabilité devient la panne.
- Pipeline de logs à fort volume : agent local (Fluent Bit/Vector) → buffer borné → bus durable si pics ou multiples sinks → index (Elasticsearch pour recherche riche, Loki pour indexation par labels) → archivage froid en stockage objet. **Drop/sample sur débordement, jamais bloquer l'app**, et mesurer le taux de perte.
- Ne **jamais** logger mots de passe, tokens ou données personnelles. Surveiller le watcher : latence d'export, taux de perte, état des collecteurs (Grafana/Prometheus, Sentry, OpenTelemetry).

---

## 💻 2. Exemples techniques

Le logger pino enrichit chaque ligne du `trace_id` et du `span_id` du span OpenTelemetry actif, ce qui permet une seule recherche par requête.

```ts
// observability/logger.ts
import { trace } from '@opentelemetry/api';
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: { service: 'orders-api' },
  mixin() {
    const spanContext = trace.getActiveSpan()?.spanContext();
    return spanContext
      ? { trace_id: spanContext.traceId, span_id: spanContext.spanId }
      : {};
  },
});

logger.info({ orderId: 4217 }, 'commande acceptée');
```

Spring Boot Actuator expose des groupes de santé séparés pour la liveness et la readiness, avec les informations de build dans la réponse.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState,db
  info:
    build:
      enabled: true
```

Une alerte Prometheus porte sur le SLO utilisateur (99,9 % de requêtes servies en moins de 300 ms) et renvoie vers un runbook, pas sur un seuil CPU.

```yaml
groups:
  - name: api-slo
    rules:
      - alert: ApiLatencySloBurn
        expr: |
          sum(rate(http_server_requests_seconds_bucket{le="0.3",status!~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m])) < 0.999
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "SLO latence API en dessous de 99,9 %"
          runbook_url: "https://wiki.example.com/runbooks/api-latency"
```

---

## 🔗 3. Voir aussi

- [Sujet 17 — Gestion d'erreurs globale](17-erreurs-globales.md)
- [Sujet 22 — Jobs asynchrones et files de tâches](22-jobs-asynchrones-files.md)
- [Sujet 31 — Conteneurisation, CI/CD, health checks](31-conteneurisation-cicd-health-checks.md)
- [Logging, monitoring et observabilité (first principles)](../backend-from-first-principles/18-logging-monitoring-observability.md)
