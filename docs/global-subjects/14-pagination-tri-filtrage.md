# 14. Pagination, tri et filtrage

> 🎯 **Résumé.** Une collection infinie ne se renvoie pas en entier : on choisit entre offset/limit, curseur keyset et slice sans total selon le volume et la stabilité des données. Le contrat de réponse reste identique pour tous les endpoints, avec un curseur opaque et un `limit` plafonné côté serveur. Le piège principal est le tri non déterministe : sans clé secondaire, les éléments bougent entre deux pages et le curseur saute des lignes.

---

## 🧭 1. Contexte

**Trois approches**
- **Offset/limit** (`page`, `size`) : simple, mais lent sur gros volumes et instable si les données bougent.
- **Curseur / keyset** (`after=<id>`) : stable et rapide, mais pas d'accès direct à la page N.
- **Slice** (sans total) : évite le `COUNT(*)` coûteux.

**Contrat de réponse stable** : `{ data: [...], next_cursor: string|null, has_more: boolean }`. Le curseur est un token opaque qui encode la clé de tri (ex. timestamp + id), pas un offset visible. Plafonner `limit` côté serveur (valeur par défaut et maximum, ex. 20 / 100), quel que soit le client.

| Stack | Solution |
|---|---|
| Spring Boot | `Pageable` / `Page<T>` / `Slice<T>` (Spring Data) ; keyset via `ScrollPosition` (Spring Data récent) ; filtrage : Specifications, QueryDSL |
| Django / DRF | `PageNumberPagination`, `LimitOffsetPagination`, **`CursorPagination`** ; filtrage : `django-filter` |
| Laravel | `paginate()`, `simplePaginate()`, `cursorPaginate()` ; filtrage : Spatie Query Builder |
| Node | Prisma `skip/take` ou `cursor` ; TypeORM `skip/take` ; Mongoose `limit/skip` |

```java
// Spring
Page<User> page = repo.findAll(PageRequest.of(2, 20, Sort.by("name")));
```
```python
# DRF
class UserList(ListAPIView):
    pagination_class = PageNumberPagination
```
```php
// Laravel
User::orderBy('name')->paginate(20);
```

> ⚠️ **Pièges.** tri non déterministe (ajouter un tri secondaire par id) ; autoriser n'importe quel champ de tri (injection / colonnes non indexées) : utiliser une liste blanche ; plafonner `size` (ex. 100 max).

---

## 💻 2. Exemples techniques

Keyset explicite avec Spring Data JPA : on lit `size + 1` lignes pour savoir s'il y a une suite, et le curseur renvoyé est le dernier `id` vu (tri secondaire inclus dans l'ordre) :

```java
public record UserSlice<T>(T data, Long next_cursor, boolean has_more) {}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("select u from User u where (:after is null or u.id > :after) order by u.id asc")
    List<User> findAfter(@Param("after") Long after, Pageable pageable);
}

@Service
public class UserService {
    private static final int MAX_LIMIT = 100;
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }

    public UserSlice<List<User>> list(Long after, int limit) {
        int size = Math.min(Math.max(limit, 1), MAX_LIMIT);
        List<User> rows = repo.findAfter(after, PageRequest.of(0, size + 1));
        boolean hasMore = rows.size() > size;
        List<User> data = hasMore ? rows.subList(0, size) : rows;
        Long next = hasMore ? data.get(data.size() - 1).getId() : null;

        return new UserSlice<>(data, next, hasMore);
    }
}
```

Avec DRF, `CursorPagination` fait le même travail ; l'ordre doit être unique, donc on ajoute `-id` après `-created_at`, et le `size` demandé est plafonné :

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.CursorPagination",
    "PAGE_SIZE": 20,
}

# users/views.py
from rest_framework import generics
from rest_framework.pagination import CursorPagination

from .models import User
from .serializers import UserSerializer


class UserCursorPagination(CursorPagination):
    page_size = 20
    page_size_query_param = "size"
    max_page_size = 100
    ordering = ("-created_at", "-id")


class UserList(generics.ListAPIView):
    serializer_class = UserSerializer
    pagination_class = UserCursorPagination

    def get_queryset(self):
        return User.objects.order_by("-created_at", "-id")
```

Côté Prisma, le curseur s'appuie sur l'index primaire et le tri est choisi dans une liste blanche : aucune colonne arbitraire ne peut arriver de la query string :

```ts
import { Prisma, PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
const SORTABLE = {
  name: 'name',
  created: 'createdAt',
} as const;

type SortKey = keyof typeof SORTABLE;

export async function listUsers(cursor: string | undefined, sizeRaw: number, sortRaw: string) {
  const take = Math.min(Math.max(sizeRaw || 20, 1), 100);
  const sort: SortKey = sortRaw in SORTABLE ? (sortRaw as SortKey) : 'created';
  const orderBy: Prisma.UserOrderByWithRelationInput[] = [
    { [SORTABLE[sort]]: 'desc' } as Prisma.UserOrderByWithRelationInput,
    { id: 'desc' },
  ];

  const rows = await prisma.user.findMany({
    take: take + 1,
    ...(cursor ? { cursor: { id: cursor }, skip: 1 } : {}),
    orderBy,
    select: { id: true, name: true, createdAt: true },
  });

  const hasMore = rows.length > take;
  const data = hasMore ? rows.slice(0, take) : rows;

  return {
    data,
    next_cursor: hasMore ? data[data.length - 1].id : null,
    has_more: hasMore,
  };
}
```

---

## 🔗 3. Voir aussi

- [Sujet 1 — Cache des données serveur et pagination](01-cache-donnees-serveur-pagination.md)
- [Sujet 16 — Validation des entrées](16-validation-entrees.md)
- [Sujet 19 — Problème N+1 et performance ORM](19-n1-performance-orm.md)
- [Sujet 27 — Documentation d'API et versioning](27-documentation-api-versioning.md)
