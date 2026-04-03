# Routes Backend Par Domaine

## Methode

Cette reference est reconstruite depuis :

- les routers FastAPI ;
- les schemas Pydantic ;
- les services et clients DB ;
- les consommateurs trouves dans `../frontend` et `../workers`.

Chaque fichier couvre un domaine fonctionnel et chaque endpoint possede son propre
diagramme Mermaid.

## Format de lecture

Chaque route suit la meme structure :

- diagramme Mermaid unique ;
- consommateurs identifies dans le code versionne ;
- securite effective ;
- inputs HTTP ;
- output HTTP ;
- erreurs explicites ;
- tables / systemes touches ;
- processus reel, aligne sur le code.

## Legende securite

- `Public` : aucune authentification requise.
- `Session user` : session web valide via `x-manifeed-session` ou `manifeed_session`.
- `Session admin` : meme mecanisme, plus `users.role == "admin"`.
- `Worker bearer` : API key worker transmise en `Authorization: Bearer <key>`.
- `Worker bearer + HMAC` : meme Bearer, plus signature HMAC dans le body.

## Fichiers

| Fichier | Domaine | Routes |
| --- | --- | --- |
| [auth.md](./auth.md) | auth web | 4 |
| [account.md](./account.md) | compte utilisateur | 6 |
| [admin.md](./admin.md) | administration | 2 |
| [health.md](./health.md) | supervision minimale | 1 |
| [rss.md](./rss.md) | catalogue RSS | 5 |
| [sources.md](./sources.md) | lecture sources | 4 |
| [jobs.md](./jobs.md) | creation et suivi des jobs | 5 |
| [workers.md](./workers.md) | gateway worker et releases | 6 |

## Points d'attention globaux

- Le protocole worker courant passe uniquement par `/workers/ping`, `/workers/sessions/open` et `/workers/tasks/*`.
- Le flag `reembed_model_mismatches` est publie sur `POST /jobs/source-embedding` mais n'est pas encore exploite dans la requete SQL courante.
- `GET /health/` est admin-only.
- `GET /workers/releases/manifest` est public.
