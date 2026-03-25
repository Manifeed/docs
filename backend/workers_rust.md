# Workers Rust

## Role

Les binaires Rust consomment exclusivement le gateway `/workers/*` du backend.
Ils n'accedent pas directement aux tables PostgreSQL.

## Worker RSS

Le worker RSS :

- ouvre une session worker
- claim des taches `rss_scrape`
- fetch les feeds RSS
- parse les items
- applique la dedup locale utile au payload
- complete ou fail la task signee
- envoie des heartbeats reguliers

Le backend reste responsable de la promotion finale vers `articles`, `article_feed_links` et `article_versions`.

## Worker embedding

Le worker embedding :

- ouvre une session worker
- claim des taches `source_embedding`
- calcule les vecteurs pour une `worker_version`
- retourne les embeddings au backend
- envoie des heartbeats reguliers

Le backend reste responsable de :

- la validation des vecteurs
- la trace PostgreSQL
- l'indexation Qdrant
- l'upsert de `embedding_manifest`

## Contrat commun

Chaque worker doit gerer :

- l'authentification Bearer via `user_api_keys`
- la signature des completes et fails
- le respect du `lease_id` et du `trace_id`
- les retries uniquement au niveau du protocole de claim

## Releases

Le backend peut servir un manifest de release via `/workers/releases/manifest`.
Cette route permet de distribuer des versions de binaire sans multiplier les contrats HTTP.
