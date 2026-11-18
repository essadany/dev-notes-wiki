# 38. Feature flags

> 🎯 **Résumé.** Une feature flag permet de livrer du code désactivé, de progresser par paliers de 5 à 50 % d'utilisateurs et de couper une fonctionnalité en une seconde sans redéployer. Le service dédié (Unleash, Flagsmith, LaunchDarkly) ou une librairie locale (Pennant, waffle, Togglz) fournit le mécanisme, mais le back doit toujours revérifier le flag. Le piège : accumulateur de flags jamais retirés et combinatoire d'états impossible à tester.

---

## 🧭 1. Contexte

**But** : activer/désactiver une fonctionnalité sans redéployer ; déploiement progressif (pourcentage d'utilisateurs), tests A/B, *kill switch*, livraison de code en production avant activation (trunk-based development).

| Stack | Solution |
|---|---|
| Service dédié | Unleash, Flagsmith, GrowthBook, LaunchDarkly (SaaS), OpenFeature (standard d'API) |
| Laravel | Laravel Pennant |
| Django | `django-waffle` |
| Spring | Togglz, FF4j, ou SDK d'un service dédié |
| Front | SDK du fournisseur (React/Vue/Angular), flags récupérés au chargement |

> ⚠️ **Pièges.** accumuler des flags jamais supprimés (dette technique : prévoir une date de retrait) ; combinatoire de flags impossible à tester ; ne pas se reposer sur un flag **front seul** pour protéger une fonctionnalité sensible (le back doit aussi le vérifier).

---

## 💻 2. Exemples techniques

Avec `django-waffle`, le contrôle est fait côté serveur : même si un client appelle directement l'endpoint, un flag désactivé renvoie l'ancien comportement :

```python
from django.http import JsonResponse
from waffle import flag_is_active


def pricing_view(request):
    if flag_is_active(request, "checkout_new_pricing"):
        return JsonResponse({"plan": "v2", "discount_percent": 10})
    return JsonResponse({"plan": "v1", "discount_percent": 0})
```

Le SDK Unleash évalue le pourcentage de déploiement à partir du `userId` fourni dans le contexte ; le code applicatif ne contient aucune condition de date ou de pourcentage :

```js
const { Unleash } = require('unleash-client');

const unleash = new Unleash({
  url: process.env.UNLEASH_URL,
  appName: 'checkout-api',
  customHeaders: { Authorization: process.env.UNLEASH_TOKEN },
});

unleash.on('error', (error) => console.error('Unleash indisponible', error.message));

function pricingFor(user) {
  if (unleash.isEnabled('checkout.new_pricing', { userId: String(user.id) })) {
    return { plan: 'v2', discount_percent: 10 };
  }
  return { plan: 'v1', discount_percent: 0 };
}

module.exports = { pricingFor };
```

Un registre versionné avec une date de retrait, associé à un test en CI, évite que les flags deviennent permanents :

```python
from datetime import date

FLAGS = {
    "checkout.new_pricing": {"owner": "team-checkout", "remove_after": date(2026, 12, 15)},
    "search.meilisearch": {"owner": "team-search", "remove_after": date(2026, 11, 30)},
}


def test_aucun_flag_perime():
    today = date(2026, 11, 18)
    expired = {name: config for name, config in FLAGS.items() if config["remove_after"] < today}
    assert not expired, f"Flags à retirer : {sorted(expired)}"
```

---

## 🔗 3. Voir aussi

- [Sujet 12 — Sécurité front](12-securite-front.md)
- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
- [Sujet 30 — Tests backend](30-tests-backend.md)
- [Sujet 34 — Monolithe modulaire vs microservices](34-monolithe-modulaire-vs-microservices.md)
