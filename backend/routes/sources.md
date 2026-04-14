# Routes Sources

## GET /api/admin/sources/

- Consommateurs : `frontend/src/services/api/sources.service.ts`
- Securite : `Session admin`
- Query : `limit`, `offset`, `author_id`
- Output : `RssSourcePageRead`

## GET /api/admin/sources/feeds/{feed_id}

- Consommateurs : `frontend/src/services/api/sources.service.ts`
- Securite : `Session admin`
- Query : `limit`, `offset`, `author_id`
- Output : `RssSourcePageRead`

## GET /api/admin/sources/companies/{company_id}

- Consommateurs : `frontend/src/services/api/sources.service.ts`
- Securite : `Session admin`
- Query : `limit`, `offset`, `author_id`
- Output : `RssSourcePageRead`

## GET /api/admin/sources/{source_id}

- Consommateurs : `frontend/src/services/api/sources.service.ts`
- Securite : `Session admin`
- Output : `RssSourceDetailRead`
