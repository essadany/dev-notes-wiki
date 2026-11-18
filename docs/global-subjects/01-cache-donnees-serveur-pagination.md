# 1. Cache des données serveur et pagination

> 🎯 **Résumé.** Évite de relancer un appel réseau à chaque retour sur une vue déjà consultée en mettant en cache les réponses serveur. Le cache est indexé par une clé incluant tous les paramètres de la requête, avec une durée de fraîcheur (`staleTime`) et une invalidation après mutation. Le piège principal est d'oublier l'invalidation des clés, de mal distinguer fraîcheur et rétention, ou de partager un cache entre utilisateurs.

---

## 🧭 1. Contexte

**Problème** : revenir à la page 2 ne doit pas relancer un appel réseau.

**Principe** : cache indexé par une clé = tous les paramètres de la requête (`['users', {page, size, sort, filter}]`), avec une durée de fraîcheur (`staleTime`) et une invalidation après mutation.

| Stack | Solution |
|---|---|
| React | TanStack Query (`useQuery`), RTK Query (si Redux), SWR |
| Vue | `@tanstack/vue-query`, Pinia Colada |
| Angular | `@tanstack/angular-query-experimental`, NgRx SignalStore, `resource()` / `rxResource()` (Angular 19+), `httpResource` (Angular 20+), `shareReplay` pour un besoin simple |
| Svelte | `@tanstack/svelte-query` |

```ts
useQuery({
  queryKey: ['users', { page, size }],
  queryFn: () => api.getUsers({ page, size }),
  staleTime: 60_000,
  placeholderData: keepPreviousData,
})
```

## ⚠️ 2. Pièges
- Oublier d'invalider les clés après un create/update/delete.
- `staleTime` à 0 par défaut : beaucoup de refetch si on ne le règle pas.
- Pagination offset : les éléments bougent pendant la navigation (préférer le curseur pour les flux vivants).
- Vider le cache à la déconnexion (`queryClient.clear()`).
- Distinguer `staleTime` (fraîcheur) et `gcTime` (rétention hors écran, ex-`cacheTime` en v5) ; versionner les clés (`users:v2:...`) quand la forme des données change.
- Inclure l'identifiant utilisateur/tenant dans la clé si les données sont propres au compte, sinon fuite entre utilisateurs.

---

## 💻 3. Exemples techniques

Exemple Vue avec TanStack Vue Query : la clé réactive inclut la page et le cache est invalidé après une mutation.

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import { keepPreviousData, useMutation, useQuery, useQueryClient } from '@tanstack/vue-query'
import { createUser, getUsers } from '@/api/users'

const route = useRoute()
const queryClient = useQueryClient()
const page = computed(() => Number(route.query.page ?? 1))
const size = 20

const usersQuery = useQuery({
  queryKey: computed(() => ['users', { page: page.value, size }]),
  queryFn: () => getUsers({ page: page.value, size }),
  staleTime: 60_000,
  placeholderData: keepPreviousData,
})

const createUserMutation = useMutation({
  mutationFn: createUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] })
  },
})
</script>

<template>
  <ul v-if="usersQuery.data.value">
    <li v-for="user in usersQuery.data.value.items" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

Exemple Angular avec `httpResource` (Angular 20+) pour une lecture paginée réactive pilotée par des signaux.

```ts
import { Component, computed, signal } from '@angular/core';
import { httpResource } from '@angular/common/http';

type UsersPage = {
  items: Array<{ id: string; name: string }>;
  total: number;
};

@Component({
  selector: 'app-users-page',
  template: `
    @if (users.isLoading()) {
      <p>Chargement…</p>
    } @else if (users.value(); as data) {
      <ul>
        @for (user of data.items; track user.id) {
          <li>{{ user.name }}</li>
        }
      </ul>
      <button type="button" (click)="prev()" [disabled]="page() === 1">Précédent</button>
      <button type="button" (click)="next()">Suivant</button>
    }
  `,
})
export class UsersPageComponent {
  readonly page = signal(1);
  readonly size = 20;

  readonly users = httpResource<UsersPage>(() => ({
    url: '/api/users',
    params: { page: this.page(), size: this.size() },
  }));

  prev(): void {
    this.page.update((p) => Math.max(1, p - 1));
  }

  next(): void {
    this.page.update((p) => p + 1);
  }
}
```

---

## 🔗 4. Voir aussi

- [Sujet 2 — Gestion d'état](02-gestion-etat.md)
- [Sujet 14 — Pagination, tri et filtrage](14-pagination-tri-filtrage.md)
- [Sujet 18 — Cache backend et cache HTTP](18-cache-backend-http.md)
- [Sujet 9 — Mises à jour optimistes](09-mises-a-jour-optimistes.md)
