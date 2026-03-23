# Relations entre tables

## Diagramme ER

```mermaid
erDiagram
    rss_company ||--o{ rss_feeds : owns
    rss_feeds ||--|| rss_feed_runtime : has_runtime
    rss_feeds ||--o{ rss_feed_tags : tagged_with
    rss_tags ||--o{ rss_feed_tags : used_by

    rss_feeds ||--o{ rss_source_feeds : references
    rss_sources ||--o{ rss_source_feeds : linked_to
    rss_sources ||--o{ rss_source_embeddings : embedded_as

    users ||--o{ user_sessions : owns
    users ||--o{ user_api_keys : grants
    user_api_keys ||--|| worker_runtime : exposes

    rss_scrape_jobs ||--o{ rss_scrape_tasks : contains
    rss_scrape_tasks ||--o{ rss_scrape_task_items : contains
    rss_feeds ||--o{ rss_scrape_task_items : queued_from
    rss_scrape_jobs ||--o{ rss_scrape_result_feeds : produces
    rss_scrape_tasks ||--o{ rss_scrape_result_feeds : produced_by
    rss_feeds ||--o{ rss_scrape_result_feeds : targets
    rss_scrape_result_feeds ||--o{ rss_scrape_result_sources : expands_to
    rss_scrape_jobs ||--o{ rss_scrape_result_sources : groups

    rss_embedding_jobs ||--o{ rss_embedding_tasks : contains
    rss_embedding_tasks ||--o{ rss_embedding_task_items : contains
    rss_sources ||--o{ rss_embedding_task_items : queued_from
    rss_embedding_jobs ||--o{ rss_embedding_results : produces
    rss_embedding_tasks ||--o{ rss_embedding_results : produced_by
    rss_sources ||--o{ rss_embedding_results : targets
```

## Vue ASCII

```text
rss_company (1) ---- (0..n) rss_feeds
rss_feeds   (1) ---- (0..1) rss_feed_runtime
rss_feeds   (1) ---- (0..n) rss_feed_tags (n..0) ---- (1) rss_tags

rss_feeds   (1) ---- (0..n) rss_source_feeds (n..0) ---- (1) rss_sources
rss_sources (1) ---- (0..n) rss_source_embeddings

users        (1) ---- (0..n) user_sessions
users        (1) ---- (0..n) user_api_keys
user_api_keys (1) --- (0..1) worker_runtime

rss_scrape_jobs (1) ---- (0..n) rss_scrape_tasks
rss_scrape_tasks (1) ---- (0..n) rss_scrape_task_items (n..0) ---- (1) rss_feeds
rss_scrape_jobs (1) ---- (0..n) rss_scrape_result_feeds
rss_scrape_result_feeds (1) ---- (0..n) rss_scrape_result_sources

rss_embedding_jobs (1) ---- (0..n) rss_embedding_tasks
rss_embedding_tasks (1) ---- (0..n) rss_embedding_task_items (n..0) ---- (1) rss_sources
rss_embedding_jobs (1) ---- (0..n) rss_embedding_results
rss_embedding_results (n..0) ---- (1) rss_sources
```

## Notes importantes

- `rss_source_embeddings` et `rss_embedding_results` sont relies logiquement par `worker_version`, pas par un `embedding_model_id`.
- `user_api_keys` est la vraie source d'identite des workers ; `worker_runtime` n'est qu'un etat courant.
- le nom de worker expose par le backend est derive de `users.pseudo + worker_type + worker_number`.
- `rss_scrape_result_*` et `rss_embedding_results` restent des tables techniques de transit avant finalisation.
