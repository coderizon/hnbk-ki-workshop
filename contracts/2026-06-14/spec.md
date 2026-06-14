# Spec: Laptop-GPU variant (Ollama on an NVIDIA laptop GPU)

**Date:** 2026-06-14
**Project:** hnbk-ki-workshop (public RAG infrastructure repo)
**Status of the thing being specified:** to-be-built, and on delivery it ships
**documented-but-unvalidated** — see "Lifecycle and validation honesty" below.

## Active-reality lifecycle context

- **Affected surface:** the compose-variant set of the repo. Today: `compose.yml`
  (DGX Spark, vLLM/GPU, five services) and `compose.laptop.yml` (laptop, Ollama/CPU,
  four services), each with its own `.env*.example`. This contract adds a third
  fixed-topology sibling.
- **Lifecycle state of the repo's variants:** the Spark variant is
  **real-use-verified** (booted and RAG-validated on `spark-c07c` 2026-06-11, per
  `docs/VALIDATION.md`). The CPU-laptop variant is **documented-but-unvalidated**:
  it has no entry in `docs/VALIDATION.md`, meaning no recorded real boot. The new
  GPU-laptop variant will enter the repo in that same documented-but-unvalidated
  state.
- **Evidence surface:** `docs/VALIDATION.md` (real-boot record) and
  `docs/Dependency Intelligence.md` (current dependency state). Governance front
  door for change rules: `AGENTS.md`.
- **Operational implication:** this contract permits adding the variant files and
  their documentation. It does **not** permit any claim that the variant is
  validated. A real boot on real Nvidia-laptop hardware is a separate, post-delivery
  act recorded as a follow-on (see "Follow-on after real validation").

## Purpose

Give a workshop participant who has a laptop with a discrete **NVIDIA GPU** (but no
DGX Spark) a faster local path than the existing CPU-only laptop variant, by running
Ollama with CUDA GPU acceleration instead of Ollama on the CPU. Same RAG stack, same
four-service shape as the CPU-laptop variant, same Open WebUI wiring; the only change
is that the Ollama container is given the GPU.

## Governing decision: a third fixed-topology sibling (not a profile, not a -f override)

**Decision:** ship `compose.laptop-gpu.yml` + `.env.laptop-gpu.example` as a third
standalone, fixed-topology compose file, parallel to the two existing variants.

**Neutral pass (if no governance existed):** for two near-identical laptop variants
differing only in whether the Ollama container claims the GPU, a Compose `profiles`
selector or a layered `-f compose.laptop.yml -f compose.laptop-gpu.override.yml`
override would be the DRY-minimal way to avoid duplicating the qdrant/tika/open-webui
block.

**Current-state pass (governance that does exist):** `AGENTS.md` is explicit —
"Two compose files, each one fixed topology … do not reintroduce the swappable-engine
plugin layout from the `dgx-spark-observables` benchmarker that this was extracted
from. Keep each readable top to bottom." The repo's design value is that a participant
can read one file top-to-bottom and run it with one documented command, with no
mental compose-merge. A profile selector or a `-f` override reintroduces exactly the
swappable/layered shape the governance forbids and breaks the readable-top-to-bottom
invariant.

**Resolution:** the sibling-file approach is the only conforming option. The DRY cost
(the qdrant/tika/open-webui/open-webui block is duplicated a third time) is accepted
deliberately as the price of the repo's fixed-topology, one-command-per-variant
invariant. The override is **not** weighed as a primary alternative; it is recorded
here only as the rejected option and why. This matches how the CPU-laptop variant was
already added.

## What gets built

Two new tracked files, mirroring the CPU-laptop pair, plus a one-line `.gitignore`
edit that keeps the new example file tracked:

