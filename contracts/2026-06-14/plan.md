# Laptop-GPU variant Implementation Plan

> **For agentic workers:** implement this plan task-by-task. Steps use checkbox
> (`- [ ]`) syntax for tracking. Each step is one small action.

**Goal:** Add a third fixed-topology compose variant, `compose.laptop-gpu.yml` +
`.env.laptop-gpu.example`, that runs the existing four-service laptop RAG stack with
Ollama using an NVIDIA laptop GPU, and update the repo's docs to reflect three
variants — shipping documented-but-unvalidated, with no claim of validation.

**Architecture:** Copy the CPU-laptop pair verbatim, then make exactly one
substantive change to the compose file (grant the `ollama` service the GPU via the
Compose `deploy.resources.reservations.devices` block) plus an optional grounded
model-tag change in the env file. Keep the file readable top-to-bottom; no profiles,
no `-f` overrides (AGENTS.md mandate). Documentation updates are part of the contract.

**Tech Stack:** Docker Compose, Ollama (CUDA GPU build, same `ollama/ollama` image as
the CPU variant — the image carries CUDA support; the difference is the GPU
reservation, not a different image), Qdrant, Apache Tika, Open WebUI, NVIDIA
Container Toolkit on the host.

**Governance note (read before editing):** `AGENTS.md` forbids the swappable-engine /
layered-override layout. This variant is a standalone fixed-topology sibling file by
deliberate decision (see `spec.md`, "Governing decision"). Do not refactor the three
variants into a shared base + overrides.

**Lifecycle gate (read before claiming done):** affected surface = the repo's
compose-variant set; its state for this new variant is **documented-but-unvalidated**
(evidence: `docs/VALIDATION.md` has no boot record for laptop variants). This plan's
completion claim is limited to "files written and statically valid", never "variant
works". A real boot on Nvidia-laptop hardware is the separate follow-on in Task 6.

---

## File structure

- **Create:** `compose.laptop-gpu.yml` — fixed-topology four-service stack, GPU on
  `ollama`.
- **Create:** `.env.laptop-gpu.example` — pinned images, model ids, ports for this
  variant.
- **Modify:** `.gitignore` — append `!.env.laptop-gpu.example` so the new example
  file is tracked rather than silently ignored by the `.env.*` rule.
- **Modify:** `README.md` — add the GPU-laptop run subsection; pending-honest wording.
- **Modify:** `AGENTS.md` — change "two compose files" to three; extend the
  "Where things live" list.

---

## Task 1: Create `.env.laptop-gpu.example`

**Files:**
- Create: `/home/user/hnbk-ki-workshop/.env.laptop-gpu.example`

- [ ] **Step 1: Copy the CPU-laptop env file as the starting point**

```bash
cp /home/user/hnbk-ki-workshop/.env.laptop.example \
   /home/user/hnbk-ki-workshop/.env.laptop-gpu.example
```

- [ ] **Step 2: Replace the header and run command**

Replace the top comment block so it reads (note the new filename and that a GPU +
NVIDIA Container Toolkit are now expected):

```
# Laptop-GPU variant (Ollama on an NVIDIA laptop GPU). Copy to .env.laptop-gpu
# (gitignored), then:
#   docker compose -f compose.laptop-gpu.yml --env-file .env.laptop-gpu up -d
# Requires a discrete NVIDIA GPU and the NVIDIA Container Toolkit (same prereq as the
# DGX Spark variant). Faster than the CPU laptop variant; still a learning setup.
```

- [ ] **Step 3: Decide and set the chat model (grounded choice)**

Decision per `decision-proposal-discipline`: a laptop GPU has more headroom than
16 GB CPU RAM, so a non-QAT or larger Gemma 4 build is reasonable. **Recommended
default: keep `gemma4:e4b-it-qat`** for first delivery — it is the tag already
grounded in `docs/Dependency Intelligence.md` (confirmed-to-exist 2026-06-11), so it
introduces no new unverified dependency claim into an unvalidated variant. A larger
tag (`gemma4:e4b`, 9.6 GB) is a noted upgrade path but must NOT be written in as the
default without re-confirming the tag exists in the Ollama library, because this
variant has no real boot to catch a bad tag.

Action: leave `CHAT_MODEL=gemma4:e4b-it-qat`. Update the inline comment to:

