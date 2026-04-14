# Frontend

## Role

Le frontend Next.js couvre la landing publique, les parcours user et l'espace admin.

## Routes API consommees

### User

- `/api/auth/*`
- `/api/account/*`
- `/workers/api/releases/desktop`

### Admin

- `/api/admin/health/`
- `/api/admin/users*`
- `/api/admin/jobs*`
- `/api/admin/rss*`
- `/api/admin/sources*`

### Assets publics

- `/api/rss/img/{icon_url:path}`

## Notes

- Le navigateur passe par l'edge `nginx`, sans proxy applicatif Next.
- Le SSR Next appelle les memes chemins publics canoniques via `BACKEND_INTERNAL_URL`.
- Les pages admin non autorisees rendent une `404` Next.
