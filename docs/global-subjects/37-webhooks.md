# 37. Webhooks (envoi et réception)

> 🎯 **Résumé.** Un webhook est un `POST` HTTP déclenché par un événement chez un tiers : paiement, push Git, inscription. Le consommateur doit vérifier la signature sur le corps **brut**, répondre immédiatement puis traiter en asynchrone, tandis que le producteur maintient une file de livraison avec retries et journal. Le piège classique est de recalculer le HMAC sur un JSON re-sérialisé : l'ordre des clés ou les espaces changent et la signature ne correspond plus.

---

## 🧭 1. Contexte

**Principe** : un service notifie un autre par un appel HTTP `POST` quand un événement se produit (paiement, push Git...).

**Réception (côté consommateur)**
1. **Vérifier la signature** (HMAC-SHA256 du corps brut avec un secret partagé) et comparer en temps constant.
2. **Répondre vite** (`2xx`), traiter en asynchrone via une file (sujet 22).
3. **Idempotence** : stocker l'identifiant d'événement, ignorer les doublons (les fournisseurs retentent).
4. Vérifier l'horodatage pour limiter les attaques par rejeu.

**Envoi (côté producteur)** : file de livraison, retries avec backoff, désactivation après échecs répétés, journal des tentatives, secret par client, signature de chaque requête.

| Stack | Remarque |
|---|---|
| Spring | lire le corps brut avant parsing pour la signature |
| Django / DRF | `request.body` brut pour la signature |
| Laravel | `spatie/laravel-webhook-client` et `laravel-webhook-server` |
| Node | middleware `express.raw()` pour garder le corps brut |

> ⚠️ **Piège classique.** calculer la signature sur le JSON déjà re-sérialisé (l'ordre ou les espaces changent) au lieu du corps brut.

---

## 💻 2. Exemples techniques

Django/DRF lit `request.body` avant tout parsing, compare en temps constant, borne l'horodatage et déduplique par identifiant d'événement avant de répondre `202` :

```python
import hashlib
import hmac
import time

from django.conf import settings
from django.db import models
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

from .tasks import process_webhook


class WebhookEvent(models.Model):
    event_id = models.CharField(max_length=255, unique=True)
    received_at = models.DateTimeField(auto_now_add=True)


def signature_is_valid(raw: bytes, header: str, secret: str, tolerance: int = 300) -> bool:
    parts = dict(part.split("=", 1) for part in header.split(",") if "=" in part)
    timestamp = int(parts["t"])
    if abs(time.time() - timestamp) > tolerance:
        return False
    signed_payload = f"{timestamp}.".encode() + raw
    expected = hmac.new(secret.encode(), signed_payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, parts["v1"])


@csrf_exempt
def payment_webhook(request):
    valid = signature_is_valid(
        request.body,
        request.headers.get("Stripe-Signature", ""),
        settings.WEBHOOK_SECRET,
    )
    if not valid:
        return JsonResponse({"error": "invalid signature"}, status=401)

    event_id = request.headers["X-Event-Id"]
    _, created = WebhookEvent.objects.get_or_create(event_id=event_id)
    if not created:
        return JsonResponse({"status": "duplicate"})

    process_webhook.delay(event_id, request.body.decode("utf-8"))
    return JsonResponse({"status": "accepted"}, status=202)
```

Côté Node, `express.raw()` conserve les octets exacts, `timingSafeEqual` évite les comparaisons non constantes dans le temps, et BullMQ sort le travail du chemin de requête :

```js
const express = require('express');
const crypto = require('crypto');
const { Queue } = require('bullmq');
const IORedis = require('ioredis');

const app = express();
const connection = new IORedis(process.env.REDIS_URL);
const webhookQueue = new Queue('webhook-events', { connection });

app.post('/webhooks/github', express.raw({ type: 'application/json' }), async (req, res) => {
  const raw = req.body;
  const expected = 'sha256=' + crypto
    .createHmac('sha256', process.env.GITHUB_WEBHOOK_SECRET)
    .update(raw)
    .digest('hex');
  const received = req.headers['x-hub-signature-256'] || '';
  const a = Buffer.from(expected);
  const b = Buffer.from(received);

  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
    return res.status(401).json({ error: 'invalid signature' });
  }

  await webhookQueue.add('github', {
    event: req.headers['x-github-event'],
    payload: raw.toString('utf8'),
  });
  return res.status(202).json({ status: 'accepted' });
});

app.listen(8080, () => console.log('listening on 8080'));
```

Le producteur Laravel signe chaque livraison avec le secret du client, limite les tentatives, journalise les échecs et désactive l'abonnement après épuisement :

```php
<?php

namespace App\Jobs;

use App\Models\WebhookDelivery;
use App\Models\WebhookEndpoint;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Http;

class DeliverWebhook implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 5;

    public array $backoff = [10, 60, 300, 1800];

    public function __construct(
        private WebhookEndpoint $endpoint,
        private string $eventId,
        private string $payload,
    ) {}

    public function handle(): void
    {
        $signature = 'sha256=' . hash_hmac('sha256', $this->payload, $this->endpoint->secret);

        $response = Http::withHeaders([
            'X-Event-Id' => $this->eventId,
            'X-Signature' => $signature,
        ])->timeout(5)->post($this->endpoint->url, $this->payload);

        WebhookDelivery::create([
            'webhook_endpoint_id' => $this->endpoint->id,
            'event_id' => $this->eventId,
            'status_code' => $response->status(),
        ]);

        if ($response->failed() && $this->attempts() >= $this->tries) {
            $this->endpoint->update(['active' => false]);
        }
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
- [Sujet 22 — Jobs asynchrones et files](22-jobs-asynchrones-files.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
