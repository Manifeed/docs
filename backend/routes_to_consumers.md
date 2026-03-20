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
| `GET` | `/health/` | frontend | `frontend/src/features/health/hooks/useHealthStatus.ts` |
| `GET` | `/rss/` | frontend | `frontend/src/app/rss/page.tsx`, `frontend/src/app/sources/page.tsx` |
| `PATCH` | `/rss/feeds/{feed_id}/enabled` | frontend | `frontend/src/app/rss/page.tsx` |
| `PATCH` | `/rss/companies/{company_id}/enabled` | frontend | `frontend/src/app/rss/page.tsx` |
| `POST` | `/rss/sync` | frontend | `frontend/src/app/rss/page.tsx` |
| `GET` | `/rss/img/{icon_url:path}` | frontend | `frontend/src/features/rss/components/FeedCard.tsx`, `frontend/src/features/rss/components/CompanyCard.tsx` |
| `POST` | `/rss/ingest` | frontend | `frontend/src/app/rss/page.tsx`, `frontend/src/app/sources/page.tsx` |
| `POST` | `/sources/embeddings/enqueue` | frontend | `frontend/src/app/sources/page.tsx` |
| `GET` | `/sources/` | frontend | `frontend/src/app/sources/page.tsx` |
| `GET` | `/sources/feeds/{feed_id}` | frontend | `frontend/src/app/sources/page.tsx` |
| `GET` | `/sources/companies/{company_id}` | frontend | `frontend/src/app/sources/page.tsx` |
| `GET` | `/sources/{source_id}` | frontend | `frontend/src/app/sources/page.tsx` |
| `GET` | `/jobs` | frontend | `frontend/src/app/jobs/page.tsx` |
| `GET` | `/jobs/{job_id}` | frontend | `frontend/src/app/jobs/page.tsx` |
| `GET` | `/jobs/{job_id}/tasks` | frontend | `frontend/src/app/jobs/page.tsx` |
| `GET` | `/jobs/{job_id}/feeds` | frontend | `frontend/src/app/jobs/page.tsx` |
| `GET` | `/jobs/{job_id}/sources` | frontend | `frontend/src/app/jobs/page.tsx` |
| `GET` | `/jobs/{job_id}/embeddings` | frontend | `frontend/src/app/jobs/page.tsx` |
| `DELETE` | `/jobs/{job_id}` | frontend | `frontend/src/app/jobs/page.tsx` |
| `POST` | `/workers/enroll` | worker-rss, worker-source-embedding | `workers/manifeed-worker-common/src/auth.rs` |
| `POST` | `/workers/auth/challenge` | worker-rss, worker-source-embedding | `workers/manifeed-worker-common/src/auth.rs` |
| `POST` | `/workers/auth/verify` | worker-rss, worker-source-embedding | `workers/manifeed-worker-common/src/auth.rs` |
| `GET` | `/workers/me` | unused | none |
| `GET` | `/workers/overview` | frontend | `frontend/src/app/workers/page.tsx` |
| `GET` | `/workers/queues/overview` | unused | none |
| `POST` | `/workers/queues/{queue_name}/purge` | unused | none |
| `POST` | `/workers/rss/claim` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/complete` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/fail` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/rss/state` | worker-rss | `workers/worker-rss/src/api.rs` |
| `POST` | `/workers/embedding/claim` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |
| `POST` | `/workers/embedding/complete` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |
| `POST` | `/workers/embedding/fail` | worker-source-embedding | `workers/worker-source-embedding/src/api.rs` |

## Notes

- Cette matrice est basee sur une analyse statique des appels dans `frontend/src` et `workers/`.
- Les tests, scripts manuels et appels externes hors repo ne sont pas comptes comme consommateurs applicatifs.
