# 8. Gestion d'erreurs côté front

> 🎯 **Résumé.** Structure le traitement des erreurs en distinguant les échecs de requête, les erreurs de rendu et les incidents globaux à monitorer. L'objectif est d'offrir un retour utilisateur compréhensible tout en capturant les informations utiles au diagnostic. Le piège principal est d'avaler les erreurs en silence, d'exposer des détails techniques bruts ou de ne pas différencier 4xx et 5xx.

---

## 🧭 1. Contexte

**Principe** : trois niveaux : erreur d'une requête (affichage local), erreur de rendu (frontière), erreur globale (monitoring).

| Stack | Outil |
|---|---|
| React | Error Boundary (`react-error-boundary`) |
| Vue | `onErrorCaptured`, `app.config.errorHandler` |
| Angular | `ErrorHandler` global + interceptor HTTP |
| Tous | Sentry (ou équivalent), toasts pour les erreurs utilisateur |

> ⚠️ **Pièges.** avaler les erreurs en silence ; afficher un message technique brut à l'utilisateur ; ne pas distinguer 4xx (action utilisateur) et 5xx (retry/support).

---

## 💻 2. Exemples techniques

Exemple React avec `react-error-boundary` pour isoler une erreur de rendu et proposer une action de reprise.

```tsx
import { ErrorBoundary, type FallbackProps } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <p>Une erreur inattendue est survenue.</p>
      <pre>{import.meta.env.DEV ? error.message : 'Réessayez plus tard.'}</pre>
      <button type="button" onClick={resetErrorBoundary}>Réessayer</button>
    </div>
  );
}

export function ProfilePanel() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback} onReset={() => location.reload()}>
      <ProfileContent />
    </ErrorBoundary>
  );
}
```

Exemple Vue avec capture locale des erreurs de rendu et gestionnaire global pour le monitoring.

```ts
import { createApp } from 'vue'
import * as Sentry from '@sentry/vue'
import App from './App.vue'

const app = createApp(App)

Sentry.init({ app, dsn: import.meta.env.VITE_SENTRY_DSN })

app.config.errorHandler = (err, instance, info) => {
  console.error('Erreur globale Vue', err, info)
  Sentry.captureException(err)
}

app.mount('#app')
```

```vue
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const hasError = ref(false)

onErrorCaptured((err) => {
  hasError.value = true
  console.error('Erreur dans le sous-arbre', err)
  return false
})
</script>

<template>
  <p v-if="hasError" role="alert">Impossible d'afficher ce bloc.</p>
  <slot v-else />
</template>
```

Exemple Angular avec `ErrorHandler` global et interceptor HTTP qui différencie 4xx et 5xx.

```ts
import { ErrorHandler, Injectable, inject } from '@angular/core';
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from './notification.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  handleError(error: unknown): void {
    console.error('Erreur applicative', error);
  }
}

export const httpErrorInterceptor: HttpInterceptorFn = (req, next) => {
  const notify = inject(NotificationService);
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status >= 400 && error.status < 500) {
        notify.warn('Vérifiez les informations saisies.');
      } else if (error.status >= 500) {
        notify.error('Service momentanément indisponible. Réessayez plus tard.');
      }
      return throwError(() => error);
    }),
  );
};
```

---

## 🔗 3. Voir aussi

- [Sujet 4 — Formulaires et validation](04-formulaires-validation.md)
- [Sujet 9 — Mises à jour optimistes](09-mises-a-jour-optimistes.md)
- [Sujet 17 — Gestion d'erreurs globale](17-erreurs-globales.md)
