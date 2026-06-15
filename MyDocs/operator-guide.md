# Operator guide: Windows 11 with NVIDIA RTX 4060 8 GB

## Executive summary

- **What Ollama does:** It manages model downloads/storage, exposes a local HTTP API, launches a local runtime, and serves LLM responses.
- **Architecture:** CLI/API server in Go, local model store under `.ollama`, scheduler/GPU discovery, and a private `llama-server` subprocess for GGUF models.
- **Local-only vs not:** Local inference with downloaded models can be fully local. Pull/push/sign-in/cloud models/model recommendations require network or Ollama-hosted services.
- **Main security risks:** Accidental network binding, unauthenticated local API access, model supply chain, prompt leakage through cloud/integrations, and DoS from large models/context.
- **Safe Windows + RTX 4060 posture:** Keep `OLLAMA_HOST=127.0.0.1:11434`, set `OLLAMA_NO_CLOUD=1` if you do not want cloud features, firewall inbound 11434, prefer 7B/8B Q4/Q5 models at 4K-8K context, and monitor VRAM.
- **Open WebUI/OpenClaw/OpenHands fit:** They should connect to Ollama's OpenAI-compatible local API at `http://127.0.0.1:11434/v1` or the Docker host gateway equivalent. Ollama supplies inference; agent tools still need their own sandbox/network controls.
- **Next Codex task:** Perform a focused dynamic validation: run Ollama on Windows-like settings, capture listening sockets and outbound connections during startup, local inference, pull, cloud-disabled mode, Open WebUI connection, and a cloud-suffixed request.

## Safest local installation assumptions

Assume:

- You are a single Windows user running Ollama for personal local inference.
- You do not want your prompts to leave the machine unless you explicitly choose cloud/network features.
- Your RTX 4060 has 8 GB VRAM.
- You want frontends/agents to connect locally, not expose Ollama to your LAN.

Use a normal user account, not Administrator, for daily operation. Keep NVIDIA drivers current. Install Ollama from the official installer or build from trusted source. Treat model files as third-party artifacts.

## Run Ollama locally only

Set user environment variables:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "127.0.0.1:11434", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_NO_CLOUD", "1", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "4096", "User")
```

Restart Ollama after changing environment variables.

Notes:

- `OLLAMA_NO_CLOUD=1` disables Ollama Cloud features such as remote inference and web search paths controlled by `internal/cloud`.
- It does **not** prevent model pulls from the registry. Pulling models still requires network access.
- It does **not** control what Open WebUI/OpenClaw/OpenHands do independently.

Alternative config file:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ollama" | Out-Null
'{"disable_ollama_cloud": true}' | Set-Content "$env:USERPROFILE\.ollama\server.json" -Encoding utf8
```

## Prevent accidental public exposure

