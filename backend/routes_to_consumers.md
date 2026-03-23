# Routes To Consumers

Ce document liste les routes HTTP exposees par le backend et les consommateurs trouves
dans le code versionne des repos `frontend/` et `workers/`.

## Legende

- `frontend` : appelee depuis la console d'administration Next.js
- `worker-rss` : appelee par le worker RSS Rust
- `worker-source-embedding` : appelee par le worker d'embeddings Rust
- `unused` : aucune consommation trouvee dans `frontend/src` ni `workers/`

## Routes backend

| Methode | Route | Consommateur(s) | Reference code |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | frontend | `frontend/src/lib/server/auth-actions.ts` |
| `POST` | `/auth/login` | frontend | `frontend/src/lib/server/auth-actions.ts` |
| `POST` | `/auth/logout` | frontend | `frontend/src/lib/server/auth-actions.ts` |
| `GET` | `/auth/session` | frontend | `frontend/src/app/api/auth/session/route.ts`, `frontend/src/lib/server/backend.ts` |
| `GET` | `/health/` | frontend | `frontend/src/services/api/health.service.ts` |
| `GET` | `/rss/` | frontend | `frontend/src/services/api/rss.service.ts`, `frontend/src/services/api/sources.service.ts` |
| `PATCH` | `/rss/feeds/{feed_id}/enabled` | frontend | `frontend/src/services/api/rss.service.ts` |
| `PATCH` | `/rss/companies/{company_id}/enabled` | frontend | `frontend/src/services/api/rss.service.ts` |
| `POST` | `/rss/sync` | frontend | `frontend/src/services/api/rss.service.ts` |
| `GET` | `/rss/img/{icon_url:path}` | frontend | `frontend/src/features/rss/components/FeedCard.tsx`, `frontend/src/features/rss/components/CompanyCard.tsx` |
| `POST` | `/rss/ingest` | frontend | `frontend/src/services/api/rss.service.ts` |
| `GET` | `/sources/` | frontend | `frontend/src/services/api/sources.service.ts` |
| `GET` | `/sources/feeds/{feed_id}` | frontend | `frontend/src/services/api/sources.service.ts` |
| `GET` | `/sources/companies/{company_id}` | frontend | `frontend/src/services/api/sources.service.ts` |
| `GET` | `/sources/{source_id}` | frontend | `frontend/src/services/api/sources.service.ts` |
| `POST` | `/sources/embeddings/enqueue` | frontend | `frontend/src/services/api/sources.service.ts` |
| `GET` | `/jobs` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/jobs/{job_id}` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/jobs/{job_id}/tasks` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/jobs/{job_id}/feeds` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/jobs/{job_id}/sources` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/jobs/{job_id}/embeddings` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `DELETE` | `/jobs/{job_id}` | frontend | `frontend/src/services/api/jobs.service.ts` |
| `GET` | `/account/me` | frontend | `frontend/src/features/user/components/UserProfilePanel.tsx` |
| `PATCH` | `/account/me` | frontend | `frontend/src/features/user/components/UserProfilePanel.tsx` |
| `GET` | `/account/api-keys` | frontend | `frontend/src/features/user/components/UserApiKeysPanel.tsx` |
| `POST` | `/account/api-keys` | frontend | `frontend/src/features/user/components/UserApiKeysPanel.tsx` |
| `DELETE` | `/account/api-keys/{api_key_id}` | frontend | `frontend/src/features/user/components/UserApiKeysPanel.tsx` |
| `GET` | `/account/workers` | frontend | `frontend/src/features/user/components/UserWorkersPanel.tsx` |
| `GET` | `/admin/users` | frontend | `frontend/src/features/user/components/AdminUsersPanel.tsx` |
| `PATCH` | `/admin/users/{user_id}` | frontend | `frontend/src/features/user/components/AdminUsersPanel.tsx` |
| `GET` | `/workers/overview` | frontend | `frontend/src/services/api/workers.service.ts` |
| `GET` | `/workers/queues/overview` | unused | none |
| `POST` | `/workers/queues/{queue_name}/purge` | unused | none |
| `GET` | `/workers/ping` | worker-rss, worker-source-embedding | `workers/manifeed-worker-common/src/diagnostics.rs` |
| `GET` | `/workers/releases/manifest` | worker-rss, worker-source-embedding | `workers/manifeed-worker-common/src/release.rs` |
| `POST` | `/workers/rss/claim` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/complete` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/fail` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/state` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/embedding/claim` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |
| `POST` | `/workers/embedding/complete` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |
| `POST` | `/workers/embedding/fail` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |

## Notes

- cette matrice est basee sur une analyse statique des appels dans `frontend/src` et `workers/` ;
- les tests, scripts manuels et appels externes hors repo ne sont pas comptes comme consommateurs applicatifs ;
- le frontend appelle souvent des routes proxy `/api/...` qui relayent ensuite vers les routes backend listees ici.