1. **`compose.laptop-gpu.yml`** — a fixed-topology, four-service stack
   (`ollama`, `ollama-models` one-shot puller, `qdrant`, `tika`, `open-webui`),
   structurally a copy of `compose.laptop.yml`, with one substantive change: the
   `ollama` service is granted the NVIDIA GPU via Compose's `deploy.resources.
   reservations.devices` GPU reservation (driver `nvidia`, `capabilities: [gpu]`).
   Everything else — service names, ports, healthchecks, the one-shot model puller,
   the Open WebUI Ollama wiring, the volumes — stays identical to the CPU-laptop file
   so the only axis of difference is GPU vs CPU.

2. **`.env.laptop-gpu.example`** — a copy of `.env.laptop.example` with its own
   header and run command. Same pinned images and ports. The chat model **may** be a
   larger/less-quantized Gemma 4 build than the CPU variant's `gemma4:e4b-it-qat`,
   since a laptop GPU has more headroom than 16 GB system RAM on CPU; the plan
   chooses and grounds the exact tag (see plan). Embedding stays `bge-m3` so the
   vector space matches the other two variants.

3. **`.gitignore` (one-line edit, part of shipping the example file):** `.gitignore`
   ignores `.env.*` by default and re-includes the tracked example files with explicit
   negations (`!.env.example`, `!.env.laptop.example`). There is currently **no**
   `!.env.laptop-gpu.example` negation, so the new example file would be silently
   git-ignored and never reach participants. The contract therefore appends a
   `!.env.laptop-gpu.example` negation line, mirroring the two existing ones. This is
   in-scope (the file does not actually ship without it), not aftercare.

### GPU-reservation specifics (to be grounded in the plan, not guessed here)

- The mechanism is the Compose `deploy.resources.reservations.devices` GPU block on
  the `ollama` service. Requires the NVIDIA Container Toolkit on the host, same as the
  Spark prerequisite already documented in the README.
- The exact YAML must copy the two-line device form already used by both GPU services
  in `compose.yml` (`engine` and `vllm-embedding`) verbatim — `driver: nvidia` then
  `capabilities: [gpu]`, with **no** `count:` key. `compose.yml` uses no `count` form
  anywhere, so introducing one here would be exactly the silent drift `AGENTS.md`
  forbids; the whole point of a near-duplicate sibling file is to minimize the
  difference surface. The plan records this literal with that rationale per
  decision-proposal-discipline; the spec fixes the intent (give Ollama the GPU), the
  plan fixes the literal.

## Documentation changes (part of the contract, not aftercare)

- **`README.md`:** add a short subsection under the existing "Running on a laptop
  instead" section (or a sibling section) describing the GPU-laptop path: who it is
  for (laptop with an NVIDIA GPU + Container Toolkit), the one `docker compose -f
  compose.laptop-gpu.yml --env-file .env.laptop-gpu up -d` command, and the honest
  expectation that it is faster than CPU but still a learning setup, not a Spark. It
  must state the same prerequisite as the Spark path: NVIDIA Container Toolkit so the
  container can see the GPU.
- **`AGENTS.md`:** update the "Two compose files" rule and the "Where things live"
  list to describe **three** fixed-topology variants, so the governance text matches
  reality and a future agent does not read "two" and treat the third as drift.
- **`docs/Dependency Intelligence.md`:** the spec does **not** add the GPU-laptop
  dependency entry now. That entry is deferred to the post-validation follow-on
  (below), because writing a dependency-intelligence entry implies a current-state,
  validated claim, and the variant is unvalidated at delivery. The plan instead adds,
  at most, a one-line forward pointer noting the variant exists and is pending real
  validation — phrased as pending, not as validated state.

## Lifecycle and validation honesty (hard requirement)

- The variant ships **documented-but-unvalidated**, exactly parallel to the
  CPU-laptop variant's provisional state. No file may imply it has been booted.
- `docs/VALIDATION.md` gets **no** success row for this variant at delivery. The
  plan adds, if anything, an explicit "not yet validated" note so the absence is
  intentional and legible rather than an oversight.
- README and AGENTS wording for the new variant must use pending/expected language,
  never "verified"/"validated"/"confirmed".

## Follow-on after real validation (out of this contract's scope)

When a participant (or the team) boots `compose.laptop-gpu.yml` on a real
Nvidia-laptop GPU and runs the end-to-end RAG acceptance test:

1. Add an **Ollama-on-GPU** entry to `docs/Dependency Intelligence.md` (runtime,
   GPU-reservation mechanism, chosen chat model and why, GPU-vs-CPU caveats).
2. Add a **validation row** to `docs/VALIDATION.md` recording host, date, the
   service checks, and the end-to-end RAG result.

These are explicitly **not** done in the implementation this contract governs; they
are gated on real-hardware evidence that does not exist yet.

## Acceptance for THIS contract (the build, not the validation)

- `compose.laptop-gpu.yml` and `.env.laptop-gpu.example` exist, are well-formed, and
  differ from the CPU-laptop pair only by the GPU reservation (and any deliberately
  grounded model-tag change). The GPU reservation copies the existing two-line device
  form from `compose.yml` verbatim, with no `count:` key.
- `.gitignore` carries a `!.env.laptop-gpu.example` negation, so `git check-ignore`
  reports the new example file as **not** ignored (it is tracked, like the other two
  example files).
- `docker compose -f compose.laptop-gpu.yml --env-file .env.laptop-gpu config`
  parses cleanly (static validation only — this is **not** a boot and is **not**
  real-use verification; `config` validates YAML and interpolation only and cannot
  see the GPU runtime, so a clean config proves nothing about whether the GPU
  reservation will satisfy at boot).
- README, AGENTS.md reflect three variants; all new wording is pending-honest.
- No file claims the variant is validated; `docs/VALIDATION.md` has no success row
  for it.

## Out of scope

- Booting the stack or any GPU runtime test (no Nvidia-laptop hardware here).
- The CPU-laptop and Spark variants' content (untouched except the AGENTS/README
  count and list).
- The post-validation Dependency Intelligence and VALIDATION entries (follow-on).
