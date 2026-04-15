# Schema PostgreSQL

Le schema de reference est celui obtenu apres `alembic upgrade head`.

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
Colonnes importantes : `session_id`, `api_key_id`, `worker_type`, `worker_version`, `expires_at`.

### `worker_leases`

Lease signee rattachee a une session worker.
Colonnes importantes :

- `lease_id`
- `session_id`
- `task_type`
- `payload_ref`
- `expires_at`
- `signature_hash`
- `result_status`
- `result_nonce`
- `result_signature_hash`

`result_status`, `result_nonce` et `result_signature_hash` servent a reserver et identifier une finalisation idempotente avant tout effet metier.

## 4. Queue de travail neutre

### `worker_jobs`

Niveau orchestration d'un lot logique.
Colonnes importantes : `job_kind`, `task_type`, `worker_version`, `status`, `task_total`, `task_processed`, `item_total`, `item_success`, `item_error`, `requested_at`, `started_at`, `finished_at`, `finalized_at`.

### `worker_tasks`

Unite de claim effective pour un worker.
Le payload est stocke en `JSONB`.
Colonnes importantes :

- `task_id`
- `job_id`
- `task_type`
- `worker_version`
- `payload`
- `execution_id`
- `status`
- `claimed_at`
- `claim_expires_at`
- `completed_at`
- `attempt_count`
- `item_total`
- `item_success`
- `item_error`

`execution_id` est attribue a chaque claim et doit correspondre a l'execution active lors de `complete` ou `fail`.

## 5. Stockage canonique des articles

### `articles`

Table principale des articles normalises.
Colonnes importantes : `article_key`, `content_key`, `published_at`, `canonical_url`, `title`, `summary`, `image_url`, `language`, `company_id`.

Notes :

- `content_key` est ajoute par `1_1_article_content_key`
- `1_2_unique_article_content_key` impose une unicite partielle sur `content_key IS NOT NULL`

### `authors`

Referentiel canonique des auteurs normalises.
Colonnes importantes : `normalized_name`, `display_name`.

### `article_authors`

Relation article <-> auteur avec ordre de presentation.
Colonnes importantes : `article_id`, `author_id`, `position`.

### `article_feed_links`

Relation article <-> feed avec horodatage de premiere observation.
Colonnes importantes : `article_id`, `feed_id`, `first_seen_at`.

## 6. Embeddings

### `embedding_manifest`

Etat de reference d'indexation pour `(article_id, worker_version)`.
Colonnes importantes :

- `article_id`
- `article_key`
- `worker_version`
- `index_backend`
- `dimensions`
- `vector_checksum`
- `status`
- `indexed_at`
- `failed_at`
- `failure_reason`

Valeurs de statut attendues : `indexed` ou `failed`.
