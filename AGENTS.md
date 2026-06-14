# AGENTS.md — working on this repo

Guidance for anyone, human or AI, making changes here.

## What this repo is

This is the **public infrastructure** half of the HNBK KI-Workshop. It is the
Docker-based RAG stack that workshop participants clone onto their own NVIDIA DGX
Spark and run. It is meant to be pushed online and cloned freely.

There is a **separate private teaching repo** (`hnbk-ki-workshop-teaching`) that
holds the workshop proposal, the presentation diagrams, and the Open WebUI
walkthrough. That material must never be merged into or committed to this repo.
The `.gitignore` here already blocks `*.docx` and `teaching/` as a safety net.

## Ground rules

- **No secrets in the repo.** Only `.env.example` is tracked; the real `.env` is
  gitignored. The default model is ungated, so no token is needed.
- **Image versions are pinned on purpose.** They live in `.env.example`. Changing
  a pinned image is a deliberate act: update it, bring the stack up, and confirm
  it still works before committing. A floating tag that breaks for a participant
  next month is the failure mode we are avoiding.
- **The GPU split is validated, not guessed.** The chat engine and the embedding
  service share one GB10 GPU. `CHAT_GPU_FRACTION` and `EMBED_GPU_FRACTION` were
  confirmed to co-exist without out-of-memory. Note: the GB10 uses unified memory,
  so `nvidia-smi` reports memory as `N/A`; judge OOM from vLLM's own logs, not
  from `nvidia-smi`.
- **Three compose files, each one fixed topology.** `compose.yml` is the DGX Spark
  variant (vLLM on the GPU); `compose.laptop.yml` is the laptop CPU variant (Ollama
  on CPU); `compose.laptop-gpu.yml` is the laptop NVIDIA-GPU variant (Ollama on the
  GPU). Keep each readable top to bottom; do not reintroduce the swappable-engine
  plugin layout from the `dgx-spark-observables` benchmarker that this was
  extracted from. A change to the shared services (Qdrant, Tika, Open WebUI) usually
  needs applying to all three files.

## Where things live

- `compose.yml` — the five-service DGX Spark stack (vLLM/GPU).
- `compose.laptop.yml` — the four-service laptop stack (Ollama/CPU).
- `compose.laptop-gpu.yml` — the four-service laptop stack with Ollama on an NVIDIA
  GPU (documented but not yet validated on real laptop-GPU hardware).
- `.env.example` / `.env.laptop.example` / `.env.laptop-gpu.example` — pinned images,
  model ids, GPU fractions, ports for each variant.
- `contracts/<date-slug>/` — **the home for design contracts going forward.** Each
  change of substance gets its own dated folder with a `spec.md` (the design) and a
  `plan.md` (the implementation plan); these are tracked and are the public design
  record for that change. Example: `contracts/2026-06-14/` (the laptop-GPU variant).
  Each contract may also carry a `handoff-trees/` subfolder, but that is local
  session/working scaffolding (progress trackers, iteration notes, backlogs) and is
  **gitignored** — same rule as the root `HANDOFF.md`. Track the contract; not the
  handoff tree.
- `docs/specs/` and `docs/plans/` — the original, foundational RAG-infra design
  records (`2026-06-10-rag-infra-design.md` and its implementation plan), including
  the decision history (notably the pivot from Gemma 4 to Nemotron Nano; read the
  spec before changing the chat model). These are kept as the historical record of
  how the stack was first built; **new** design contracts go under
  `contracts/<date-slug>/` (above), not here.
- `docs/Dependency Intelligence.md` — current state of the vLLM / DGX Spark /
  model dependencies, with pinned versions and the freshness rule. Consult it
  before any version-sensitive change.
- `docs/VALIDATION.md` — what has actually been booted and tested, and when.

## Verifying a change

Bring the stack up and probe each service before claiming a change works:

```bash
docker compose up -d
curl -s http://localhost:8000/v1/models          # engine
curl -s http://localhost:8081/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"BAAI/bge-m3","input":"test"}'     # embedding
curl -s http://localhost:6333/healthz             # qdrant
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/   # open-webui
```

The full RAG path (upload a document, ask a grounded question, see a citation)
is verified in the Open WebUI browser interface; it is the real acceptance test.
