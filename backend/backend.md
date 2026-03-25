# Backend Manifeed

## Role

Le backend expose l'API web, gere l'authentification, orchestre les jobs RSS et embeddings, et sert de gateway unique pour les workers externes.

Il ne scrape pas directement les feeds et ne calcule pas les embeddings lui-meme.

## Flux principaux

### Catalogue RSS

`POST /rss/sync` synchronise le depot `rss_feeds`, applique les upserts sur `rss_company`, `rss_feeds` et `rss_feed_tags`, puis met a jour `rss_catalog_sync_state`.

### Ingestion RSS

`POST /rss/ingest` ouvre un job de type `rss_scrape` dans `worker_jobs`, puis cree des `worker_tasks` dont le payload contient les feeds a traiter.

Les workers RSS suivent ensuite le protocole commun :

- `POST /workers/sessions/open`
- `POST /workers/tasks/claim`
- `POST /workers/tasks/complete`
- `POST /workers/tasks/fail`
- `POST /workers/heartbeat`

Au `complete`, le backend persiste le fetch et la normalisation dans :

- `staging_feed_fetch_results`
- `staging_article_candidates`
- `articles`
- `article_feed_links`
- `article_versions`
- `ingest_events`
- `dedup_decisions`

`rss_feed_runtime` est mis a jour au fil des resultats utiles du pipeline.

### Embeddings

`POST /sources/embeddings/enqueue` ouvre un job `source_embedding` dans `worker_jobs`, puis cree des `worker_tasks` contenant les articles a vectoriser.

Les workers embeddings utilisent le meme gateway `/workers/*` et renvoient des vecteurs valides pour une `worker_version` donnee.

Le backend :

- controle le checksum et la dimension du vecteur
- ecrit une trace dans `staging_embedding_results`
- pousse le vecteur dans Qdrant
- met a jour `embedding_manifest`
- ecrit `embedding_dead_letters` en cas d'echec d'indexation

## Identite worker

L'identite d'affichage d'un worker est derivee de :

- `users.pseudo`
- `user_api_keys.worker_type`
- `user_api_keys.worker_number`

Le backend conserve aussi les sessions ouvertes et les snapshots de heartbeat dans :

- `worker_sessions`
- `worker_leases`
- `worker_runtime_snapshots`

## Routes structurantes

### Web

- `/auth/*` : comptes, sessions web
- `/account/*` : profil et cles API worker de l'utilisateur courant
- `/admin/*` : administration des utilisateurs
- `/rss/*` : catalogue et declenchement d'ingestion
- `/sources/*` : lecture des articles consolides et enqueue embedding
- `/jobs/*` : lecture et suppression des jobs techniques

### Workers

- `/workers/sessions/open`
- `/workers/tasks/claim`
- `/workers/tasks/complete`
- `/workers/tasks/fail`
- `/workers/heartbeat`
- `/workers/overview`
- `/workers/queues/overview`
- `/workers/queues/{queue_name}/purge`
- `/workers/releases/manifest`
- `/workers/ping`

## Variables importantes

### Runtime general

- `DATABASE_URL`
- `CORS_ORIGINS`
- `DB_POOL_SIZE`
- `DB_MAX_OVERFLOW`
- `DB_POOL_TIMEOUT_SECONDS`
- `DB_POOL_RECYCLE_SECONDS`

### Catalogue RSS

- `RSS_FEEDS_REPOSITORY_URL`
- `RSS_FEEDS_REPOSITORY_BRANCH`
- `RSS_FEEDS_REPOSITORY_PATH`
- `RSS_SCRAPE_TASK_BATCH_SIZE`

### Embeddings

- `SOURCE_EMBEDDING_WORKER_VERSION`
- `SOURCE_EMBEDDING_TASK_BATCH_SIZE`

### Monitoring worker

- `WORKER_CONNECTED_IDLE_THRESHOLD_MS`
- `WORKER_ACTIVE_IDLE_THRESHOLD_MS`
- `WORKER_RELEASE_MANIFEST_PATH`
- `WORKER_RELEASE_MANIFEST_JSON`

## Commandes utiles

```bash
cd infra
make db-reset
make db-migrate
make test-backend
make logs SERVICE=backend
```
