# Ollama security map

## Summary

Ollama's default posture is acceptable for a single-user local workstation **only if it remains bound to loopback**. The major security risks are unauthenticated local HTTP access, accidental binding to the LAN/public interfaces, cloud/model-registry supply-chain trust, prompt leakage through cloud/proxy/integrations, and local resource exhaustion from large models, high context, or parallel requests.

## Security-relevant areas

| Area | Affected files | Risk description | Realistic misuse scenario | Mitigation | Investigate next |
|---|---|---|---|---|---|
| Local API exposure | `envconfig/config.go`, `cmd/cmd.go`, `server/routes.go`, `api/client.go` | Server listens on `OLLAMA_HOST`, default loopback. Routes include model management and inference. Default local use appears unauthenticated in reviewed server paths. | Malware or another local user posts to `/api/chat`, pulls/deletes models, or reads model list. | Keep loopback binding; use OS user isolation; set firewall rules; consider separate Windows account/VM for untrusted tools. | Verify whether `OLLAMA_AUTH` has server-side enforcement in your exact release/build. |
| Host/port binding | `envconfig/config.go`, `cmd/cmd.go` | `OLLAMA_HOST=0.0.0.0:11434` or LAN IP exposes the API beyond the local host. | A device on the same Wi-Fi sends requests to your Ollama server and uses your GPU or deletes/pulls models. | Use `OLLAMA_HOST=127.0.0.1:11434`; block inbound TCP 11434 in Windows Firewall; never port-forward 11434. | Confirm effective listener with `netstat`/PowerShell after service startup. |
| Allowed Host/CORS checks | `server/routes.go`, `envconfig/config.go` | Host header filtering protects loopback listeners from some DNS-rebinding/browser attacks. When bound non-loopback, middleware allows requests through. CORS allows local/app/file/vscode origins plus `OLLAMA_ORIGINS`. | A browser app on the same host or allowed origin sends local API calls; if LAN-bound, remote clients can call directly. | Keep loopback; restrict `OLLAMA_ORIGINS`; do not add broad origins; avoid LAN binding. | Test browser-based access paths used by Open WebUI in your deployment. |
| Unauthenticated local access | `server/routes.go`, `api/client.go`, `auth/auth.go`, `envconfig/config.go` | Client can sign local requests when `OLLAMA_AUTH` is set, but this analysis did not find a general auth middleware applied to local routes. | Any process that can connect to the socket can invoke inference/model-management APIs. | Treat local port as privileged; firewall; do not expose to shared machines; front with an authenticated reverse proxy only if remote access is unavoidable. | Search release branch for auth middleware or service-manager ACLs. |
| Remote network exposure | `cmd/cmd.go`, `server/routes.go` | Remote exposure combines unauthenticated API, model management, and high compute cost. | Attacker on LAN triggers large pulls, long contexts, or repeated prompts, causing data usage and GPU DoS. | Bind loopback; firewall inbound; if remote UI needed, use VPN/SSH tunnel/reverse proxy with auth and rate limits. | Add network monitoring and access logs at proxy if intentionally exposed. |
| Model download/supply chain | `server/images.go`, `server/download.go`, `server/auth.go`, `manifest/paths.go` | Pull trusts registry manifests and blobs. Blob integrity is SHA-256 verified, but model provenance/license/safety remain human trust issues. `--insecure` permits HTTP registries. | Malicious or compromised registry/model provides weights with unsafe behavior, poisoned prompts/templates, or huge resource requirements. | Pull from trusted registries/authors; avoid `--insecure`; pin tags/digests when possible; inspect `ollama show --modelfile`; use firewall/offline mode after download. | Signature/provenance support beyond digest verification; policy controls for registries. |
| Model file trust boundaries | `parser/parser.go`, `server/create.go`, `fs/ggml`, `convert/*`, `llm/llama_server.go` | Local GGUF/safetensors files are parsed and may be converted/loaded by native code. Parser bugs in model formats or backend libraries can be security-relevant. | Opening a malicious GGUF triggers a parser/backend bug or consumes excessive memory. | Only import trusted files; keep Ollama updated; isolate untrusted model testing in a VM/container; avoid running as admin. | Fuzzing/history of GGUF/safetensors parsers and llama.cpp backend. |
| Prompt/data leakage | `server/cloud_proxy.go`, `server/model_recommendations.go`, `server/routes.go`, integrations in `cmd/launch/*` | Normal local inference stays local, but cloud model suffixes and some experimental endpoints can send prompts externally. Integrations may have their own cloud/network behavior. | User accidentally selects `model:cloud` in an OpenAI-compatible client; prompts are proxied to Ollama Cloud. Agent tool uses web/search/shell and leaks data. | Set `OLLAMA_NO_CLOUD=1`; avoid cloud-suffixed models; block Ollama outbound when validating; configure integrations for local-only models; disable web-search tools. | Audit each integration's network/tool settings separately. |
| Environment/config risks | `envconfig/config.go`, `llm/server.go`, `llm/llama_server.go`, `discover/*` | Env vars control binding, model root, origins, GPU selection, library path, cloud enablement, proxy settings, context, queue, debug logging. | A wrapper sets `OLLAMA_HOST=0.0.0.0`; `HTTPS_PROXY` sends registry/cloud traffic through an unwanted proxy; `OLLAMA_DEBUG_LOG_REQUESTS=1` stores prompts. | Use explicit service environment; inspect `ollama serve` logs; clear proxy env vars; avoid debug request logging unless needed; protect `~/.ollama`. | Windows service/app environment persistence. |
| Filesystem access paths | `envconfig/config.go`, `manifest/paths.go`, `manifest/layer.go`, `parser/parser.go`, `server/model_recommendations.go`, `auth/auth.go` | Ollama writes blobs/manifests, cache snapshots, request logs if enabled, and keys under `~/.ollama`. `OLLAMA_MODELS` redirects storage. | Models fill disk; prompt logs remain in temp; keys or models readable by other users if permissions are weak. | Store models on a dedicated volume; monitor disk; keep user profile private; disable request logging; check ACLs on `.ollama`. | Exact Windows ACLs applied by installer/app. |
| GPU/runtime risks | `discover/*`, `ml/device.go`, `server/sched.go`, `llm/llama_server.go` | Native GPU libraries and `llama-server` process are complex native code; OOM/stalls can destabilize sessions. | Malicious/buggy model or huge context crashes runner/GPU driver or degrades system responsiveness. | Keep NVIDIA drivers updated; avoid admin privileges; cap queue/parallel/context; use smaller quantizations; watch Event Viewer and Ollama logs. | Backend CVEs and crash isolation boundaries. |
| Integration risks: Open WebUI | Not in repo; Ollama side in `server/routes.go` and `middleware/openai.go` | WebUI often runs in Docker and may need host access to 11434. Publishing Ollama broadly for container access creates exposure. | Docker compose maps Ollama to `0.0.0.0`; anyone on LAN can use it. | Keep Ollama loopback; have Open WebUI connect via host gateway; publish only WebUI behind auth; do not publish 11434. | WebUI auth, user management, model permission settings. |
| Integration risks: OpenClaw/OpenHands/agents | `cmd/launch/openclaw.go`, other `cmd/launch/*`; external projects | Agent tools can execute shell/browser/file actions. Ollama only supplies model inference. | Agent uses local model but executes destructive commands or leaks files through tool/network actions. | Run agents in isolated workspaces/containers; require approvals; disable YOLO modes; keep Ollama local; restrict outbound network. | Tool-specific sandboxing and approval behavior. |
| Denial of service | `envconfig/config.go`, `server/sched.go`, `llm/llama_server.go`, `server/download.go` | Large models, large `num_ctx`, high `num_parallel`, large queues, and pulls can consume disk/RAM/VRAM/network. | Local webpage or LAN attacker submits repeated high-context requests, causing GPU saturation and disk fill. | `OLLAMA_MAX_QUEUE`, `OLLAMA_MAX_LOADED_MODELS`, moderate context, firewall, monitor disk, avoid LAN binding. | Per-route auth/rate limiting if exposed through proxy. |
| Debug request logging | `envconfig/config.go`, `server/inference_request_log.go` | `OLLAMA_DEBUG_LOG_REQUESTS` logs inference request bodies/replay commands to disk. | Sensitive prompts are stored and later read from temp/log directory. | Keep disabled unless actively debugging; delete logs; avoid sensitive prompts during debug. | Exact temp path and retention behavior for your OS. |
| pprof exposure | `server/routes.go` | Server uses `http.DefaultServeMux` and comments note pprof is available by default. If API is remotely exposed, debugging endpoints may also be exposed. | Remote user queries pprof data when Ollama is LAN-bound. | Loopback-only; firewall; do not expose default mux publicly. | Confirm which pprof endpoints are registered in production builds. |