```
# gemma4:e4b-it-qat (same tag as the CPU variant, confirmed to exist 2026-06-11).
# A laptop GPU can also run the larger gemma4:e4b (9.6 GB) for better quality —
# confirm the tag still exists before switching, since this variant is unvalidated.
CHAT_MODEL=gemma4:e4b-it-qat
EMBEDDING_MODEL=bge-m3
```

**Known carried-over inconsistency (copy knowingly, do NOT fix here):** the copied
`.env.laptop.example` comment block (its lines ~13-14) calls `gemma4:e2b` "smaller"
than the ~6.1 GB QAT build, while `docs/Dependency Intelligence.md` (line ~102) lists
`gemma4:e2b` at 7.2 GB — i.e. larger, not smaller. Copying `.env.laptop.example`
verbatim propagates that pre-existing drift. This contract does **not** fix it (it is
not this variant's defect); reconciling the e2b size between the two surfaces is a
deferred `@sync` follow-on. Copy the comment as-is, knowing it carries this flaw.

- [ ] **Step 4: Verify the file is well-formed**

```bash
grep -E '^(OLLAMA_IMAGE|QDRANT_IMAGE|TIKA_IMAGE|OPENWEBUI_IMAGE|CHAT_MODEL|EMBEDDING_MODEL|OLLAMA_PORT|QDRANT_PORT|QDRANT_GRPC_PORT|TIKA_PORT|WEBUI_PORT)=' \
  /home/user/hnbk-ki-workshop/.env.laptop-gpu.example
```
Expected: all 11 keys present, identical image digests and ports to
`.env.laptop.example`. This is the correct, complete key set for a laptop variant:
do **not** expect or add `CHAT_GPU_FRACTION` / `EMBED_GPU_FRACTION` here. Those keys
exist only in the Spark `.env.example` because vLLM splits one shared GB10 by an
explicit memory fraction; Ollama on a laptop GPU takes no such fraction key, so the
laptop env files (CPU and GPU) deliberately omit them.

- [ ] **Step 5: Append the `.gitignore` negation so this file is actually tracked**

`.gitignore` ignores `.env.*` by default and re-includes the tracked example files
with explicit negations. Without a matching negation, the file created in Steps 1-3
is silently git-ignored and never reaches participants. Append the negation line
after the existing `!.env.laptop.example` line, mirroring the two existing negations:

```
!.env.laptop-gpu.example
```

The relevant region of `.gitignore` should then read:

```
.env
.env.*
*.env
!.env.example
!.env.laptop.example
!.env.laptop-gpu.example
```

Verify the new example file is no longer ignored:

```bash
git -C /home/user/hnbk-ki-workshop check-ignore -v .env.laptop-gpu.example \
  && echo "STILL IGNORED — negation missing or wrong" \
  || echo "tracked (not ignored) — correct"
```
Expected: `tracked (not ignored) — correct` (i.e. `git check-ignore` exits non-zero
because the path is no longer ignored).

---

## Task 2: Create `compose.laptop-gpu.yml` from the CPU variant

**Files:**
- Create: `/home/user/hnbk-ki-workshop/compose.laptop-gpu.yml`

- [ ] **Step 1: Copy the CPU-laptop compose file**

```bash
cp /home/user/hnbk-ki-workshop/compose.laptop.yml \
   /home/user/hnbk-ki-workshop/compose.laptop-gpu.yml
```

- [ ] **Step 2: Replace the top comment block**

Replace the existing header comment with one naming this variant and its prereq:

```yaml
# Laptop-GPU variant: same RAG stack as compose.laptop.yml, but Ollama uses an
# NVIDIA laptop GPU instead of the CPU. For participants who have a discrete NVIDIA
# GPU but no DGX Spark. Requires the NVIDIA Container Toolkit on the host.
#
# Run:
#   cp .env.laptop-gpu.example .env.laptop-gpu
#   docker compose -f compose.laptop-gpu.yml --env-file .env.laptop-gpu up -d
#
# Ollama serves BOTH the chat model and the embedding model on the GPU, so this is
# four services instead of five. Fixed topology, readable top to bottom (AGENTS.md).
```

- [ ] **Step 3: Grant the GPU to the `ollama` service**

In the `ollama:` service block, add a `deploy.resources.reservations.devices` GPU
reservation. Insert it after the `volumes:` block and before `healthcheck:` (placement
within the service is not significant; keep it readable). The exact YAML to add,
indented under `ollama:`:

```yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu]
```

Grounded literals (per `decision-proposal-discipline`):
- This is the exact two-line device form already used by both GPU services in
  `compose.yml` — `engine` (lines ~23-28) and `vllm-embedding` (lines ~55-60).
  Copying it verbatim is the grounded choice: it minimizes the difference surface
  from the existing variants, which is the entire justification for a near-duplicate
  sibling file.
- **No `count:` key.** `compose.yml` uses no `count` form anywhere; introducing
  `count: 1` (or `count: all`) here would be a form the repo never uses elsewhere —
  exactly the silent drift `AGENTS.md` forbids. `driver: nvidia` plus
  `capabilities: [gpu]` already requests the host's NVIDIA GPU(s); on a single-GPU
  laptop this gives Ollama that GPU. Do not add `count:`.

The `ollama-models` one-shot puller does NOT need the GPU (it only calls `ollama pull`
against the already-running `ollama` service); leave it unchanged.

- [ ] **Step 4: Confirm nothing else changed vs the CPU variant**

```bash
diff /home/user/hnbk-ki-workshop/compose.laptop.yml \
     /home/user/hnbk-ki-workshop/compose.laptop-gpu.yml
```
Expected: only the header comment block and the added `deploy:` GPU block differ.
Service names, ports, healthchecks, the model puller, the Open WebUI wiring, and the
volumes must be byte-identical otherwise.

- [ ] **Step 5: Static compose validation (NOT a boot)**

```bash
docker compose -f /home/user/hnbk-ki-workshop/compose.laptop-gpu.yml \
  --env-file /home/user/hnbk-ki-workshop/.env.laptop-gpu.example config >/dev/null \
  && echo "compose config: OK"
```
Expected: `compose config: OK`. This proves the YAML and variable interpolation are
well-formed. It is explicitly NOT a boot and NOT real-use verification. Note honestly:
`config` validates only YAML and variable interpolation — it cannot see the GPU
runtime. It emits a clean config even on a host with no GPU and no NVIDIA Container
Toolkit, so a green `config` proves nothing about whether the GPU reservation will
actually satisfy at boot. The only thing that proves the reservation works is a real
boot, which is the deferred follow-on (Task 6), not this step.
If `docker` is unavailable in this environment, record that `config` could not be run
and leave the variant flagged as not-yet-statically-checked rather than claiming OK.

---

## Task 3: Add the GPU-laptop run section to `README.md`

**Files:**
- Modify: `/home/user/hnbk-ki-workshop/README.md` (after the "Running on a laptop
  instead (no DGX Spark)" section, lines ~141-169)

- [ ] **Step 1: Add a new subsection after the CPU-laptop section**

Insert, immediately after the existing laptop section's closing paragraph (the
"Gemma 4 E4B is a normal instruct model…" line) and before "## A note on the model's
'thinking'":

```markdown
### Faster on a laptop with an NVIDIA GPU

If your laptop has a discrete NVIDIA GPU and the NVIDIA Container Toolkit installed
(the same prerequisite as the DGX Spark variant), there is a third compose file that
runs the same four-service Ollama stack with the GPU instead of the CPU:

```bash
cp .env.laptop-gpu.example .env.laptop-gpu
docker compose -f compose.laptop-gpu.yml --env-file .env.laptop-gpu up -d
docker compose -f compose.laptop-gpu.yml ps
# then open http://localhost:3000
```

This is faster than the CPU laptop variant but is still a learning setup, not a
Spark. Note: this variant ships documented but not yet validated on real laptop-GPU
hardware — if you boot it, the team would welcome a validation report.
```

(The wording is pending-honest: "not yet validated" / "if you boot it", never
"verified".)

- [ ] **Step 2: Verify the insertion reads cleanly**

```bash
grep -n "Faster on a laptop with an NVIDIA GPU" /home/user/hnbk-ki-workshop/README.md
grep -n "not yet validated" /home/user/hnbk-ki-workshop/README.md
```
Expected: both lines found, in the laptop region of the file.

---

## Task 4: Update `AGENTS.md` to describe three variants

**Files:**
- Modify: `/home/user/hnbk-ki-workshop/AGENTS.md` (the "Two compose files" ground
  rule, lines ~29-34; and the "Where things live" list, lines ~37-48)

- [ ] **Step 1: Update the compose-files ground rule**

Change the bullet that begins "**Two compose files, each one fixed topology.**" to
name three, while keeping the existing fixed-topology / no-swappable-layout mandate
intact. Replace its first two sentences with:

```markdown
- **Three compose files, each one fixed topology.** `compose.yml` is the DGX Spark
  variant (vLLM on the GPU); `compose.laptop.yml` is the laptop CPU variant (Ollama
  on CPU); `compose.laptop-gpu.yml` is the laptop NVIDIA-GPU variant (Ollama on the
  GPU). Keep each readable top to bottom; do not reintroduce the swappable-engine
  plugin layout from the `dgx-spark-observables` benchmarker that this was
  extracted from. A change to the shared services (Qdrant, Tika, Open WebUI) usually
  needs applying to all three files.
```

- [ ] **Step 2: Extend the "Where things live" list**

After the `compose.laptop.yml` line, add:

```markdown
- `compose.laptop-gpu.yml` — the four-service laptop stack with Ollama on an NVIDIA
  GPU (documented but not yet validated on real laptop-GPU hardware).
```

And update the `.env.example` line so it mentions all three env files:

```markdown
- `.env.example` / `.env.laptop.example` / `.env.laptop-gpu.example` — pinned images,
  model ids, GPU fractions, ports for each variant.
```

- [ ] **Step 3: Verify**

```bash
grep -n "Three compose files" /home/user/hnbk-ki-workshop/AGENTS.md
grep -n "compose.laptop-gpu.yml" /home/user/hnbk-ki-workshop/AGENTS.md
```
Expected: the three-variant rule and the new where-things-live line both present.

---

## Task 5: Confirm the unvalidated state is legible (no false validation)

**Files:**
- Read-only check across `docs/VALIDATION.md` and the new/edited files.

- [ ] **Step 1: Confirm VALIDATION.md has NO success row for the GPU-laptop variant**

```bash
grep -ni "laptop-gpu\|laptop gpu" /home/user/hnbk-ki-workshop/docs/VALIDATION.md \
  || echo "no laptop-gpu validation row — correct for delivery"
```
Expected: no success/validation row. (Leaving it absent is the intended honest state;
do not add a success row.)

- [ ] **Step 2: Confirm no new file claims validation**

```bash
grep -rniE "verified|validated|confirmed" \
  /home/user/hnbk-ki-workshop/compose.laptop-gpu.yml \
  /home/user/hnbk-ki-workshop/.env.laptop-gpu.example
```
Expected: no hits asserting this variant is verified/validated. (A match only in
prereq prose like "confirm the tag exists" is fine; a claim that the variant works is
not.)

- [ ] **Step 3: Static-validity self-check recap**

Confirm Task 2 Step 5 (`docker compose config`) passed, or that its non-availability
was recorded. The build is "done" only at static-validity + docs-consistent, never at
"works on hardware".

---

## Task 6: Record the post-validation follow-on (do NOT execute here)

This task documents what happens AFTER a real boot; it is out of this contract's
implementation scope and must not be performed without real Nvidia-laptop evidence.

- [ ] **Step 1: Note the two follow-on edits, gated on real validation**

When a participant or the team boots `compose.laptop-gpu.yml` on a real laptop GPU and
passes the end-to-end RAG acceptance test (upload a doc, ask a grounded question, see a
citation — the same acceptance test AGENTS.md defines):

1. Add an **Ollama-on-GPU** entry to `docs/Dependency Intelligence.md`: runtime, the
   GPU-reservation mechanism, the chosen chat model and why, and GPU-vs-CPU caveats.
2. Add a **validation row** to `docs/VALIDATION.md`: host, date, per-service checks,
   and the end-to-end RAG result.

These are a separate `@capture`/`@sync` follow-on once evidence exists. Do not write
them as part of this plan.

---

## Self-review

- **Spec coverage:** sibling-file decision (Tasks 1-2), GPU reservation (Task 2 Step
  3), env + grounded model tag (Task 1), `.gitignore` negation so the example file
  ships (Task 1 Step 5), README (Task 3), AGENTS three-variant update (Task 4),
  lifecycle honesty / no false validation (Task 5), post-validation follow-on deferred
  (Task 6). All spec sections map to a task.
- **Placeholder scan:** GPU-reservation YAML, env header, README block, and AGENTS
  wording are all given literally; the one deferred literal (a larger model tag) is
  deliberately deferred with its grounding rule, not left as a TODO.
- **Type/name consistency:** filenames `compose.laptop-gpu.yml` /
  `.env.laptop-gpu.example`, run command, service name `ollama`, and the GPU device
  block (two-line `driver: nvidia` / `capabilities: [gpu]`, no `count:`, matching
  `compose.yml`) are used identically across all tasks.
- **No-boot discipline:** every "verify" step is static (grep / `compose config` /
  diff); no task boots the stack or claims real-use verification.
