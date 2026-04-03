# Schema PostgreSQL

Le schema de reference est celui cree par `1_0_baseline`.

## 1. Catalogue RSS

### `rss_company`

Entite editoriale source d'un ou plusieurs feeds.
Colonnes importantes : `name`, `host`, `icon_url`, `country`, `language`, `fetchprotection`, `enabled`.

### `rss_feeds`

Catalogue des feeds RSS connus.
Colonnes importantes : `company_id`, `url`, `section`, `trust_score`, `enabled`.

### `rss_feed_runtime`

Etat runtime consolide d'un feed.
Colonnes importantes : `last_status`, `last_scraped_at`, `last_error_at`, `last_error_code`, `etag`, `last_feed_update`, `last_article_published_at`.

### `rss_tags` et `rss_feed_tags`

Taxonomie du catalogue.

### `rss_catalog_sync_state`

Trace de la derniere synchro du depot catalogue.

## 2. Authentification

### `users`

Compte applicatif : `email`, `pseudo`, `password_hash`, `role`, `is_active`, `api_access_enabled`.

### `user_sessions`

Sessions web du frontend/admin.

### `user_api_keys`

Cles Bearer dediees aux workers.
Colonnes importantes : `label`, `worker_type`, `worker_number`, `key_prefix`, `key_hash`, `last_used_at`, `revoked_at`.

## 3. Control plane worker

### `worker_sessions`

Session HTTP logique ouverte par un worker authentifie.

### `worker_leases`

Lease signee rattachee a une session et a une task.

### `worker_runtime_snapshots`

Snapshots de heartbeat : charge active, dernier label de task, dernier message d'erreur, hints reseau/CPU/GPU.

### `worker_quotas`

Quotas et priorites par scope et type de task.

## 4. Queue de travail neutre

### `worker_jobs`

Niveau orchestration d'un lot logique.
Colonnes importantes : `job_kind`, `task_type`, `queue_lane`, `worker_version`, `status`, `task_total`, `item_total`, `item_success`, `item_error`, `finalized_at`.

### `worker_tasks`

Unite de claim effective pour un worker.
Le payload est stocke en `JSONB`.
Colonnes importantes : `task_type`, `queue_lane`, `worker_version`, `status`, `attempt_count`, `claim_expires_at`, `lease_id`.

## 5. Stockage canonique des articles

### `articles`

Table principale des articles normalises.
Colonnes importantes : `article_key`, `published_at`, `canonical_url`, `title`, `summary`, `image_url`, `language`, `company_id`.

### `authors`

Referentiel canonique des auteurs normalises.
Colonnes importantes : `normalized_name`, `display_name`.

### `article_authors`

Relation article <-> auteur avec ordre de presentation.
Colonnes importantes : `article_id`, `author_id`, `position`.

### `article_feed_links`

Relation article <-> feed avec horodatage de premiere observation.

### `article_versions`

Historique leger des variantes capturees d'un article.

## 6. Buffers de pipeline

### `staging_feed_fetch_results`

Trace d'un fetch RSS brut par `trace_id` et `lease_id`.

### `staging_article_candidates`

Candidats d'articles derives du parsing RSS avant promotion canonique.

### `staging_embedding_requests`

Demandes d'embedding generees pour des articles.

### `staging_embedding_results`

Trace des embeddings recus avant indexation definitive.

## 7. Embeddings

### `embedding_manifest`

Etat de reference d'indexation pour `(article_id, worker_version)`.
Valeurs de statut attendues : `indexed` ou `failed`.

### `embedding_dead_letters`

Echecs d'indexation ou de validation conserves avec snapshot minimal du payload.

## 8. Audit

### `ingest_events`

Evenements structures du pipeline par `trace_id`.

### `dedup_decisions`

Journal des decisions de dedup par `article_key`.
