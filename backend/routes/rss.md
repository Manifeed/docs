# Routes RSS

## GET /api/admin/rss/

- Consommateurs : `frontend/src/services/api/rss.service.ts`
- Securite : `Session admin`
- Output : `list[RssFeedRead]`

## PATCH /api/admin/rss/feeds/{feed_id}/enabled

- Consommateurs : `frontend/src/services/api/rss.service.ts`
- Securite : `Session admin`
- Input : `RssEnabledTogglePayload`
- Output : `RssFeedEnabledToggleRead`

## PATCH /api/admin/rss/companies/{company_id}/enabled

- Consommateurs : `frontend/src/services/api/rss.service.ts`
- Securite : `Session admin`
- Input : `RssEnabledTogglePayload`
- Output : `RssCompanyEnabledToggleRead`

## POST /api/admin/rss/sync

- Consommateurs : `frontend/src/services/api/rss.service.ts`
- Securite : `Session admin`
- Query : `force`
- Output : `RssSyncRead`

## GET /api/rss/img/{icon_url}

- Consommateurs : `frontend/src/features/rss/components/FeedCard.tsx`, `frontend/src/features/rss/components/CompanyCard.tsx`
- Securite : `Public`
- Output : `image/svg+xml`
