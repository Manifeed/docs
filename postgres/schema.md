# Schema PostgreSQL

Le schema de reference est defini par les migrations Alembic de `infra/postgres_migration/alembic/versions/`.
La baseline actuelle est `v1_initialization`, completee par `v1_1` a `v1_5`.

## Conventions

- les tables metier et techniques vivent dans le meme schema `public`
- les dates sont stockees en `TIMESTAMPTZ`
- les jobs sont separes entre pipeline RSS et pipeline embeddings
- les workers n'ecrivent jamais directement dans les tables finales

## 1. Catalogue RSS

### `rss_company`

Reference des entites editoriales.

Colonnes clefs :

- `id`
- `name` unique
- `host`
- `icon_url`
- `country`
- `language`
- `fetchprotection` (`0..2`)
- `enabled`

### `rss_feeds`

Reference des flux RSS connus.

Colonnes clefs :

- `id`
- `company_id` nullable
- `url` unique
- `section`
- `trust_score` (`0.0..1.0`)
- `enabled`

Indexes notables :

- `idx_rss_feeds_company_id`
- `idx_rss_feeds_enabled` (partiel sur `enabled = true`)

### `rss_feed_runtime`

Etat courant d'un feed apres finalisation des jobs RSS.

Colonnes clefs :

- `feed_id` (PK et FK vers `rss_feeds`)
- `last_status`
- `last_scraped_at`
- `last_success_at`
- `last_error_at`
- `last_error_message`
- `consecutive_error_count`
- `etag`
- `last_feed_update`
- `last_article_published_at`

### `rss_tags` / `rss_feed_tags`

Taxonomie simple pour le catalogue.

## 2. Sources consolidees

### `rss_sources`

Table canonique des articles consolides.

Colonnes clefs :

- `id`
- `ingested_at`
- `published_at`
- `url`
- `title`
- `summary`
- `author`
- `image_url`
- `identity_key` unique

Notes :

- l'identite est basee sur une URL normalisee ou un fallback derive du contenu ;
- la migration `v1_4_rss_source_identity` recentre la canonisation autour des URLs normalisees.

### `rss_source_feeds`

Table de liaison many-to-many entre sources consolidees et feeds.

Colonnes clefs :

- `feed_id`
- `source_id`
- `ingested_at`

PK composite :

- `(feed_id, source_id)`

## 3. Embeddings

### `embedding_models`

Catalogue des modeles d'embeddings connus.

Colonnes clefs :

- `id`
- `code` unique
- `label`
- `dimensions`
- `created_at`
- `updated_at`

### `rss_source_embeddings`

Stockage final des embeddings actifs par source et modele.

Colonnes clefs :

- `source_id`
- `embedding_model_id`
- `embedding` (`float[]`)
- `updated_at`

PK composite :

- `(source_id, embedding_model_id)`

La revision `v1_5_embed_finalization_idx` ajoute un index supplementaire sur
`(embedding_model_id, source_id)` pour accelerer la finalisation et la projection.

## 4. Pipeline RSS

### `rss_scrape_jobs`

Un job par requete backend `POST /rss/ingest`.

Colonnes clefs :

- `id` (`rss-...`)
- `status`
- `requested_at`
- `started_at`
- `finished_at`
- `task_total`
- `task_processed`
- `feed_total`
- `feed_success`
- `feed_error`
- `finalized_at`

### `rss_scrape_tasks`

Tasks batch attribuees au worker RSS.

Colonnes clefs :

- `id`
- `job_id`
- `status`
- `claimed_at`
- `completed_at`
- `claim_expires_at`
- `feed_total`
- `feed_success`
- `feed_error`

### `rss_scrape_task_items`

Membres d'une task RSS, un item par feed.

Colonnes clefs :

- `task_id`
- `feed_id`
- `fetchprotection`
- `etag`
- `last_feed_update`
- `last_article_published_at`

Notes :

- la revision `v1_2_rss_task_feed_unique` rend `feed_id` unique a l'echelle de la queue ;
- cela evite que le meme feed soit simultanement attache a plusieurs tasks RSS.

### `rss_scrape_result_feeds`

Resultat brut par feed traite.

Colonnes clefs :

- `id`
- `job_id`
- `task_id`
- `feed_id`
- `worker_id`
- `fetchprotection_used`
- `resolved_fetchprotection`
- `status`
- `status_code`
- `error_message`
- `new_etag`
- `last_feed_update`
- `last_article_published_at`
- `created_at`

