# Routes Auth

## POST /api/auth/register

- Consommateurs : `frontend/src/features/user/components/AuthFormCard.tsx`
- Securite : `Public`
- Input : `AuthRegisterRequestSchema`
- Output : `AuthRegisterRead`
- Effet principal : cree un compte utilisateur avec mot de passe Argon2 et role `user`

## POST /api/auth/login

- Consommateurs : `frontend/src/features/user/components/AuthFormCard.tsx`
- Securite : `Public`
- Input : `AuthLoginRequestSchema`
- Output : `AuthLoginRead`
- Effet principal : cree une session web et retourne `session_token` + user

## POST /api/auth/logout

- Consommateurs : `frontend/src/features/user/components/LogoutButton.tsx`
- Securite : `Session user`
- Output : `AuthLogoutRead`
- Effet principal : revoque la session courante

## GET /api/auth/session

- Consommateurs : `frontend/src/lib/server/backend.ts`
- Securite : `Session user`
- Output : `AuthSessionRead`
- Erreurs : `401` session absente/invalide, `403` compte inactif, `404` user introuvable apres resolution
