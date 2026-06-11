# Dependency Intelligence

Current state of the technologies this stack depends on. Update the latest-check
date and the affected entry whenever you research or change a dependency; this is
a living current-state report, not a log.

- **Latest check:** 2026-06-11
- **Updated:** 2026-06-11

## Hardware

- NVIDIA DGX Spark, GB10 Grace-Blackwell, aarch64, unified CPU/GPU memory (~121 GiB).
- **Unified-memory caveat:** `nvidia-smi` reports `memory.used` / `memory.free` as
  `N/A` on the GB10. Judge GPU pressure and out-of-memory from vLLM's own logs
  (look for "Available KV cache memory" and any CUDA OOM), not from `nvidia-smi`.

## Pinned images

vLLM is pinned by its stable dated NGC tag (NVIDIA does not move dated tags); the
others are pinned by digest because they were pulled from floating tags. Values
live in `.env.example`.

| Service | Pin | Version |
| --- | --- | --- |
| engine + embedding | `nvcr.io/nvidia/vllm:25.10-py3` | vLLM 0.10.2+nv25.10 |
| qdrant | `qdrant/qdrant@sha256:0fb8897412abc81d1c0430a899b9a81eb8328aa634e7242d1bc804c1fe8fe863` | 1.15.5 |
| tika | `apache/tika@sha256:21d8052de04e491ccf66e8680ade4da6f3d453a56d59f740b4167e54167219b7` | 3.2.3 (full) |
| open-webui | `ghcr.io/open-webui/open-webui@sha256:53a4d2fc8c7a7cc620cd18e6fe416ed9940f2db87fddf837e3aa55111bec6995` | 0.6.36 |

## Chat model

- **Chosen: `nvidia/NVIDIA-Nemotron-Nano-9B-v2-NVFP4`** on the stock NGC vLLM
  image. Ungated (gated=False, NVIDIA open model license, no token), NVFP4 for the
  Blackwell GPU, 9B, loads from cache in ~3.5 minutes. Validated booting and
  answering on this host 2026-06-11.
- **Why not Gemma 4 26B-A4B NVFP4:** it is ungated and looked ideal on paper
  (~16.5 GB, multimodal), but on 2026-06-11 it needed the special
  `vllm/vllm-openai:gemma4-cu130` image and then failed with
  `Engine core initialization failed` during vLLM init on this host. Parked as a
  fallback to revisit when the vLLM/Gemma-4 NVFP4 path stabilizes.
- **Secondary fallback / gate rule:** `RedHatAI/Qwen3.6-35B-A3B-NVFP4` (Apache-2.0,
  ungated). Use it if a future chosen checkpoint turns out to be gated.

## Embedding model

- `BAAI/bge-m3` via vLLM pooling mode, 1024-dim vectors. Open, non-gated, validated.

## GPU memory split

- One GB10 shared by two vLLM processes. `--gpu-memory-utilization` 0.80 (chat) and
  0.10 (embedding) coexist without OOM; the embedding service reaches healthy while
  the engine reserves ~88 GiB KV cache. Lower `CHAT_GPU_FRACTION` if a future model
  needs more headroom.

## Reasoning / "thinking" behaviour (important, partially unresolved)

Nemotron Nano v2 is a hybrid reasoning model. Its chat template wraps the
reasoning in `<think> ... </think>` and supports toggling:

- Thinking is **on by default**.
- Put `/no_think` in a system or user message to turn it **off** (clean answer, no
  reasoning, slightly weaker on hard prompts). `/think` turns it back on. The
  template also honours an `enable_thinking` chat-template kwarg.

**Clean server-side separation does not work on this image, by design of the
parsers available:**

- vLLM 25.10 ships reasoning parsers for `deepseek_r1`, `qwen3`, `granite`,
  `hunyuan_a13b`, `mistral`, `glm45`, `openai_gptoss`, `step3`. There is **no
  Nemotron parser** in this build.
