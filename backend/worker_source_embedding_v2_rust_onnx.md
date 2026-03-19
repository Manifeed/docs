# Worker Source Embedding Rust + ONNX

Le nom du fichier historique est conserve, mais ce document decrit le worker d'embeddings
actuel present dans :

- `worker-source-embedding/`
- `worker-source-embedding-desktop/`

## Role

Le worker `worker-source-embedding` :

- s'enrole et s'authentifie aupres du backend ;
- claim des tasks d'embedding ;
- telecharge ou reutilise localement les artefacts ONNX du modele demande ;
- charge ONNX Runtime ;
- calcule les embeddings des sources ;
- renvoie les resultats au backend ;
- maintient un fichier d'etat JSON local ;
- alimente une UI desktop Linux facultative.

Le worker ne parle pas directement a PostgreSQL.

## Binaire CLI

Le binaire accepte trois commandes :

```bash
worker-source-embedding run
worker-source-embedding probe
worker-source-embedding enroll
```

### `run`

Lance la boucle de travail normale.

### `probe`

Retourne un JSON de diagnostic sur :

- le backend d'execution recommande ;
- le bundle ONNX Runtime recommande ;
- les remarques de compatibilite detectees.

### `enroll`

Force une sequence d'enrolement/authentification puis affiche un resume JSON.

## Configuration

### Variables principales

| Variable | Defaut | Usage |
| --- | --- | --- |
| `MANIFEED_API_URL` | `http://127.0.0.1:8000` | URL du backend |
| `MANIFEED_EMBEDDING_POLL_SECONDS` | `30` | pause entre deux claims vides |
| `MANIFEED_EMBEDDING_LEASE_SECONDS` | `300` | lease demandee au backend |
| `MANIFEED_EMBEDDING_INFERENCE_BATCH_SIZE` | `1` | taille du lot infere en local |
| `MANIFEED_EMBEDDING_EXECUTION_BACKEND` | `auto` | `auto`, `cpu`, `cuda`, `webgpu` |
| `MANIFEED_EMBEDDING_ORT_DYLIB_PATH` | vide | chemin explicite vers ONNX Runtime |
| `MANIFEED_EMBEDDING_STATUS_FILE` | `~/.local/state/manifeed/worker-source-embedding/status.json` | etat local |
| `MANIFEED_EMBEDDING_CACHE_DIR` | `~/.cache/manifeed/worker-source-embedding/models` | cache des modeles |
| `MANIFEED_EMBEDDING_HF_BASE_URL` | `https://huggingface.co` | endpoint Hugging Face |
| `MANIFEED_EMBEDDING_HF_DEFAULT_REVISION` | `main` | revision Hugging Face |
| `MANIFEED_EMBEDDING_HF_TOKEN` | vide | token HF optionnel |
| `HF_TOKEN` | vide | fallback du token HF |
| `MANIFEED_EMBEDDING_IDENTITY_DIR` | `~/.config/manifeed/worker-source-embedding` | identite locale |
| `MANIFEED_EMBEDDING_ENROLLMENT_TOKEN` | vide | token d'enrolement initial |

## Authentification et endpoints

Comme le worker RSS, le worker d'embeddings utilise actuellement le prefixe legacy `/workers/*`.

Routes utilisees :

| Methode | Route | Role |
| --- | --- | --- |
| `POST` | `/workers/enroll` | enrolement initial |
| `POST` | `/workers/auth/challenge` | challenge d'auth |
| `POST` | `/workers/auth/verify` | verification de signature |
| `POST` | `/workers/embedding/claim` | claim d'une task |
| `POST` | `/workers/embedding/complete` | completion d'une task |
| `POST` | `/workers/embedding/fail` | echec d'une task |

Le backend publie la version officielle de ces routes sous `/internal/workers/*`.

## Boucle d'execution

1. `probe_system()` inspecte la machine ;
2. `ensure_ort_runtime_loaded()` charge ONNX Runtime ;
3. le worker initialise son fichier d'etat local ;
4. `claim()` demande au backend une task ;
5. le manager de modele Hugging Face charge le modele requis si necessaire ;
6. le worker calcule les embeddings ;
7. `complete()` renvoie `result_payload.sources[]` ;
8. en cas d'erreur, `fail()` ou marque l'etat local en erreur.

En cas d'erreur reseau, le worker :

- marque le serveur comme deconnecte dans son status file ;
- attend `5` secondes ;
- retente automatiquement.

## Fichier d'etat local

Le worker persiste un JSON atomique contenant notamment :

- `worker_type`
- `execution_backend`
- `pid`
- `phase`
- `server_connection`
- `started_at`
- `last_updated_at`
- `last_server_contact_at`
- `current_task`
- `completed_task_count`
- `network_totals`
- `last_error`

Ce fichier est consomme par `worker-source-embedding-desktop`.

## UI desktop Linux

Le binaire `worker-source-embedding-desktop` :

- lit le fichier d'environnement et le status file ;
- demarre ou arrete le worker comme processus enfant ;
- affiche la connexion serveur, la phase courante, la CPU et le trafic reseau ;
- empeche de lancer un second worker si un processus externe est deja actif.

Le bundle Linux et l'installeur prepares dans ce depot ciblent cette variante.

## Construction et execution

### Build des binaires

```bash
cd ../workers
cargo build --release -p worker-source-embedding -p worker-source-embedding-desktop
```

### Run manuel

```bash
export MANIFEED_API_URL=http://127.0.0.1:8000
export MANIFEED_EMBEDDING_ENROLLMENT_TOKEN=manifeed-embedding-enroll
./target/release/worker-source-embedding run
```

### Probe

```bash
./target/release/worker-source-embedding probe
```

### Enroll

```bash
./target/release/worker-source-embedding enroll
```

### Bundle Linux

```bash
./installers/linux/worker-source-embedding/build-bundle.sh
./dist/linux/worker-source-embedding/install.sh --cli
```

## Notes d'exploitation

- le modele logique actif cote backend est determine par `EMBEDDING_MODEL_NAME` ;
- le worker telecharge les artefacts du modele au besoin, puis les garde en cache local ;
- le backend fusionne les resultats dans `rss_source_embeddings` et tente ensuite une mise a jour
  incrementielle de la projection pour les sources du job ;
- l'installeur Linux provisionne aujourd'hui les bundles ONNX Runtime CPU ou CUDA, pas un runtime
  WebGPU partage ;
- le fichier `dist/linux/worker-source-embedding/` est un artefact de distribution produit par script.
