# Routes Admin

## GET /api/admin/users

- Consommateurs : `frontend/src/features/user/components/AdminUsersPanel.tsx`
- Securite : `Session admin`
- Query : `role`, `is_active`, `api_access_enabled`
- Output : `AdminUserListRead`
- Erreurs : `404` masque pour utilisateur authentifie non admin

## PATCH /api/admin/users/{user_id}

- Consommateurs : `frontend/src/features/user/components/AdminUsersPanel.tsx`
- Securite : `Session admin`
- Input : `AdminUserUpdateRequestSchema`
- Output : `AdminUserRead`
- Erreurs : `404` masque pour utilisateur authentifie non admin
