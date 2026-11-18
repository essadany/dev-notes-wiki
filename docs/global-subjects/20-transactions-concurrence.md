# 20. Transactions et concurrence

> 🎯 **Résumé.** Une transaction garantit le « tout ou rien » d'un bloc ACID, mais elle ne protège pas à elle seule contre deux requêtes concurrentes qui modifient la même ligne. On choisit donc un verrouillage optimiste (version) ou pessimiste (`SELECT ... FOR UPDATE`) selon la fréquence des conflits, et une clé d'idempotence pour les écritures réessayées. Le piège principal est de garder une transaction ouverte pendant un appel réseau long : les verrous s'accumulent, le pool de connexions s'épuise et l'événement à publier se perd si le commit échoue — d'où le pattern outbox.

---

## 🧭 1. Contexte

**Transaction** : un bloc « tout ou rien » (ACID).

| Stack | Solution |
|---|---|
| Spring | `@Transactional` (attention : appel interne à la même classe = pas de proxy) |
| Django | `transaction.atomic()`, `ATOMIC_REQUESTS` |
| Laravel | `DB::transaction(fn)` |
| Prisma | `$transaction([...])` |

**Concurrence**
- **Verrouillage optimiste** : colonne de version, la mise à jour échoue si la version a changé (JPA `@Version`, ou champ version manuel ailleurs).
- **Verrouillage pessimiste** : `SELECT ... FOR UPDATE` (Django `select_for_update`, Laravel `lockForUpdate`, JPA `@Lock`).
- **Idempotence** : header `Idempotency-Key` pour éviter un double paiement sur retry.

**Niveau de cohérence** : choisir la garantie la plus faible qui suffit (sujet 35). « Lire ses propres écritures » (*read-your-writes*) coûte bien moins qu'une cohérence forte globale ; réserver cette dernière aux invariants (solde, stock, unicité). Les lectures `read-your-writes` vont au primaire, jamais à une réplique en retard.

**Contrat d'idempotence** : clé UUID générée par le client, unique par opération logique, stockée avec la réponse pendant une fenêtre bornée (ex. 24 h). Même clé en retry → réponse mémorisée, sans réexécution ; même clé avec un corps différent → 422. GET/PUT/DELETE sont sûrs à réessayer, POST ne l'est pas sans clé.

> ⚠️ **Piège.** ne pas laisser une transaction ouverte pendant un appel réseau long (verrous maintenus, épuisement du pool de connexions) — publier un événement/outbox et traiter après commit.

---

## 💻 2. Exemples techniques

Spring : `@Transactional` entoure le transfert, les deux comptes sont verrouillés en `PESSIMISTIC_WRITE` dans un ordre fixe pour éviter les deadlocks, et `@Version` fournit le verrou optimiste sur les autres écritures :

```java
@Entity
public class Account {
    @Id
    private Long id;
    private long balanceCents;

    @Version
    private long version;

    public void debit(long amountCents) {
        this.balanceCents -= amountCents;
    }

    public void credit(long amountCents) {
        this.balanceCents += amountCents;
    }
}

@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select a from Account a where a.id = :id")
    Account findByIdForUpdate(@Param("id") Long id);
}

@Service
public class TransferService {
    private final AccountRepository repo;

    public TransferService(AccountRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, long amountCents) {
        Account first = repo.findByIdForUpdate(Math.min(fromId, toId));
        Account second = repo.findByIdForUpdate(Math.max(fromId, toId));

        Account from = first.getId().equals(fromId) ? first : second;
        Account to = first.getId().equals(fromId) ? second : first;

        from.debit(amountCents);
        to.credit(amountCents);
    }
}
```

Django : `select_for_update()` sécurise la lecture du stock et `atomic()` limite la portée de la transaction ; la ligne `PaymentAttempt` immuable sert de mémoire d'idempotence :

```python
from django.db import models, transaction


class PaymentAttempt(models.Model):
    idempotency_key = models.UUIDField(unique=True)
    response = models.JSONField()


def reserve_stock(product_id: int, quantity: int) -> None:
    with transaction.atomic():
        product = Product.objects.select_for_update().get(pk=product_id)
        if product.stock < quantity:
            raise ValueError("Stock insuffisant")
        product.stock -= quantity
        product.save(update_fields=["stock"])
```

Laravel : la route de paiement vérifie la clé dans la même transaction que le débit, renvoie la réponse mémorisée sur retry et rejette une clé réutilisée avec un autre corps (422) :

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Route;

Route::post('/payments', function (Request $request) {
    $key = $request->header('Idempotency-Key');

    if (! $key) {
        abort(400, 'Idempotency-Key requise');
    }

    return DB::transaction(function () use ($request, $key) {
        $payload = $request->validate([
            'amount' => 'required|integer|min:1',
            'account_id' => 'required|integer',
        ]);
        $hash = hash('sha256', json_encode($payload));

        $existing = PaymentAttempt::where('idempotency_key', $key)->first();
        if ($existing) {
            if ($existing->request_hash !== $hash) {
                abort(422, 'Clé d’idempotence réutilisée avec un corps différent');
            }
            return response()->json($existing->response);
        }

        $account = Account::whereKey($payload['account_id'])
            ->lockForUpdate()
            ->firstOrFail();
        $account->decrement('balance_cents', $payload['amount']);

        $response = ['status' => 'paid', 'account_id' => $account->id];
        PaymentAttempt::create([
            'idempotency_key' => $key,
            'request_hash' => $hash,
            'response' => $response,
        ]);

        return response()->json($response);
    });
});
```

---

## 🔗 3. Voir aussi

- [Sujet 9 — Mises à jour optimistes](09-mises-a-jour-optimistes.md)
- [Sujet 21 — Migrations de base de données](21-migrations-base-de-donnees.md)
- [Sujet 22 — Jobs asynchrones et files de tâches](22-jobs-asynchrones-files.md)
- [Mastering databases with Postgres](../backend-from-first-principles/12-mastering-databases-with-postgres.md)
