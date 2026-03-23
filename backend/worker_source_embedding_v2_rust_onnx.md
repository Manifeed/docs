# Worker Source Embedding Rust + ONNX

Le nom du fichier historique est conserve, mais ce document decrit le worker d'embeddings
actuel present dans :

- `worker-source-embedding/`
- `worker-source-embedding-desktop/`

## Role

Le worker `worker-source-embedding` :

- consomme l'API workers du backend avec une cle API Bearer ;
- claim des tasks embedding filtrees par `worker_version` ;
- telecharge ou reutilise localement les artefacts ONNX du modele fixe ;
- charge ONNX Runtime ;
- calcule les embeddings des sources ;
- renvoie les resultats au backend ;
- maintient un fichier d'etat JSON local ;
- alimente l'app desktop partagee `Manifeed Workers`.

Le worker ne parle pas directement a PostgreSQL.

## Contrat de pipeline

- modele fixe : `Xenova/multilingual-e5-large`
- version logique envoyee au backend : `MANIFEED_EMBEDDING_WORKER_VERSION`
- valeur par defaut : `e5-large-v1`

Le backend :

- n'accepte qu'une version cible a la fois (`SOURCE_EMBEDDING_WORKER_VERSION`) ;
- n'envoie au worker que les tasks rattachees a cette version ;
- attend la meme version sur `claim`, `complete` et `fail`.

## Binaire CLI

Le binaire expose maintenant un vrai mode CLI persistant :

```bash
worker-source-embedding install --api-url ... --api-key ...
worker-source-embedding run
worker-source-embedding probe
worker-source-embedding config show
worker-source-embedding config set api-url https://api.example.com
worker-source-embedding config set api-key mfk_live_xxxxx
worker-source-embedding config set acceleration gpu
worker-source-embedding doctor
worker-source-embedding version
worker-source-embedding service install
worker-source-embedding service start
worker-source-embedding service stop
worker-source-embedding service uninstall
```

### `run`

Lance la boucle de travail normale a partir de la configuration locale persistante.

### `probe`

Retourne un JSON de diagnostic sur :

- le backend d'execution recommande ;
- le bundle ONNX Runtime recommande ;
- les remarques de compatibilite detectees.

## Configuration

La configuration nominale est stockee dans :

- `~/.config/manifeed/workers.json`
- `~/.cache/manifeed/embedding/worker.log`
- `~/.local/state/manifeed/embedding/status.json`

L'installation standard ne demande que :

- `api_url`
- `api_key`

L'interface desktop et `config set` permettent ensuite de modifier `api_url`, `api_key`,
`acceleration_mode` et le mode service sans repasser par l'environnement.

## Manuel ou service utilisateur

- `Manuel` : le worker tourne seulement quand tu le lances depuis `worker-source-embedding run` ou depuis l'app desktop. C'est le mode adapte pour les tests et les lancements ponctuels.
- `Service utilisateur` : le worker est installe comme service OS utilisateur. Il peut continuer a tourner sans garder l'application ouverte et convient a une machine dediee a l'inference.

Cette distinction est maintenant rendue explicite dans la page `Embedding` de l'app `Manifeed Workers`.

### Variables d'environnement expertes

