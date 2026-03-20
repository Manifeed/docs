# PostgreSQL

## Role

PostgreSQL est le stockage unique de Manifeed.
Il contient a la fois :

- le catalogue RSS ;
- les sources consolidees ;
- les embeddings ;
- les jobs et tasks techniques ;
- les identites et etats runtime des workers.

## Configuration actuelle

Dans `infra/docker-compose.yml` :

- image : `postgres:15`
- conteneur : `manifeed_postgres`
- volume : `pgdata:/var/lib/postgresql/data`
- port expose : `5432:5432`
- reseau principal : `manifeed_internal`

Variables standard :

- `POSTGRES_DB` (defaut `manifeed`)
- `POSTGRES_USER` (defaut `manifeed`)
- `POSTGRES_PASSWORD` (defaut `manifeed`)

## Migrations

Les migrations sont gerees par Alembic dans `infra/postgres_migration/alembic/`.
Leur orchestration appartient a `infra/`, pas au cycle de vie du conteneur backend.

Sequence normale :

1. `infra` demarre PostgreSQL ;
2. le service one-shot `db_migrations` execute `alembic upgrade head` ;
3. les revisions Alembic sont appliquees ;
4. le backend peut ensuite devenir disponible.

Commandes utiles :

Depuis le repo `infra/` :

```bash
make db-migrate
make db-reset
```

`make db-reset` :

- drop le schema `public`
- le recree
- relance les migrations via le service `db_migrations`

Cette commande est destructive.

## Extensions et fonctions SQL

La revision `v1_3_enable_pgcrypto` active `pgcrypto`.
Le schema courant declare aussi plusieurs fonctions utilitaires :

- `normalize_rss_source_url(text)` : normalisation d'URL pour l'identite des sources ;
- `cleanup_expired_job_data(interval)` : purge des jobs anciens ;
- `cleanup_expired_worker_auth_challenges(interval)` : purge des challenges workers anciens.

Important :

- ces fonctions de retention existent en base ;
- leur execution periodique n'est pas planifiee par l'application ;
- en production, il faut les scheduler explicitement via l'infra (`pg_cron`, cron externe, job Ops).

## Exploitation

### Commandes utiles

```bash
cd ../infra
make logs SERVICE=postgres
docker compose -f docker-compose.yml exec postgres psql -U manifeed -d manifeed
docker compose -f docker-compose.yml exec postgres pg_isready -U manifeed -d manifeed
```

### Sauvegarde

Le volume `pgdata` doit etre sauvegarde comme donnee persistante critique.
La base contient les donnees metier finales, pas seulement des files d'attente techniques.

### Securite

- n'exposez pas PostgreSQL publiquement ;
- isolez-le derriere le reseau interne du stack ;
- changez les credentials par defaut hors developpement ;
- surveillez la taille des tables de jobs et resultats si aucun nettoyage periodique n'est en place.

## Documents associes

- `postgres/schema.md`
- `postgres/relations.md`
