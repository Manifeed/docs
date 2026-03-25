# Relations entre tables

## Diagramme ER

```mermaid
erDiagram
    rss_company ||--o{ rss_feeds : owns
    rss_feeds ||--|| rss_feed_runtime : has_runtime
    rss_feeds ||--o{ rss_feed_tags : tagged_with
    rss_tags ||--o{ rss_feed_tags : used_by
    rss_company ||--o{ articles : references
    rss_feeds ||--o{ article_feed_links : links
    articles ||--o{ article_feed_links : appears_in
    articles ||--o{ article_versions : versions

    users ||--o{ user_sessions : owns
    users ||--o{ user_api_keys : grants
    user_api_keys ||--o{ worker_sessions : opens
    worker_sessions ||--o{ worker_leases : issues
    worker_sessions ||--o{ worker_runtime_snapshots : reports

    worker_leases ||--o{ worker_tasks : confirms
    worker_jobs ||--o{ worker_tasks : contains

    worker_leases ||--o{ staging_feed_fetch_results : traces
    rss_feeds ||--o{ staging_feed_fetch_results : fetched
    rss_feeds ||--o{ staging_article_candidates : emits
    articles ||--o{ staging_embedding_requests : requests
    worker_leases ||--o{ staging_embedding_results : returns

    articles ||--o{ embedding_manifest : indexed_as
```

## Vue ASCII

```text
rss_company (1) ---- (0..n) rss_feeds
rss_feeds   (1) ---- (0..1) rss_feed_runtime
rss_feeds   (1) ---- (0..n) rss_feed_tags (n..0) ---- (1) rss_tags
rss_company (1) ---- (0..n) articles
rss_feeds   (1) ---- (0..n) article_feed_links (n..0) ---- (1) articles
articles    (1) ---- (0..n) article_versions

users         (1) ---- (0..n) user_sessions
users         (1) ---- (0..n) user_api_keys
user_api_keys (1) ---- (0..n) worker_sessions
worker_sessions (1) -- (0..n) worker_leases
worker_sessions (1) -- (0..n) worker_runtime_snapshots
worker_jobs   (1) ---- (0..n) worker_tasks
worker_leases (1) ---- (0..n) worker_tasks

worker_leases (1) ---- (0..n) staging_feed_fetch_results (n..0) ---- (1) rss_feeds
rss_feeds     (1) ---- (0..n) staging_article_candidates
articles      (1) ---- (0..n) staging_embedding_requests
worker_leases (1) ---- (0..n) staging_embedding_results
articles      (1) ---- (0..n) embedding_manifest
```

## Notes

- `worker_jobs` et `worker_tasks` remplacent les anciennes familles techniques RSS et embedding.
- `articles` et `article_feed_links` remplacent les anciennes tables `rss_sources` et `rss_source_feeds`.
- `worker_sessions` + `worker_runtime_snapshots` remplacent l'ancien stockage runtime unique.
- `embedding_manifest` est la source relationnelle de reference pour l'etat des embeddings, tandis que Qdrant stocke les vecteurs.
