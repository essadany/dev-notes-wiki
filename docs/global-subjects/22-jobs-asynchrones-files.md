# 22. Jobs asynchrones et files de tâches

> 🎯 **Résumé.** Un envoi d'e-mail ou un import qui bloque la requête HTTP transforme un pic de trafic en timeouts pour tout le monde. On sort le travail du chemin de requête via une file, des workers idempotents et des retries bornés, avec un `202 Accepted` et un sondage quand le résultat doit être consulté. Le piège : croire au « exactly-once » annoncé par un broker alors qu'un message peut être rejoué — et laisser un backlog infini cacher la panne.

---

## 🧭 1. Contexte

**Quand** : envoi d'emails, génération de PDF, imports lourds, appels externes lents.

**D'abord trancher synchrone vs asynchrone** : si l'appelant a besoin du résultat immédiatement, garder l'appel direct. Une file sert à sortir du chemin de requête, absorber un pic (buffer) et découpler des services. Ne pas installer un broker « pour plus tard » sans besoin chiffré, et ne pas construire de request/reply sur une file pour simuler du synchrone.

**Choisir la forme la moins coûteuse**
- **File de travail** : un groupe de workers se partage les messages ; tâches en arrière-plan, scale indépendant.
- **Pub/sub** : chaque abonné reçoit une copie ; un événement déclenche plusieurs réactions (notifier, indexer, auditer).
- **Stream / log réplayable** : ordre par partition, rejeu, plusieurs positions de lecture ; plus lourd à opérer.
- **Workflow durable** : processus multi-étapes avec retries, timers et compensation intégrés, plutôt qu'un enchevêtrement de files + flags + cron.

| Stack | Solution |
|---|---|
| Spring | `@Async`, `@Scheduled`, Spring AMQP (RabbitMQ), Kafka, Spring Batch |
| Django | **Celery** (+ RabbitMQ/Redis), `django-celery-beat` pour le planifié |
| Laravel | Queues (`dispatch(new Job)`), Horizon, Scheduler |
| Node | BullMQ (Redis), `node-cron` |

**Contrat d'un message** : `message_id` (déduplication), `type` + `schema_version`, `partition_key`, `timestamp`, `trace_id`, payload d'événement factuel et petit. Ack **après** traitement → *at-least-once* : les doublons sont possibles, donc **consommateurs idempotents**. Un « exactly-once » annoncé par un broker ne vaut souvent qu'à l'intérieur de son cluster.

**Planification / allocation**
- **Pull + lease** (défaut) : le worker loue la tâche pour un *visibility timeout* supérieur au p99 de sa durée, ack à la fin, **heartbeat** pour prolonger les longues tâches. Un timeout trop court est la première cause de doubles exécutions.
- **Déclencheurs** : cron (une machine suffit), scheduler distribué avec élection de leader (plusieurs instances, pas de double tir), file à délai (`not_before`), DAG (Airflow) seulement si dépendances et historique sont nécessaires.
- Mettre du **jitter** dans les crons (ne pas tout lancer à `:00`) ; `dedup_key = (job, scheduled_time)` pour qu'un double enqueue soit un no-op.

**Pattern « fire and poll »** : l'API répond `202 Accepted` avec un id de tâche, le front interroge `GET /tasks/{id}` (ou reçoit un SSE/WebSocket).

## ⚠️ 2. Pièges

- Tâches **idempotentes** ; retries plafonnés avec backoff + jitter, puis **dead-letter queue** pour les messages empoisonnés.
- **Backpressure** : file bornée ; au-delà, refuser/ralentir le producteur plutôt que bufferiser sans limite (un backlog infini cache la panne).
- Ne pas passer d'objets lourds dans le message (passer un id).
- Surveiller la profondeur de file et **l'âge du plus vieux message** (meilleur signal de retard), le taux de redelivery/DLQ et débit producteur vs consommateur.

---

## 💻 3. Exemples techniques

Une tâche Celery qui envoie un e-mail est idempotente grâce à un marqueur en base : un doublon de message ne renvoie rien, et les retries sont plafonnés avec backoff et jitter.

```python
# orders/tasks.py
from celery import shared_task
from django.conf import settings
from django.core.mail import send_mail
from django.utils import timezone
from orders.models import Order


@shared_task(
    bind=True,
    autoretry_for=(TimeoutError,),
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
    max_retries=5,
)
def send_order_confirmation(self, order_id: int) -> None:
    order = Order.objects.get(pk=order_id)
    if order.confirmation_sent_at is not None:
        return

    send_mail(
        "Confirmation de commande",
        f"Bonjour {order.customer_email}, votre commande {order.id} est confirmée.",
        settings.DEFAULT_FROM_EMAIL,
        [order.customer_email],
    )
    Order.objects.filter(pk=order_id, confirmation_sent_at__isnull=True).update(
        confirmation_sent_at=timezone.now()
    )
```

Côté Node, BullMQ illustre le *jobId* de déduplication, le backoff exponentiel, le *lease* long et une file bornée qui refuse les producteurs au-delà d'un seuil.

```ts
// jobs/thumbnails.ts
import IORedis from 'ioredis';
import { Queue, Worker } from 'bullmq';
import { renderThumbnail } from './render-thumbnail';

const connection = new IORedis({ host: 'redis', maxRetriesPerRequest: null });

export const thumbnails = new Queue('thumbnails', {
  connection,
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: { count: 1000 },
    removeOnFail: { count: 5000 },
  },
});

export async function enqueueThumbnail(assetId: string): Promise<void> {
  if ((await thumbnails.getWaitingCount()) >= 10_000) {
    throw new Error('QUEUE_FULL');
  }
  await thumbnails.add(
    'generate',
    { assetId },
    { jobId: `thumbnail:${assetId}`, delay: 0 },
  );
}

new Worker(
  'thumbnails',
  async (job) => {
    await renderThumbnail((job.data as { assetId: string }).assetId);
  },
  { connection, concurrency: 8, lockDuration: 60_000 },
);
```

Le Scheduler Laravel montre une tâche planifiée avec une heure décalée, une exécution unique sur la flotte et un verrou anti-chevauchement.

```php
// app/Console/Kernel.php
namespace App\Console;

use App\Jobs\GenerateDailyReport;
use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->job(new GenerateDailyReport())
            ->dailyAt('02:17')
            ->timezone('Europe/Paris')
            ->withoutOverlapping(120)
            ->onOneServer();
    }
}
```

---

## 🔗 4. Voir aussi

- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
- [Sujet 23 — Rate limiting](23-rate-limiting.md)
- [Sujet 37 — Webhooks (envoi et réception)](37-webhooks.md)
- [Files de tâches et jobs en arrière-plan (first principles)](../backend-from-first-principles/14-task-queues-and-background-jobs.md)
