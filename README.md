# Manifeed Docs

Depot de documentation du split multi-repo Manifeed.

## Carte des repos

- `../backend` : API FastAPI et tests Python
- `../frontend` : console d'administration Next.js
- `../workers` : workspace Rust et installeurs Linux
- `../infra` : orchestration locale Docker, migrations et commandes transverses
- `../api` : contrat OpenAPI publie

## Contenu

- `backend/` : backend, workers et notes d'architecture
  - `backend/backend.md` : vue d'ensemble backend et diagrammes des flux critiques
  - `backend/routes/` : reference detaillee route par route, splittee par domaine
  - `backend/routes_review.md` : index de navigation vers la reference detaillee
  - `backend/routes_to_consumers.md` : matrice route -> consommateurs versionnes
- `frontend_admin/` : documentation de la console d'administration
- `postgres/` : schema et exploitation PostgreSQL
- `migrations/` : historiques de migration d'architecture

## Ajouts notables

- `migrations/hyper_scale_v2/` : plan cible hyper-scale, schema DB cible et migration par phases

Le dossier `frontend_admin/` conserve son nom historique, mais il documente bien le repo `frontend`.
