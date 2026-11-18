# 16. Validation des entrées

> 🎯 **Résumé.** Toute donnée venant du client est hostile : on valide d'abord la forme, le type et les bornes, puis les règles métier. La validation est portée par des DTO distincts des entités pour éviter le *mass assignment* (`isAdmin: true` envoyé dans un JSON). Le piège principal est de faire confiance à la validation front : elle sert l'UX, jamais de garde-fou serveur, et le back doit revalider systématiquement.

---

## 🧭 1. Contexte

**Principe** : ne jamais faire confiance aux données du client. Valider forme, type, bornes, puis règles métier.

| Stack | Solution |
|---|---|
| Spring Boot | Bean Validation (`@Valid`, `@NotBlank`, `@Size`...) sur les DTO |
| Django / DRF | Serializers (`validate_<champ>`, `validate`) |
| Laravel | `FormRequest` (`rules()`), `Validator` |
| NestJS | `class-validator` + `ValidationPipe` |
| Express | Zod, Joi, express-validator |

Utiliser des **DTO** distincts des entités (éviter le *mass assignment* : un client qui envoie `isAdmin: true`).

---

## 💻 2. Exemples techniques

NestJS : le DTO porte les contraintes `class-validator` et le `ValidationPipe` global avec `whitelist` rejette les champs non déclarés (dont un éventuel `isAdmin`) :

```ts
// create-user.dto.ts
import { IsEmail, IsInt, IsString, Length, Max, Min } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @Length(2, 80)
  name!: string;

  @IsEmail()
  email!: string;

  @IsInt()
  @Min(18)
  @Max(120)
  age!: number;
}

// main.ts
import { ValidationPipe } from '@nestjs/common';

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

DRF : les validateurs par champ puis le validateur global ajoutent les règles métier après le contrôle de forme :

```python
from rest_framework import serializers

from .models import User


class CreateUserSerializer(serializers.Serializer):
    name = serializers.CharField(min_length=2, max_length=80)
    email = serializers.EmailField()
    age = serializers.IntegerField(min_value=18, max_value=120)

    def validate_email(self, value):
        if User.objects.filter(email__iexact=value).exists():
            raise serializers.ValidationError("Cet email est déjà utilisé.")
        return value

    def validate(self, attrs):
        if attrs["name"].strip().lower() == "admin":
            raise serializers.ValidationError({"name": "Nom réservé."})
        return attrs
```

Spring Boot : un record sert de DTO, les annotations Bean Validation sont posées sur les composants et le contrôleur déclenche la validation avec `@Valid` :

```java
public record CreateUserRequest(
        @NotBlank @Size(max = 80) String name,
        @NotBlank @Email String email,
        @Min(18) @Max(120) int age
) {}

@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService service;

    public UserController(UserService service) {
        this.service = service;
    }

    @PostMapping
    public User create(@Valid @RequestBody CreateUserRequest body) {
        return service.create(body);
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 4 — Formulaires et validation](04-formulaires-validation.md)
- [Sujet 17 — Gestion d'erreurs globale](17-erreurs-globales.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Validations and transformations for backend engineers](../backend-from-first-principles/09-validations-and-transformations-for-backend-engineers.md)