Do:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "127.0.0.1:11434", "User")
```

Do not do this unless you have a secured reverse proxy/VPN/firewall design:

```powershell
# Dangerous: exposes Ollama on all interfaces
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11434", "User")
```

Add a Windows Firewall block rule for inbound TCP 11434 as defense-in-depth:

```powershell
New-NetFirewallRule -DisplayName "Block inbound Ollama 11434" -Direction Inbound -Action Block -Protocol TCP -LocalPort 11434
```

If Windows asks whether to allow Ollama through the firewall on public/private networks, deny unless you intentionally designed remote access.

## Verify host/port listening

PowerShell:

```powershell
Get-NetTCPConnection -LocalPort 11434 -State Listen | Select-Object LocalAddress,LocalPort,OwningProcess
```

Expected safe output:

```text
LocalAddress LocalPort OwningProcess
------------ --------- -------------
127.0.0.1        11434 <pid>
```

Also check with:

```powershell
netstat -ano | findstr :11434
```

Danger signs:

- `0.0.0.0:11434`
- `[::]:11434`
- your LAN IP, e.g. `192.168.x.y:11434`

Verify API:

```powershell
curl.exe http://127.0.0.1:11434/api/version
curl.exe http://127.0.0.1:11434/api/tags
```

## Choose models for 8 GB VRAM

Practical starting points:

| Model class | Quantization | Expected RTX 4060 8 GB fit | Notes |
|---|---|---|---|
| 3B-4B | Q4/Q5/Q8 | Comfortable | Good for tool routing, small assistants, coding helpers with lower quality needs. |
| 7B-8B | Q4 | Comfortable to moderate | Best default for 8 GB. Leaves room for context/KV cache. |
| 7B-8B | Q5 | Moderate | Often good balance; reduce context if VRAM is tight. |
| 7B-8B | Q8 | Tight | May spill/offload or require lower context. |
| 13B-14B | Q4 | Tight/partial | Usually slower and may use CPU/RAM offload on 8 GB. |
| 30B+ | Any | Not recommended locally on 8 GB | Expect heavy CPU/RAM offload or failure. |

Start with:

```powershell
ollama pull llama3.2:3b
ollama pull qwen2.5-coder:7b
ollama pull mistral:7b
```

Exact tags available can change over time; list models from Ollama's library or your registry when choosing.

## Compare Q4/Q5/Q8 models

General trade-off:

- **Q4:** smallest, fastest, lowest VRAM, most quality loss.
- **Q5:** balanced; often worth it for 7B/8B on 8 GB.
- **Q8:** better fidelity but much larger; can reduce speed and fit.

Test method:

```powershell
ollama run model-q4 "Summarize this repository architecture in 10 bullets."
ollama ps
ollama run model-q5 "Summarize this repository architecture in 10 bullets."
ollama ps
ollama run model-q8 "Summarize this repository architecture in 10 bullets."
ollama ps
```

Compare:

- VRAM from `ollama ps`.
- Windows Task Manager GPU memory.
- Response latency and tokens/sec if verbose mode is available.
- Quality on your real prompts.

## Monitor RAM/VRAM usage

Ollama:

```powershell
ollama ps
```

NVIDIA:

```powershell
nvidia-smi
nvidia-smi -l 1
```

PowerShell process view:

```powershell
Get-Process | Where-Object {$_.ProcessName -match "ollama|llama"} | Select-Object ProcessName,Id,CPU,WorkingSet64
```

Windows GUI:

- Task Manager -> Performance -> GPU -> Dedicated GPU memory.
- Task Manager -> Details -> add columns for GPU/GPU memory if useful.
- Resource Monitor -> Network tab for outbound connections.

## Control memory pressure

Recommended environment for 8 GB VRAM:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "4096", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_NUM_PARALLEL", "1", "User")
[Environment]::SetEnvironmentVariable("OLLAMA_MAX_LOADED_MODELS", "1", "User")
```

Use request-level options when testing:

```powershell
curl.exe http://127.0.0.1:11434/api/chat `
  -H "Content-Type: application/json" `
  -d '{"model":"qwen2.5-coder:7b","messages":[{"role":"user","content":"hello"}],"options":{"num_ctx":4096}}'
```

If a model is too slow or spills to RAM:

1. Stop it: `ollama stop <model>`.
2. Lower context (`num_ctx` or `OLLAMA_CONTEXT_LENGTH`).
3. Use Q4 instead of Q5/Q8.
4. Use a smaller parameter model.
5. Ensure only one loaded model with `OLLAMA_MAX_LOADED_MODELS=1`.

## Clean up model storage

List models:

```powershell
ollama list
```

Remove a model:

```powershell
ollama rm <model>
```

Find storage size:

```powershell
Get-ChildItem "$env:USERPROFILE\.ollama\models" -Recurse | Measure-Object -Property Length -Sum
```

Inspect large blobs:

```powershell
Get-ChildItem "$env:USERPROFILE\.ollama\models\blobs" | Sort-Object Length -Descending | Select-Object -First 20 Name,Length
```

Ollama prunes unused layers on startup by default. Do not set `OLLAMA_NOPRUNE=1` unless you understand the disk growth trade-off.

## Connect Open WebUI safely

### If Open WebUI runs directly on Windows

Set Ollama URL in Open WebUI to:

```text
http://127.0.0.1:11434
```

For OpenAI-compatible configuration:

```text
Base URL: http://127.0.0.1:11434/v1
API key: local-placeholder
```

### If Open WebUI runs in Docker Desktop

Common Docker host URL:

```text
http://host.docker.internal:11434
```

Keep Ollama bound to `127.0.0.1`. Do **not** switch Ollama to `0.0.0.0` unless Docker networking absolutely requires it and you add firewall controls.

Publish only the Open WebUI port and put authentication on the UI. Avoid publishing Ollama's port.

## Connect OpenClaw safely

Use Ollama's local OpenAI-compatible endpoint:

```text
Base URL: http://127.0.0.1:11434/v1
Model: your-local-model, e.g. qwen2.5-coder:7b
API key: placeholder if required
```

Safety controls:

- Keep `OLLAMA_NO_CLOUD=1` if you do not want cloud fallback.
- Avoid `:cloud` models.
- Run OpenClaw in a dedicated workspace.
- Keep approval prompts enabled.
- Avoid YOLO/autonomous modes until you trust the setup.
- Restrict outbound network if you are testing prompt locality.

