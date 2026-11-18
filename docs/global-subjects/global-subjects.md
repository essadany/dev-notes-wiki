# Guide des sujets classiques d'une app web pro (front & back)

Ce guide regroupe les décisions d'architecture, de sécurité, de performance et de qualité qui reviennent dans la majorité des applications web professionnelles — du cache TanStack Query au graceful shutdown, en passant par l'idempotence des webhooks et le multi-tenancy.

> 🎯 **Résumé.** Une page par sujet : le problème, le principe, les pièges, les outils par stack, puis des exemples de code. Ces fiches sont volontairement indépendantes : on les lit selon le besoin du moment, pas dans l'ordre.

```
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
│     🎨 FRONTEND    │   │     ⚙️ BACKEND     │   │  🏗️ ARCHITECTURE   │
│      13 sujets     │   │      18 sujets     │   │      9 sujets      │
│   rendu, état, UX  │   │  API, DB, sécurité │   │  tenant, jobs, SLO │
└─────────┬──────────┘   └─────────┬──────────┘   └─────────┬──────────┘
          └────────────────────────┼────────────────────────┘
                                   ▼
                   ✅ Checklist avant mise en production
```

---

## 🧭 1. Comment utiliser ce guide

> 📌 **Règle.** Tout ce qui varie entre deux environnements (URL de base, clés d'API, credentials S3, port) vit dans les variables d'environnement ou un Secret Manager, jamais dans le code — c'est le facteur 4 du 12-Factor App.

**Principes directeurs**
- **Mesurer avant de deviner.** On n'ajoute un cache ou une couche qu'après avoir identifié le goulot avec le p95/p99 et le tracing.
- **Ne jamais faire confiance au client** — ni au navigateur, ni à un autre service.
- **Être idempotent par défaut** — clés d'idempotence, retries avec jitter, outbox.
- **Dégrader explicitement** — réponse partielle avec `stale: true` plutôt qu'erreur silencieuse.
- **Documents et code vivent ensemble** — OpenAPI, migrations et schémas versionnés.
- **Designer pour l'évolution** — monolithe modulaire d'abord, Extraction de service ensuite.

---

## 🗺️ 2. Sommaire

> 💡 **Navigation.** Chaque ligne pointe vers une fiche dédiée : résumé, pièges classiques, stack et exemples techniques.

### 🎨 2.1 Partie 1 — Frontend (13 sujets)

| # | Sujet | L'essentiel | Piège classique |
| :--- | :--- | :--- | :--- |
| 1 | [Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md) | clé = paramètres de requête, `staleTime`, invalidation | clés non invalidées ou partagées entre comptes |
| 2 | [Gestion d'état](02-gestion-etat.md) | état serveur (cache) / UI global (store) / local | recoder le cache dans Redux/Pinia |
| 3 | [Authentification côté front](03-authentification-front.md) | PKCE, cookie `HttpOnly`, refresh mutualisé | guard UX ≠ sécurité réelle |
| 4 | [Formulaires et validation](04-formulaires-validation.md) | schéma partagé, revalidation back, 422 | doubles soumissions |
| 5 | [Routing, guards et code splitting](05-routing-guards-code-splitting.md) | lazy loading par route, préchargement | pagination/filtres hors de l'URL |
| 6 | [Performance](06-performance-front.md) | virtualisation, debounce + annulation | optimiser sans mesurer |
| 7 | [Modes de rendu](07-modes-de-rendu.md) | CSR / SSR / SSG / ISR | `window` côté serveur, hydratation |
| 8 | [Gestion d'erreurs front](08-erreurs-front.md) | requête / rendu / global, toasts + monitoring | avaler les erreurs |
| 9 | [Mises à jour optimistes](09-mises-a-jour-optimistes.md) | `onMutate` → `onError` → `onSettled` | l'appliquer à un paiement |
| 10 | [Temps réel](10-temps-reel.md) | polling / SSE / WebSocket | reconnexion et cleanup oubliés |
| 11 | [i18n](11-i18n.md) | lib i18n + `Intl` | concaténer traductions |
| 12 | [Sécurité front](12-securite-front.md) | XSS, CSP, CSRF, dépendances | secret dans le bundle |
| 13 | [Tests front](13-tests-front.md) | Testing Library, MSW, Playwright | tester l'implémentation |

### ⚙️ 2.2 Partie 2 — Backend (18 sujets)

| # | Sujet | L'essentiel | Piège classique |
| :--- | :--- | :--- | :--- |
| 14 | [Pagination, tri et filtrage](14-pagination-tri-filtrage.md) | offset / curseur / slice, enveloppe stable | tri non déterministe, `limit` non plafonné |
| 15 | [Authentification et autorisation](15-authentification-autorisation.md) | session/JWT/OIDC, RBAC/ABAC | IDOR |
| 16 | [Validation des entrées](16-validation-entrees.md) | DTO distincts, forme puis métier | faire confiance au front |
| 17 | [Erreurs globales](17-erreurs-globales.md) | RFC 9457, `request_id`, `retryable` | enveloppe incohérente |
| 18 | [Cache backend et HTTP](18-cache-backend-http.md) | cache-aside, TTL+jitter, single-flight | clés partagées, invalidation |
| 19 | [N+1 et performance ORM](19-n1-performance-orm.md) | `JOIN FETCH`, `select_related`, `with` | JOIN massif sur collections |
| 20 | [Transactions et concurrence](20-transactions-concurrence.md) | ACID, verrous, idempotence, outbox | transaction ouverte sur un appel réseau |
| 21 | [Migrations de base de données](21-migrations-base-de-donnees.md) | schéma versionné, expand/contract | modifier une migration déployée |
| 22 | [Jobs asynchrones et files](22-jobs-asynchrones-files.md) | lease, DLQ, backpressure | croire à l'exactly-once |
| 23 | [Rate limiting](23-rate-limiting.md) | token bucket, Redis, `429` | store de comptage SPOF |
| 24 | [Logging et observabilité](24-logging-observabilite.md) | métriques/logs/traces, `/livez`≠`/readyz` | liveness qui vérifie la DB |
| 25 | [Configuration et secrets](25-configuration-secrets.md) | 12-factor, env, coffre | `.env` non chiffré |
| 26 | [Upload de fichiers](26-upload-fichiers.md) | URL pré-signée, blob store, clé en base | croire le MIME du client |
| 27 | [Documentation API et versioning](27-documentation-api-versioning.md) | OpenAPI, `/api/v1`, client TS généré | changer le sens d'un champ |
| 28 | [Soft delete et audit](28-soft-delete-audit.md) | `deleted_at`, filtre centralisé, Envers | `UNIQUE`, RGPD |
| 29 | [Sécurité backend](29-securite-backend.md) | OWASP, Argon2/bcrypt, contrôle objet | CORS `*` |
| 30 | [Tests backend](30-tests-backend.md) | pyramide, Testcontainers | SQLite en test, Postgres en prod |
| 31 | [Conteneurisation, CI/CD et health checks](31-conteneurisation-cicd-health-checks.md) | image SHA, pipeline, rollback | health checks trop agressifs |

### 🏗️ 2.3 Partie 3 — Architecture et sujets avancés (9 sujets)

| # | Sujet | L'essentiel | Piège classique |
| :--- | :--- | :--- | :--- |
| 32 | [Recherche full-text](32-recherche-full-text.md) | Postgres → Meilisearch → Elasticsearch | écrire sur le chemin de requête |
| 33 | [Architecture en couches, hexagonale, Clean](33-architecture-couches-hexagonale-clean.md) | domaine sans framework, ports/adapters, ArchUnit | sur-ingénierie, couches passe-plat |
| 34 | [Monolithe modulaire vs microservices](34-monolithe-modulaire-vs-microservices.md) | modulaire d'abord, statelessness, scale d'équipes | monolithe distribué |
| 35 | [Communication inter-services et résilience](35-communication-inter-services-resilience.md) | timeout=p99, retry budget, outbox, graceful shutdown | retry storm, fallback qui ment |
| 36 | [Multi-tenancy](36-multi-tenancy.md) | `tenant_id`, schéma/base, RLS | oublier caches et files |
| 37 | [Webhooks](37-webhooks.md) | HMAC sur corps brut, async, idempotence | signature sur JSON re-sérialisé |
| 38 | [Feature flags](38-feature-flags.md) | rollout progressif, kill switch | flags jamais retirés |
| 39 | [Export et génération de documents](39-export-documents.md) | synchrone streamé / async + URL pré-signée | charger tout en mémoire |
| 40 | [Dates, fuseaux, montants et identifiants](40-dates-fuseaux-montants-ids.md) | UTC/ISO 8601, décimal, UUIDv7/ULID | `float`, skew d'horloge |

---

## ✅ 3. Checklist rapide avant mise en production

> ⚠️ **À cocher avant chaque livraison.** Une case non cochée vaut une dette à traiter explicitement, pas un oubli à découvrir en incidents.

- [ ] Health checks (`/livez`, `/readyz`) distincts et non destructeurs
- [ ] Retry (importants) et backoff avec jitter ; timeouts calibrés sur le p99 mesuré
- [ ] Debounce + annulation des requêtes sur la recherche et le scroll
- [ ] Clés de cache complètes (params + utilisateur/tenant) et invalidées après mutation
- [ ] Validation des entrées côté serveur avec DTO dédiés
- [ ] Clé d'idempotence sur toute write retryable
- [ ] Cache HTTP avec `ETag`/`Cache-Control` et clés versionnées
- [ ] Test d'autorisation réel (IDOR) sur chaque ressource
- [ ] Error tracking actif avec `request_id` corrélé aux logs
- [ ] Secrets externalisés (12-factor), jamais dans le code ou le dépôt
- [ ] Migration réversible (expand/contract) testée sur une copie de prod
- [ ] Observabilité en place : métriques RED/USE, logs structurés, traces
- [ ] i18n : pluriels, formats de date/devise, débordements de texte vérifiés
- [ ] Documentation d'architecture et contrat OpenAPI à jour
- [ ] Pipeline CI/CD vert + rollback testé
- [ ] Requêtes multi-tenant isolées et testées
- [ ] Accessibilité : navigation clavier, labels, contrastes
- [ ] Feature flags avec kill switch documenté et date de retrait

---

## 🔗 4. Voir aussi

- [Backend from first principles](../backend-from-first-principles/index.md) — les raisonnements qui sous-tendent ces fiches
- [Langages de programmation](../langages/index.md) — sync/async et clients HTTP
- [Déploiement web](../deploiement-web/index.md) — EC2, DNS et Nginx
