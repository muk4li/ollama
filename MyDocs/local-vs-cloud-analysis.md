# Ollama local vs cloud analysis

## Bottom line

Ollama can perform core model inference fully locally after the model files are present on disk. However, the repository contains several intentional internet/cloud paths:

- Pulling models from the default registry requires internet access.
- Pushing models requires registry access.
- Signing in/out and cloud-model access use Ollama-hosted services.
- Explicit `:cloud` model requests can be proxied to Ollama Cloud.
- Model recommendations are refreshed from `https://ollama.com/api/experimental/model-recommendations` unless cloud is disabled.
- Experimental web search/fetch are not proven local-only from this pass and should be treated as network-capable.

Setting `OLLAMA_NO_CLOUD=1` or `~/.ollama/server.json` with `{"disable_ollama_cloud": true}` disables Ollama Cloud features, but it does **not** make all networking impossible: registry pulls/pushes and any explicitly networked integrations/tools can still use the internet.

## Evidence by code path

### Local-only inference path

For a normal local model name without a cloud suffix:

1. HTTP request reaches `server/routes.go` inference handler.
2. `scheduleRunner` resolves local model metadata with `GetModel`.
3. Scheduler loads local files through `llm.LoadModel`.
4. `llama-server` is launched with `--offline` and `--host 127.0.0.1`.
5. Ollama calls runner endpoints on `http://127.0.0.1:<port>`.

Evidence files:

- `server/routes.go`: local inference handlers and `scheduleRunner`.
- `server/sched.go`: scheduler load path.
- `llm/server.go`: `NewLlamaServer` for GGML/GGUF models.
- `llm/llama_server.go`: runner launch flags and localhost runner HTTP calls.
- `manifest/paths.go`: local model paths.

Conclusion: inference for already-downloaded local GGUF models does not require external services in the code path reviewed.

### Model download path

`PullModel` builds registry URLs from model name defaults and calls:

```text
GET /v2/<namespace>/<model>/manifests/<tag>
GET /v2/<namespace>/<model>/blobs/<digest>
```

Short names default to `registry.ollama.ai/library/<model>:latest`. Downloads are verified by SHA-256 digest before the manifest is written.

Evidence files:

- `types/model/name.go`: default host/protocol/namespace/tag.
- `server/images.go`: `PullModel`, `pullModelManifest`, `makeRequestWithRetry`, `verifyBlob`.
- `server/download.go`: blob download/resume/range logic.

Conclusion: downloading from the public model registry requires internet access and external registry services unless using a local/private registry reference or preexisting local files.

### Cloud model path

OpenAI-compatible and related routes include `cloudPassthroughMiddleware`. It reads the request body, extracts `model`, parses it, and if the model source is cloud, proxies to a cloud base URL. Default cloud proxy base URL is `https://ollama.com:443`. Requests to the default cloud signing host are signed with the user's local Ollama key.

Evidence files:

- `server/routes.go`: cloud middleware attached to `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/responses`, `/v1/images/*`, `/v1/messages`, and `/v1/models/:model` details path.
- `server/cloud_proxy.go`: cloud proxy base URL, request body inspection, proxying, signing, cloud disable errors.
- `auth/auth.go`: local key signing.
- `internal/cloud/cloud.go`: cloud disable status.

Conclusion: explicitly cloud-suffixed model usage can leave the machine and use Ollama-hosted inference. This differs from local inference because prompts/request bodies are proxied to Ollama Cloud.

### Model recommendations path

The model recommendations cache starts at server startup and refreshes from `https://ollama.com/api/experimental/model-recommendations`, with a local snapshot under `~/.ollama/cache/model-recommendations.json`. It short-circuits if `envconfig.NoCloud()` is true.

Evidence files:

- `server/model_recommendations.go`: URL, background cache, refresh logic, snapshot path.
- `server/routes.go`: model cache startup and recommendations endpoint.
- `envconfig/config.go`: `OLLAMA_NO_CLOUD` and `server.json` disable logic.

Conclusion: by default, this is an outbound network check to Ollama-hosted services. It is disabled by `OLLAMA_NO_CLOUD`/server config.

### Sign-in/out and registry auth

Ollama creates an Ed25519 keypair under `~/.ollama` and signs challenges. Cloud sign-in URL generation points at Ollama-hosted sign-in. Registry auth follows `WWW-Authenticate` challenges but validates that token realm host equals the original registry host before requesting a token.

Evidence files:

- `cmd/cmd.go`: `initializeKeypair`, sign-in/sign-out command wiring.
- `server/routes.go`: `/api/me`, `/api/signout`, `signinURL`.
- `server/auth.go`: registry challenge URL and same-host validation.
- `auth/auth.go`: key load/sign/public-key functions.

