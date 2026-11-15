# Ollama model storage and runtime

## Where models are stored

By default, models are stored under:

```text
$HOME/.ollama/models
```

The root can be changed with `OLLAMA_MODELS`. Under that root, the important directories are:

```text
models/
  blobs/
    sha256-<digest>
  manifests/
    <host>/<namespace>/<model>/<tag>
```

Evidence:

- `envconfig.Models()` returns `OLLAMA_MODELS` or `$HOME/.ollama/models`.
- `manifest.Path()` appends `manifests` to the model root.
- `manifest.BlobsPath()` appends `blobs/sha256-...` to the model root.
- `manifest.PathForName()` maps a qualified model name to `manifests/<host>/<namespace>/<model>/<tag>`.

## How models are named and versioned

Ollama model names are structured as:

```text
<host>/<namespace>/<model>:<tag>
```

Short forms are merged with defaults:

| Field | Default |
|---|---|
| host | `registry.ollama.ai` |
| namespace | `library` |
| tag | `latest` |
| protocol | `https` |

The tag is the version-like selector in normal use. `model:tag` becomes a manifest file under the host/namespace/model/tag path. The model name parser validates allowed characters and lengths for host, namespace, model, and tag.

## How model metadata is represented

Local metadata is layered:

1. A manifest JSON file with schema version, Docker manifest media type, one config layer, and model layers.
2. Content-addressed blobs referenced by each layer's SHA-256 digest.
3. Layer media types for model weights, projectors, adapters, templates, params, license, messages, image tensors, and draft models.
4. Model configuration (`model.ConfigV2`) assembled by server model loading code from the manifest/config layers and GGUF metadata.

The manifest code computes a manifest digest when parsing, can sum total layer size, and can remove a manifest/layers that are not in use elsewhere.

## GGUF and related formats

Ollama uses GGUF/GGML metadata heavily for local LLM execution:

- `llm.LoadModel` states that the model must be in GGML format and decodes it with `fs/ggml`.
- `server/model.go` has `layerGGML`, `isGGUF`, and content-type detection through `ggml.DetectContentType`.
- `llm.NewLlamaServer` says all GGML models are served via upstream `llama-server` subprocess.
- `model.New` initializes a model instance by constructing an ML backend from a model path and GGUF-style configuration.

Safetensors also appear in create/import and newer MLX/image paths:

- The parser can gather safetensors model directories.
- `server.CreateHandler` has an experimental local safetensors creation path.
- `Model.IsMLX()` returns true when `Config.ModelFormat == "safetensors"`.
- The scheduler sends safetensors/image models to `mlxrunner` or `imagegen` rather than `llama-server`.

Conclusion: GGUF/GGML is the primary local runtime format for normal Ollama LLM inference; safetensors support exists in creation/import and separate runner paths.

## How Modelfiles work

A Modelfile is parsed into commands. Important commands include:

- `FROM`: base model name or local model path.
- `DRAFT`: draft model path for speculative/MTP-like paths.
- `ADAPTER`: adapter file/path.
- `TEMPLATE`: prompt template.
- `SYSTEM`: system prompt.
- `LICENSE`: license text.
- `MESSAGE`: seed/chat messages.
- `PARAMETER`: runtime/model options such as `num_ctx`, temperature, top-k, etc.
- `RENDERER`, `PARSER`, `REQUIRES`: additional model metadata/compatibility controls.

`parser.Modelfile.CreateRequest` converts these commands into an API create request. If `FROM` resolves to a local file/directory, files are digested and uploaded/imported into the create request. If the path does not exist, `FROM` is treated as an existing model reference.

## How custom models are created or referenced

There are two main paths:

1. **Derived from an existing Ollama model**: `FROM llama3.2` references a local or remote model. If not local, `parseFromModel` can pull it.
2. **Imported from local files**: `FROM ./model.gguf` or a safetensors directory causes the parser/create code to digest files and create local layers.

Create flow:

1. CLI reads the Modelfile and converts it to `api.CreateRequest`.
2. Server `CreateHandler` validates the model name and request.
3. Local files become content-addressed layers.
4. Optional adapters/projectors/templates/params/messages/license become additional layers.
5. The manifest is written with `manifest.WriteManifest`.