The repository includes OpenClaw launch helper code, but OpenClaw itself is an external agent application with its own security model.

## Connect OpenHands safely

Configure OpenHands as an OpenAI-compatible/custom provider:

```text
Base URL: http://127.0.0.1:11434/v1
Model: qwen2.5-coder:7b or another local model
API key: placeholder if required
```

If OpenHands runs in Docker, use the host gateway address (`host.docker.internal` on Docker Desktop) and keep Ollama loopback/firewalled.

OpenHands can execute code and access files/network depending on configuration. Ollama does not sandbox OpenHands actions. Use a disposable workspace/container and do not mount sensitive directories unless required.

## Verify prompts are not leaving the machine during local inference

### Static configuration checks

```powershell
[Environment]::GetEnvironmentVariable("OLLAMA_HOST", "User")
[Environment]::GetEnvironmentVariable("OLLAMA_NO_CLOUD", "User")
Get-Content "$env:USERPROFILE\.ollama\server.json" -ErrorAction SilentlyContinue
```

Expected:

```text
OLLAMA_HOST = 127.0.0.1:11434
OLLAMA_NO_CLOUD = 1
server.json contains disable_ollama_cloud true, or env var is set
```

Check model names:

```powershell
ollama list
```

Avoid names ending in `:cloud` or `-cloud` for local-only runs.

### Network observation

Open one terminal:

```powershell
netstat -ano 1 | findstr /i "ollama llama 11434 https established"
```

Or use Resource Monitor:

1. Start -> Resource Monitor.
2. Network tab.
3. Watch `ollama.exe`, `ollama app.exe`, and `llama-server`/runner process.
4. Run a local prompt.
5. Confirm no new remote connections are created during inference.

Firewall test:

1. Pull a model while online.
2. Set `OLLAMA_NO_CLOUD=1`.
3. Block outbound network for Ollama in Windows Firewall.
4. Run a local prompt.
5. It should still work if the model is local.
6. `ollama pull`, sign-in, cloud models, and recommendations should fail or be disabled.

### API-level check

Run a request to a known local model:

```powershell
curl.exe http://127.0.0.1:11434/api/chat `
  -H "Content-Type: application/json" `
  -d '{"model":"qwen2.5-coder:7b","messages":[{"role":"user","content":"Say local only."}],"stream":false}'
```

Then test a cloud-suffixed model while cloud is disabled:

```powershell
curl.exe http://127.0.0.1:11434/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d '{"model":"some-model:cloud","messages":[{"role":"user","content":"Should not run."}]}'
```

Expected with cloud disabled: an error indicating cloud features/remote inference are unavailable.

## Recommended daily workflow

1. Keep Ollama local/firewalled.
2. Pull models intentionally.
3. Disconnect/block outbound if validating privacy.
4. Run local UI/agent against `http://127.0.0.1:11434/v1`.
5. Use Q4/Q5 7B/8B models at 4096 context.
6. Monitor `ollama ps` and `nvidia-smi`.
7. Remove unused models with `ollama rm`.
8. Avoid cloud-suffixed models unless you explicitly want hosted inference.

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Open WebUI cannot reach Ollama in Docker | Container cannot resolve host loopback | Use `http://host.docker.internal:11434`; keep Ollama loopback. |
| Model loads but is very slow | CPU/RAM offload or too large context | Use smaller/Q4 model, lower `num_ctx`, stop other models. |
| Out-of-memory on RTX 4060 | Model/quant/context too large | Q4, 4K context, `OLLAMA_MAX_LOADED_MODELS=1`, `OLLAMA_NUM_PARALLEL=1`. |
| `ollama pull` fails offline | Registry needs network | Pull while online or import local GGUF. |
| Prompts unexpectedly leave machine | Cloud model/integration/web tool/proxy | Set `OLLAMA_NO_CLOUD=1`, avoid cloud suffixes, block outbound, audit frontend. |
| Port exposed on LAN | `OLLAMA_HOST` set to non-loopback | Reset to `127.0.0.1:11434`, restart, firewall. |

## Next Codex task recommendation

Run a dynamic network and process audit with these phases:

1. Clean startup with `OLLAMA_HOST=127.0.0.1:11434` and `OLLAMA_NO_CLOUD=1`.
2. Capture listening sockets.
3. Run local inference with a pre-pulled model while collecting outbound connection logs.
4. Attempt `:cloud` inference and confirm it is blocked.
5. Pull a model and document registry endpoints contacted.
6. Connect Open WebUI/OpenClaw/OpenHands and confirm only local `/v1` requests hit Ollama.
7. Produce a reproducible Windows PowerShell script for these checks.