| Variable | Defaut | Usage |
| --- | --- | --- |
| `MANIFEED_API_URL` | `http://127.0.0.1:8000` | URL du backend |
| `MANIFEED_EMBEDDING_WORKER_VERSION` | `e5-large-v1` | version logique envoyee au backend |
| `MANIFEED_EMBEDDING_POLL_SECONDS` | `30` | pause entre deux claims vides |
| `MANIFEED_EMBEDDING_LEASE_SECONDS` | `300` | lease demandee au backend |
| `MANIFEED_EMBEDDING_INFERENCE_BATCH_SIZE` | `1` | taille du lot infere en local |
| `MANIFEED_EMBEDDING_EXECUTION_BACKEND` | `auto` | `auto`, `cpu`, `cuda`, `webgpu` |
| `MANIFEED_EMBEDDING_ORT_DYLIB_PATH` | vide | chemin explicite vers ONNX Runtime |
| `MANIFEED_EMBEDDING_STATUS_FILE` | `~/.local/state/manifeed/worker-source-embedding/status.json` | etat local |
| `MANIFEED_EMBEDDING_CACHE_DIR` | `~/.cache/manifeed/worker-source-embedding/models` | cache du modele fixe |
| `MANIFEED_EMBEDDING_HF_BASE_URL` | `https://huggingface.co` | endpoint Hugging Face |
| `MANIFEED_EMBEDDING_HF_DEFAULT_REVISION` | `main` | revision Hugging Face |
| `MANIFEED_EMBEDDING_HF_TOKEN` | vide | token HF optionnel |
| `HF_TOKEN` | vide | fallback du token HF |

## Endpoints consommes

Le worker d'embeddings consomme le prefixe officiel `/workers/*`.

Routes utilisees :

| Methode | Route | Role |
| --- | --- | --- |
| `GET` | `/workers/ping` | test de connexion local |
| `GET` | `/workers/releases/manifest` | verification de version |
| `POST` | `/workers/embedding/claim` | claim d'une task pour `worker_version` |
| `POST` | `/workers/embedding/complete` | completion d'une task |
| `POST` | `/workers/embedding/fail` | echec d'une task |

## Boucle d'execution

1. `probe_system()` inspecte la machine ;
2. `ensure_ort_runtime_loaded()` charge ONNX Runtime ;
3. le worker initialise son fichier d'etat local ;
4. `claim()` demande au backend une task avec `worker_version` ;
5. le manager Hugging Face charge le modele fixe si necessaire ;
6. le worker calcule les embeddings ;
7. `complete()` renvoie `result_payload.sources[]` et `worker_version` ;
8. en cas d'erreur, `fail()` renvoie aussi `worker_version`.

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

`current_task` expose la `worker_version` en cours de traitement.

## App desktop partagee

Le binaire desktop partage :

- lit `workers.json` et les status files RSS/embedding ;
- permet de piloter les deux workers dans une interface unique ;
- expose deux pages clairement separees :
  - `Scraping`
  - `Embedding`
- permet de changer `api_url`, `api_key`, `service_mode` et `acceleration_mode` ;
- verifie la connectivite avec `/workers/ping` ;
- verifie la version avec `/workers/releases/manifest` ;
- empeche de demarrer un second processus manuel si un worker externe est deja actif.

Le bundle Linux RSS et le bundle Linux embedding installent la meme app `Manifeed Workers`.

## Construction et execution

### Build des binaires

```bash
cd ../workers
cargo build --release -p worker-source-embedding -p worker-source-embedding-desktop
```

### Run manuel

```bash
./target/release/worker-source-embedding install \
  --api-url http://127.0.0.1:8000 \
  --api-key mfk_live_xxxxx
./target/release/worker-source-embedding run
```

### Probe

```bash
./target/release/worker-source-embedding probe
```

### Bundle Linux

```bash
./installers/linux/worker-source-embedding/build-bundle.sh
./dist/linux/worker-source-embedding/install.sh --cli
```

## Notes d'exploitation

- le worker n'accepte plus de modele dynamique dans le payload ;
- le backend garde un champ de compatibilite `embedding_model_name` dans le claim tant que la transition n'est pas totalement retiree ;
- le cache local reste organise par repo Hugging Face et revision ;
- l'installeur Linux provisionne aujourd'hui les bundles ONNX Runtime CPU ou CUDA, pas un runtime WebGPU partage ;
- l'acceleration exposee a l'utilisateur est `auto|cpu|gpu`; `npu` reste reserve pour une extension future ;
- la page `Embedding` de l'app desktop montre explicitement le probe runtime GPU, les chemins de cache et les modes `Manuel` / `Service utilisateur` ;
- le fichier `dist/linux/worker-source-embedding/` est un artefact de distribution produit par script.
