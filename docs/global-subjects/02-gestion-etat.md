# 2. Gestion d'état

> 🎯 **Résumé.** Organiser l'état d'une application en distinguant clairement données serveur, état UI global et état local évite une grande partie de la complexité front. Les données d'API doivent vivre dans une couche de cache dédiée, pas dans un store global recodé à la main. Le piège principal est de confondre ces catégories et de réimplémenter cache, chargement et erreurs dans Redux ou Pinia.

---

## 🧭 1. Contexte

**Principe** : distinguer 3 types d'état.
1. **État serveur** (données de l'API) → lib de cache (sujet 1), pas un store global.
2. **État UI global** (thème, utilisateur connecté, panier) → store.
3. **État local** (modale ouverte, champ en cours) → état du composant.

| Stack | Solutions |
|---|---|
| React | `useState`/`useReducer`, Context (petit état), Zustand, Redux Toolkit, Jotai |
| Vue | `ref`/`reactive`, composables, **Pinia** (standard) |
| Angular | services + signals, NgRx Store / SignalStore |
| Svelte | stores, runes (`$state`) |

**Précisions Angular** : `computed` pour l'état dérivé, `linkedSignal` pour un état réinitialisable lié à une source, `effect` uniquement pour des effets de bord (jamais pour mettre à jour un autre signal).

**Précisions Vue** : Setup Store Pinia (plutôt qu'Options Store), `computed`, `shallowRef` pour les grosses structures remplacées en bloc, `v-memo` sur les listes qui changent peu.

> ⚠️ **Pièges.** mettre les données serveur dans Redux/Pinia et réécrire à la main cache, loading et erreurs ; Context React qui re-render tout l'arbre ; `effect` Angular utilisé comme un simple `computed`.

---

## 💻 2. Exemples techniques

Exemple React avec Zustand pour un état UI global (thème et panier) sans y stocker les données serveur.

```tsx
import { create } from 'zustand';

type CartItem = { productId: string; quantity: number };

type UiStore = {
  theme: 'light' | 'dark';
  cart: CartItem[];
  toggleTheme: () => void;
  addToCart: (productId: string) => void;
};

export const useUiStore = create<UiStore>((set) => ({
  theme: 'light',
  cart: [],
  toggleTheme: () =>
    set((state) => ({ theme: state.theme === 'light' ? 'dark' : 'light' })),
  addToCart: (productId) =>
    set((state) => {
      const existing = state.cart.find((i) => i.productId === productId);
      if (!existing) return { cart: [...state.cart, { productId, quantity: 1 }] };
      return {
        cart: state.cart.map((i) =>
          i.productId === productId ? { ...i, quantity: i.quantity + 1 } : i,
        ),
      };
    }),
}));
```

Exemple Vue avec un Setup Store Pinia pour l'état UI global, en gardant l'état dérivé dans `computed`.

```ts
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

export const useSessionStore = defineStore('session', () => {
  const user = ref<{ id: string; name: string } | null>(null)
  const locale = ref<'fr' | 'en'>('fr')

  const isAuthenticated = computed(() => user.value !== null)

  function setUser(nextUser: { id: string; name: string }) {
    user.value = nextUser
  }

  function logout() {
    user.value = null
  }

  return { user, locale, isAuthenticated, setUser, logout }
})
```

Exemple Angular avec un service basé sur les signals : `computed` pour le dérivé, `linkedSignal` pour un état réinitialisable.

```ts
import { Injectable, computed, linkedSignal, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class PreferencesStore {
  readonly theme = signal<'light' | 'dark'>('light');
  readonly itemsPerPageOptions = signal([10, 20, 50] as const);

  readonly itemsPerPage = linkedSignal(() => this.itemsPerPageOptions()[1]);

  readonly isDark = computed(() => this.theme() === 'dark');

  toggleTheme(): void {
    this.theme.update((t) => (t === 'light' ? 'dark' : 'light'));
  }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 3 — Authentification côté front](03-authentification-front.md)
- [Sujet 5 — Routing, guards et code splitting](05-routing-guards-code-splitting.md)
