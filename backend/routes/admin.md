# Routes Admin

## GET /admin/users

```mermaid
flowchart LR
    admin["Frontend admin"] --> route["GET /admin/users"]
    route --> auth["Verifier role admin"]
    auth --> users[(users)]
    users --> response[["200 AdminUserListRead"]]
```

- Consommateurs : `frontend/src/features/user/components/AdminUsersPanel.tsx`.
- Securite : `Session admin`.
- Inputs :
  - Query optionnelles :
    - `role`: `user | admin`
    - `is_active`: `true | false`
    - `api_access_enabled`: `true | false`
- Output :
  - `200` `AdminUserListRead { items[] }`.
- Tables / systemes :
  - lecture `users`.
- Processus :
  1. verifie la session admin ;
  2. applique les filtres fournis avec une logique `AND` ;
  3. liste les users tries par creation ;
  4. retourne la vue admin complete.

## PATCH /admin/users/{user_id}

```mermaid
flowchart LR
    admin["Frontend admin"] --> route["PATCH /admin/users/{user_id}"]
    route --> normalize["Normaliser pseudo si present"]
    normalize --> users[(users)]
    users --> response[["200 AdminUserRead"]]
```

- Consommateurs : `frontend/src/features/user/components/AdminUsersPanel.tsx`.
- Securite : `Session admin`.
- Inputs :
  - Path `user_id >= 1`.
  - Body `AdminUserUpdateRequestSchema { is_active?, api_access_enabled? }`.
- Output :
  - `200` `AdminUserRead`.
- Erreurs :
  - `404` user inconnu.
  - `422` si le body contient `pseudo`, `role` ou tout autre champ inattendu.
- Tables / systemes :
  - lecture `users` ;
  - mise a jour `users`.
- Processus :
  1. charge le user cible ;
  2. applique uniquement `is_active` et `api_access_enabled` ;
  3. commit et retourne le user final.
