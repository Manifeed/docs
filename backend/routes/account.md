# Routes Account

## GET /api/account/me

- Consommateurs : `frontend/src/features/user/components/UserProfilePanel.tsx`
- Securite : `Session user`
- Output : `AccountMeRead`

## PATCH /api/account/me

- Consommateurs : `frontend/src/features/user/components/UserProfilePanel.tsx`
- Securite : `Session user`
- Input : `AccountProfileUpdateRequestSchema`
- Output : `AccountProfileUpdateRead`

## PATCH /api/account/password

- Consommateurs : `frontend/src/features/user/components/UserProfilePanel.tsx`
- Securite : `Session user`
- Input : `AccountPasswordUpdateRequestSchema`
- Output : `AccountPasswordUpdateRead`

## GET /api/account/api-keys

- Consommateurs : `frontend/src/features/user/components/UserApiKeysPanel.tsx`
- Securite : `Session user`
- Output : `UserApiKeyListRead`
- Erreurs : `404` masque si `api_access_enabled=false`

## POST /api/account/api-keys

- Consommateurs : `frontend/src/features/user/components/UserApiKeysPanel.tsx`
- Securite : `Session user`
- Input : `UserApiKeyCreateRequestSchema`
- Output : `UserApiKeyCreateRead`
- Erreurs : `404` masque si `api_access_enabled=false`

## DELETE /api/account/api-keys/{api_key_id}

- Consommateurs : `frontend/src/features/user/components/UserApiKeysPanel.tsx`
- Securite : `Session user`
- Output : `UserApiKeyDeleteRead`
- Erreurs : `404` masque si `api_access_enabled=false` ou ressource absente
