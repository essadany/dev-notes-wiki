# 32. Recherche full-text

> 🎯 **Résumé.** `LIKE '%mot%'` ignore les index, la pertinence, les accents et les fautes de frappe. La solution commence dans PostgreSQL avec `tsvector`/GIN, puis monte vers Meilisearch ou Elasticsearch selon le volume ; l'index n'est qu'une copie dérivée de la base. Le piège : écrire dans l'index sur le chemin de requête et oublier la réindexation complète ainsi que la latence source → searchable.

---

## 🧭 1. Contexte

**Problème** : `LIKE '%mot%'` est lent (pas d'index utilisable), sans pertinence, sans gestion des accents, des pluriels ni des fautes de frappe.

**Trois niveaux de solution**
1. **Recherche native de la base** : PostgreSQL (`tsvector` + index **GIN**, extension `pg_trgm` pour la recherche approximative). Suffisant dans beaucoup de projets, et aucun service en plus à gérer.
2. **Moteur léger** : Meilisearch, Typesense (tolérance aux fautes, très rapides, simples à déployer).
3. **Moteur complet** : Elasticsearch / OpenSearch (gros volumes, agrégations, analyse linguistique poussée).

| Stack | Solution |
|---|---|
| Django | `django.contrib.postgres.search` (`SearchVector`, `SearchQuery`, `SearchRank`, `TrigramSimilarity`) |
| Laravel | **Laravel Scout** (drivers Meilisearch, Typesense, Algolia, ou driver base de données) |
| Spring Boot | Hibernate Search, Spring Data Elasticsearch, requêtes natives PostgreSQL |
| Node | clients Meilisearch/Typesense/Elasticsearch, Prisma (full-text search sur PostgreSQL, selon la version) |

**L'index est une copie dérivée** : la source de vérité reste la base ; prévoir une **réindexation complète**. Alimenter l'index par événements/jobs via un buffer durable (sujets 22 et 35), jamais par des écritures synchrones sur le chemin de requête. Surveiller la **latence d'indexation** (source → searchable), distincte de la latence de requête.

**Ranking et dimensionnement** : démarrer avec BM25 (pertinence lexicale), n'ajouter popularité/fraischeur que si nécessaire. L'analyzer (tokenizer, stemming, synonymes, accents) fait partie du contrat : le changer impose un reindex. Dimensionner les shards depuis le volume total et viser des shards **homogènes** plutôt que minuscules (le fan-out de requête coûte cher) ; répliques pour le QPS de lecture. Autocomplétion : prefix trie/FST ou edge-n-grams, avec debounce côté front.

```python
# Django + PostgreSQL
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank
vector = SearchVector('title', weight='A') + SearchVector('body', weight='B')
query = SearchQuery('cache pagination', config='french')
Article.objects.annotate(rank=SearchRank(vector, query)).filter(rank__gt=0.1).order_by('-rank')
```

## ⚠️ 2. Pièges
- **Synchronisation** entre la base (source de vérité) et l'index externe : passer par des événements/jobs (voir sujets 22 et 35) et prévoir une réindexation complète.
- Configuration linguistique (`french`, stemming, accents).
- Avec Elasticsearch, la pagination profonde (`from` + `size` élevés) est coûteuse : utiliser `search_after`.
- Autocomplétion : index dédié (préfixes/n-grams) plutôt que des `LIKE`.
- Côté front : debounce + annulation (sujet 6) sur la barre de recherche.

---

## 💻 3. Exemples techniques

Le schéma PostgreSQL porte un vecteur généré en français, un index GIN pour la pertinence et un index trigram pour la tolérance aux fautes :

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

ALTER TABLE blog_article
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('french', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('french', coalesce(body, '')), 'B')
  ) STORED;

CREATE INDEX blog_article_search_gin ON blog_article USING GIN (search_vector);
CREATE INDEX blog_article_title_trgm ON blog_article USING GIN (title gin_trgm_ops);

SELECT id, title, ts_rank(search_vector, query) AS rank
FROM blog_article, plainto_tsquery('french', 'cache pagination') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

Côté Node, la configuration Meilisearch est versionnée avec le code et l'alimentation passe par un job qui attend la fin de la tâche d'indexation :

```js
const { MeiliSearch } = require('meilisearch');

const client = new MeiliSearch({
  host: process.env.MEILI_HOST,
  apiKey: process.env.MEILI_ADMIN_KEY,
});

async function configureSearch() {
  const index = client.index('articles');
  await index.updateSettings({
    searchableAttributes: ['title', 'body'],
    displayedAttributes: ['id', 'title', 'body', 'published_at'],
    filterableAttributes: ['tenant_id', 'published_at'],
    sortableAttributes: ['published_at'],
    typoTolerance: {
      enabled: true,
      minWordSizeForTypos: { oneTypo: 4, twoTypos: 8 },
    },
    synonyms: { cache: ['memoisation', 'mise en cache'] },
  });

  const task = await index.addDocuments([
    {
      id: 101,
      title: 'Cache serveur et pagination',
      body: 'Mettre en cache les réponses et paginer avec un curseur.',
      tenant_id: 1,
      published_at: '2026-11-18T14:30:00Z',
    },
    {
      id: 102,
      title: 'Recherche full-text PostgreSQL',
      body: 'Index GIN et tsvector pour la recherche en français.',
      tenant_id: 1,
      published_at: '2026-11-17T09:00:00Z',
    },
  ]);
  await client.waitForTask(task.taskUid);
}

configureSearch();
```

Pour Elasticsearch, l'analyzer français et l'edge-n-gram d'autocomplétion sont déclarés dans le mapping, puis la pagination profonde utilise `search_after` :

```json
PUT articles
{
  "settings": {
    "analysis": {
      "filter": {
        "stop_fr": { "type": "stop", "stopwords": "_french_" },
        "stem_fr": { "type": "stemmer", "language": "french" },
        "edge_ngram_filter": { "type": "edge_ngram", "min_gram": 2, "max_gram": 20 }
      },
      "analyzer": {
        "search_fr": {
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "stop_fr", "stem_fr"]
        },
        "autocomplete_fr": {
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "edge_ngram_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "search_fr",
        "fields": { "prefix": { "type": "text", "analyzer": "autocomplete_fr" } }
      },
      "body": { "type": "text", "analyzer": "search_fr" },
      "published_at": { "type": "date" }
    }
  }
}
```

La requête suivante reprend le curseur renvoyé par la page précédente :

```json
GET articles/_search
{
  "size": 20,
  "query": {
    "match": { "title": { "query": "cache pagination", "analyzer": "search_fr" } }
  },
  "sort": [
    { "_score": "desc" },
    { "published_at": "desc" },
    { "id": "asc" }
  ],
  "search_after": [1.8723, "2026-11-17T09:00:00Z", "101"]
}
```

---

## 🔗 4. Voir aussi

- [Sujet 14 — Pagination, tri et filtrage](14-pagination-tri-filtrage.md)
- [Sujet 18 — Cache backend et HTTP](18-cache-backend-http.md)
- [Sujet 22 — Jobs asynchrones et files](22-jobs-asynchrones-files.md)
- [Recherche full-text et Elasticsearch](../backend-from-first-principles/15-full-text-search-elasticsearch.md)
