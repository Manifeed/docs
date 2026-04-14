# Backend Manifeed

## Role

Le backend FastAPI est la source de verite de toutes les routes publiques web et worker de Manifeed.
`nginx` reste limite au proxy edge, au rate limiting, au blocage bot et aux pages HTML custom pour ses propres erreurs.

## Surface HTTP publiee

### Web public et utilisateur

- `/api/auth/*`
- `/api/account/*`

### Web admin

- `/api/admin/users*`
- `/api/admin/health/`
- `/api/admin/jobs*`
- `/api/admin/rss*`
- `/api/admin/sources*`

### Assets publics

- `/api/rss/img/{icon_url}`

### Workers

- `/workers/api/*`

## Modele d'acces

- Les parcours web utilisent une session via `x-manifeed-session` ou `manifeed_session`.
- Les zones browser masquees renvoient `404` pour un utilisateur authentifie non autorise.
- Les workers utilisent une API key Bearer, avec HMAC pour `complete` et `fail`.

## Notes importantes

- Le frontend navigateur appelle les routes publiques canoniques via l'edge.
- Le SSR Next appelle ces memes routes canoniques via `BACKEND_INTERNAL_URL`.
- Les anciennes surfaces brutes `/auth/*`, `/account/*`, `/admin/*`, `/health/*`, `/jobs/*`, `/rss/*`, `/sources/*` ne font plus partie du contrat public.
