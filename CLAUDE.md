# CLAUDE.md

This repo is a restore runbook for a local LLM setup: LM Studio (model server)
+ Continue (VS Code extension). Full details, rationale, and exact file
contents live in README.md — this file is the execution entry point.

## When asked to restore this setup on a new machine

Execute in order. Don't ask for confirmation between steps unless one fails —
this is a known, pre-validated checklist, not an open design decision.

1. Check prerequisites (README.md §0): LM Studio installed, `lms` CLI on
   PATH, Continue extension installed, GPU with ≥8GB VRAM.
2. Diff the installed Continue extension's version against `continue-2.0.0`.
   If different, do NOT paste config.yaml blindly — re-validate every field
   against that version's `config-yaml-schema.json` first (see §5.2, this
   already broke once silently: `completionOptions`, `systemMessage`,
   `tabAutocompleteModel`, `embeddingsProvider` all looked plausible and were
   all wrong).
3. Download the 8 models (README.md §1) via `lms get`, falling back to the
   HuggingFace links in the table if `lms get` can't resolve an ID.
4. Create `~/bin/lms-pick` with the exact content from README.md §2,
   `chmod +x` it.
5. Ensure PATH entries from README.md §3 are in `~/.bashrc`.
6. Create `~/.continue/config.yaml` with the exact content from README.md §4.
   Per-model `temperature`/`systemMessage`/`reasoning` values are intentional,
   not defaults — see the rationale table in §4 ("Чому саме такі параметри
   тюнінгу") before changing any of them.
7. Run the jsonschema validation snippet in README.md §4 — treat a failure
   here as blocking, not a warning. Config that "looks right" but fails
   schema validation has silently done nothing before.
8. Run `lms-pick`, confirm preset 1 loads with zero errors, before declaring
   the restore complete.

## Standing facts to apply without re-deriving

- **VRAM hard limit is 7.2GB, not 8GB**, on 8GB cards — confirmed by an
  actual `SIGABRT`/`cudaMalloc failed: out of memory` crash, not caution.
  Don't relax this threshold in `lms-pick` because the math looks like it
  should fit.
- **Qwen 3.5 9B cannot coexist with any second model**, regardless of
  context-length or load order — it silently pulls a vision/CLIP submodule
  (~0.9GB) that isn't reflected in `--estimate-only` output. Load it solo.
- If `lms-pick` reports "insufficient VRAM?" for even the smallest model
  while VRAM is actually free, suspect auth desync first, not sizing:
  ```
  lms ps
  ```
  `Invalid passkey for lms CLI client` in the output means `lms load`/`lms
  unload` have been failing silently (the script redirects stderr). Find and
  kill the orphaned `llama-server` process holding VRAM, kill `llmster`, then
  `lms server start --port 1234` to resync. This has recurred more than once
  — don't treat it as a one-off.
- Sampling/prompt tuning is role-specific by design: low temperature
  (0.1–0.2) for autocomplete/coding where determinism matters, higher
  (0.6–0.7) for chat/reasoning where some variation is fine. Don't normalize
  all models to one temperature "for consistency" — that undoes the point.
- Every file in this repo has been round-tripped: the embedded script/config
  blocks were diffed byte-for-byte against the live files they came from, and
  config.yaml was validated against the installed extension's JSON Schema
  (not just checked for valid YAML). If you regenerate either file, redo both
  checks before considering the update done.
