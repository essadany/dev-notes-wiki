# 27. Documentation d'API et versioning

> 🎯 **Résumé.** Un client front écrit à la main dérive dès que le back ajoute un champ, et un renommage de champ casse silencieusement tous les consommateurs. On génère OpenAPI depuis le code, on versionne le préfixe d'URL et on ne retire une version qu'après avoir vu son trafic tomber. Le piège : supposer qu'ajouter un champ est toujours anodin — changer le sens d'un champ existant casse déjà le contrat.

---

## 🧭 1. Contexte

**Documentation** : OpenAPI (Swagger) généré depuis le code.

| Stack | Outil |
|---|---|
| Spring | springdoc-openapi |
| DRF | drf-spectacular |
| Laravel | Scribe, L5-Swagger |
| NestJS | `@nestjs/swagger` |

On peut ensuite **générer le client TypeScript** du front (openapi-typescript, orval) pour garder types et API synchronisés.

**Versioning** : préfixe d'URL (`/api/v1/...`, le plus simple), ou header/Accept. Règle : ne jamais casser un contrat existant ; ajouter des champs est sûr, en supprimer ou en changer le sens ne l'est pas ; déprécier avant de retirer.

**Contrat d'interface complet** : page par défaut et max plafonnées, enveloppe d'erreur stable (`code`, `message`, `request_id`, `retryable`), `Idempotency-Key` sur les écritures non idempotentes (sujet 20), verbes choisis selon la sûreté du retry (GET/PUT/DELETE sûrs, POST non). Surveiller le trafic par version pour savoir quand retirer une ancienne version.

---

## 💻 2. Exemples techniques

springdoc-openapi génère le document depuis le code Spring : le serveur documenté inclut le préfixe versionné `/api/v1`.

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI usersApi() {
        return new OpenAPI()
            .info(new Info().title("Users API").version("1.0.0"))
            .addServersItem(new Server().url("/api/v1"));
    }
}
```

NestJS combine le versioning par URI, un préfixe global et la génération du document Swagger au démarrage.

```ts
// src/main.ts
import { VersioningType } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api');
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });

  const config = new DocumentBuilder()
    .setTitle('Users API')
    .setVersion('1.0.0')
    .build();
  SwaggerModule.setup('api/docs', app, SwaggerModule.createDocument(app, config));

  await app.listen(3000);
}
bootstrap();
```

Avec DRF, `drf-spectacular` produit le schéma et sert Swagger/ReDoc ; le schéma généré devient la source du client TypeScript.

```python
# settings.py
INSTALLED_APPS += ['drf_spectacular']

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'Users API',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
}
```

---

## 🔗 3. Voir aussi

- [Sujet 14 — Pagination, tri et filtrage](14-pagination-tri-filtrage.md)
- [Sujet 16 — Validation des entrées](16-validation-entrees.md)
- [Sujet 17 — Gestion d'erreurs globale](17-erreurs-globales.md)
- [Sujet 20 — Transactions et concurrence](20-transactions-concurrence.md)
