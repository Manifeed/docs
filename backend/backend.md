# Backend Manifeed

## Role

Le repo `backend/` est le centre de controle du systeme.

Il assure :

- l'API publique pour le frontend Next.js ;
- l'auth web (`register`, `login`, `session`, `logout`) ;
- l'API workers consommee par les binaires Rust ;
- la creation et la finalisation des jobs RSS et embedding ;
- la synchronisation du catalogue RSS depuis le depot Git.

Il ne scrape pas les flux RSS lui-meme et ne calcule pas les embeddings.

## Demarrage et cycle de vie

Au demarrage normal de `main.py` :

1. les logs applicatifs sont configures ;
2. les CORS sont derives de `CORS_ORIGINS` ;
3. les routers `auth`, `account`, `admin`, `health`, `workers`, `jobs`, `rss` et `sources` sont montes ;
4. le schema OpenAPI est normalise avant export.

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
2. charge les feeds actives ou la selection `feed_ids` ;
3. cree un job `rss-...` dans `rss_scrape_jobs` ;
4. cree des tasks batch dans `rss_scrape_tasks` ;
5. cree les items par feed dans `rss_scrape_task_items`.

Les workers RSS :

1. s'authentifient avec une cle API worker (`Authorization: Bearer ...`) ;
2. claim des tasks ;
3. postent `complete` ou `fail` ;
4. poussent leur etat courant via `POST /workers/rss/state`.

Lorsque toutes les tasks sont traitees, le backend finalise le job :

- mise a jour de `rss_feed_runtime` ;
- fusion des sources dans `rss_sources` ;
- creation des liens `rss_source_feeds` ;
- marquage du job comme finalise.

### Job d'embedding versionne

`POST /sources/embeddings/enqueue` :

1. resout la version cible via `SOURCE_EMBEDDING_WORKER_VERSION` ;
2. verifie qu'aucun job actif n'existe deja pour cette version ;
3. cherche les sources sans embedding pour cette version ;
4. cree un job `embedding-...` dans `rss_embedding_jobs` ;
5. cree des tasks dans `rss_embedding_tasks` et `rss_embedding_task_items`.

Choix d'architecture :

- le modele applicatif est fixe a `Xenova/multilingual-e5-large` ;
- le backend ne stocke plus de `embedding_model_id` ;
- l'isolement de pipeline repose sur `worker_version` ;
- un worker embedding ne claim/complete/fail que pour sa propre version.

Les workers d'embeddings :

1. s'authentifient avec une cle API worker ;
2. envoient `worker_version` sur `claim`, `complete` et `fail` ;
3. calculent les embeddings avec le modele fixe ;
4. remontent les resultats au backend.

La finalisation :

- garde le dernier resultat par `(source_id, worker_version)` ;
- upsert `rss_source_embeddings` ;
- derive la dimension d'embedding en lecture via `array_length(...)`.

### Identite workers

Le worker ne choisit jamais son nom.
Le backend le derive automatiquement depuis :

- `users.pseudo`
- `user_api_keys.worker_type`
- `user_api_keys.worker_number`

Format :

- `pseudo-rss-1`
- `pseudo-embedding-1`

Le numero est monotone par `(user_id, worker_type)` et n'est pas recycle.

## Routes principales

### Auth

| Methode | Route | Description |
| --- | --- | --- |
| `POST` | `/auth/register` | creation d'un compte avec `email`, `pseudo`, `password` |
| `POST` | `/auth/login` | ouverture de session |
| `GET` | `/auth/session` | lecture de la session courante |
| `POST` | `/auth/logout` | fermeture de session |

### Account et admin

| Methode | Route | Description |
| --- | --- | --- |
| `GET` | `/account/me` | profil courant |
| `PATCH` | `/account/me` | mise a jour du `pseudo` |
| `PATCH` | `/account/password` | changement de mot de passe |
| `GET` | `/account/api-keys` | cles API worker de l'utilisateur |
| `POST` | `/account/api-keys` | creation d'une cle API worker |
| `DELETE` | `/account/api-keys/{api_key_id}` | revocation d'une cle API worker |
| `GET` | `/account/workers` | vue runtime des workers de l'utilisateur |
| `GET` | `/admin/users` | liste des utilisateurs |
| `PATCH` | `/admin/users/{user_id}` | edition admin (`pseudo`, role, activation, acces API) |

### RSS

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/rss/` | catalogue RSS consolide | inclut company et `fetchprotection` resolu |
| `PATCH` | `/rss/feeds/{feed_id}/enabled` | active ou desactive un feed | body `{ "enabled": bool }` |
| `PATCH` | `/rss/companies/{company_id}/enabled` | active ou desactive une company | body `{ "enabled": bool }` |
| `POST` | `/rss/sync` | synchro du depot `rss_feeds` | query `force=true` pour relecture complete |
| `GET` | `/rss/img/{icon_url:path}` | sert une icone SVG | refuse les chemins absolus, `..` et les non-SVG |
| `POST` | `/rss/ingest` | cree un job RSS | query optionnelle `feed_ids` multiple |

### Sources

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/sources/` | liste paginee des sources | `limit` max `100`, `offset` |
| `GET` | `/sources/feeds/{feed_id}` | sources pour un feed | pagination identique |
| `GET` | `/sources/companies/{company_id}` | sources pour une company | pagination identique |
| `GET` | `/sources/{source_id}` | detail d'une source | renvoie company names et sections |
| `POST` | `/sources/embeddings/enqueue` | cree un job d'embedding | expose `worker_version`; le flag `reembed_model_mismatches` reste un alias de compatibilite temporaire |

