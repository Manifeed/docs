# PostgreSQL

## Role

PostgreSQL porte le stockage relationnel de Manifeed : catalogue RSS, auth, control plane worker, queue de jobs, stockage canonique des articles, buffers de pipeline et audit.

Les vecteurs finaux sont indexes dans Qdrant, mais leur etat de reference reste suivi dans PostgreSQL via `embedding_manifest`.

## Migrations

Les migrations vivent dans `infra/postgres_migration/alembic/`.
La reference courante est une baseline unique : `1_0_baseline`.

Flux normal en developpement :

1. reset de la base ou du schema
2. `alembic upgrade head`
3. lancement du backend

Le projet ne maintient plus de migration progressive vers une architecture `v2`.

## Exploitation locale

Depuis `infra/` :

```bash
make db-migrate
make db-reset
make logs SERVICE=postgres
```

`make db-reset` est destructif : il recree un schema vierge puis reapplique la baseline.

## Configuration standard

- image Docker : `postgres:15`
- base par defaut : `manifeed`
- utilisateur par defaut : `manifeed`
- mot de passe par defaut : `manifeed`
- port local : `5432`

## Vigilances

- ne pas exposer PostgreSQL publiquement
- changer les credentials hors developpement
- surveiller la croissance des tables `worker_*`, `staging_*`, `ingest_events` et `dedup_decisions`
- considerer les buffers `staging_*` comme des traces de pipeline, pas comme des tables de lecture produit
