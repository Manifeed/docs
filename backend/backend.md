# Backend Manifeed

## Role

Le repo `backend/` est le centre de controle du systeme.

Il assure :

- l'API publique pour l'interface d'administration ;
- l'API d'enrolement, d'authentification et de gateway pour les workers ;
- la synchronisation du catalogue RSS depuis un depot Git ;
- la creation des jobs RSS et embeddings ;
- la finalisation des jobs dans les tables metier ;
- les migrations Alembic au demarrage ;
- la maintenance de la projection d'embeddings pour le visualizer.

Il ne scrape pas les flux RSS lui-meme et ne calcule pas les embeddings.

## Demarrage et cycle de vie

Au demarrage normal de `main.py` :

1. les logs applicatifs sont configures ;
2. les CORS sont derives de `CORS_ORIGINS` ;
3. les routers `health`, `jobs`, `rss`, `sources` et `workers` sont montes ;
4. les migrations Alembic sont appliquees ;
5. une boucle asynchrone de synchronisation des projections est lancee.

La variable `MANIFEED_DISABLE_STARTUP_TASKS=1` desactive les migrations et la boucle de projection.
Elle est utile pour les tests ou certains scripts ponctuels.

## Flux metier

### Synchronisation du catalogue RSS

`POST /rss/sync` :

1. synchronise le depot Git `rss_feeds` ;
2. detecte le mode `noop` ou `full_reconcile` selon la revision appliquee ;
3. parse les fichiers `json/*.json` ;
4. upsert les companies, feeds et tags ;
5. retire les feeds absents du catalogue ;
6. met a jour `rss_catalog_sync_state`.

`GET /rss/img/{icon_url:path}` sert ensuite les icones SVG depuis ce meme depot.

### Job RSS

`POST /rss/ingest` :

1. verifie qu'aucun job RSS actif n'est deja en cours ;
2. charge les feeds actives (ou la selection `feed_ids`) ;
3. cree un job `rss-...` dans `rss_scrape_jobs` ;
4. cree des tasks batch dans `rss_scrape_tasks` ;
5. cree les items par feed dans `rss_scrape_task_items`.

Les workers RSS :

1. s'authentifient contre l'API workers ;
2. claim des tasks ;
3. postent `complete` ou `fail` ;
4. poussent leur etat courant via `POST /internal/workers/rss/state`.

Lorsque toutes les tasks sont traitees, le backend finalise le job :

- mise a jour de `rss_feed_runtime` ;
- fusion des sources dans `rss_sources` ;
- creation des liens `rss_source_feeds` ;
- marquage du job comme finalise.

### Job d'embedding

`POST /sources/embeddings/enqueue` :

1. resout le modele actif via `EMBEDDING_MODEL_NAME` ;
2. verifie qu'aucun job actif n'existe deja pour ce modele ;
3. cherche les sources sans embedding, ou sans embedding du modele courant ;
4. cree un job `embedding-...` dans `rss_embedding_jobs` ;
5. cree des tasks dans `rss_embedding_tasks` et `rss_embedding_task_items`.

Les workers d'embeddings :

1. s'authentifient ;
2. claim une task ;
3. calculent les embeddings ;
4. postent `complete` ou `fail`.

La finalisation :

- garde le dernier resultat par `(source_id, embedding_model_id)` ;
- upsert `rss_source_embeddings` ;
- met a jour les dimensions du modele dans `embedding_models` ;
- tente un rafraichissement incrementiel de la projection si l'etat courant est deja valide.

## Routes publiques

### Health

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/health/` | etat du backend et acces base | retourne `ok` ou `degraded` |

### RSS

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/rss/` | catalogue RSS consolide | inclut company et `fetchprotection` resolu |
| `PATCH` | `/rss/feeds/{feed_id}/enabled` | active ou desactive un feed | body `{ "enabled": bool }` |
| `PATCH` | `/rss/companies/{company_id}/enabled` | active ou desactive une company | body `{ "enabled": bool }` |
| `POST` | `/rss/sync` | synchro du depot `rss_feeds` | query `force=true` pour relecture complete |
| `GET` | `/rss/img/{icon_url:path}` | sert une icone SVG | refuse les chemins absolus, `..` et les non-SVG |
| `POST` | `/rss/ingest` | cree un job RSS | query optionnelle `feed_ids` multiple |

Les endpoints RSS mutateurs sont proteges par un verrou SQL (`job_lock`) pour eviter
les executions concurrentes de meme nature.

### Sources

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/sources/` | liste paginee des sources | `limit` max `100`, `offset` |
| `GET` | `/sources/feeds/{feed_id}` | sources pour un feed | pagination identique |
| `GET` | `/sources/companies/{company_id}` | sources pour une company | pagination identique |
| `GET` | `/sources/{source_id}` | detail d'une source | renvoie company names et sections |
| `POST` | `/sources/embeddings/enqueue` | cree un job d'embedding | query `reembed_model_mismatches=true` |
| `GET` | `/sources/visualizer` | projection 2D | filtres `date_from`, `date_to` |
| `GET` | `/sources/visualizer/{source_id}/neighbors` | voisins semantiques | `neighbor_limit` entre `1` et `24` |

### Jobs

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/jobs` | liste des jobs RSS et embeddings | `limit` max `500` |
| `GET` | `/jobs/{job_id}` | detail d'un job | prefixes actuels `rss-` et `embedding-` |
| `GET` | `/jobs/{job_id}/tasks` | tasks du job | pagination `limit`/`offset` |
| `GET` | `/jobs/{job_id}/feeds` | resultats feed d'un job RSS | 404 si job non RSS |
| `GET` | `/jobs/{job_id}/sources` | resultats source d'un job RSS | 404 si job non RSS |
| `GET` | `/jobs/{job_id}/embeddings` | resultats d'un job embedding | 404 si job non embedding |
| `DELETE` | `/jobs/{job_id}` | supprime un job technique | ne retire pas les donnees metier deja fusionnees |

