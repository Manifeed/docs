# Routes Backend Par Domaine

Reference des routes publiques exposees par le backend.

## Fichiers

| Fichier | Domaine | Surface principale |
| --- | --- | --- |
| [auth.md](./auth.md) | auth web | `/api/auth/*` |
| [account.md](./account.md) | compte utilisateur | `/api/account/*` |
| [admin.md](./admin.md) | administration users | `/api/admin/users*` |
| [health.md](./health.md) | supervision admin | `/api/admin/health/` |
| [jobs.md](./jobs.md) | jobs admin | `/api/admin/jobs*` |
| [rss.md](./rss.md) | catalogue RSS | `/api/admin/rss*`, `/api/rss/img/*` |
| [sources.md](./sources.md) | lecture sources | `/api/admin/sources*` |
| [workers.md](./workers.md) | gateway worker et releases | `/workers/api/*` |

## Points d'attention

- Les routes web canoniques sont directement exposees par FastAPI; `nginx` ne les recrit plus.
- Les zones browser masquees renvoient `404` cote backend pour un utilisateur authentifie non autorise.
- Les workers restent sur `/workers/api/*`.
