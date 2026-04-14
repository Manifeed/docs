# Routes Jobs

## GET /api/admin/jobs

- Consommateurs : `frontend/src/services/api/jobs.service.ts`
- Securite : `Session admin`
- Query : `limit`
- Output : `JobsOverviewRead`

## GET /api/admin/jobs/{job_id}

- Consommateurs : `frontend/src/services/api/jobs.service.ts`
- Securite : `Session admin`
- Output : `JobStatusRead`

## GET /api/admin/jobs/{job_id}/tasks

- Consommateurs : `frontend/src/services/api/jobs.service.ts`
- Securite : `Session admin`
- Query : `limit`, `offset`
- Output : `list[JobTaskRead]`

## POST /api/admin/jobs/rss-scrape

- Consommateurs : `frontend/src/services/api/jobs.service.ts`, `frontend/src/app/rss/RssAdminPageClient.tsx`, `frontend/src/app/sources/SourcesAdminPageClient.tsx`
- Securite : `Session admin`
- Input : `RssScrapeJobCreateRequestSchema`
- Output : `JobEnqueueRead`

## POST /api/admin/jobs/source-embedding

- Consommateurs : `frontend/src/services/api/jobs.service.ts`, `frontend/src/app/sources/SourcesAdminPageClient.tsx`
- Securite : `Session admin`
- Input : `SourceEmbeddingJobCreateRequestSchema`
- Output : `JobEnqueueRead`