- Adding `--reasoning-parser deepseek_r1` **crash-loops the engine**: that parser
  requires `<think>`/`</think>` to be registered *special tokens* in the
  tokenizer, and Nemotron's are plain text inserted by the template, so it raises
  `DeepSeek R1 reasoning parser could not locate think start/end tokens` and the
  engine never starts. Do not add this flag. (Confirmed 2026-06-11.)

**What this means in practice:**

- As shipped (no reasoning parser), Open WebUI renders the model's reasoning
  inline before the answer. It is usable; output quality is fine.
- For clean fast answers, create a second Open WebUI model preset whose system
  prompt is `/no_think`. That gives a "Nemotron (fast)" entry alongside the default
  "Nemotron (reasoning)", both served by the same engine.
- To get a collapsible "Thinking" panel, the realistic paths are: (a) Open WebUI
  client-side reasoning-tag handling, which needs testing in the UI because
  Nemotron's output carries the closing `</think>` without an opening tag (the
  opening tag sits in the prompt); or (b) a future vLLM image that ships a Nemotron
  reasoning parser. Treat clean folding as an open item, not a solved one.

## Laptop variant (Ollama on CPU)

A second compose file, `compose.laptop.yml`, runs the same RAG stack on a laptop
without a GPU, for participants who don't have a Spark. Ollama replaces both vLLM
services (it serves the chat model and the embedding model), so it's four services
instead of five.

- **Runtime:** `ollama/ollama` pinned by digest (version 0.30.7 at pin time).
- **Chat model:** `gemma4:e4b-it-qat` — Gemma 4 E4B, 4-bit QAT, instruction-tuned,
  ~6.1 GB. Chosen over the default `gemma4:e4b` (9.6 GB) because the QAT build plus
  bge-m3, Qdrant, Tika, and Open WebUI fits a 16 GB-RAM laptop with headroom; the
  9.6 GB default would crowd it. `gemma4:e2b` (7.2 GB) is a lighter alternative,
  plain `gemma4:e4b` for machines with more RAM. (Sizes from the Ollama library;
  confirm the tag still exists before relying on it.)
- **Embedding:** `bge-m3` on Ollama (~1.2 GB), same model as the Spark variant so
  the vector space matches.
- **Open WebUI wiring:** `ENABLE_OLLAMA_API=true`, `OLLAMA_BASE_URL`,
  `RAG_EMBEDDING_ENGINE=ollama`, `RAG_OLLAMA_BASE_URL`; the OpenAI path is disabled.
- **Caveats:** CPU inference is slow (a few tokens/sec). Ollama inside Docker on a
  Mac is CPU-only (no Metal passthrough); Mac users wanting speed run Ollama
  natively. Gemma 4 E4B is a plain instruct model, so no reasoning/thinking step.
- **Model pull:** a one-shot `ollama-models` service pulls both models on first
  start; Open WebUI waits for it via `service_completed_successfully`.

## Currency notes

- vLLM officially supports the DGX Spark; NVFP4 is the Blackwell-native 4-bit path.
- The Gemma-4 NVFP4 serving path on Spark is still stabilising (special image +
  occasional model patch); watch for it landing in a stock or NGC vLLM image.
- A Nemotron reasoning parser in a newer vLLM would resolve the thinking-panel
  item above; watch the vLLM release notes.

## Freshness rule

Re-check before any version-sensitive change (model swap, vLLM image bump, Open
WebUI upgrade). "Fresh" means this file's latest-check date covers the dependency
within the last few days. Update the date and the affected entry on every check,
even when nothing changed.

## Sources

vLLM DGX Spark blog (2026-06-01); "Now Serving NVIDIA Nemotron with vLLM"
(2025-10-23); vLLM reasoning-outputs docs; Open WebUI reasoning-models docs;
Hugging Face model cards (gating checks); live boot/parse tests on spark-c07c
2026-06-11.
