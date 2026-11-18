# 6. Performance (listes longues, debounce, annulation)

> 🎯 **Résumé.** Améliore la réactivité perçue et réelle de l'interface en réduisant le travail de rendu et les requêtes inutiles. Les leviers principaux sont la virtualisation des longues listes, le debounce couplé à l'annulation des requêtes obsolètes, et des optimisations ciblées après mesure. Le piège principal est d'optimiser sans mesurer, d'oublier des clés stables ou de ne pas nettoyer abonnements et contrôleurs d'annulation.

---

## 🧭 1. Contexte

**Listes longues → virtualisation** : ne rendre que les lignes visibles.

| Stack | Outil |
|---|---|
| React | TanStack Virtual, react-window |
| Vue | vue-virtual-scroller, TanStack Virtual |
| Angular | CDK `cdk-virtual-scroll-viewport` |

**Recherche instantanée → debounce + annulation**
- React/Vue : `useDebounce` / `refDebounced` (VueUse) + TanStack Query (il passe un `signal` d'annulation).
- Angular : `debounceTime` + `distinctUntilChanged` + **`switchMap`** (annule la requête précédente).
- Natif : `AbortController`.

```ts
const ctrl = new AbortController();
fetch('/api/search?q=abc', { signal: ctrl.signal });
ctrl.abort(); // annule la requête obsolète
```

**Autres leviers** : mémoïsation (`useMemo`, `computed`, signals), `v-memo` / `v-once` / `<KeepAlive>` (Vue), `trackBy` (Angular), images lazy + formats modernes (WebP/AVIF), bundle analysis, Core Web Vitals (LCP, INP, CLS), compression Brotli côté serveur/CDN, assets statiques servis par CDN avec URLs fingerprintées (sujet 18).

> ⚠️ **Pièges.** optimiser avant de mesurer ; oublier `trackBy`/`key` stable dans les listes ; ne pas nettoyer les watchers/abonnements (`onUnmounted`, `onWatcherCleanup` en Vue 3.5+, `AbortController`) → fuites mémoire et race conditions.

---

## 💻 2. Exemples techniques

Exemple React avec TanStack Virtual pour ne rendre que les lignes visibles d'une longue liste.

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

type Row = { id: string; label: string };

export function VirtualizedList({ rows }: { rows: Row[] }) {
  const parentRef = useRef<HTMLDivElement | null>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 400, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const row = rows[virtualRow.index];
          return (
            <div
              key={row.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              {row.label}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

Exemple Angular avec RxJS : debounce, suppression des doublons et annulation de la requête précédente via `switchMap`.

```ts
import { Component, inject } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

type SearchResult = { id: string; title: string };

@Component({
  selector: 'app-search-box',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <input [formControl]="query" placeholder="Rechercher" />
    <ul>
      @for (item of results() ?? []; track item.id) {
        <li>{{ item.title }}</li>
      }
    </ul>
  `,
})
export class SearchBoxComponent {
  private readonly http = inject(HttpClient);
  readonly query = new FormControl('', { nonNullable: true });

  readonly results = toSignal(
    this.query.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap((q) =>
        this.http.get<SearchResult[]>('/api/search', { params: { q } }),
      ),
    ),
    { initialValue: [] as SearchResult[] },
  );
}
```

Exemple Vue avec `refDebounced` et `watch` nettoyé via `AbortController` pour annuler les requêtes obsolètes.

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'
import { refDebounced } from '@vueuse/core'

const query = ref('')
const debouncedQuery = refDebounced(query, 300)
const results = ref<Array<{ id: string; title: string }>>([])

watch(debouncedQuery, async (q, _prev, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  if (!q) {
    results.value = []
    return
  }

  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
    signal: controller.signal,
  })
  if (res.ok) results.value = await res.json()
})
</script>

<template>
  <input v-model="query" placeholder="Rechercher" />
  <ul>
    <li v-for="item in results" :key="item.id">{{ item.title }}</li>
  </ul>
</template>
```

---

## 🔗 3. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 5 — Routing, guards et code splitting](05-routing-guards-code-splitting.md)
- [Sujet 18 — Cache backend et cache HTTP](18-cache-backend-http.md)
