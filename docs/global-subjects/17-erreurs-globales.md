# 17. Gestion d'erreurs globale

> 🎯 **Résumé.** Une API doit renvoyer la même enveloppe d'erreur sur tous ses endpoints, avec un `request_id` pour corréler logs et traces, sans jamais exposer de stack trace. Le standard RFC 9457 « Problem Details » fournit les champs de base, enrichis ici de `retryable` pour que le client sache s'il peut réessayer. Le piège principal est l'incohérence : un endpoint qui renvoie `{error: "..."}` et un autre une chaîne brute rendent le client incapable de réagir correctement, surtout face à un `429`.

---

## 🧭 1. Contexte

**Principe** : format d'erreur uniforme et cohérent, jamais de stack trace au client. Standard : **RFC 9457 « Problem Details »** (`type`, `title`, `status`, `detail`), enrichi de `request_id` (corrélation avec les logs) et `retryable` (le client sait s'il peut réessayer).

| Stack | Solution |
|---|---|
| Spring Boot | `@RestControllerAdvice` + `@ExceptionHandler`, `ProblemDetail` |
| Django / DRF | `EXCEPTION_HANDLER` personnalisé, `APIException` |
| Laravel | `bootstrap/app.php` / `Handler` (rendu des exceptions), exceptions custom |
| NestJS | `ExceptionFilter`, `HttpException` |
| Express | middleware d'erreur à 4 paramètres `(err, req, res, next)` |

Codes HTTP : 400 requête invalide, 401 non authentifié, 403 interdit, 404 introuvable, 409 conflit, 422 validation, 429 trop de requêtes, 5xx erreur serveur. Une même enveloppe sur **tous** les endpoints ; un `429` porte `Retry-After` (et idéalement `X-RateLimit-*`).

---

## 💻 2. Exemples techniques

Spring Boot : un `@RestControllerAdvice` transforme les erreurs de validation et les erreurs inattendues en `ProblemDetail`, avec l'identifiant de requête posé par un filtre amont :

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail handleValidation(MethodArgumentNotValidException ex,
                                   HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY, "Validation échouée");
        problem.setType(URI.create("urn:problem:validation"));
        problem.setTitle("Champs invalides");
        problem.setProperty("request_id", request.getAttribute("request_id"));
        problem.setProperty("retryable", false);
        return problem;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "Erreur interne");
        problem.setTitle("Erreur serveur");
        problem.setProperty("request_id", request.getAttribute("request_id"));
        problem.setProperty("retryable", true);
        return problem;
    }
}
```

NestJS : un filtre global `@Catch()` garantit l'enveloppe, ajoute `Retry-After` sur un `429` et masque le message technique sur les 5xx :

```ts
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class ProblemFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
    const title =
      status === HttpStatus.NOT_FOUND
        ? 'Introuvable'
        : status >= 500
          ? 'Erreur serveur'
          : 'Requête invalide';

    if (status === HttpStatus.TOO_MANY_REQUESTS) {
      response.setHeader('Retry-After', '30');
    }

    response.status(status).json({
      type: `urn:problem:http-${status}`,
      title,
      status,
      detail:
        status >= 500
          ? 'Erreur interne'
          : exception instanceof Error
            ? exception.message
            : 'Requête invalide',
      request_id: request.header('x-request-id') ?? 'non-disponible',
      retryable: status >= 500 || status === HttpStatus.TOO_MANY_REQUESTS,
    });
  }
}
```

DRF : un `EXCEPTION_HANDLER` custom habille les erreurs gérées par DRF et fournit une enveloppe pour les exceptions non gérées (qui, sinon, échappent au handler) :

```python
# settings.py
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "core.errors.problem_exception_handler",
}

# core/errors.py
import uuid

from rest_framework.response import Response
from rest_framework.views import exception_handler as drf_exception_handler


def problem_exception_handler(exc, context):
    response = drf_exception_handler(exc, context)
    request_id = context["request"].headers.get("X-Request-Id") or str(uuid.uuid4())

    if response is None:
        return Response(
            {
                "type": "urn:problem:server-error",
                "title": "Erreur serveur",
                "status": 500,
                "detail": "Erreur interne",
                "request_id": request_id,
                "retryable": True,
            },
            status=500,
            headers={"X-Request-Id": request_id},
        )

    data = response.data if isinstance(response.data, dict) else {}
    status = response.status_code
    response.data = {
        "type": f"urn:problem:http-{status}",
        "title": str(data.get("detail", "Requête invalide")),
        "status": status,
        "detail": (
            "Consultez les erreurs par champ."
            if data and "detail" not in data
            else str(data.get("detail", ""))
        ),
        "request_id": request_id,
        "retryable": status == 429 or status >= 500,
    }
    if status == 429:
        response["Retry-After"] = "30"
    return response
```

---

## 🔗 3. Voir aussi

- [Sujet 8 — Gestion d'erreurs côté front](08-erreurs-front.md)
- [Sujet 23 — Rate limiting](23-rate-limiting.md)
- [Sujet 24 — Logging et observabilité](24-logging-observabilite.md)
- [Error handling and fault tolerance](../backend-from-first-principles/16-error-handling-and-fault-tolerance.md)
