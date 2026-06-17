# configs

The voice server's llama.cpp setup: [config.ini](config.ini) defines the models, and [docker-compose.yml](docker-compose.yml) runs the server that hosts them.

`config.ini` is INI-style: a global `[*]` section applies to every model, and each `[<hf-repo>:<quant>]` section configures one model. Both the chat LLM (`Gemma4`) and the STT model (`Qwen3-ASR`) set `load-on-startup` and stay resident, so they run at the same time and either can be hit without a load delay.

## Global defaults — `[*]`

Applied to all models unless overridden.

| Setting | Value | Why |
| --- | --- | --- |
| `jinja` | `true` | Use the model's built-in chat template for prompt formatting. |
| `n-gpu-layers` | `-1` | Offload all layers to the GPU. |
| `flash-attn` | `on` | Flash attention — lower memory, faster prefill. |
| `fit` | `off` | Don't auto-shrink offload to fit VRAM; layout is tuned by hand. |
| `prio` / `prio-batch` | `3` | High scheduling priority for generation and batch work. |
| `batch-size` / `ubatch-size` | `2048` / `1024` | Larger batches for faster prompt processing. |
| `reasoning-budget-message` | *(text)* | Injected to cut off runaway thinking so the assistant answers promptly. |

## Chat LLM — Gemma4

`unsloth/gemma-4-26B-A4B-it-qat-GGUF`, aliased `Gemma4`. A 26B-A4B MoE, QAT-trained and served at a Q4_K_XL quant — the quality/VRAM tradeoff this setup targets.

- **Speculative decoding** (`spec-type = draft-mtp,ngram-mod,ngram-map-k4v`) — combines the model's MTP draft head with n-gram drafting from the context (no separate draft model), which speeds up the repetitive, templated output voice replies tend to produce. The `spec-ngram-*` keys tune match length and the K4V draft cache.
- **Sampling** — `temp 1.0`, `top-p 0.95`, `top-k 64`, Gemma's recommended settings.
- **Vision** — `image-min-tokens`/`image-max-tokens` bound how many tokens an image consumes (used for Frigate camera frames).
- **Context** — `ctx-size 128000` with `parallel 6` (six concurrent slots), `cache-idle-slots off` (don't clear KV for idle slots), `kvu on` (unified KV cache), and `cache-ram 4096` for host-side KV offload.

## STT — Qwen3-ASR

`ggml-org/Qwen3-ASR-1.7B-GGUF:Q8_0`, aliased `Qwen3-ASR`. Small enough to run at a high-quality Q8_0 quant alongside the LLM.

- **`ctx-size 896`, `parallel 1`, `cache-ram 0`** — transcription is short and single-stream, so it's given a tiny context, one slot, and no KV offload to leave resources for the LLM.
- **`no-webui = true`** — disables the web UI for this model.

## Running it

[docker-compose.yml](docker-compose.yml) runs the official `llama.cpp:server-vulkan` image and points it at this config:

- **GPU** — Vulkan build on an AMD Radeon card (`VK_DRIVER_FILES` → `radeon_icd.json`, `/dev/dri` passed through, `GGML_VK_ALLOW_GRAPHICS_QUEUE=1`).
- **Volumes** — the Hugging Face cache is mounted so model downloads persist, and `./config` is mounted to `/config` (where `config.ini` lives).
- **Serving** — `--models-preset /config/config.ini` loads every model defined here; the server listens on port `8080`.

## Editing

- Add a model by appending a new `[<hf-repo>:<quant>]` section; global `[*]` defaults apply automatically.
- Swap the LLM by changing its section header to a different repo/quant — size the quant to your GPU's VRAM.
- Both resident models set `load-on-startup = true`. Drop it to `false` for a model you call rarely, trading first-request latency for memory.