### Jobs

| Methode | Route | Description | Notes |
| --- | --- | --- | --- |
| `GET` | `/jobs` | liste des jobs RSS et embedding | `limit` max `500` |
| `GET` | `/jobs/{job_id}` | detail d'un job | prefixes actuels `rss-` et `embedding-` |
| `GET` | `/jobs/{job_id}/tasks` | tasks du job | pagination `limit`/`offset` |
| `GET` | `/jobs/{job_id}/feeds` | resultats feed d'un job RSS | 404 si job non RSS |
| `GET` | `/jobs/{job_id}/sources` | resultats source d'un job RSS | 404 si job non RSS |
| `GET` | `/jobs/{job_id}/embeddings` | resultats d'un job embedding | expose `worker_version` |
| `DELETE` | `/jobs/{job_id}` | supprime un job technique | ne retire pas les donnees metier deja fusionnees |

### Workers

Les routes workers sont sous le prefixe unique `/workers/*`.
Elles utilisent uniquement des cles API Bearer.

#### Pipeline RSS

| Methode | Route | Description |
| --- | --- | --- |
| `POST` | `/workers/rss/claim` | claim de tasks RSS |
| `POST` | `/workers/rss/complete` | completion avec `result_events` |
| `POST` | `/workers/rss/fail` | echec technique de task |
| `POST` | `/workers/rss/state` | etat runtime remonte par le worker RSS |

#### Pipeline embedding

| Methode | Route | Description |
| --- | --- | --- |
| `POST` | `/workers/embedding/claim` | claim de task embedding avec `worker_version` |
| `POST` | `/workers/embedding/complete` | completion avec vecteurs et `worker_version` |
| `POST` | `/workers/embedding/fail` | echec technique de task avec `worker_version` |

#### Monitoring

| Methode | Route | Description |
| --- | --- | --- |
| `GET` | `/workers/ping` | verification rapide d'une cle API worker |
| `GET` | `/workers/releases/manifest` | manifest de version/publication pour bundles et CLI |
| `GET` | `/workers/overview` | etat des workers par type |
| `GET` | `/workers/queues/overview` | etat des queues techniques |
| `POST` | `/workers/queues/{queue_name}/purge` | purge brute d'une queue |

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

### Catalogue RSS

| Variable | Defaut | Usage |
| --- | --- | --- |
| `RSS_FEEDS_REPOSITORY_URL` | `https://github.com/Dorn-15/rss_feeds` | depot source du catalogue |
| `RSS_FEEDS_REPOSITORY_BRANCH` | `main` | branche synchronisee |
| `RSS_FEEDS_REPOSITORY_PATH` | `/tmp/rss_feeds` ou override compose | chemin local du clone |

### Embeddings

| Variable | Defaut | Usage |
| --- | --- | --- |
| `SOURCE_EMBEDDING_WORKER_VERSION` | `e5-large-v1` | version cible du pipeline embedding |
| `SOURCE_EMBEDDING_TASK_BATCH_SIZE` | `128` | taille des tasks embedding |

### Jobs RSS

| Variable | Defaut | Usage |
| --- | --- | --- |
| `RSS_SCRAPE_TASK_BATCH_SIZE` | `20` | taille des tasks RSS, plafonnee a `20` cote code |

### Monitoring workers

| Variable | Defaut | Usage |
| --- | --- | --- |
| `WORKER_CONNECTED_IDLE_THRESHOLD_MS` | `300000` | seuil "connecte" pour l'overview |
| `WORKER_ACTIVE_IDLE_THRESHOLD_MS` | `30000` | seuil "actif" pour l'overview |
| `WORKER_RELEASE_MANIFEST_PATH` | vide | chemin JSON des manifests de release workers |
| `WORKER_RELEASE_MANIFEST_JSON` | vide | payload JSON inline des manifests de release workers |

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
cd ../infra
make db-migrate
cd ../backend
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Notes d'exploitation

- les bundles desktop/CLI des workers verifient au lancement leur version via `/workers/releases/manifest` ;
- `latest_version` declenche un warning, `minimum_supported_version` bloque le demarrage ;
- l'API `GET /workers/ping` sert aux commandes `doctor` et a l'UI desktop pour tester la cle API stockee localement.

- le backend est la seule porte d'entree des workers ; ils n'accedent jamais directement a PostgreSQL ;
- les routes admin et account reposent sur la session web courante ;
- les finalizers transforment les resultats techniques en donnees metier ;
- supprimer un job ne retire donc pas automatiquement les sources ou embeddings deja fusionnes.