Conclusion: authentication/sign-in uses external services when interacting with Ollama Cloud/registry auth. Local model inference does not need sign-in.

### Update checks and telemetry

This pass found no obvious general telemetry pipeline or automatic update-check code in the prioritized server/model/runtime paths. There are outbound calls for model recommendations, cloud proxy, registry transfer, and sign-in/auth. Uncertainty remains because the repository is large and platform app code may contain update behavior outside the analyzed server/CLI paths.

## Feature table

| Feature | Local Only | Internet Required | Optional Cloud | Evidence | Notes |
|---|---:|---:|---:|---|---|
| Run already-downloaded GGUF model through `/api/chat` or `/api/generate` | Yes | No | No | `server/routes.go`, `server/sched.go`, `llm/llama_server.go` | Runner is started locally with `--offline` and localhost-only runner binding. |
| OpenAI-compatible local inference at `/v1/chat/completions` with a local model | Yes | No | No | `server/routes.go`, `middleware/openai.go` | Cloud middleware passes through when model is not cloud-sourced. |
| Embeddings with a local model | Yes | No | No | `server/routes.go`, `llm/llama_server.go` | Local runner `/v1/embeddings` endpoint is used. |
| Model pull from default short name | No | Yes | No | `types/model/name.go`, `server/images.go`, `server/download.go` | Defaults to `registry.ollama.ai`; requires registry availability. |
| Model pull from local/private LAN registry | No | Network maybe | No | `types/model/name.go`, `server/images.go` | Internet not required if the registry host is local, but networking is still used. |
| Create model from local GGUF file | Yes | No | No | `parser/parser.go`, `server/create.go`, `manifest/layer.go` | Source-backed for local files. Derived base models may pull if not already present. |
| Create model `FROM` missing local model | No | Maybe | No | `server/model.go` | `parseFromModel` pulls if manifest is missing. |
| Store/list/show/delete local models | Yes | No | No | `manifest/*`, `server/routes.go` | Operates on local manifests/blobs. |
| Push model | No | Yes | Maybe | `server/images.go`, `server/upload.go` | Uploads to a registry endpoint. |
| Cloud model inference (`:cloud` / `-cloud`) | No | Yes | Yes | `server/cloud_proxy.go`, `server/routes.go` | Proxies request to Ollama Cloud by default. |
| Model recommendations | No | Yes by default | Yes | `server/model_recommendations.go` | Disabled by `OLLAMA_NO_CLOUD` or server config. |
| Sign in/out / `/api/me` | No | Yes | Yes | `server/routes.go`, `auth/auth.go` | Uses Ollama-hosted sign-in/user endpoints. |
| Registry auth token retrieval | No | Yes when challenged | No/registry | `server/auth.go`, `server/images.go` | Contacts auth realm on same registry host. |
| Experimental web search/fetch | No | Likely | Maybe | `server/routes.go` | Network behavior should be separately audited before use. |
| API heartbeat/version/status | Yes | No | No | `server/routes.go`, `api/client.go` | Local HTTP only. |
| GPU discovery | Yes | No | No | `discover/*`, `ml/device.go` | Uses local OS/runtime probes and runner probes. |
| Update checks | Unproven | Unproven | Unproven | No prioritized-path evidence found | Investigate platform app/updater code if high assurance is required. |
| Telemetry | Unproven absent | Unproven | Unproven | No prioritized-path evidence found | This analysis found no general telemetry pipeline in reviewed paths, but absence is not fully proven. |

## How to force local-only operation as much as Ollama supports

1. Keep `OLLAMA_HOST=127.0.0.1:11434`.
2. Set `OLLAMA_NO_CLOUD=1`.
3. Avoid model names ending in `:cloud` or `-cloud`.
4. Avoid `/api/pull`, `/api/push`, sign-in/out, model recommendations, and experimental web endpoints unless network use is intended.
5. Download models ahead of time, then block the Ollama process with Windows Firewall and test local inference.
6. Use packet capture or Resource Monitor to verify no outbound network connections while running a local model.

## Explicit uncertainties

- This pass prioritized server startup, API exposure, model storage, model download, inference runtime, config/env handling, network behavior, and security boundaries. It did not exhaustively audit every app/updater/platform integration file.
- I did not prove the absence of telemetry in packaged desktop app code or installer/updater code.
- Experimental web search/fetch internals should be separately audited before treating them as safe/local.
- Open WebUI/OpenClaw/OpenHands themselves can perform network/tool actions independently of Ollama; this repository can only prove Ollama's side of the connection.
