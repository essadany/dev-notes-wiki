# 40. Dates, fuseaux horaires, montants et identifiants

> 🎯 **Résumé.** Une date stockée sans fuseau, un montant en `float` ou un identifiant 64 bits envoyé tel quel en JSON sont trois sources de bugs silencieux. La règle est de stocker et échanger en UTC/ISO 8601, de convertir seulement à l'affichage, d'utiliser des décimaux ou des centiers entiers, et de choisir un schéma d'ID adapté au nombre de générateurs. Le piège : un Snowflake dont l'horloge recule, ou un UUIDv4 aléatoire qui fragmente les index.

---

## 🧭 1. Contexte

**Dates et fuseaux**
- Stocker et échanger en **UTC**, format **ISO 8601** (`2026-11-18T14:30:00Z`) ; convertir au fuseau de l'utilisateur **à l'affichage** uniquement.
- Distinguer un **instant** (moment précis) d'une **date locale** (anniversaire, jour férié) : ne pas leur appliquer le même type.

| Stack | Outils |
|---|---|
| Java | `Instant`, `OffsetDateTime`, `ZonedDateTime` (jamais `java.util.Date` pour du nouveau code) |
| Django | `USE_TZ = True`, objets *aware* |
| Laravel | Carbon, `APP_TIMEZONE` en UTC |
| JavaScript | `Intl.DateTimeFormat`, date-fns, Day.js, Luxon ; l'API `Temporal` arrive progressivement |

**Montants** : jamais de `float`/`double`. Utiliser `BigDecimal` (Java), `Decimal` (Python), ou stocker des **entiers en centimes** + code devise. Définir explicitement l'arrondi.

**Identifiants**
- Auto-incrément : simple, compact, mais prévisible (énumération possible) et problématique en multi-nœuds.
- **UUID** : imprévisible, générable côté client ; UUIDv4 aléatoire fragmente les index, **UUIDv7 / ULID** (triables par date) sont meilleurs pour les performances.
- Ne jamais compter sur l'identifiant pour la sécurité : le contrôle d'accès reste obligatoire (sujet 15).

**Choisir selon le besoin** : un seul nœud DB → `BIGINT AUTO_INCREMENT` suffit (pas de service d'ID distribué). Plusieurs générateurs : UUIDv4 si seule l'unicité compte ; **ULID / UUIDv7** (préfixe temps, triables, meilleurs pour les index) = défaut moderne ; **Snowflake 64 bits** `[timestamp:41 | node:10 | seq:12]` (~4 M IDs/noeud/s) pour un entier compact et quasi-trié ; ticket/range allocation pour des entiers quasi-monotones.

> ⚠️ **Pièges.** changement d'heure (jours de 23 ou 25 h), comparaison de dates sans fuseau, JSON qui perd la précision des grands entiers côté JavaScript (envoyer les identifiants 64 bits en chaînes). Côté IDs : une horloge qui recule (NTP, pause VM) peut réémettre un timestamp et créer des doublons → refuser d'émettre tant que `now < last_timestamp` et alerter sur le skew ; allouer les `node_id` par lease plutôt qu'en config statique (autoscaling) ; ne pas utiliser un ID monotone comme clé de sharding (hot shard). Séparer la **clé interne** (triable, révèle la date) de l'**ID externe** opaque si l'énumération pose problème.

---

## 💻 2. Exemples techniques

En Java, l'instant est un `Instant`, le fuseau n'est appliqué qu'à l'affichage, la date locale reste un `LocalDate` et le prix un `BigDecimal` à l'échelle de la devise :

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.util.Currency;

public final class DateAndMoneyExample {

    public record Price(BigDecimal amount, Currency currency) {
        public Price {
            amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_EVEN);
        }
    }

    public static void main(String[] args) {
        Instant publishedAt = Instant.parse("2026-11-18T14:30:00Z");
        ZonedDateTime inParis = publishedAt.atZone(ZoneId.of("Europe/Paris"));
        LocalDate birthday = LocalDate.of(1990, 5, 12);
        Price price = new Price(new BigDecimal("19.99"), Currency.getInstance("EUR"));

        System.out.println(publishedAt);
        System.out.println(inParis);
        System.out.println(birthday);
        System.out.println(price.amount() + " " + price.currency());
    }
}
```

Un générateur Snowflake 64 bits refuse d'émettre si l'horloge recule ; le `node_id` doit venir d'un lease et non d'une configuration figée :

```python
import threading
import time

EPOCH_MS = 1735689600000
NODE_ID_BITS = 10
SEQ_BITS = 12
MAX_SEQ = (1 << SEQ_BITS) - 1


class Snowflake:
    def __init__(self, node_id: int):
        if not 0 <= node_id < (1 << NODE_ID_BITS):
            raise ValueError("node_id hors plage")
        self.node_id = node_id
        self.last_ts = 0
        self.seq = 0
        self.lock = threading.Lock()

    def next_id(self) -> int:
        with self.lock:
            now = int(time.time() * 1000) - EPOCH_MS
            if now < self.last_ts:
                raise RuntimeError("horloge en arrière : émission refusée")
            if now == self.last_ts:
                self.seq += 1
                if self.seq > MAX_SEQ:
                    while now <= self.last_ts:
                        time.sleep(0.001)
                        now = int(time.time() * 1000) - EPOCH_MS
                    self.seq = 0
            else:
                self.seq = 0
            self.last_ts = now
            return (now << (NODE_ID_BITS + SEQ_BITS)) | (self.node_id << SEQ_BITS) | self.seq
```

Côté API, l'identifiant 64 bits est sérialisé en chaîne pour ne pas perdre de précision dans un navigateur :

```js
res.status(200).json({
  id: String(order.id),
  external_id: order.externalId,
  created_at: order.createdAt.toISOString(),
  amount_cents: order.amountCents,
});
```

Django stocke en UTC et l'affichage au fuseau de l'utilisateur se fait avec `Intl.DateTimeFormat`, y compris les jours de 23 ou 25 heures :

```python
USE_TZ = True
TIME_ZONE = "UTC"
```

L'affichage dans le fuseau de l'utilisateur se fait seulement à ce moment-là :

```js
const instant = "2026-11-18T14:30:00Z";

const affiche = new Intl.DateTimeFormat("fr-FR", {
  timeZone: "Europe/Paris",
  dateStyle: "long",
  timeStyle: "short",
}).format(new Date(instant));

console.log(affiche);
```

---

## 🔗 3. Voir aussi

- [Sujet 11 — i18n](11-i18n.md)
- [Sujet 16 — Validation des entrées](16-validation-entrees.md)
- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
- [Sujet 27 — Documentation d'API et versioning](27-documentation-api-versioning.md)
