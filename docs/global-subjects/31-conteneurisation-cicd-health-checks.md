# 31. Conteneurisation, CI/CD, health checks

> 🎯 **Résumé.** Une image reconstruite différemment entre staging et prod, ou un déploiement sans procédure de rollback, transforme chaque livraison en loterie. On fige l'artefact par son SHA, on sépare strictement build, release et run, et on distingue `/livez` de `/readyz` pour ne jamais redémarrer toute la flotte à cause d'une dépendance lente. Le piège principal : des health checks trop agressifs qui DDoS un service qui redémarre.

---

## 🧭 1. Contexte

- **Docker** : build multi-stage, image minimale, utilisateur non-root, pas de secret dans l'image, `.dockerignore`, **limites de ressources** (CPU/mémoire), tag immuable (SHA) — jamais `latest` en prod.
- **Infrastructure as Code** (Terraform/Pulumi) et **GitOps** (ArgoCD/Flux) pour Kubernetes : aucune modification manuelle, diffs vérifiés avant apply.
- **CI/CD** (GitLab CI, GitHub Actions) : lint → tests → build → **scan d'image et de dépendances** (Trivy, Dependabot) → staging → smoke tests → prod avec **approbation explicite**. Même image du staging à la prod. **Documenter et vérifier la procédure de rollback** (ex. `kubectl rollout undo`) avant de livrer.
- **Health checks** : `/livez` (processus seulement) vs `/readyz` (DB et dépendances) — voir sujet 24. Spring : Actuator (`/actuator/health`) ; Laravel : route `/up` ; Django/Node : endpoint dédié. Ne jamais mettre de dépendance lente dans la liveness, sous peine de restart loop de toute la flotte.
- **Déploiement sans coupure** : rolling update / blue-green / canary, migrations rétro-compatibles (sujet 21), *graceful shutdown* et drain des connexions, *slow-start* derrière le LB pour ne pas écraser une instance qui revient.
- **Reverse proxy / LB** (Nginx, Traefik) : TLS, compression, en-têtes de sécurité, rate limiting en amont, health checks à seuils généreux (N échecs avant retrait, M succès avant retour) — un check trop agressif peut DDoS un service qui redémarre. Actif-actif ou actif-passif pour ne pas faire du LB un SPOF.

---

## 💻 2. Exemples techniques

Un Dockerfile multi-stage garde le compilateur hors de l'image finale, exécute sous un utilisateur non-root et ne contient aucun secret :

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S -G app app
COPY --from=build --chown=app:app /app/package.json /app/package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build --chown=app:app /app/dist ./dist
USER app
EXPOSE 8080
CMD ["node", "dist/server.js"]
```

Le pipeline pousse **une seule image taggée par SHA**, la scanne, la smoke-teste en staging puis attend une approbation sur l'environnement de production ; le rollback est intégré au job :

```yaml
name: ci

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --runInBand
      - run: npm run build
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          scan-ref: .
          severity: CRITICAL,HIGH
          exit-code: "1"
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: image
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: CRITICAL
          exit-code: "1"

  staging:
    needs: verify
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Smoke test
        run: curl --fail --silent --show-error "${{ vars.STAGING_URL }}/readyz"

  production:
    needs: staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Rollout de l'image validée en staging
        run: kubectl set image deployment/api "api=ghcr.io/${{ github.repository }}:${{ github.sha }}"
      - name: Attente de la fin du rollout
        run: kubectl rollout status deployment/api --timeout=180s
      - name: Rollback automatique en cas d'échec
        if: failure()
        run: kubectl rollout undo deployment/api
```

Cette spécification Kubernetes sépare les sondes, plafonne les ressources, garde `maxUnavailable: 0` et laisse 45 s pour le drain :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: api
          image: api:9f3c1ab2e4f6
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          startupProbe:
            httpGet:
              path: /livez
              port: 8080
            periodSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /livez
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5
            successThreshold: 2
            failureThreshold: 3
```

---

## 🔗 3. Voir aussi

- [Sujet 21 — Migrations base de données](21-migrations-base-de-donnees.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Sujet 25 — Configuration et secrets](25-configuration-secrets.md)
- [12-Factor App — Build, Release, Run](../backend-from-first-principles/27-twelve-factor-app.md)
