---
name: editscore-ladder-run
description: Run an EditScore-7B pairwise ladder for an image-edit generator (SDEdit / diffusion / LoRA / distilled step-count sweep) on held-out shard pairs. Use when a new candidate generator needs to be measured against an existing baseline for a shipping-decision (RFD 2186 dressing overlay is the reference case), or when a step-count sweet spot needs to be found before committing distillation compute. Aggregate reports wins / mean / median on the 0-25 overall axis with distribution-level N. Ships as an HF dataset per RFD 2196 shape.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# EditScore ladder run

Measure a candidate image-edit generator against a baseline on held-out `(source, edited, instruction)` triples with EditScore-7B pairwise. Ships an HF dataset carrying the images and per-pair scores, and an aggregate that decides gating questions like "does the candidate clear the 80% wins-over-baseline threshold?"

The reference case is the LLaDA-o work: the n=1 → n=5 → n=20 chain confirmed 16-step SDEdit as the sweet spot and cleared RFD 2198's distillation gate. See `logbook-lladao-n1-quality-beats-omnigen2.md`, `logbook-lladao-5pair-step-sweep.md`, `logbook-lladao-n20-held-out-sweep.md`.

## When to invoke

- A new generator candidate needs to be compared against an existing baseline for a shipping decision. The baseline is a real number, not "we think it's better."
- A step-count sweep is needed before committing distillation compute. Cheap to run 5-pair first, then n=20 if the shape holds.
- A LoRA vs no-LoRA comparison at fixed step count needs a decision-grade N.

## Pipeline

**1. Pick the pair source.** `chibifire/editscore-reward-train` is the canonical shard corpus; shard 90 is the standing hold-out. Pick pair indices that span task_type variety — subject_remove, material_alter, color_alter, background, style, subject_add — so the aggregate captures cross-type performance rather than the model's best task.

**2. Hold-out discipline.** If your model was trained on this corpus or a subset of it, pick pairs the training set never saw. For SDEdit / distilled generators that did NOT touch the training pairs, any shard works. Skip indices that overlap with prior ladders (the 5-pair pilot at 0/1/2/5/17 comes back around in n=20 as pairs 20-39 to avoid overlap).

**3. Task-type exclusions.** Single-image SDEdit cannot do `motion_change` (needs frame sequence) or `text_change` (needs OCR-conditioned generation the base model may not have). Exclude those from the pair set unless the candidate explicitly handles them; see memory `parked-language-to-vision-edit-pair`.

**4. Run the generator on each pair.** For a step sweep, run the same pair at each step count separately with a fixed seed. For a fixed-step ladder, one edit per pair. Save every output PNG next to its input for the parquet later.

**5. Score with EditScore-7B pairwise.** `evaluate([source, edited], text_prompt)` returns `{PF, SC, PQ, overall}` on the 0-25 scale. `overall` is the shipping axis; PF (prompt-following), SC (source-consistency), PQ (perceptual-quality) are the axes to read when overall zeros to explain the failure mode (a zero on any axis zeroes overall via the multiplicative aggregator). See memory `editscore-api-surface`.

**6. Aggregate.** Report on the same table: mean overall, median overall, wins-over-baseline count and percent (with the baseline number stated inline), non-zero count. If a gating threshold exists (RFD 2198's 80% for LLaDA-o), state it and whether the run cleared it.

**7. Publish as HF dataset per RFD 2196 shape.** Repo naming: `chibifire/<generator>-<experiment>-<yyyymmdd>` (`chibifire/lladao-step-sweep-shard90-20260904` is the reference case). Parquet schema: image columns as `struct<bytes:binary, path:string>`, ZStandard compression, row_groups ≤ 100 for HF viewer safety. `CITATION.cff` at repo root; `dataset_info` YAML in the README for HF viewer type-inference. See memory `cuda-tests-ship-to-hf`.

**8. Write a logbook entry.** Named `logbook-<generator>-<experiment>.md`, extending the prior entry in the chain (n=1 extends nothing, n=5 extends n=1, n=20 extends n=5). Structure: Question / Apparatus / Result / Reading / Verdict / Not measured / Cost paid. Retractions stay in place next to what they retract. Draft ends with the AI attestation canary if AI drafted it.

## Rules that catch failure modes

- **PQ-only single-image scoring is blocked for shipping decisions** (memory `pq-only-single-image-blocklisted`). Always pairwise `(source, edited, instruction)`, never PQ alone on the edited output.
- **VRAM will bite with three models loaded** — generator + EditScore-7B often exceed the 3090's 24 GB even with offload. Serialize: generate all pairs first, unload generator, then load EditScore and score. See memory `three-model-vram-serialization`.
- **Held-out means never inspected during dev** (CLAUDE.md standing constraint). If you looked at the pair to pick it, you've trained on it by hand.
- **flash_attn 2.8.3 wheel NaNs OmniGen2's forward on Windows** — monkey-patch `_flash_attn_available=False` before importing omnigen2 modules. Memory `omnigen2-flash-attn-nan` records the exact site.
- **A baseline without an N is not a baseline.** State the baseline's mean and the N it was measured on, on the same table row as your candidate's. `OmniGen2 mean 3.36 on n=20 shard-90 pairs 0-19` is the shape.
- **Xet-split shards from EditScore repo don't reassemble** (memory `xet-download-blocklist`); wait for peer reupload rather than debugging client paths. `EditScore-7B` main weights download cleanly.
- **task_type stratification matters for the aggregate.** A mean over a task-type-imbalanced sample tells you about the imbalance, not the model. Report per-task-type mean alongside overall mean when N per type is ≥ 3.
- **Zero-overall pairs deserve a per-axis look.** A PQ=0 with PF=10 SC=10 is a scoring artefact (subject_remove that removed too well); a PF=0 SC=10 PQ=0 is a punt (model returned the source unchanged). Both zeroes hurt the mean without a corresponding model-quality issue at the mode. Call them out in the Reading section rather than letting them silently drag the aggregate.

## Not scoped

- **Training a scorer:** RFD 1157 covers EditScore-7B's LoRA training. This skill uses the trained scorer, doesn't retrain it.
- **Comparing scorers:** if the question is "does EditScore agree with human raters," that's a separate calibration study, not a ladder run.
- **Multi-frame edits:** the pair schema is single-image. Video edits need a different pipeline.
