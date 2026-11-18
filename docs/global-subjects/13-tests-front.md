# 13. Tests front

> 🎯 **Résumé.** Un test front utile vérifie ce que l'utilisateur voit et fait, pas les détails internes d'un composant. On combine des tests unitaires/composant rapides, un mock API réaliste avec MSW et quelques parcours Playwright ou Cypress. Le piège principal est de tester l'implémentation (props internes, appels de hooks) : ces tests cassent à chaque refactoring sans rien garantir sur le comportement.

---

## 🧭 1. Contexte

| Niveau | Outils |
|---|---|
| Unitaire / composant | Vitest ou Jest + Testing Library (React, Vue, Svelte) ; Angular : **Vitest + TestBed + Component Harnesses** (défaut des CLI récents), Jasmine/Karma pour les anciens projets |
| Mock API | MSW (Mock Service Worker) |
| End-to-end | Playwright, Cypress |

Tester le comportement visible par l'utilisateur plutôt que l'implémentation interne.

---

## 💻 2. Exemples techniques

Test de composant Vitest + Testing Library : on tape dans le champ, on clique, et on vérifie le message affiché et l'absence d'appel :

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('affiche une erreur et ne soumet pas un email invalide', async () => {
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await userEvent.type(screen.getByLabelText('Email'), 'pas-un-email');
    await userEvent.click(screen.getByRole('button', { name: 'Se connecter' }));

    expect(screen.getByRole('alert').textContent).toContain('Email invalide');
    expect(onSubmit).not.toHaveBeenCalled();
  });
});
```

Serveur MSW partagé par les tests : les composants appellent `fetch('/api/users')` comme en production, mais la réponse est contrôlée (y compris la page 2 et son curseur) :

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { afterAll, afterEach, beforeAll } from 'vitest';

export const server = setupServer(
  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page') ?? '1');

    return HttpResponse.json({
      data: page === 1 ? [{ id: 1, name: 'Ada' }] : [],
      next_cursor: page === 1 ? 'eyJpZCI6MX0' : null,
      has_more: page === 1,
    });
  }),
);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Parcours end-to-end Playwright : le filtre doit être visible dans l'URL pour être partageable et survivre au bouton retour :

```ts
import { expect, test } from '@playwright/test';

test('le filtre est reflété dans l’URL et la ligne filtrée est visible', async ({ page }) => {
  await page.goto('/users');
  await page.getByLabel('Recherche').fill('ada');

  await expect(page).toHaveURL(/q=ada/);
  await expect(page.getByRole('row', { name: /Ada/ })).toBeVisible();
});
```

---

## 🔗 3. Voir aussi

- [Sujet 4 — Formulaires et validation](04-formulaires-validation.md)
- [Sujet 5 — Routing, guards et code splitting](05-routing-guards-code-splitting.md)
- [Sujet 8 — Gestion d'erreurs côté front](08-erreurs-front.md)
- [Sujet 30 — Tests backend](30-tests-backend.md)
