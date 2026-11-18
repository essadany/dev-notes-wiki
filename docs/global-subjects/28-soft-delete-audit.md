# 28. Soft delete et audit

> 🎯 **Résumé.** Supprimer une ligne référencée par des commandes casse l'historique, mais l'effacer vraiment empêche aussi de comprendre ce qui s'est passé. On marque `deleted_at`, on centralise le filtre dans un manager ou une restriction SQL, et on enregistre qui a modifié quoi avec Envers ou `django-simple-history`. Le piège : oublier que le soft delete ne supprime pas la donnée personnelle (RGPD) et qu'un `UNIQUE` sur l'e-mail bloque la réinscription.

---

## 🧭 1. Contexte

**Soft delete** : marquer comme supprimé (`deleted_at`) au lieu d'effacer.

| Stack | Solution |
|---|---|
| Hibernate | `@SoftDelete` (Hibernate 6.4+) ou `@SQLDelete` + `@SQLRestriction` |
| Django | manager personnalisé, `django-safedelete` |
| Laravel | trait `SoftDeletes` |
| Prisma | extension / middleware |

**Audit (qui a modifié quoi)** : Spring Data Auditing (`@CreatedBy`, `@LastModifiedDate`) + Hibernate Envers ; Django `django-simple-history` ; Laravel `spatie/laravel-activitylog`.
> ⚠️ **Piège.** contraintes d'unicité et RGPD (le soft delete ne supprime pas réellement la donnée personnelle).

---

## 💻 2. Exemples techniques

Laravel fournit le trait `SoftDeletes` : la colonne `deleted_at` est ajoutée par la migration et toutes les requêtes du modèle l'excluent automatiquement.

```php
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class User extends Model
{
    use SoftDeletes;
}

// database/migrations/2026_01_15_000000_add_deleted_at_to_users.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->softDeletes();
            $table->index('deleted_at');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropIndex(['deleted_at']);
            $table->dropSoftDeletes();
        });
    }
};
```

Django sépare le manager des actifs et le manager complet, tandis que la contrainte d'unicité ne porte que sur les lignes actives — un e-mail supprimé peut être réutilisé.

```python
# users/models.py
from django.conf import settings
from django.db import models
from simple_history.models import HistoricalRecords


class ActiveUserManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)


class User(models.Model):
    email = models.EmailField()
    deleted_at = models.DateTimeField(null=True, blank=True)
    history = HistoricalRecords()

    objects = ActiveUserManager()
    all_objects = models.Manager()

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['email'],
                condition=models.Q(deleted_at__isnull=True),
                name='uniq_email_among_active_users',
            )
        ]
```

Spring Data Auditing remplit les auteurs et dates, et `@Audited` d'Envers conserve l'historique des modifications dans des tables dédiées.

```java
@Entity
@Audited
@EntityListeners(AuditingEntityListener.class)
class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedBy
    private String createdBy;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedBy
    private String lastModifiedBy;

    @LastModifiedDate
    private Instant lastModifiedAt;
}

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
class AuditingConfig {
    @Bean
    AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(
            SecurityContextHolder.getContext().getAuthentication()
        ).map(Authentication::getName);
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Sujet 36 — Multi-tenancy](36-multi-tenancy.md)