## Routes workers

Le backend monte deux prefixes :

- `/internal/workers/*` : prefixe officiel expose dans la documentation OpenAPI ;
- `/workers/*` : alias legacy, masque du schema, conserve pour les binaires Rust actuels.

### Authentification et monitoring

| Methode | Route officielle | Description |
| --- | --- | --- |
| `POST` | `/internal/workers/enroll` | enrolement initial par cle publique Ed25519 |
| `POST` | `/internal/workers/auth/challenge` | emission d'un challenge court |
| `POST` | `/internal/workers/auth/verify` | verification de signature et emission d'un JWT worker |
| `GET` | `/internal/workers/me` | profil du worker authentifie |
| `GET` | `/internal/workers/overview` | etat des workers par type |
| `GET` | `/internal/workers/queues/overview` | etat des queues techniques |
| `POST` | `/internal/workers/queues/{queue_name}/purge` | purge brute d'une queue |

### Pipeline RSS

| Methode | Route officielle | Description |
| --- | --- | --- |
| `POST` | `/internal/workers/rss/claim` | claim de tasks RSS |
| `POST` | `/internal/workers/rss/complete` | completion avec `result_events` |
| `POST` | `/internal/workers/rss/fail` | echec technique de task |
| `POST` | `/internal/workers/rss/state` | etat runtime remonte par le worker RSS |

### Pipeline embeddings

| Methode | Route officielle | Description |
| --- | --- | --- |
| `POST` | `/internal/workers/embedding/claim` | claim de task embedding |
| `POST` | `/internal/workers/embedding/complete` | completion avec vecteurs |
| `POST` | `/internal/workers/embedding/fail` | echec technique de task |

## Variables d'environnement

### Base et runtime

| Variable | Defaut | Usage |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql://manifeed:manifeed@localhost:5432/manifeed` | connexion SQLAlchemy |
| `DB_POOL_SIZE` | `20` | taille du pool SQLAlchemy |
| `DB_MAX_OVERFLOW` | `40` | overflow du pool |
| `DB_POOL_TIMEOUT_SECONDS` | `30` | timeout du pool |
| `DB_POOL_RECYCLE_SECONDS` | `1800` | recycle des connexions |
| `CORS_ORIGINS` | `*` | liste CSV ou `*` |
| `MANIFEED_DISABLE_STARTUP_TASKS` | vide | desactive migrations et projection loop |

### Catalogue RSS

| Variable | Defaut | Usage |
| --- | --- | --- |
| `RSS_FEEDS_REPOSITORY_URL` | `https://github.com/Dorn-15/rss_feeds` | depot source du catalogue |
| `RSS_FEEDS_REPOSITORY_BRANCH` | `main` | branche synchronisee |
| `RSS_FEEDS_REPOSITORY_PATH` | `/tmp/rss_feeds` ou override compose | chemin local du clone |

### Embeddings et projections

| Variable | Defaut | Usage |
| --- | --- | --- |
| `EMBEDDING_MODEL_NAME` | `Xenova/multilingual-e5-large` | modele logique courant |
| `BACKEND_PROJECTION_POLL_SECONDS` | `30` | frequence de la boucle de projection |
| `SOURCE_EMBEDDING_TASK_BATCH_SIZE` | `128` | taille des tasks embedding |

### Jobs RSS

| Variable | Defaut | Usage |
| --- | --- | --- |
| `RSS_SCRAPE_TASK_BATCH_SIZE` | `20` | taille des tasks RSS, plafonnee a `20` cote code |

### Auth workers

| Variable | Defaut | Usage |
| --- | --- | --- |
| `WORKER_ENROLLMENT_TOKENS` | vide | mapping CSV `worker_type:token` |
| `WORKER_ENROLLMENT_TOKEN` | vide | token global fallback |
| `WORKER_ENROLLMENT_TOKEN_RSS_SCRAPPER` | vide | token specifique RSS |
| `WORKER_ENROLLMENT_TOKEN_SOURCE_EMBEDDING` | vide | token specifique embedding |
| `WORKER_TOKEN_SECRET` | requis | secret JWT workers |
| `WORKER_TOKEN_TTL_SECONDS` | `3600` | duree des sessions worker |
| `WORKER_CHALLENGE_TTL_SECONDS` | `300` | duree des challenges |

### Monitoring workers

| Variable | Defaut | Usage |
| --- | --- | --- |
| `WORKER_CONNECTED_IDLE_THRESHOLD_MS` | `300000` | seuil "connecte" pour l'overview |
| `WORKER_ACTIVE_IDLE_THRESHOLD_MS` | `30000` | seuil "actif" pour l'overview |

## Commandes utiles

Depuis le repo `infra/` :

```bash
make up SERVICE=backend
make logs SERVICE=backend
make db-migrate
make test-backend
```

Pour lancer le backend seul hors Compose :

```bash
cd ../backend
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Notes d'exploitation

- Le backend est aujourd'hui la seule porte d'entree des workers ; ils n'accedent jamais directement a PostgreSQL.
- Les finalizers transforment les resultats techniques en donnees metier. Supprimer un job ne retire donc pas
  automatiquement les sources ou embeddings deja fusionnes.
- La projection 2D est maintenue en fond et peut etre reconstruite integralement quand l'etat n'est plus courant.
- Les routes admin publiques n'embarquent pas d'authentification applicative a ce stade ; il faut proteger
  l'exposition reseau en production.
