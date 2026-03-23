# Worker RSS Rust

Le nom du fichier historique est conserve, mais ce document decrit l'etat actuel du worker RSS
versionne dans `worker-rss/` au sein du repo `workers`.

## Role

Le worker RSS est un binaire Rust natif charge de :

- s'enroler et s'authentifier aupres du backend ;
- claim des tasks RSS ;
- recuperer les flux HTTP/HTTPS ;
- parser RSS/Atom ;
- normaliser les resultats ;
- poster `complete` ou `fail` au backend ;
- publier son etat runtime via `/workers/rss/state`.

Le worker ne parle pas directement a PostgreSQL.

## Configuration

Le worker charge d'abord une configuration locale persistante stockee dans `workers.json`.
Les variables d'environnement historiques restent supportees comme overrides experts.

### Configuration nominale

L'installation standard renseigne seulement :

- `api_url`
- `api_key`

Puis le binaire lit automatiquement :

- `~/.config/manifeed/workers.json`
- `~/.cache/manifeed/rss/worker.log`
- `~/.local/state/manifeed/rss/status.json`

### Variables d'environnement expertes

| Variable | Defaut | Usage |
| --- | --- | --- |
| `MANIFEED_API_URL` | `http://127.0.0.1:8000` | base URL du backend |
| `MANIFEED_RSS_POLL_SECONDS` | `5` | attente quand aucune task n'est claimable |
| `MANIFEED_RSS_LEASE_SECONDS` | `300` | duree de lease demandee au backend |
| `MANIFEED_RSS_HOST_MAX_REQUESTS_PER_SECOND` | `20` | rate limit par host |
| `MANIFEED_RSS_MAX_IN_FLIGHT_REQUESTS` | `5` | parallellisme global |
| `MANIFEED_RSS_MAX_IN_FLIGHT_REQUESTS_PER_HOST` | `5` | parallellisme par host |
| `MANIFEED_RSS_MAX_CLAIMED_TASKS` | `5` | nombre max de tasks claim simultanement |
| `MANIFEED_RSS_REQUEST_TIMEOUT_SECONDS` | `10` | timeout HTTP |
| `MANIFEED_RSS_FETCH_RETRY_COUNT` | `1` | retries de fetch |
## CLI

Le binaire expose maintenant :

```bash
worker-rss install --api-url ... --api-key ...
worker-rss run
worker-rss config show
worker-rss config set api-url https://api.example.com
worker-rss config set api-key mfk_live_xxxxx
worker-rss config set service-mode background
worker-rss doctor
worker-rss version
worker-rss service install
worker-rss service start
worker-rss service stop
worker-rss service uninstall
```

## Manuel ou service utilisateur

- `Manuel` : le worker tourne seulement quand tu le lances depuis `worker-rss run` ou depuis l'app desktop. C'est le mode simple pour tester, depanner ou lancer ponctuellement.
- `Service utilisateur` : le worker est installe comme service OS utilisateur. Il peut continuer a tourner sans garder la fenetre desktop ouverte et convient a une machine dediee.

L'app `Manifeed Workers` expose explicitement ces deux modes dans la page `Scraping`.

## Authentification

L'authentification utilise une cle API worker Bearer.
Le nom du worker est derive uniquement par le backend a partir du pseudo utilisateur.

## Endpoints utilises

| Methode | Route utilisee par le binaire | Role |
| --- | --- | --- |
| `GET` | `/workers/ping` | test de connexion local |
| `GET` | `/workers/releases/manifest` | verification de version |
| `POST` | `/workers/rss/claim` | claim de tasks |
| `POST` | `/workers/rss/complete` | completion de task |
| `POST` | `/workers/rss/fail` | echec de task |
| `POST` | `/workers/rss/state` | publication de l'etat courant |

## Boucle d'execution

La boucle du worker est simple :

1. initialisation du client backend et du fetcher HTTP ;
2. `claim()` ;
3. si rien n'est claim, `sleep(poll_seconds)` ;
4. sinon fetch et parse des feeds de la task ;
5. `complete()` avec les resultats par feed ;
6. ou `fail()` si la task echoue globalement ;
7. reprise immediate d'une nouvelle iteration.

En cas d'erreur de loop non recuperable, le worker loggue l'erreur puis attend `3` secondes avant
de retenter.

## Construction et execution

Depuis la racine du repo `workers` :

```bash
cargo build --release -p worker-rss
```

Initialisation persistante :

```bash
./target/release/worker-rss install \
  --api-url http://127.0.0.1:8000 \
  --api-key mfk_live_xxxxx
```

Puis lancement :

```bash
./target/release/worker-rss run
```

Avec logs stdout renforces :

```bash
./target/release/worker-rss --log
```

## Points d'exploitation

- le bundle Linux `installers/linux/worker-rss/` fournit la meme experience d'installation que l'embedding ;
- la desktop app partagee `Manifeed Workers` expose maintenant une page dediee `Scraping` avec :
  - actions de demarrage / arret / redemarrage ;
  - edition de `api_url`, `api_key` et du mode `Manuel` / `Service utilisateur` ;
  - lecture du status file local, des logs et du runtime local ;
- le worker verifie sa version au lancement et bloque seulement si elle est sous la version minimale supportee ;
- le backend reste la source de verite pour l'etat runtime visible dans la page `Workers`.
