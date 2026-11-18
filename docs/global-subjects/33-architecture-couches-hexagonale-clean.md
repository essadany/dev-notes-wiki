# 33. Architecture en couches, hexagonale, Clean Architecture

> 🎯 **Résumé.** Quand la logique métier se retrouve dans les contrôleurs, les vues et les modèles ORM, chaque changement devient risqué et les tests unitaires impossibles. On restaure une frontière nette : HTTP d'un côté, règles métier pures de l'autre, persistence et services externes derrière des ports. Le piège est double : sur-ingénierer un CRUD pauvre, et laisser des couches qui se contentent de retransmettre l'appel.

---

## 🧭 1. Contexte

**Problème** : la logique métier éparpillée dans les contrôleurs, les vues et les modèles ORM devient impossible à tester et à faire évoluer.

**Trois niveaux d'organisation**
- **Couches classiques** : controller → service → repository. Simple, adapté à la majorité des projets.
- **Hexagonale (ports & adapters)** : le domaine est au centre et ne dépend de rien. Il expose des **ports** (interfaces : `UserRepository`, `PaymentGateway`) ; l'infrastructure (base de données, API externes, HTTP) fournit les **adapters**.
- **Clean Architecture / DDD** : même règle de dépendance (les dépendances pointent vers le domaine) avec des couches nommées (entités, cas d'usage, interfaces, infrastructure).

```
   HTTP / CLI / jobs ───▶ ┌──────────────────────────┐
                          │        ADAPTERS          │  controllers, ORM, Stripe
                          │    (infrastructure)      │  Redis, S3, Kafka...
                          └────────────┬─────────────┘
                                       │ ports (interfaces)
                                       ▼
                          ┌──────────────────────────┐
                          │  APPLICATION / SERVICES  │  cas d'usage, transactions
                          └────────────┬─────────────┘
                                       ▼
                          ┌──────────────────────────┐
                          │     🎯 DOMAINE (pur)     │  entités, règles métier
                          └──────────────────────────┘

   Les dépendances pointent TOUJOURS vers le centre.
```

> 💡 **Règle d'or.** Le domaine ne connaît ni le framework, ni l'ORM, ni HTTP.

| Stack | Mise en pratique |
|---|---|
| Spring Boot | packages `domain` / `application` / `infrastructure`, interfaces de ports, **ArchUnit** pour vérifier les règles de dépendance dans les tests |
| Django | couche `services.py` (écritures) + `selectors.py` (lectures), modèles fins, pas de logique dans les vues/serializers |
| Laravel | classes Action/Service, Repository (si besoin), DTO ; Form Requests pour la validation |
| NestJS | modules + providers, injection de dépendances par interfaces/tokens |
| Front (React/Vue/Angular) | structure **par fonctionnalité** (`features/users/...`) plutôt que par type de fichier, composants « bêtes » + composables/hooks/services pour la logique |

> ⚠️ **Pièges.** sur-ingénierie sur un CRUD simple (hexagonale = coûteuse si le domaine est pauvre) ; couches qui se contentent de renvoyer l'appel à la suivante ; entités ORM exposées directement dans l'API.

## 🧠 2. Logique first-principles

**Mesurer avant de deviner, tracer avant d'optimiser.** Un `GET /products/:id` à 550 ms n'est pas forcément un problème de base : un chronométrage peut révéler 10 ms de SQL, 5 ms de Redis et 500 ms de logger synchrone vers Elasticsearch. Suivez le **p95/p99** plutôt que la moyenne et utilisez le *distributed tracing* (OpenTelemetry, Jaeger, Tempo) — 95 % des ralentissements sont IO-bound, pas CPU-bound. On n'ajoute une couche ou un cache qu'après avoir identifié le goulot.

**Le domaine ne connaît ni framework ni ORM ni HTTP.** Le controller est la seule couche qui parle HTTP : il extrait, désérialise, valide, appelle le service puis choisit le code de statut. Dès qu'on entre dans le service, le protocole n'existe plus : des données pures entrent, un résultat pur sort. Le repository, lui, n'exécute qu'une tâche de base précise (`findBookById`, `getAllBooks`). Si une fonction de service laisse deviner qu'elle est appelée par une API HTTP, la frontière a été percée.

**Django : `services.py` pour les écritures, `selectors.py` pour les lectures.** Les modèles restent fins, les vues et serializers ne contiennent aucune règle métier, et la logique d'écriture à transaction est regroupée dans le service tandis que les requêtes de lecture optimisées (`select_related`, `prefetch_related`) vivent dans le selector.

**Ports et adapters, et backing services.** Le domaine déclare des interfaces (`PaymentGateway`, `UserRepository`) ; l'infrastructure fournit l'implémentation concrète, Stripe ou PostgreSQL. C'est exactement le facteur 4 du 12-Factor App : Postgres local en conteneur et RDS en production doivent être interchangeables par une simple URL d'attachement, sans toucher au code métier.

**ArchUnit pour vérifier les règles de dépendance.** Une règle d'or non testée se dégrade en quelques semaines : un test d'architecture échoue en CI dès que `..domain..` importe Spring, JPA, Servlet ou `..infrastructure..`.

**Config par environnement.** Tout ce qui varie entre déploiements (URL de base, clés d'API, credentials S3, port) vit dans les variables d'environnement ou un Secret Manager, jamais dans le code. Un service qui ne dépend que de ses ports reste portable de Docker à Kubernetes sans recompilation.

---

## 💻 3. Exemples techniques

Un test ArchUnit exécuté en CI interdit au domaine de dépendre de l'infrastructure et des API du framework :

```java
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.ArchRule;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

class DependencyRulesTest {

    private static final JavaClasses CLASSES =
            new ClassFileImporter().importPackages("com.example.shop");

    @Test
    void le_domaine_ne_depend_de_rien_d_externe() {
        ArchRule regle = noClasses()
                .that().resideInAPackage("..domain..")
                .should().dependOnClassesThat()
                .resideInAnyPackage("..infrastructure..", "..web..",
                        "org.springframework..", "jakarta.persistence..", "jakarta.servlet..");

        regle.check(CLASSES);
    }
}
```

Dans une application Django, le cas d'usage d'écriture et la lecture optimisée sont séparés, la vue ne fait que déléguer :

```python
# orders/services.py
from decimal import Decimal
from django.db import transaction

from .models import Order, OrderLine


@transaction.atomic
def place_order(*, user, items: list[tuple[str, int]], unit_prices: dict[str, Decimal]) -> Order:
    total = sum((unit_prices[sku] * quantity for sku, quantity in items), Decimal("0"))
    order = Order.objects.create(user=user, total_cents=int(total * 100), currency="EUR")
    OrderLine.objects.bulk_create([
        OrderLine(
            order=order,
            sku=sku,
            quantity=quantity,
            unit_price_cents=int(unit_prices[sku] * 100),
        )
        for sku, quantity in items
    ])
    return order


# orders/selectors.py
def orders_for_user(*, user):
    return (
        Order.objects.filter(user=user)
        .select_related("user")
        .prefetch_related("lines")
        .order_by("-created_at")
    )


# orders/views.py
from rest_framework.response import Response
from rest_framework.decorators import api_view

from .selectors import orders_for_user


@api_view(["GET"])
def my_orders(request):
    return Response([{"id": str(order.id), "total_cents": order.total_cents}
                     for order in orders_for_user(user=request.user)])
```

Côté Spring, le port reste dans le domaine, l'adaptateur consomme une URL et une clé injectées par l'environnement :

```java
package com.example.shop.domain.port;

public interface PaymentGateway {
    String charge(long amountCents, String currency, String orderId);
}
```

L'adaptateur HTTP implémente ce port et consomme les valeurs injectées :

```java
package com.example.shop.infrastructure.payment;

import com.example.shop.domain.port.PaymentGateway;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class StripePaymentGateway implements PaymentGateway {

    private final RestClient restClient;

    public StripePaymentGateway(RestClient.Builder builder,
                                @Value("${payments.stripe.base-url}") String baseUrl,
                                @Value("${payments.stripe.api-key}") String apiKey) {
        this.restClient = builder.baseUrl(baseUrl)
                .defaultHeader("Authorization", "Bearer " + apiKey)
                .defaultHeader("Content-Type", MediaType.APPLICATION_JSON_VALUE)
                .build();
    }

    private record ChargeRequest(long amount, String currency, String description) {}

    @Override
    public String charge(long amountCents, String currency, String orderId) {
        return restClient.post()
                .uri("/v1/charges")
                .body(new ChargeRequest(amountCents, currency, orderId))
                .retrieve()
                .body(String.class);
    }
}
```

La configuration de l'adaptateur ne change pas d'un environnement à l'autre, seules les variables d'environnement le font :

```yaml
payments:
  stripe:
    base-url: ${STRIPE_BASE_URL}
    api-key: ${STRIPE_API_KEY}
```

---

## 🔗 4. Voir aussi

- [Sujet 19 — N+1 et performance ORM](19-n1-performance-orm.md)
- [Sujet 30 — Tests backend](30-tests-backend.md)
- [Sujet 25 — Configuration et secrets](25-configuration-secrets.md)
- [Architecture en couches, middlewares et request context](../backend-from-first-principles/10-controllers-services-repositories-middlewares-request-context.md)
