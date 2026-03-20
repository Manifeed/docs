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

Le worker charge sa configuration depuis l'environnement.

### Variables principales

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
| `MANIFEED_RSS_IDENTITY_DIR` | auto | stockage de l'identite locale |
| `MANIFEED_RSS_ENROLLMENT_TOKEN` | vide | token d'enrolement initial |

## Authentification

L'authentification est geree par `manifeed-worker-common`.

Flux :

1. creation ou chargement d'une identite locale ;
2. si le worker n'est pas enrole, `POST /workers/enroll` ;
3. signature du challenge renvoye par le backend ;
4. `POST /workers/auth/verify` ;
5. reutilisation du JWT tant qu'il est valide ;
6. re-challenge automatique si la session expire.

Important :

- le binaire et le backend utilisent maintenant un prefixe unique `/workers/*`.

## Endpoints utilises

| Methode | Route utilisee par le binaire | Role |
| --- | --- | --- |
| `POST` | `/workers/enroll` | enrolement initial |
| `POST` | `/workers/auth/challenge` | challenge d'auth |
| `POST` | `/workers/auth/verify` | verification de signature |
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

Puis :

```bash
export MANIFEED_API_URL=http://127.0.0.1:8000
export MANIFEED_RSS_ENROLLMENT_TOKEN=manifeed-rss-enroll
./target/release/worker-rss
```

Avec logs stdout renforces :

```bash
./target/release/worker-rss --log
```

## Points d'exploitation

- le worker ne dispose pas d'installeur dedie dans ce depot ;
- son identite locale persiste entre les runs ;
- si le backend oublie l'identite, le mecanisme commun peut reenroller automatiquement le worker ;
- le backend reste la source de verite pour l'etat runtime visible dans la page `Workers`.
