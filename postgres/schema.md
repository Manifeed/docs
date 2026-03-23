# Schema PostgreSQL

Le schema de reference est defini par les migrations Alembic de
`infra/postgres_migration/alembic/versions/`.
La baseline courante est `v1_8_embedding_workers_pseudos`.

## Conventions

- les tables metier et techniques vivent dans le schema `public`
- les dates sont stockees en `TIMESTAMPTZ`
- les jobs RSS et embedding sont separes
- les workers n'ecrivent jamais directement dans les tables metier finales
- le pipeline embedding est versionne par `worker_version`, pas par catalogue de modeles

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
- la migration de canonisation recentre l'unicite sur `identity_key`.

### `rss_source_feeds`

Table de liaison many-to-many entre sources consolidees et feeds.

Colonnes clefs :

- `feed_id`
- `source_id`
- `ingested_at`

PK composite :

- `(feed_id, source_id)`

### `rss_source_embeddings`

Stockage final des embeddings actifs par source et version de worker.

Colonnes clefs :

- `source_id`
- `worker_version`
- `embedding` (`float[]`)
- `updated_at`

PK composite :

- `(source_id, worker_version)`

Indexes notables :

- `idx_rss_source_embeddings_worker_version_source` sur `(worker_version, source_id)`

Important :

- le backend ne stocke plus de table `embedding_models` ;
- le modele applicatif est fixe a `Xenova/multilingual-e5-large` ;
- seule la version logique du pipeline (`worker_version`) est persistee.

## 3. Authentification applicative et identite workers

### `users`

Utilisateurs applicatifs.

Colonnes clefs :

- `id`
- `email` unique
- `pseudo` unique
- `password_hash`
- `role`
- `is_active`
- `api_access_enabled`
- `created_at`
- `updated_at`

`pseudo` est normalise cote backend et sert a calculer automatiquement les noms de workers.

### `user_sessions`

Sessions web du frontend/admin.

Colonnes clefs :

- `id`
- `user_id`
- `token_hash`
- `expires_at`
- `revoked_at`
- `created_at`

### `user_api_keys`

Cles Bearer destinees aux workers.

Colonnes clefs :

- `id`
- `user_id`
- `label`
- `worker_type`
- `worker_number`
- `key_prefix`
- `key_hash`
- `last_used_at`
- `revoked_at`
- `created_at`

Contraintes notables :

- unicite `(user_id, worker_type, worker_number)`

Le backend derive le nom expose du worker a partir de :

- `slug(users.pseudo)`
- un type court (`rss` ou `embedding`)
- `worker_number`

Exemple : `alice-rss-1`, `alice-embedding-2`.

### `worker_runtime`

Etat runtime courant vu par le backend pour une cle API worker.

Colonnes clefs :

- `api_key_id`
- `worker_name`
- `worker_version` nullable
- `connection_state`
- `desired_state`
- `active`
- `last_seen_at`
- `current_job_kind`
- `current_task_id`
- `current_execution_id`
- `current_task_label`
- `current_feed_id`
- `current_feed_url`
- `active_claim_count`
- `last_error`

Notes :

- `worker_name` reste stocke comme information technique de dernier heartbeat ;
- les vues backend recalculent le nom canonique a partir de `users.pseudo` et `user_api_keys.worker_number`.

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

- `feed_id` est unique a l'echelle de la queue ;
- cela evite qu'un meme feed soit simultanement present dans plusieurs tasks RSS.

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

## 5. Pipeline embedding versionne

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
- `worker_version`
- `finalized_at`

Indexes notables :

- `idx_rss_embedding_jobs_worker_version_status`

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
- `worker_version`
- `worker_id`
- `embedding`
- `created_at`

Index notable :

- `idx_rss_embedding_results_job_latest` sur
  `(job_id, source_id, worker_version, created_at, id)` ;
- la finalisation relit ensuite le dernier resultat par `(source_id, worker_version)`.

## 6. Etat de sync RSS

### `rss_catalog_sync_state`

Table ajoutee pour memoriser l'etat de synchronisation du depot RSS.

Colonnes clefs :

- `id`
- `last_applied_revision`
- `last_seen_revision`
- `last_sync_status`
- `last_sync_error`
- `updated_at`

Elle permet au backend de savoir si un `rss/sync` doit rejouer un reconcile complet ou non.

## 7. Fonctions de retention

- `normalize_rss_source_url(text)` normalise les URLs sources ;
- `cleanup_expired_job_data(interval)` supprime les jobs RSS et embedding anciens.

Ces fonctions existent, mais leur ordonnancement reste externe au code applicatif.
