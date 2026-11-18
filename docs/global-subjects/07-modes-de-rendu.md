# 7. Modes de rendu (CSR, SSR, SSG, ISR)

> 🎯 **Résumé.** Choisit la stratégie de rendu (client, serveur, statique, régénération incrémentale) en fonction des besoins de SEO, de fraîcheur des données et de complexité applicative. Chaque mode implique des compromis différents entre coût serveur, performance perçue et dynamique du contenu. Le piège principal est d'utiliser des API navigateur côté serveur et de provoquer des erreurs d'hydratation.

---

## 🧭 1. Contexte

| Mode | Idée | Quand |
|---|---|---|
| CSR | rendu dans le navigateur | back-office, apps privées |
| SSR | HTML généré à chaque requête | SEO + contenu dynamique |
| SSG | HTML généré au build | contenu statique |
| ISR | SSG régénéré périodiquement | catalogue, blog volumineux |

| Framework | Solution |
|---|---|
| React | Next.js (App Router, Server Components, `revalidate`) |
| Vue | Nuxt (`routeRules` pour mixer SSR/SSG/ISR par route) |
| Angular | `@angular/ssr` + hydratation |
| Svelte | SvelteKit (`prerender`, `ssr`) |

> ⚠️ **Pièges.** accès à `window`/`localStorage` côté serveur ; erreurs d'hydratation (HTML serveur ≠ client).

---

## 💻 2. Exemples techniques

Exemple Next.js (App Router) avec une page statique régénérée périodiquement (ISR via `revalidate`).

```tsx
type Article = { id: string; title: string; body: string };

export const revalidate = 60;

export default async function ArticlesPage() {
  const res = await fetch('https://api.example.com/articles', {
    next: { revalidate: 60 },
  });
  const articles: Article[] = await res.json();

  return (
    <main>
      <h1>Articles</h1>
      <ul>
        {articles.map((article) => (
          <li key={article.id}>{article.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

Exemple Nuxt avec `routeRules` pour mixer SSR, SSG et ISR selon les routes.

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },
    '/blog/**': { isr: 300 },
    '/account/**': { ssr: true },
    '/admin/**': { ssr: false },
  },
})
```

Exemple SvelteKit avec pré-rendu statique et chargement de données côté serveur sans accès aux API navigateur.

```ts
// src/routes/blog/+page.ts
import type { PageLoad } from './$types';

export const prerender = true;

export const load: PageLoad = async ({ fetch }) => {
  const response = await fetch('/api/posts');
  if (!response.ok) {
    throw new Error('Impossible de charger les articles');
  }
  const posts: Array<{ id: string; title: string }> = await response.json();
  return { posts };
};
```

---

## 🔗 3. Voir aussi

- [Sujet 5 — Routing, guards et code splitting](05-routing-guards-code-splitting.md)
- [Sujet 6 — Performance (listes longues, debounce, annulation)](06-performance-front.md)
- [Sujet 18 — Cache backend et cache HTTP](18-cache-backend-http.md)
