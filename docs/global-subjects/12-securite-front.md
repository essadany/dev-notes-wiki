# 12. Sécurité front

> 🎯 **Résumé.** Le navigateur exécute tout ce qui se trouve dans le bundle et affiche tout ce qu'on lui donne : HTML utilisateur, scripts tiers, tokens. La parade est une combinaison de précautions — échappement et sanitisation, CSP, cookies `SameSite`, sanity check des dépendances — sans jamais y mettre de secret. Le piège principal est de croire qu'un contrôle côté navigateur protège le serveur : la sécurité réelle reste vérifiée côté back.

---

## 🧭 1. Contexte

- **XSS** : les frameworks échappent par défaut. Danger : `dangerouslySetInnerHTML` (React), `v-html` (Vue), `[innerHTML]` (Angular le sanitise, mais reste prudent). Nettoyer avec DOMPurify si HTML utilisateur nécessaire.
- **CSP** (Content-Security-Policy) en header pour limiter les scripts autorisés.
- **CSRF** : cookies `SameSite`, token CSRF si auth par cookie.
- **Dépendances** : `npm audit`, Dependabot/Renovate.
- Ne jamais mettre de secret dans le bundle front (tout y est lisible).

---

## 💻 2. Exemples techniques

Sanitisation explicite avant d'injecter du HTML utilisateur avec React (la même fonction DOMPurify s'utilise avant `v-html` côté Vue) :

```tsx
import DOMPurify from 'dompurify';

export function ArticleBody({ html }: { html: string }) {
  const safeHtml = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'strong', 'em', 'a', 'ul', 'li'],
    ALLOWED_ATTR: ['href'],
  });

  return <article dangerouslySetInnerHTML={{ __html: safeHtml }} />;
}
```

En-têtes de sécurité servis par le reverse proxy (Nginx) : une CSP stricte bloque les scripts injectés, `frame-ancestors` empêche le clickjacking et `nosniff` évite les interprétations de MIME surprises :

```nginx
server {
    listen 443 ssl http2;
    server_name app.example.org;

    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; object-src 'none'; frame-ancestors 'none'; base-uri 'self'" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        try_files $uri /index.html;
    }
}
```

Envoi du token CSRF lu dans un cookie `SameSite=Lax` posé par le serveur (ici convention Django/DRF) avec `fetch` en `same-origin` :

```ts
export async function apiPost<T>(url: string, body: unknown): Promise<T> {
  const csrfToken = document.cookie
    .split('; ')
    .find((cookie) => cookie.startsWith('csrftoken='))
    ?.split('=')[1];

  const response = await fetch(url, {
    method: 'POST',
    credentials: 'same-origin',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRFToken': csrfToken ?? '',
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw new Error(`Requête refusée : HTTP ${response.status}`);
  }

  return response.json() as Promise<T>;
}
```

---

## 🔗 3. Voir aussi

- [Sujet 3 — Authentification côté front](03-authentification-front.md)
- [Sujet 8 — Gestion d'erreurs côté front](08-erreurs-front.md)
- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
