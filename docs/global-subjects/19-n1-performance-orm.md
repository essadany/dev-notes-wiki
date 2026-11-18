# 19. Problème N+1 et performance ORM

> 🎯 **Résumé.** Le N+1 apparaît quand on récupère une liste en une requête puis les relations de chaque ligne une par une : 1 + N allers-retours, et une latence qui explose avec la taille de la page. Chaque ORM a son outil de chargement anticipé (`JOIN FETCH`, `select_related`/`prefetch_related`, `with`, `include`). Le piège principal est de corriger le N+1 avec un `JOIN` massif sur des collections : il faut parfois préférer un chargement groupé et limiter les colonnes sélectionnées.

---

## 🧭 1. Contexte

**Problème** : 1 requête pour la liste + N requêtes pour les relations de chaque élément.

| Stack | Solution |
|---|---|
| Spring / JPA | `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, projections DTO |
| Django | `select_related` (FK/OneToOne, jointure), `prefetch_related` (M2M, inverse) |
| Laravel / Eloquent | `with('relation')`, `Model::preventLazyLoading()` en dev |
| Prisma | `include` / `select`, `relationLoadStrategy` |
| SQLAlchemy | `selectinload`, `joinedload` |

**Autres leviers** : index sur les colonnes filtrées/triées, `EXPLAIN` pour lire un plan d'exécution, ne sélectionner que les colonnes utiles, éviter `COUNT(*)` coûteux, activer le log SQL en dev (django-debug-toolbar, Laravel Debugbar, `spring.jpa.show-sql`).

---

## 💻 2. Exemples techniques

Django : `select_related` pour la FK (une jointure), `prefetch_related` pour les relations multiples (requêtes groupées), et `only()` pour ne ramener que les colonnes utilisées par le serializer :

```python
from rest_framework import generics

from .models import Author
from .serializers import AuthorSerializer


class AuthorList(generics.ListAPIView):
    serializer_class = AuthorSerializer

    def get_queryset(self):
        return (
            Author.objects
            .select_related("country")
            .prefetch_related("books", "books__tags")
            .only("id", "name", "country__name")
            .order_by("name")
        )
```

Spring Data JPA : `@EntityGraph` sur la méthode de repository charge en une seule requête les relations lues par le DTO, et `@BatchSize` limite le nombre de requêtes si une relation doit rester lazy :

```java
@Entity
public class Author {
    @Id
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Country country;

    @BatchSize(size = 50)
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private Set<Book> books = new HashSet<>();
}

@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @EntityGraph(attributePaths = {"country", "books", "books.tags"})
    @Query("select distinct a from Author a order by a.name")
    List<Author> findAllWithRelations();
}
```

Prisma : `select` imbriqué remplace `include` quand on ne veut que quelques champs, et `take` borne le nombre de relations chargées par auteur :

```ts
const authors = await prisma.author.findMany({
  select: {
    id: true,
    name: true,
    country: { select: { name: true } },
    books: {
      select: {
        title: true,
        tags: { select: { label: true } },
      },
      orderBy: { publishedAt: 'desc' },
      take: 5,
    },
  },
  orderBy: { name: 'asc' },
});
```

---

## 🔗 3. Voir aussi

- [Sujet 14 — Pagination, tri et filtrage](14-pagination-tri-filtrage.md)
- [Sujet 18 — Cache backend et cache HTTP](18-cache-backend-http.md)
- [Mastering databases with Postgres](../backend-from-first-principles/12-mastering-databases-with-postgres.md)