## Local API exposure details

Default:

```text
OLLAMA_HOST unset -> http://127.0.0.1:11434
```

Dangerous examples:

```text
OLLAMA_HOST=0.0.0.0:11434
OLLAMA_HOST=<LAN-IP>:11434
```

Loopback binding means only local processes can connect. Non-loopback binding turns Ollama into a network server. Because the same server exposes pull, delete, create, and inference endpoints, network exposure is high impact even without arbitrary code execution.

## Supply-chain trust boundary

Digest verification proves that a blob matches the digest in the manifest. It does not prove that:

- the author is trustworthy,
- the manifest was intended by the author,
- the tag has not changed,
- the model template/system prompt is safe,
- the license permits your use,
- the model will not trigger parser/runtime bugs,
- the model will fit your hardware safely.

Treat pulled model weights and Modelfile content like third-party executable-adjacent artifacts.

## Prompt/data leakage boundaries

Prompts are local for normal local models. They may leave the machine if:

- the model name is cloud-suffixed and cloud proxy is enabled,
- you use experimental web/fetch/search endpoints,
- a frontend/agent logs or uploads prompts independently,
- `OLLAMA_DEBUG_LOG_REQUESTS` writes prompts to disk,
- you configure a remote `OLLAMA_HOST` or remote OpenAI-compatible backend.

## Recommended safe defaults

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "127.0.0.1:11434", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_NO_CLOUD", "1", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_ORIGINS", "http://localhost:3000,http://127.0.0.1:3000", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "4096", "User")
```

Adjust the UI origin if Open WebUI runs on a different local port. Do not set `OLLAMA_HOST=0.0.0.0` unless you have a firewall/reverse-proxy design with authentication.
