# Ollama architecture overview

## Executive summary

Ollama is a local model manager, HTTP API server, CLI, and runtime orchestrator for running LLM-style models on a user machine. The repository is primarily Go code around model naming, registry transfer, manifest/blob storage, request routing, scheduling, GPU discovery, and launcher integration. GGUF-backed models are executed by a local `llama-server` subprocess bound to `127.0.0.1` on an ephemeral port; Ollama's public local API is the Go server that listens on `OLLAMA_HOST`, defaulting to `127.0.0.1:11434`.

## Major components

| Component | Responsibility | Important source files |
|---|---|---|
| CLI entrypoint | Creates Cobra commands and starts the app/server or calls the HTTP API. | `main.go`, `cmd/cmd.go`, `cmd/start_windows.go` |
| API client | Builds requests to the configured Ollama host. | `api/client.go`, `api/types.go` |
| HTTP server/router | Exposes `/api/*`, OpenAI-compatible `/v1/*`, Anthropic-compatible `/v1/messages`, and selected experimental endpoints. | `server/routes.go`, `middleware/openai.go`, `middleware/anthropic.go` |
| Cloud proxy path | Proxies explicitly cloud-suffixed models to Ollama-hosted endpoints unless cloud is disabled. | `server/cloud_proxy.go`, `internal/cloud/cloud.go` |
| Configuration | Reads environment variables and `~/.ollama/server.json` for cloud-disable state. | `envconfig/config.go` |
| Model naming | Normalizes names to host/namespace/model/tag with defaults. | `types/model/name.go` |
| Model storage | Stores content-addressed blobs and Docker/OCI-like manifests under `OLLAMA_MODELS` or `~/.ollama/models`. | `manifest/paths.go`, `manifest/manifest.go`, `manifest/layer.go` |
| Registry transfer | Pulls/pushes manifests and blobs from/to registry endpoints and verifies downloaded blob SHA-256. | `server/images.go`, `server/download.go`, `server/upload.go`, `server/auth.go` |
| Modelfile creation | Parses Modelfiles, imports local GGUF/safetensors/adapters, writes manifests. | `parser/parser.go`, `server/create.go`, `convert/*` |
| Runtime scheduler | Queues requests, resolves options, discovers memory, chooses placement, loads/unloads runners. | `server/sched.go` |
| GPU/device discovery | Detects CUDA/ROCm/Vulkan/Metal/CPU capabilities and free memory. | `discover/*`, `ml/device.go` |
| Runtime backend | Starts and talks to `llama-server` for GGUF models; MLX/image runners exist for newer model formats/features. | `llm/server.go`, `llm/llama_server.go`, `x/mlxrunner/*`, `x/imagegen/*` |
| Tool launch integrations | Configures tools such as OpenClaw/OpenCode/Codex/Copilot to point at Ollama's OpenAI-compatible API. | `cmd/launch/openclaw.go`, `cmd/launch/opencode.go`, `cmd/launch/codex.go`, `cmd/launch/copilot.go` |

## Process model

```text
CLI or external client
  -> Ollama HTTP server on OLLAMA_HOST, default http://127.0.0.1:11434
      -> route/middleware layer
          -> model resolution + scheduler
              -> local runner subprocess/client
                  -> llama-server on 127.0.0.1:<ephemeral port> for GGUF
                  -> GPU/CPU backend libraries chosen by discovery and env
```

The default public surface is the Go HTTP server, not the runner. The runner is an internal subprocess that Ollama starts with `--host 127.0.0.1`, `--no-webui`, and `--offline`.

## CLI flow

1. `main.go` calls `cmd.NewCLI().ExecuteContext(...)`.
2. `cmd.NewCLI` constructs commands such as `serve/start`, `run`, `pull`, `create`, `show`, `delete`, `stop`, `ps`, and launch helpers.
3. Most user-facing commands run `checkServerHeartbeat` first. If the local server is not reachable and the platform supports app startup, Windows starts `ollama app.exe --hide --fast-startup`; macOS uses the app bundle; other platforms return the connection error.
4. CLI commands then use `api.ClientFromEnvironment()`, which targets `envconfig.Host()`.
5. `ollama serve` runs `RunServer`, initializes the keypair, listens on `envconfig.Host().Host`, then calls `server.Serve`.

Important paths:

- `main.go` -> `cmd.NewCLI`.
- `cmd/cmd.go` -> `NewCLI`, `checkServerHeartbeat`, `RunServer`.
- `cmd/start_windows.go` -> Windows GUI app startup fallback.
- `api/client.go` -> `ClientFromEnvironment`, request/stream methods.

## Server/API flow

`server.Serve` performs startup housekeeping, builds routes, initializes the scheduler and model caches, logs GPU details, then serves HTTP. The router exposes:

- General: `/`, `/api/version`, `/api/status`.
- Local model management: `/api/pull`, `/api/push`, `/api/tags`, `/api/show`, `/api/delete`, `/api/create`, `/api/blobs/:digest`, `/api/copy`.
- Local inference: `/api/generate`, `/api/chat`, `/api/embed`, `/api/embeddings`.
- Runtime status: `/api/ps`.
- OpenAI-compatible endpoints: `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/models`, `/v1/responses`, `/v1/images/*`, `/v1/audio/transcriptions`.
- Anthropic-compatible endpoint: `/v1/messages`.
- Experimental web/model endpoints: `/api/experimental/web_search`, `/api/experimental/web_fetch`, `/api/experimental/model-recommendations`.

The server installs CORS with `envconfig.AllowedOrigins()` and an `allowedHostsMiddleware`. When the listener address is loopback, the middleware rejects Host headers outside local/private/allowed names. When the listener itself is non-loopback, the middleware allows requests through; this makes network binding a major security boundary.

## Model download flow

1. CLI `ollama pull MODEL` or `POST /api/pull` reaches `PullHandler`.
2. `PullHandler` parses and normalizes the model reference.
3. `PullModel` parses the model name. Short names default to `registry.ollama.ai/library/<model>:latest`.
4. `pullModelManifest` requests `GET <scheme>://<host>/v2/<namespace>/<model>/manifests/<tag>` with Docker manifest media type.
5. For normal GGUF-style layers, each layer digest is downloaded from `/v2/<namespace>/<model>/blobs/<digest>` into the local blob store.
6. Downloads are resumable/parallelized using partial files and HTTP range requests.
7. Downloaded blobs are verified by recomputing SHA-256 and matching the digest from the manifest.
8. The raw manifest JSON is written under `models/manifests/<host>/<namespace>/<model>/<tag>`.
9. Unused old layers are pruned unless `OLLAMA_NOPRUNE` is set.

Important paths:

- `server/routes.go` -> `PullHandler`.
- `server/images.go` -> `PullModel`, `pullModelManifest`, `verifyBlob`.
- `server/download.go` -> `downloadBlob`, resumable/ranged download internals.
- `manifest/paths.go` -> blob/manifest filesystem paths.
- `types/model/name.go` -> registry host/namespace/tag defaults.

## Model loading flow

1. `/api/generate`, `/api/chat`, `/api/embed`, and compatible routes bind JSON request types.
2. The handler calls `scheduleRunner` with required model capabilities.
3. `scheduleRunner` resolves the model with `GetModel`, validates capabilities, merges model/request/default options, and asks the scheduler for a runner.
4. The scheduler checks whether a compatible runner is already loaded. If not, it queues a load request.
5. For GGUF-backed models, the scheduler opens model metadata with `llm.LoadModel`, predicts VRAM with `llm.PredictServerVRAM`, chooses GPU placement, applies automatic context/batch/mmap defaults, and creates a `llama-server` runner.
6. The runner is started as a subprocess with a model path, private localhost port, context size, parallelism, optional adapters/projector, KV cache type, flash-attention mode, batch size, and GPU offload flags.
7. The scheduler waits for `/health` on the runner to report ready, then marks the runner loaded.
8. Runners are kept alive for `OLLAMA_KEEP_ALIVE`, request `keep_alive`, or unloaded on expiry/eviction.

Important paths:

- `server/routes.go` -> `scheduleRunner`, `GenerateHandler`, `ChatHandler`, `EmbedHandler`.
- `server/sched.go` -> `getRunner`, `load`, placement and retry decisions.
- `llm/server.go` -> runtime interface and `NewLlamaServer`.
- `llm/llama_server.go` -> runner subprocess launch and HTTP calls to runner endpoints.

## Inference/request flow

For local GGUF inference:

1. Client posts to `/api/chat` or `/api/generate`, or to OpenAI-compatible `/v1/chat/completions` or `/v1/completions`.
2. Middleware maps compatible schemas into Ollama's internal API structs where needed.
3. The handler schedules/loads the model.
4. Prompt rendering uses the model template unless raw mode or compatible middleware alters behavior.
5. Ollama sends the formatted completion/chat request to the runner at `http://127.0.0.1:<runner-port>/completion` or `/v1/chat/completions`.
6. The runner streams tokens back; Ollama transforms them into Ollama/OpenAI/Anthropic response formats and streams to the client.

If the requested model has a cloud suffix and the route is cloud-enabled, `cloudPassthroughMiddleware` can proxy the request to Ollama Cloud instead of local inference.

## Configuration system

Primary configuration comes from environment variables read through `envconfig.Var` and exposed in `envconfig.AsMap`. Important variables include:

- `OLLAMA_HOST`: server/client host, default `http://127.0.0.1:11434`.
- `OLLAMA_MODELS`: model storage root, default `$HOME/.ollama/models`.
- `OLLAMA_ORIGINS`: extra browser CORS origins.
- `OLLAMA_KEEP_ALIVE`: loaded model retention, default 5 minutes.
- `OLLAMA_CONTEXT_LENGTH`: default context length when not explicitly set.
- `OLLAMA_NUM_PARALLEL`: parallel requests per model, default 1.
- `OLLAMA_MAX_QUEUE`: pending request queue size, default 512.
- `OLLAMA_MAX_LOADED_MODELS`: cap on loaded models.
- `OLLAMA_LLM_LIBRARY`: override backend selection.
- `OLLAMA_FLASH_ATTENTION`, `OLLAMA_KV_CACHE_TYPE`, `OLLAMA_GPU_OVERHEAD`, `OLLAMA_SCHED_SPREAD`.
- `CUDA_VISIBLE_DEVICES`, `HIP_VISIBLE_DEVICES`, `ROCR_VISIBLE_DEVICES`, `GGML_VK_VISIBLE_DEVICES`.
- `OLLAMA_NO_CLOUD`: disables cloud features.
- `OLLAMA_NOPRUNE`: disables startup/blob pruning.

Cloud-disable can also come from `~/.ollama/server.json` using `{"disable_ollama_cloud": true}`.

## Model registry/storage system

Ollama's local model store resembles an OCI/Docker registry layout:

```text
$OLLAMA_MODELS/
  blobs/
    sha256-<64 hex chars>
  manifests/
    <host>/<namespace>/<model>/<tag>
```

Manifests contain schema version, media type, config layer, and layers. Layers hold media type, digest, size, optional source model, and optional name. Blob paths only accept `sha256:<hex>` or `sha256-<hex>` digests, reducing path traversal risk for content-addressed data.

Model names default to:

- host: `registry.ollama.ai`
- namespace: `library`
- tag: `latest`
- protocol: `https`

Thus `llama3.2` becomes roughly `https://registry.ollama.ai/library/llama3.2:latest` for pull/storage purposes.

## Runtime/backend selection

The scheduler uses discovered devices to choose placement. For GGUF models, `llama-server` is always the runtime. Ollama configures dynamic library paths and GPU visibility variables for the subprocess. Device discovery covers CPU plus CUDA/ROCm/Vulkan/Metal where supported. The scheduler generally prefers a single discrete GPU when the predicted model memory fits; otherwise it chooses a GPU group by available memory, unless `OLLAMA_SCHED_SPREAD` changes behavior or the request explicitly sets `main_gpu`/`num_gpu`.

`num_gpu` behavior:

- `-1` default: let `llama-server` auto-detect offload.
- `0`: explicit CPU-only (`-ngl 0`).
- positive value: explicit GPU layer offload count (`-ngl N`).

## GPU/CPU execution path

- GPU discovery happens at server startup and before/around scheduling through `discover.GPUDevices` and `ml.DeviceInfo`.
- Scheduler predicts VRAM for model, context, and batch sizing, logs system/GPU memory, and spawns the runner.
- Runner launch sets `PATH` on Windows, `LD_LIBRARY_PATH` on Linux, or `DYLD_LIBRARY_PATH` on macOS so `llama-server` can find CPU/GPU backend libraries.
- CPU-only can be requested by setting `num_gpu: 0` in options.
- Partial GPU offload is controlled by `llama-server` when `num_gpu` is default; explicit positive `num_gpu` can force a partial layer count.

## How Ollama exposes its local API

By default, `envconfig.Host()` returns `http://127.0.0.1:11434`; `RunServer` listens on that host/port. External clients can talk directly to this local API. OpenAI-compatible clients commonly use:

```text
Base URL: http://127.0.0.1:11434/v1
API key: any placeholder unless the client insists on one
```

Ollama itself does not require an API key for default local API use in the code paths reviewed. The client library only signs local requests when `OLLAMA_AUTH` is set, but this analysis did not find a corresponding general server-side auth middleware applied to local routes. Treat the local HTTP API as unauthenticated unless you have verified otherwise in your exact build.

## How Open WebUI, OpenClaw, and OpenHands connect

Open WebUI and similar web clients should point their Ollama/OpenAI-compatible backend URL at `http://host.docker.internal:11434` from Docker on Windows/macOS or `http://127.0.0.1:11434` when running on the same host namespace. The safest pattern is to keep Ollama bound to loopback and publish only the UI port you actually need.

OpenClaw and agent tools can use Ollama's OpenAI-compatible endpoints at `/v1`. This repository includes launch helpers that set provider/base URL environment variables for several tools. For example, OpenClaw launch code starts/configures OpenClaw with Ollama model and provider settings; OpenCode/Copilot helpers set base URLs ending in `/v1`.

For OpenHands, use its OpenAI-compatible/custom LLM configuration pointed at `http://127.0.0.1:11434/v1` or, if OpenHands runs in a container, the host gateway address. The same network warning applies: do not bind Ollama to `0.0.0.0` merely to make a container work unless you firewall it.
