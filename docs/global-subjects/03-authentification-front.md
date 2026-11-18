# 3. Authentification côté front

> 🎯 **Résumé.** Gère l'identité de l'utilisateur côté client en privilégiant des flux sûrs (Authorization Code + PKCE ou session cookie sur domaine partagé) et un stockage de jetons résistant au XSS. Le front orchestre l'envoi des credentials, le rafraîchissement des tokens et la protection des routes pour l'UX. Le piège principal est de confondre garde-front et sécurité réelle, et de mal gérer les refresh concurrents en cas de multiples 401.

---

## 🧭 1. Contexte

**Principe**
- Flux recommandé pour une SPA : **Authorization Code + PKCE** (OIDC), ou session cookie classique si front et back partagent le domaine.
- Stockage du token : un cookie `HttpOnly; Secure; SameSite` est plus sûr que `localStorage` (vulnérable au XSS).
- **Refresh token** : rotation, durée courte pour l'access token.
- Pattern **BFF** (Backend For Frontend) : le serveur garde les tokens, le navigateur n'a qu'un cookie de session.

| Stack | Mécanisme |
|---|---|
| React | interceptors axios/fetch wrapper, routes protégées (React Router loaders), NextAuth/Auth.js |
| Vue | `router.beforeEach`, interceptors axios, store Pinia pour l'utilisateur |
| Angular | `HttpInterceptorFn` (`withInterceptors`), guards `CanActivateFn` |
| Svelte | `hooks.server.ts` (SvelteKit), `load` protégés |

```ts
// Interceptor Angular fonctionnel : ajoute le token et gère le 401
export const authInterceptor: HttpInterceptorFn = (req, next) =>
  next(req.clone({ withCredentials: true }));
```

> ⚠️ **Pièges.** refresh concurrent (plusieurs 401 en parallèle déclenchent plusieurs refresh : mutualiser avec une seule promesse) ; guard front = confort UX, **jamais** une vraie sécurité (le back revérifie tout).

---

## 💻 2. Exemples techniques

Exemple React avec un wrapper `fetch` qui mutualise le refresh sur 401 pour éviter plusieurs appels simultanés.

```ts
let refreshPromise: Promise<void> | null = null;

async function refreshSession(): Promise<void> {
  const res = await fetch('/api/auth/refresh', {
    method: 'POST',
    credentials: 'include',
  });
  if (!res.ok) throw new Error('Session expirée');
}

export async function apiFetch(input: RequestInfo, init: RequestInit = {}): Promise<Response> {
  const request = () =>
    fetch(input, { ...init, credentials: 'include', headers: { ...(init.headers ?? {}) } });

  let response = await request();
  if (response.status !== 401) return response;

  if (!refreshPromise) {
    refreshPromise = refreshSession().finally(() => {
      refreshPromise = null;
    });
  }

  await refreshPromise;
  response = await request();
  return response;
}
```

Exemple Vue avec garde de navigation global et store de session pour rediriger les routes protégées.

```ts
import { createRouter, createWebHistory } from 'vue-router'
import { useSessionStore } from '@/stores/session'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/login', name: 'login', component: () => import('@/pages/LoginPage.vue') },
    {
      path: '/dashboard',
      name: 'dashboard',
      component: () => import('@/pages/DashboardPage.vue'),
      meta: { requiresAuth: true },
    },
  ],
})

router.beforeEach((to) => {
  const session = useSessionStore()
  if (to.meta.requiresAuth && !session.isAuthenticated) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})

export default router
```

Exemple Angular avec `HttpInterceptorFn` et `CanActivateFn` en mode cookie de session (pattern BFF).

```ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  return next(req.clone({ withCredentials: true })).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        router.navigate(['/login']);
      }
      return throwError(() => error);
    }),
  );
};

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isAuthenticated() ? true : router.createUrlTree(['/login']);
};
```

---

## 🔗 3. Voir aussi

- [Sujet 5 — Routing, guards et code splitting](05-routing-guards-code-splitting.md)
- [Sujet 12 — Sécurité front](12-securite-front.md)
- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
