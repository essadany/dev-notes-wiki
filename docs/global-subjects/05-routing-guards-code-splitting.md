# 5. Routing, guards et code splitting

> 🎯 **Résumé.** Structure la navigation d'une SPA en chargeant uniquement le code des routes affichées et en contrôlant l'accès aux écrans selon le contexte utilisateur. Les guards améliorent l'expérience (redirections, préchargements) mais ne remplacent pas les contrôles serveur. Le piège principal est de ne pas synchroniser l'état de pagination et de filtres avec l'URL, ce qui casse le partage de lien et le bouton retour.

---

## 🧭 1. Contexte

**Principe** : ne charger que le code de la page affichée (lazy loading par route).

| Stack | Lazy loading |
|---|---|
| React | `React.lazy` + `Suspense`, routes lazy de React Router |
| Vue | `component: () => import('./Page.vue')` dans le router, `defineAsyncComponent` |
| Angular | `loadComponent` / `loadChildren`, bloc `@defer` dans les templates |
| Svelte | découpage automatique par route avec SvelteKit |

Guards : `beforeEach` (Vue), `CanActivateFn` / `CanMatchFn` (Angular — `CanMatch` peut refuser le chargement d'une route lazy), `ResolveFn` pour précharger les données, loaders/redirect (React Router). Côté Angular, les View Transitions animent les changements de route.

> ⚠️ **Pièges.** état de l'URL non synchronisé (page, filtres) : mettre pagination et filtres dans la **query string** pour permettre partage de lien et bouton retour.

---

## 💻 2. Exemples techniques

Exemple React Router avec routes lazy, loader protégé et pagination synchronisée dans la query string.

```tsx
import { Suspense, lazy } from 'react';
import { createBrowserRouter, redirect, useSearchParams } from 'react-router-dom';

const UsersPage = lazy(() => import('./pages/UsersPage'));

async function protectedLoader() {
  const res = await fetch('/api/me', { credentials: 'include' });
  if (res.status === 401) throw redirect('/login');
  return null;
}

export const router = createBrowserRouter([
  {
    path: '/users',
    loader: protectedLoader,
    element: (
      <Suspense fallback={<p>Chargement…</p>}>
        <UsersPage />
      </Suspense>
    ),
  },
]);

export function UsersPagination() {
  const [params, setParams] = useSearchParams();
  const page = Number(params.get('page') ?? '1');
  const setPage = (next: number) => setParams({ page: String(next), sort: params.get('sort') ?? 'name' });
  return <button onClick={() => setPage(page + 1)}>Page suivante ({page + 1})</button>;
}
```

Exemple Vue Router avec import dynamique des pages et `beforeEach` pour protéger une route.

```ts
import { createRouter, createWebHistory } from 'vue-router'
import { useSessionStore } from '@/stores/session'

export const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: () => import('@/pages/HomePage.vue') },
    {
      path: '/admin',
      meta: { requiresAuth: true, role: 'admin' },
      component: () => import('@/pages/AdminPage.vue'),
    },
  ],
})

router.beforeEach((to) => {
  const session = useSessionStore()
  if (to.meta.requiresAuth && !session.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  if (to.meta.role && session.user?.role !== to.meta.role) {
    return { path: '/' }
  }
})
```

Exemple Angular avec `loadComponent`, `CanMatchFn` pour empêcher même le chargement d'une route lazy, et `ResolveFn` pour précharger des données.

```ts
import { inject } from '@angular/core';
import { CanMatchFn, ResolveFn, Routes } from '@angular/router';
import { HttpClient } from '@angular/common/http';
import { AuthService } from './auth.service';

type User = { id: string; name: string };

export const canMatchAdmin: CanMatchFn = () => {
  const auth = inject(AuthService);
  return auth.hasRole('admin');
};

export const usersResolver: ResolveFn<User[]> = () => {
  return inject(HttpClient).get<User[]>('/api/users');
};

export const routes: Routes = [
  {
    path: 'admin/users',
    canMatch: [canMatchAdmin],
    loadComponent: () =>
      import('./admin/users-page.component').then((m) => m.UsersPageComponent),
    resolve: { users: usersResolver },
  },
];
```

---

## 🔗 3. Voir aussi

- [Sujet 3 — Authentification côté front](03-authentification-front.md)
- [Sujet 6 — Performance (listes longues, debounce, annulation)](06-performance-front.md)
- [Sujet 7 — Modes de rendu (CSR, SSR, SSG, ISR)](07-modes-de-rendu.md)
