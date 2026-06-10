# RAG Infrastructure Showcase — Design Spec

- **Date:** 2026-06-10
- **Status:** Approved (design); implementation plan pending
- **Owner:** Coderizon (Ruzbeh Nagafi, Julius)
- **Repo:** `hnbk-ki-workshop` (public infrastructure repo)

## Background

The Heinz-Nixdorf-Berufskolleg is standing up a local, on-premises AI server on
NVIDIA DGX Spark hardware: an open-source LLM with a chat front-end and a
knowledge base built from the school's own documents, with everything processed
locally and no data leaving the building. Coderizon is delivering a 2.5-day
hands-on workshop (see the proposal, kept in the private teaching repo) that
teaches the school's teachers and admins to build and run that stack.

This repo is the Docker-based RAG infrastructure for that workshop. It is the
artifact participants clone onto their own DGX Spark and bring up with a single
command. It deliberately covers only the retrieval-augmented-generation path;
the monitoring and load-testing services from the full proposal architecture are
out of scope here.

## Goals

- A self-contained Docker Compose stack that brings up a complete local RAG
  system on one DGX Spark with `docker compose up`.
- Faithful to the real school deployment: GPU-backed vLLM for both inference and
  embeddings, single node, exactly the operational shape the teachers will run.
- Clone-and-run with minimal configuration. No license-gated models, no required
  tokens. Any unavoidable build friction is paid once by the authors (a pinned
  image or a shipped Dockerfile), not by each participant.
- Reproducible over time: every image is pinned so a clone next month behaves
  like a clone today.

## Non-goals

- No Prometheus, Grafana, DCGM exporter, or k6 loadtester. None touch the
  retrieval path.
- No bundled sample documents, ingest scripts, or scripted demo. The stack comes
  up empty; the user uploads documents and asks questions through Open WebUI.
- No portability layer for non-Spark hardware. The target is the DGX Spark.

## Architecture

Five services. A user talks only to Open WebUI in the browser; everything else is
internal plumbing.

| Service | Image (to be pinned) | Role |
| --- | --- | --- |
| `engine` | vLLM (NGC or pinned) | Serves the chat model on the GPU, OpenAI-compatible API |
| `vllm-embedding` | vLLM (NGC or pinned) | Serves BGE-m3 in pooling mode; text to vectors |
| `qdrant` | qdrant/qdrant (pinned) | Vector store and similarity search |
| `tika` | apache/tika (pinned) | Extracts text from uploaded PDF/DOCX/etc. |
| `open-webui` | open-webui (pinned) | Chat UI and RAG orchestrator |

Open WebUI is not just a front-end here; it is the RAG orchestrator. On document
upload it calls Tika to extract text, chunks the text, calls the embedding
service, and writes the vectors to Qdrant. On a question it embeds the query,
searches Qdrant, and feeds the retrieved context to the chat engine. The other
four services cannot be dropped without rebuilding that glue by hand.

### Data flow on a question

1. Browser sends the question to Open WebUI.
2. Open WebUI embeds the query via `vllm-embedding`.
3. Open WebUI searches `qdrant` for the most relevant chunks.
4. Open WebUI stuffs those chunks as context into a call to `engine`.
5. `engine` streams the grounded answer back to the browser.

The polished topology and sequence diagrams for the presentation live in the
private teaching repo, not here.

## Key decisions and rationale

- **DGX Spark-faithful, GPU/vLLM throughout.** The workshop teaches teachers to
  operate this exact stack, so the showcase runs what they will run rather than a
  lighter portable stand-in.
- **Bare runnable infra, no demo data.** Keeps the repo clean and the teaching
  hands-on; participants see RAG work against their own documents.
- **Non-gated model.** Distribution to participants rules out anything behind a
  Hugging Face license gate or token, so first boot just works.
- **Chat model: `nvidia/NVIDIA-Nemotron-Nano-9B-v2-NVFP4`.** Ungated
  (gated=False, NVIDIA open model license, no token), NVFP4 for native Blackwell
  performance, and small (9B) so it loads in minutes. Chosen on 2026-06-10 after
  the Gemma 4 26B-A4B NVFP4 path failed in execution: it required the special
  `vllm/vllm-openai:gemma4-cu130` image, took about an hour to download ~16.5 GB,
  and then died with `Engine core initialization failed` during vLLM init on this
  host. Nemotron Nano is already validated on this exact Spark (three benchmarker
  runs), runs on the stock NGC `vllm:25.10-py3` image with no patch, was already
  cached on the host, and booted and answered a chat request in ~3.5 minutes. It
  keeps both the NVFP4 property and the non-gated requirement. Caveat: it is a
  hybrid reasoning model, so raw output includes chain-of-thought; rendering that
  cleanly in Open WebUI is a polish item, not a blocker.
  Documented fallbacks: `nvidia/Gemma-4-26B-A4B-NVFP4` (ungated, but engine-init
  failure on the available image) and `RedHatAI/Qwen3.6-35B-A3B-NVFP4` (the
  gate-then-Qwen fallback).