### `rss_scrape_result_sources`

Sources brutes extraites depuis un resultat feed.

Colonnes clefs :

- `id`
- `result_feed_id`
- `job_id`
- `published_at`
- `url`
- `title`
- `summary`
- `author`
- `image_url`
- `created_at`

## 5. Pipeline embeddings

### `rss_embedding_jobs`

Un job par requete backend `POST /sources/embeddings/enqueue`.

Colonnes clefs :

- `id` (`embedding-...`)
- `status`
- `requested_at`
- `started_at`
- `finished_at`
- `task_total`
- `task_processed`
- `embedding_total`
- `embedding_success`
- `embedding_error`
- `embedding_model_id`
- `finalized_at`

### `rss_embedding_tasks`

Tasks batch attribuees au worker d'embeddings.

Colonnes clefs :

- `id`
- `job_id`
- `status`
- `claimed_at`
- `completed_at`
- `claim_expires_at`
- `embedding_total`
- `embedding_success`
- `embedding_error`

### `rss_embedding_task_items`

Membres d'une task embedding, un item par source.

Colonnes clefs :

- `task_id`
- `source_id`
- `item_no`

Contraintes :

- PK `(task_id, source_id)`
- unicite `(task_id, item_no)`

### `rss_embedding_results`

Resultats bruts produits par le worker d'embeddings.

Colonnes clefs :

- `id`
- `job_id`
- `task_id`
- `source_id`
- `embedding_model_id`
- `worker_id`
- `embedding`
- `created_at`

La revision `v1_5_embed_finalization_idx` ajoute un index sur
`(job_id, source_id, embedding_model_id, created_at DESC, id DESC)` pour accelerer
la selection du dernier embedding par source lors de la finalisation.

## 6. Workers

### `worker_registry`

Identite persistante des workers.

Colonnes clefs :

- `id`
- `worker_kind`
- `device_id`
- `public_key`
- `fingerprint`
- `display_name`
- `hostname`
- `platform`
- `arch`
- `worker_version`
- `enabled`
- `enrolled_at`
- `last_auth_at`

Contraintes :

- unicite `(worker_kind, device_id)`
- unicite `fingerprint`

### `worker_auth_challenges`

Challenges courts pour `enroll` et `auth`.

Colonnes clefs :

- `id`
- `worker_id`
- `purpose`
- `challenge`
- `expires_at`
- `used_at`
- `created_at`

### `worker_runtime`

Etat runtime courant du worker vu par le backend.

Colonnes clefs :

- `worker_id`
- `worker_name`
- `runtime_kind`
- `connection_state`
- `desired_state`
- `active`
- `last_seen_at`
- `current_job_kind`
- `current_task_id`
- `current_execution_id`
- `active_claim_count`
- `last_error`

### `worker_capabilities`

Capacites declarees par worker.

Colonnes clefs :

- `worker_id`
- `job_kind`
- `embedding_model_id`
- `max_batch_size`
- `enabled`

## 7. Projection d'embeddings

### `embedding_projections`

Etat versionne d'une projection 2D pour un modele donne.

Colonnes clefs :

- `id`
- `embedding_model_id`
- `projector_kind`
- `projection_version`
- `projector_state`
- `fitted_source_total`
- `last_embedding_updated_at`
- `active`
- `created_at`
- `updated_at`

Contrainte :

- unicite `(embedding_model_id, projection_version)`

### `embedding_projection_points`

Coordonnees 2D par source pour une projection donnee.

Colonnes clefs :

- `projection_id`
- `source_id`
- `x`
- `y`
- `embedding_updated_at`
- `projected_at`

PK composite :

- `(projection_id, source_id)`

## 8. Etat de sync RSS

### `rss_catalog_sync_state`

Table ajoutee par `v1_1_add_rss_catalog_sync_state`.

Colonnes clefs :

- `id`
- `last_applied_revision`
- `last_seen_revision`
- `last_sync_status`
- `last_sync_error`
- `updated_at`

Elle permet au backend de savoir si un `rss/sync` doit rejouer un reconcile complet ou non.

## 9. Fonctions de retention

- `cleanup_expired_job_data(interval)` supprime les jobs RSS et embeddings anciens ;
- `cleanup_expired_worker_auth_challenges(interval)` supprime les challenges expires ou consommes.

Ces fonctions existent, mais leur ordonnancement est externe au code applicatif.