Experimental safetensors creation is deliberately local-only in CLI code: it checks that `OLLAMA_HOST` is loopback/unspecified and rejects remote experimental safetensor creation.

## How quantized models are handled

Quantization appears in several places:

- The CLI supports `ollama create --quantize <level>` and `--draft-quantize`.
- Create code includes quantization validation and conversion paths.
- GGUF metadata exposes quantization information used by model display and runtime sizing.
- The runtime ultimately loads the selected GGUF/converted model file; quantization affects file size, memory footprint, and speed/quality trade-offs.
- KV cache quantization is separately configurable with `OLLAMA_KV_CACHE_TYPE`, passed to `llama-server` as `--cache-type-k` and `--cache-type-v`.

Operationally:

- Q4 variants are much smaller/faster and usually fit 8 GB VRAM better, but lose more accuracy.
- Q5 variants are a common balance for 7B/8B-class models.
- Q8 variants are larger and closer to full precision but often spill to system RAM on 8 GB cards for larger models.

## Context length configuration

Context length can come from multiple layers:

1. Request options, e.g. JSON `options: {"num_ctx": 4096}`.
2. Modelfile/model options, e.g. `PARAMETER num_ctx 4096`.
3. Environment default `OLLAMA_CONTEXT_LENGTH`.
4. Automatic defaults when none of the above is set.
5. Model training context limit from GGUF metadata.

The scheduler clamps minimum context to 4 and raises multimodal models to at least 2048. `llm.NewLlamaServer` warns and clamps if requested context exceeds the model's training context. Runner launch passes `-c num_ctx * num_parallel`, so parallelism multiplies effective runner context allocation.

If automatic context causes an out-of-memory load failure, the scheduler can retry once with the next lower automatic context tier, e.g. from a very large context to 32768 or 4096.

## VRAM/RAM usage determination

Ollama estimates and observes memory through several mechanisms:

- Device discovery reports total/free GPU memory and system memory.
- `llm.PredictServerVRAM` estimates model/context VRAM before runner launch.
- Scheduler adds batch surcharges for larger automatic generation batches.
- Scheduler logs system memory and per-GPU available/free/minimum/overhead numbers.
- Runner logs are parsed to update memory accounting after `llama-server` loads.
- `/api/ps` reports loaded model size/VRAM state through runner refs.

Relevant controls:

- `OLLAMA_GPU_OVERHEAD`: reserves VRAM per GPU.
- `OLLAMA_NUM_PARALLEL`: increases effective context allocation.
- `OLLAMA_CONTEXT_LENGTH` or request `num_ctx`: directly affects KV cache memory.
- `OLLAMA_KV_CACHE_TYPE`: can reduce/increase KV cache memory depending on type.
- `num_batch`: affects temporary compute memory; scheduler can downshift automatic batch size.
- `use_mmap`: affects system memory/file mapping behavior.

## What happens when a model does not fully fit in GPU memory

Behavior depends on options and runtime outcome:

- With default `num_gpu: -1`, Ollama does not pass `-ngl`; `llama-server` auto-detects GPU layer offload. This can result in partial offload when full GPU residency is not possible.
- With `num_gpu: 0`, Ollama passes `-ngl 0`, forcing CPU-only execution.
- With positive `num_gpu`, Ollama passes that explicit layer count. If less than total layers, this is explicit partial offload.
- The scheduler predicts fit and may evict loaded models before trying to load another model.
- If loading fails with a recoverable out-of-memory condition and context was automatic, the scheduler may lower the automatic context and retry once.
- If other models are loaded and OOM occurs, the scheduler can evict other models and retry once.
- If full load is required and cannot fit, `llm.ErrLoadRequiredFull` is returned; otherwise local inference may run partially on CPU/RAM and be slower.

For a Windows RTX 4060 8 GB, expect 7B/8B Q4/Q5 models to be practical, while larger or Q8 models may partially offload to RAM/CPU or fail at high context sizes.

## Practical inspection commands

```powershell
# Where Ollama stores models by default
$env:USERPROFILE + "\.ollama\models"

# Check model blobs/manifests
Get-ChildItem "$env:USERPROFILE\.ollama\models" -Recurse | Select-Object FullName,Length

# Show model metadata
ollama show llama3.2:latest --verbose

# List locally installed models
ollama list

# Show loaded models and VRAM usage
ollama ps
```
