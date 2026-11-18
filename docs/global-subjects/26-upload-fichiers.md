# 26. Upload de fichiers

> 🎯 **Résumé.** Faire passer un fichier de 200 Mo par le backend sature la mémoire et le pool de threads. On garde les octets dans un stockage objet avec seulement une clé et des métadonnées en base, des URL pré-signées courtes pour l'envoi direct et un CDN pour les lectures. Le piège : croire le nom ou le type MIME annoncé par le client — il faut valider le contenu réel, renommer, borner la taille et expirer les uploads incomplets.

---

## 🧭 1. Contexte

**Approches**
- **Multipart** classique via le backend (petits fichiers).
- **URL pré-signée** (S3/GCS) : le front envoie directement au stockage, le back ne fait que signer (moins de charge).
- **Upload par morceaux / reprise** (tus, multipart S3) pour gros fichiers.

**Stockage objet (blob store)** : garder les **octets dans le store** et un **pointeur (clé + métadonnées) en base**, jamais le fichier dans une ligne. Choisir une clé répartie (`userId/2026/uuid.jpg`) pour éviter un hot prefix. Prévoir versions et *lifecycle* (expiration des anciennes versions, arrêt des uploads multipart incomplets, descente en tiers froid). Mettre un CDN devant les lectures publiques pour que l'origine ne serve que des cache fills.

| Stack | Solution |
|---|---|
| Spring | `MultipartFile`, limites `spring.servlet.multipart.*` |
| Django | `FileField`, `django-storages` |
| Laravel | `Storage` facade (disques locaux/S3) |
| Node | Multer, busboy |

> ⚠️ **Pièges.** valider type MIME **et** contenu réel, limiter la taille, renommer le fichier (ne jamais utiliser le nom client), stocker hors de la racine web, scan antivirus si besoin. Multipart/reprise au-delà d'un seuil de taille et checksum/ETag pour l'intégrité. URL pré-signée courte et limitée à une opération/une clé, jamais un accès large et durable.

---

## 💻 2. Exemples techniques

Laravel signe une URL PUT valable cinq minutes pour une seule clé et un seul type de contenu : le fichier ne transite pas par le backend.

```php
// app/Http/Controllers/AvatarUploadController.php
namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class AvatarUploadController extends Controller
{
    public function sign(Request $request): JsonResponse
    {
        $data = $request->validate([
            'content_type' => 'required|in:image/jpeg,image/png',
        ]);

        $key = $request->user()->id . '/2026/' . Str::uuid() . '.img';

        return response()->json([
            'key' => $key,
            'url' => Storage::disk('s3')->temporaryUrl(
                $key,
                now()->addMinutes(5),
                ['ContentType' => $data['content_type']],
            ),
            'expires_in' => 300,
        ], 201);
    }
}
```

Django stocke les octets sur S3 via `django-storages` ; le modèle ne garde qu'un `FileField` et les métadonnées de contrôle.

```python
# config/settings.py
INSTALLED_APPS += ['storages']

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3.S3Storage",
        "OPTIONS": {
            "bucket_name": "wiki-uploads",
            "region_name": "eu-west-3",
            "default_acl": None,
        },
    },
}

# documents/models.py
from django.conf import settings
from django.db import models


class Attachment(models.Model):
    file = models.FileField(upload_to='attachments/%Y/%m/')
    content_type = models.CharField(max_length=100)
    size_bytes = models.PositiveBigIntegerField()
    uploaded_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
```

Spring valide le type et la taille avant de confier les octets au stockage objet, avec une clé générée côté serveur et jamais reprise du client.

```java
@RestController
@RequiredArgsConstructor
class AvatarController {
    private final ObjectStorage objectStorage;

    @PostMapping("/avatars")
    Map<String, String> upload(@RequestParam("file") MultipartFile file,
                               @AuthenticationPrincipal String username) throws IOException {
        if (!Set.of("image/jpeg", "image/png").contains(file.getContentType())) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "INVALID_MIME");
        }
        if (file.getSize() > 5L * 1024 * 1024) {
            throw new ResponseStatusException(HttpStatus.PAYLOAD_TOO_LARGE, "TOO_LARGE");
        }
        String key = username + "/2026/" + UUID.randomUUID() + ".jpg";
        objectStorage.put(key, file.getInputStream(), file.getContentType());
        return Map.of("key", key);
    }
}
```

---

## 🔗 3. Voir aussi

- [Sujet 18 — Cache backend et cache HTTP](18-cache-backend-http.md)
- [Sujet 23 — Rate limiting](23-rate-limiting.md)
- [Sujet 29 — Sécurité backend](29-securite-backend.md)
- [Stockage objet, partie 1 (first principles)](../backend-from-first-principles/23-object-storage-part1.md)
