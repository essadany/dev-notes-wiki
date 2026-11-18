# 29. Sécurité backend

> 🎯 **Résumé.** Une requête concaténée, un contrôle d'accès oublié sur un identifiant et un mot de passe hashé en MD5 suffisent à vider une base. On suit OWASP Top 10 et OWASP API Security Top 10 : requêtes paramétrées, DTO explicites, Argon2/bcrypt, contrôle au niveau objet et moindre privilège. Le piège : se contenter du contrôle côté front ou d'un CORS à `*` — le back doit revérifier chaque droit et chaque origine.

---

## 🧭 1. Contexte

Référence : **OWASP Top 10** et **OWASP API Security Top 10**.

| Risque | Parade |
|---|---|
| Injection SQL | requêtes paramétrées / ORM, jamais de concaténation |
| Broken access control (IDOR) | contrôle d'accès au niveau objet systématique |
| Mots de passe | hash lent et salé : **Argon2** ou **bcrypt** (Spring `PasswordEncoder`, Django hashers, Laravel `Hash`) |
| CORS mal configuré | liste blanche d'origines, pas de `*` avec credentials |
| CSRF | tokens CSRF ou `SameSite`, selon le mode d'auth |
| Mass assignment | DTO / `fillable` (Laravel) / serializers explicites |
| Exposition de données | ne renvoyer que les champs nécessaires |
| SSRF, désérialisation | valider les URLs sortantes, éviter les formats non sûrs |
| Dépendances vulnérables | Dependabot, `npm audit`, OWASP Dependency-Check, `pip-audit` |
| Headers | HSTS, `X-Content-Type-Options`, CSP |

Principe de moindre privilège partout (comptes DB, rôles cloud).

---

## 💻 2. Exemples techniques

Spring Security hash les mots de passe avec Argon2, active le contrôle par méthode et pose HSTS ainsi que la protection CSRF par cookie.

```java
@Configuration
@EnableMethodSecurity
class SecurityConfig {
    @Bean
    PasswordEncoder passwordEncoder() {
        return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .csrf(csrf -> csrf.csrfTokenRepository(
                CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .headers(headers -> headers
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true))
                .contentTypeOptions(Customizer.withDefaults()));
        return http.build();
    }
}
```

Le serializer DRF explicite limite les champs exposés (pas de mass assignment) et la requête SQL reste paramétrée.

```python
# api/serializers.py
from rest_framework import serializers
from users.models import User


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'email', 'display_name']
        read_only_fields = ['id', 'is_staff']

# api/views.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from users.models import User
from api.serializers import UserSerializer


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def search_users(request):
    prefix = request.query_params.get('q', '')
    users = User.objects.raw(
        'SELECT id, email, display_name FROM users_user '
        'WHERE email LIKE %s ORDER BY id LIMIT 20',
        [prefix + '%'],
    )
    return Response(UserSerializer(users, many=True).data)
```

Laravel valide l'URL sortante et la limite à une liste blanche d'hôtes avant tout appel HTTP, ce qui bloque le SSRF.

```php
// app/Http/Controllers/AvatarProxyController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Validator;

class AvatarProxyController extends Controller
{
    public function show(Request $request)
    {
        $data = Validator::make($request->all(), [
            'url' => ['required', 'url', 'regex:/^https:\/\//'],
        ])->validate();

        $host = parse_url($data['url'], PHP_URL_HOST);
        if (!in_array($host, ['avatars.example.com', 'cdn.example.com'], true)) {
            abort(422, 'Hôte non autorisé');
        }

        return Http::timeout(5)->get($data['url'])->body();
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 15 — Authentification et autorisation](15-authentification-autorisation.md)
- [Sujet 16 — Validation des entrées](16-validation-entrees.md)
- [Sujet 25 — Configuration et secrets](25-configuration-secrets.md)
- [Sécurité backend (first principles)](../backend-from-first-principles/20-backend-security.md)
