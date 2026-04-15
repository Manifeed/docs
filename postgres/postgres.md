# PostgreSQL

## Role

PostgreSQL porte le stockage relationnel de Manifeed : catalogue RSS, auth, control plane worker, queue de jobs, stockage canonique des articles et etat de reference des embeddings.

Les vecteurs finaux sont indexes dans Qdrant, mais leur etat de reference reste suivi dans PostgreSQL via `embedding_manifest`.

## Migrations

Les migrations vivent dans `infra/postgres_migration/alembic/`.
La chaine courante est :

- `1_0_baseline`
- `1_1_article_content_key`
- `1_2_unique_article_content_key`
- `1_3_worker_execution_guards`

Flux normal en developpement :

1. reset de la base ou du schema
2. `alembic upgrade head`
3. lancement du backend

Points cle de la chaine actuelle :

- `1_1` ajoute `articles.content_key`
- `1_2` impose l'unicite partielle sur `articles.content_key`
- `1_3` ajoute les garde-fous worker :
  - `worker_tasks.execution_id`
  - sequence `worker_task_execution_id_seq`
  - `worker_leases.result_status`
  - `worker_leases.result_signature_hash`

## Exploitation locale

Depuis `infra/` :

```bash
make db-migrate
make db-reset
make logs SERVICE=postgres
```

`make db-reset` est destructif : il recree un schema vierge puis reapplique toute la chaine Alembic jusqu'a `head`.

## Configuration standard

- image Docker : `postgres:15`
- base par defaut : `manifeed`
- utilisateur par defaut : `manifeed`
- mot de passe par defaut : `manifeed`
- port local : `5432`

## Vigilances

- ne pas exposer PostgreSQL publiquement
- changer les credentials hors developpement
- surveiller la croissance des tables `worker_*`, `articles`, `article_feed_links` et `embedding_manifest`
- appliquer `alembic upgrade head` avant de lancer des workers, car le protocole courant depend de `worker_tasks.execution_id` et de la finalisation idempotente des `worker_leases`