- **Embedding model: `BAAI/bge-m3`.** Open, non-gated, already validated on this
  hardware via the benchmarker.
- **Two repos.** This public infra repo ships to participants. A separate private
  teaching repo holds the topology diagram, sequence diagram, Open WebUI
  walkthrough, slides, and the proposal `.docx`. None of that reaches the public
  repo.
- **Pinned images.** Reproducibility for a repo that re-runs months apart.

## Repo structure (public infra repo)

```
hnbk-ki-workshop/
  compose.yml            # all five services, self-contained
  .env.example           # model ids, pinned image tags, GPU fractions, ports
  .gitignore             # .env, *.docx, caches, teaching/ — already in place
  README.md              # participant-facing: prerequisites, bring-up, usage
  AGENTS.md              # governance for future agents in this repo
  docs/
    specs/               # this design spec
    Dependency Intelligence.md   # model / image / vLLM currency note
```

One self-contained `compose.yml` rather than the split/plugin layout of the
`dgx-spark-observables` source project. That project splits engine from webui
because it is a benchmarking harness with swappable engines; this repo has one
fixed topology, so a single file is easier to read top to bottom.

## Open validation tasks (resolve by booting, not by assertion)

1. **Engine image + model — RESOLVED (2026-06-10).** Gemma 4 26B-A4B NVFP4 failed
   engine-init on the `vllm/vllm-openai:gemma4-cu130` image after the ~16.5 GB
   download. Pivoted to `nvidia/NVIDIA-Nemotron-Nano-9B-v2-NVFP4` on the stock NGC
   `vllm:25.10-py3` image, which booted and answered in ~3.5 minutes with no patch.
   No special image or Dockerfile is needed, so bring-up stays a plain
   `docker compose up`.
2. **GPU memory split.** Two vLLM processes share the one GPU. Start with the chat
   engine at `gpu-memory-utilization` ~0.80 and embedding ~0.10, then confirm both
   come up without OOM on first boot. The source project's memory notes that 0.95
   alone caused OOM; with an MoE on a 121 GiB box there is ample headroom, but it
   still gets checked. Adjust down if needed.
3. **Pin every image** to a specific tag or digest and record the chosen versions
   and why in `Dependency Intelligence.md`.

## Dependency intelligence snapshot (seed, 2026-06-10)

Research performed today; no prior surface existed in either project.

- vLLM officially supports the DGX Spark (GB10 Blackwell); NVFP4 is the
  Blackwell-native 4-bit path.
- NVIDIA's own large NVFP4 models (Nemotron Super) are gated; some community NVFP4
  builds with speculative decoding need source-built vLLM plus patches.
- **Chosen chat model: `nvidia/NVIDIA-Nemotron-Nano-9B-v2-NVFP4`** on the stock
  NGC `vllm:25.10-py3` image. Confirmed ungated (gated=False, NVIDIA open model
  license) and validated booting on this Spark in ~3.5 minutes from cache.
- Gemma 4 26B-A4B NVFP4 (`nvidia/Gemma-4-26B-A4B-NVFP4`) is ungated and looked
  ideal on paper (~16.5 GB, ~52 tok/s, multimodal) but failed engine-init on the
  `vllm/vllm-openai:gemma4-cu130` image on this host on 2026-06-10. Kept as a
  fallback to revisit when the vLLM/Gemma-4 NVFP4 path stabilizes.
- `RedHatAI/Qwen3.6-35B-A3B-NVFP4` (Qwen3, Apache-2.0, ungated) is the secondary
  fallback and the target of the user's gate-then-Qwen rule.
- Embedding model `BAAI/bge-m3` is open and validated on this host.
- Sources: dgx-spark-observables benchmarker (validated Nemotron Nano runs);
  live boot test 2026-06-10; HF model cards (gating checks); NVIDIA Gemma 4 blog
  and ai-muninn Spark benchmark (for the Gemma fallback); vLLM DGX Spark blog
  (2026-06-01).

## Participant experience (target)

```
git clone <repo>
cd hnbk-ki-workshop
cp .env.example .env        # defaults work as-is; no token needed
docker compose up           # first run pulls images + downloads the model
# wait for services healthy, then open http://localhost:3000
# upload documents, ask a grounded question, read the cited sources
```
