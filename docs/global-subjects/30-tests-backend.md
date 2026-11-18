# 30. Tests backend

> 🎯 **Résumé.** Un projet sans tests se limite à « ça marche en local », et un mock de base masque précisément les bugs de contraintes et de transactions. On construit une pyramide : beaucoup de tests unitaires, quelques tests d'API et une vraie base du même moteur qu'en prod via Testcontainers. Le piège : utiliser SQLite en test alors que la prod tourne sur PostgreSQL, ou déployer sans test de contrat entre services.

---

## 🧭 1. Contexte

| Niveau | Spring | Django | Laravel | Node |
|---|---|---|---|---|
| Unitaire | JUnit 5 + Mockito | pytest / `unittest` | PHPUnit / Pest | Jest / Vitest |
| Intégration (API) | `@SpringBootTest`, MockMvc | `APIClient` (DRF) | tests Feature | Supertest |
| Base de données réelle | **Testcontainers** | pytest-django + DB de test | `RefreshDatabase` | Testcontainers |

Bonnes pratiques : pyramide de tests (beaucoup d'unitaires, quelques intégrations, peu d'E2E) ; tester avec une vraie DB (même moteur qu'en prod, pas SQLite si prod = PostgreSQL) ; tests de contrat si plusieurs services.

---

## 💻 2. Exemples techniques

Le test unitaire JUnit 5 isole la logique de calcul avec Mockito et une assertion numérique tolérant les différences d'échelle `BigDecimal`.

```java
@ExtendWith(MockitoExtension.class)
class PriceServiceTest {
    @Mock
    private TaxRepository taxRepository;

    @InjectMocks
    private PriceService priceService;

    @Test
    void appliesFrenchVatRate() {
        when(taxRepository.rateFor("FR")).thenReturn(new BigDecimal("20"));

        BigDecimal total = priceService.ttc(new BigDecimal("100.00"), "FR");

        assertThat(total).isEqualByComparingTo("120.00");
    }
}
```

Côté DRF, `APIClient` vérifie le contrat HTTP : authentification requise et utilisateurs inactifs absents de la réponse paginée.

```python
# tests/test_users.py
import pytest
from rest_framework.test import APIClient


@pytest.mark.django_db
def test_user_list_requires_authentication():
    response = APIClient().get('/api/v1/users/')
    assert response.status_code == 401


@pytest.mark.django_db
def test_user_list_hides_inactive_users(django_user_model):
    active = django_user_model.objects.create_user(username='alice', password='motdepasse')
    django_user_model.objects.create_user(
        username='bob', password='motdepasse', is_active=False
    )

    client = APIClient()
    client.force_authenticate(user=active)
    response = client.get('/api/v1/users/')

    assert response.status_code == 200
    assert [row['username'] for row in response.json()['results']] == ['alice']
```

Le test d'intégration Spring démarre un vrai PostgreSQL dans un conteneur : les contraintes et le dialecte SQL sont ceux de la production.

```java
@Testcontainers
@SpringBootTest
class UserRepositoryIT {
    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine").withDatabaseName("wiki_test");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository repository;

    @Test
    void findsUserByEmail() {
        repository.save(new User("alice@example.com"));

        assertThat(repository.findByEmail("alice@example.com")).isPresent();
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 21 — Migrations de base de données](21-migrations-base-de-donnees.md)
- [Sujet 27 — Documentation d'API et versioning](27-documentation-api-versioning.md)
- [Sujet 31 — Conteneurisation, CI/CD, health checks](31-conteneurisation-cicd-health-checks.md)
- [Tester un backend (first principles)](../backend-from-first-principles/26-testing-backend.md)
