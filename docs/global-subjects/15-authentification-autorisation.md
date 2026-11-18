# 15. Authentification et autorisation

> 🎯 **Résumé.** L'authentification répond à « qui es-tu », l'autorisation à « qu'as-tu le droit de faire » ; ce sont deux couches distinctes, avec des mécanismes distincts selon la stack. Le piège principal est l'IDOR : un token valide ne donne pas le droit de modifier n'importe quelle ressource, le contrôle doit aussi porter sur l'objet visé. JWT sans expiration, secrets faibles et mots de passe non hachés sont les autres fautes classiques.

---

## 🧭 1. Contexte

**Distinction** : authentification = qui es-tu ; autorisation = qu'as-tu le droit de faire.

**Mécanismes** : session + cookie (apps classiques), JWT (APIs/mobile), OAuth2/OIDC (SSO, Google, Keycloak, Auth0).
**Autorisation** : RBAC (rôles), ABAC/permissions fines, contrôle au niveau de l'objet (« cet utilisateur peut-il modifier **cette** ressource ? »).

| Stack | Solution |
|---|---|
| Spring Boot | Spring Security (`SecurityFilterChain`, `oauth2ResourceServer`, `@PreAuthorize`) |
| Django / DRF | sessions, `djangorestframework-simplejwt`, `permission_classes`, `django-allauth`/`dj-rest-auth` |
| Laravel | Sanctum (SPA cookie / tokens API), Passport (OAuth2), Gates & Policies |
| NestJS | Passport strategies, `@UseGuards`, CASL pour les permissions |
| Express | Passport.js, middleware JWT |

> ⚠️ **Pièges.** oublier le contrôle d'accès au niveau objet (faille IDOR, n°1 des failles API) ; JWT sans expiration ni révocation ; secrets JWT faibles ; stocker les mots de passe sans hash adapté (voir sujet 29).

---

## 💻 2. Exemples techniques

Spring Security 6 : chaîne de filtres avec resource server JWT, règles par rôle, et expression `@PreAuthorize` déléguant à un bean qui vérifie la propriété de la facture :

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth -> oauth.jwt(org.springframework.security.config.Customizer.withDefaults()))
            .build();
    }
}

@Component("invoiceAccess")
public class InvoiceAccess {
    private final InvoiceRepository repo;

    public InvoiceAccess(InvoiceRepository repo) {
        this.repo = repo;
    }

    public boolean canEdit(String username, Long invoiceId) {
        return repo.existsByIdAndOwnerEmail(invoiceId, username);
    }
}

@Service
public class InvoiceService {
    private final InvoiceRepository repo;

    public InvoiceService(InvoiceRepository repo) {
        this.repo = repo;
    }

    @PreAuthorize("@invoiceAccess.canEdit(authentication.name, #invoiceId)")
    public Invoice edit(Long invoiceId, InvoiceUpdate update) {
        Invoice invoice = repo.findById(invoiceId).orElseThrow();
        invoice.apply(update);
        return repo.save(invoice);
    }
}
```

Avec DRF, l'authentification JWT est globale mais le `get_object()` refait un contrôle sur l'utilisateur courant : c'est ce geste qui bloque l'IDOR, pas le token :

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
}

# invoices/views.py
from rest_framework import generics, permissions
from rest_framework.exceptions import PermissionDenied

from .models import Invoice
from .serializers import InvoiceSerializer


class InvoiceDetail(generics.RetrieveUpdateAPIView):
    queryset = Invoice.objects.select_related("owner")
    serializer_class = InvoiceSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_object(self):
        invoice = super().get_object()
        if invoice.owner_id != self.request.user.id:
            raise PermissionDenied("Vous ne pouvez pas modifier cette facture.")
        return invoice
```

Côté NestJS, un guard lit le rôle porté par `request.user` (posé par la stratégie Passport) et refuse l'accès avant même d'entrer dans le contrôleur :

```ts
import {
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  Injectable,
} from '@nestjs/common';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context
      .switchToHttp()
      .getRequest<{ user?: { roles: string[] } }>();
    const required = Reflect.getMetadata('roles', context.getHandler()) as
      | string[]
      | undefined;

    if (!required) {
      return true;
    }

    if (!request.user?.roles.some((role) => required.includes(role))) {
      throw new ForbiddenException('Rôle insuffisant');
    }

    return true;
  }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 3 — Authentification côté front](03-authentification-front.md)
- [Sujet 12 — Sécurité front](12-securite-front.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Authentication and authorization for backend engineers](../backend-from-first-principles/08-authentication-and-authorization-for-backend-engineers.md)
