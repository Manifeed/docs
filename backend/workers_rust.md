# Workers Rust

## Role

Les binaires Rust n'accedent ni directement a PostgreSQL, ni a Qdrant.
Ils parlent exclusivement au backend via le gateway HTTP `/workers/*`.

## Endpoints consommes actuellement

### Worker RSS

- `GET /workers/ping`
- `POST /workers/sessions/open`
- `POST /workers/tasks/claim`
- `POST /workers/tasks/complete`
- `POST /workers/tasks/fail`
- `GET /workers/releases/manifest`

### Worker embedding

- `GET /workers/ping`
- `POST /workers/sessions/open`
- `POST /workers/tasks/claim`
- `POST /workers/tasks/complete`
- `POST /workers/tasks/fail`
- `GET /workers/releases/manifest`

## Protocole commun

```mermaid
sequenceDiagram
    participant Worker
    participant API as Backend
    participant DB as PostgreSQL

    Worker->>API: Bearer API key
    Worker->>API: POST /workers/sessions/open
    API->>DB: worker_sessions
    loop tant que du travail existe
        Worker->>API: POST /workers/tasks/claim
        API->>DB: worker_tasks + worker_leases
        alt succes
            Worker->>API: POST /workers/tasks/complete
            API->>DB: finalize task/job/lease
        else erreur
            Worker->>API: POST /workers/tasks/fail
            API->>DB: finalize task/job/lease en echec
        end
    end
```

## Authentification et integrite

- l'API key worker provient de `POST /account/api-keys` ;
- la cle brute n'est renvoyee qu'une seule fois a la creation ;
- le backend stocke uniquement `user_api_keys.key_hash` ;
- `complete` et `fail` exigent une signature HMAC du body ;
- le backend verifie aussi la coherence `task_type`, `worker_version`, `lease_id`, `trace_id`.

## Ce que fait le backend apres reception

### Quand le worker RSS complete

Le backend :

- valide le contrat `WorkerRssTaskResultPayloadSchema` ;
- persiste les traces de fetch dans `staging_feed_fetch_results` ;
- persiste les candidats dans `staging_article_candidates` ;
- promeut les articles dans `articles`, `article_feed_links`, `article_versions` ;
- met a jour `rss_feed_runtime` ;
- journalise `ingest_events` et `dedup_decisions` ;
- met a jour `worker_tasks`, `worker_jobs`, `worker_leases`, `worker_sessions`.

### Quand le worker embedding complete

Le backend :

- valide les dimensions et la norme du vecteur ;
- pousse le vecteur dans Qdrant ;
- met a jour `embedding_manifest` ;
- met a jour `worker_tasks`, `worker_jobs`, `worker_leases`, `worker_sessions`.

## Releases

Le backend expose maintenant un catalogue de releases desktop / RSS / embedding.
Le crate partage `manifeed-worker-common` utilise `GET /workers/releases/manifest`
pour resoudre l'artefact compatible avec le host courant et verifier la compatibilite
de version.

Points cle :

- le desktop se met a jour contre la famille `desktop` uniquement ;
- RSS et Embedding se mettent a jour independamment avec leurs propres versions de bundle ;
- `embedding` transporte aussi un `worker_version` metier distinct du `package.version` ;
- le telechargement desktop est public ;
- les bundles RSS / Embedding sont telecharges depuis `GET /workers/releases/download/{artifact_name}`
  avec Bearer worker ;
- le backend derive la version worker active du catalogue au lieu d'une variable frontend ou d'un
  hardcode partage.
