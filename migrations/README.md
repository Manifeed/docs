# Migrations d'architecture

Cette documentation ne decrit plus une migration progressive `v1 -> v2`.
La base de reference a ete reinitialisee autour d'une seule baseline Alembic : `1_0_baseline`.

## Etat retenu

Le schema conserve uniquement :

- le catalogue RSS : `rss_company`, `rss_feeds`, `rss_feed_runtime`, `rss_tags`, `rss_feed_tags`, `rss_catalog_sync_state`
- l'authentification : `users`, `user_sessions`, `user_api_keys`
- le control plane worker : `worker_sessions`, `worker_leases`, `worker_runtime_snapshots`, `worker_quotas`
- la queue de travail neutre : `worker_jobs`, `worker_tasks`
- le stockage canonique des articles : `articles`, `article_feed_links`, `article_versions`
- les buffers de pipeline : `staging_feed_fetch_results`, `staging_article_candidates`, `staging_embedding_requests`, `staging_embedding_results`
- le suivi des embeddings : `embedding_manifest`, `embedding_dead_letters`
- l'audit de pipeline : `ingest_events`, `dedup_decisions`

## Etat supprime

Les familles suivantes n'existent plus dans le schema cible :

- `rss_scrape_*`
- `rss_embedding_*`
- `rss_sources`, `rss_source_feeds`
- les vues `*_compat`
- `worker_runtime`
- toute nomenclature fonctionnelle `v2`

## Regle d'exploitation

En developpement et en test, la migration s'applique sur une base vide :

1. reset du schema PostgreSQL
2. `alembic upgrade head`
3. chargement optionnel des donnees applicatives

Il n'y a plus de chaine de migrations transitoires a rejouer pour reconstruire l'etat courant.

## Documents de reference

- `docs/postgres/postgres.md`
- `docs/postgres/schema.md`
- `docs/postgres/relations.md`
- `docs/backend/backend.md`
- `docs/backend/workers_rust.md`
