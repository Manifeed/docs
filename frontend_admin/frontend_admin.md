# Frontend Admin

## Role

Le repo `frontend/` est la console d'administration de Manifeed.
Elle s'appuie directement sur l'API HTTP du backend et couvre six zones fonctionnelles :

- `Dashboard` : sante du service ;
- `RSS` : sync du catalogue, toggles company/feed, lancement d'ingest ;
- `Sources` : navigation dans les sources consolidees et lancement des embeddings ;
- `Visualizer` : carte 2D des embeddings et recherche de voisins ;
- `Jobs` : suivi detaille des jobs et suppression technique ;
- `Workers` : presence, activite et etat courant des workers.

## Stack technique

- Next.js `14.1.3`
- React `18.2.0`
- TypeScript `5.9.x`

Le `Dockerfile` du frontend installe les dependances puis lance `yarn dev`.
La stack fournie est donc orientee developpement local.

## Configuration

Variable indispensable :

- `NEXT_PUBLIC_API_URL`

Dans `infra/docker-compose.yml`, elle est alimentee par :

- `NEXT_PUBLIC_API_URL_ADMIN`, defaut `http://localhost:8000`

Le navigateur appelle cette URL directement. Elle doit donc etre resolvable depuis le poste
de l'utilisateur, pas seulement depuis le reseau Docker.

## Pages et usages

### `/`

- appelle `GET /health/`
- affiche l'etat global du backend et de la base

### `/rss`

- appelle `GET /rss/`
- declenche `POST /rss/sync`
- declenche `POST /rss/sync?force=true`
- declenche `POST /rss/ingest`
- applique `PATCH /rss/feeds/{feed_id}/enabled`
- applique `PATCH /rss/companies/{company_id}/enabled`
- construit les URLs d'icones via `GET /rss/img/{icon_url:path}`

### `/sources`

- appelle `GET /sources/`
- appelle `GET /sources/feeds/{feed_id}`
- appelle `GET /sources/companies/{company_id}`
- appelle `GET /sources/{source_id}`
- declenche `POST /rss/ingest`
- declenche `POST /sources/embeddings/enqueue`

### `/visualizer`

- appelle `GET /sources/visualizer`
- appelle `GET /sources/visualizer/{source_id}/neighbors`
- appelle `GET /sources/{source_id}` pour la modal detail

### `/jobs`

- appelle `GET /jobs`
- appelle `GET /jobs/{job_id}`
- appelle `GET /jobs/{job_id}/tasks`
- appelle `GET /jobs/{job_id}/feeds`
- appelle `GET /jobs/{job_id}/sources`
- appelle `GET /jobs/{job_id}/embeddings`
- appelle `DELETE /jobs/{job_id}`

### `/workers`

- appelle `GET /internal/workers/overview`

## Organisation du code

- `src/app/` : pages Next.js par route
- `src/components/` : primitives et composants de layout
- `src/features/` : logique UI par domaine (`rss`, `health`, `navigation`)
- `src/services/api/` : client HTTP et appels backend
- `src/types/` : contrats TypeScript alignes sur l'API

## Commandes utiles

Depuis le repo `infra/` :

```bash
make up SERVICE=frontend_admin
make logs SERVICE=frontend_admin
```

Depuis le repo `frontend/` :

```bash
yarn dev
yarn build
yarn start
```

## Notes de developpement

- Le frontend utilise `fetch` cote navigateur, sans proxy applicatif.
- Le client HTTP leve `ApiRequestError` et reprend en priorite `payload.message` ou `payload.detail`.
- Les pages `Jobs` et `Workers` se rafraichissent periodiquement pour offrir un suivi quasi temps reel.
- Le depot ne contient pas aujourd'hui de suite de tests frontend versionnee.

## Notes de production

- Le `Dockerfile` courant lance `yarn dev` ; pour la production, privilegiez une image `yarn build && yarn start`.
- L'UI n'embarque pas d'authentification ; la securisation doit etre geree en amont.
- `NEXT_PUBLIC_API_URL` doit pointer vers une URL publique stable et securisee.
