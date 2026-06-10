# Validation Record

What has actually been booted and tested, and when. Update this when you verify
the stack on new hardware or after a meaningful change.

## 2026-06-11 — first full bring-up on `spark-c07c` (DGX Spark, GB10)

**Stack:** all five services up via `docker compose up -d`.

| Service | Check | Result |
| --- | --- | --- |
| engine (Nemotron Nano 9B v2 NVFP4) | `/v1/models`, chat completion | healthy, answers |
| vllm-embedding (BGE-m3) | `/v1/embeddings` | healthy, 1024-dim vectors |
| qdrant | `/healthz` | passed |
| tika | text extraction | extracts |
| open-webui | HTTP `/` | 200 |

**GPU split:** chat `--gpu-memory-utilization 0.80` and embedding `0.10` coexist on
the one GB10 with no OOM; engine reserves ~88 GiB KV cache. (GB10 unified memory
means `nvidia-smi` reports memory N/A; judged from vLLM logs.)

**End-to-end RAG (validated in the Open WebUI browser by the user):** created an
admin account, uploaded a Markdown file and a PDF into a knowledge collection,
saw Tika parse them to text, attached the collection to a question, and got a
grounded answer. Time-to-first-token is slowed by the model's reasoning step,
then generation is fast; output quality judged acceptable.

**Engine load time:** ~3.5 min from cache for a plain boot; ~5-6 min when a full
torch.compile + CUDA-graph capture runs. Plan healthchecks and waits accordingly.

## Known open item

- **Clean reasoning rendering** is not solved on this image. Nemotron emits a
  closing `</think>` without an opening tag (the opening tag is in the prompt), and
  vLLM 25.10 has no Nemotron reasoning parser; `--reasoning-parser deepseek_r1`
  crash-loops the engine. See `docs/Dependency Intelligence.md`. Current behaviour:
  reasoning shows inline; `/no_think` gives clean answers.
