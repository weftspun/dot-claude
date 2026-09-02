---
name: emit-etnf-maskscore-stub
description: Emit a MaskScore stub as three ZSTD-compressed parquets in Essential Tuple Normal Form — a root row per input, a candidates satellite (N candidates × axes per row), and a long-form scores satellite (metric_name/metric_value). Interned vocabularies, satellite relations rather than nullable columns, no nulls, no derivable columns. Use when adding a new MaskScore stub (mesh/depth/pose/keypoints/multimodal/speech/text/video), when extending a stub with a new axis or metric, or when auditing an existing emit for ETNF compliance.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# Emit ETNF 3-parquet MaskScore stub

A MaskScore stub is three parquets. The shape is the same across every stub —
mesh, depth, pose, keypoints, multimodal, speech, text, video — so the family
concatenates cleanly for a downstream consumer, and only the interned
`input_column`, `input_asset`, `input_asset_kind` values differ.

## When to invoke

The user asks for a new stub, extends an existing stub with a new axis
(audio + transcript for speech), adds a new metric to the scores satellite,
or asks whether an emit script complies with CLAUDE.md's ETNF rule.

## The three parquets

    maskscore_<stub>.parquet             root row(s) — one per input asset
    maskscore_<stub>_candidates.parquet  candidates satellite — N rows per root
    maskscore_<stub>_scores.parquet      scores satellite — long-form metric per candidate

All three compressed `zstd`. CLAUDE.md's archive-format rule: ZStandard
parquet for bulk storage; zip and gzip are not acceptable.

## Root schema (interned)

Every stub root shares these columns:

| column             | notes |
| ------------------ | ----- |
| `key`              | `rung<N>/<stub>/<stem>` — human-readable, joins the satellites |
| `task_type`        | interned: `pose_change`, `depth_edit`, `expression_change`, `cross_modal_compose`, `speech_edit`, ... |
| `dimension`        | interned: `instruction_following` or `overall` |
| `input_column`     | interned: name of the extras field this input plays under |
| `input_asset`      | on-disk path (relative to the emit script) |
| `input_asset_kind` | interned: `soma_mesh`, `aov_npz`, `soma_rotations`, `keypoints_json`, `png`, `wav_16khz`, ... |

A stub-specific column may append (e.g. `canonical_text` for speech, `poses`
pointer for mesh). No nullable columns — extend via a satellite instead.

## Candidates satellite

| column                 | notes |
| ---------------------- | ----- |
| `row_key`              | joins the root row's `key` |
| `candidate_axis`       | interned, per-stub: `audio`, `transcript`, `edit`, `view` |
| `rank`                 | 1..N (10 for MaskScore 10-rank ladders, 2 for walking-skeleton rank1/rank5) |
| `candidate_asset`      | on-disk path |
| `candidate_asset_kind` | interned |
| `candidate_kind`       | interned per-stub: `identity`, `pitch_up_mild`, `wrong_subject`, edit-family name |
| `candidate_target_text`| stub-specific (speech only); other stubs drop the column rather than null it |

## Scores satellite (long-form)

    row_key           joins root
    candidate_axis    joins candidate
    candidate_rank    joins candidate
    metric_name       interned: depth_l1, normal_l1, normal_dot, wavlm_cos, wer, ...
    metric_value      float64

Long-form so a new metric adds rows, not a schema migration. The join
(`row_key`, `candidate_axis`, `candidate_rank`, `metric_name`) is the PK.

## Pipeline

**1. Read CLAUDE.md's ETNF rule** if it has been a while: interned
vocabularies, satellites rather than nullable columns, no nulls, no
derivable columns. `-1` for "no parent" is a value; NULL is not.

**2. Write the emit script.** Reference: `6-datasource/anny-render-corpus/emit_speech_stub.py`.
Its docstring cites the RFD OID and lists the three output paths.

    root_rows, cand_rows, score_rows = [], [], []
    for stem in stems:
        key = f"rung1/<stub>/{stem}"
        root_rows.append({"key": key, "task_type": ..., ...})
        for r in RANKS:
            cand_rows.append({"row_key": key, "candidate_axis": ..., "rank": r, ...})
            for m in ("depth_l1", "normal_dot"):
                score_rows.append({"row_key": key, ..., "metric_name": m, "metric_value": ...})

    root_t   = pa.Table.from_pylist(root_rows)
    cands_t  = pa.Table.from_pylist(cand_rows)
    scores_t = pa.Table.from_pylist(score_rows)

    pq.write_table(root_t,   out / "maskscore_<stub>.parquet",            compression="zstd")
    pq.write_table(cands_t,  out / "maskscore_<stub>_candidates.parquet", compression="zstd")
    pq.write_table(scores_t, out / "maskscore_<stub>_scores.parquet",     compression="zstd")

Use `pa.Table.from_pylist(...)`, not `pa.table(list_of_dicts)` — the second
form fails with "Must pass names or schema".

**3. Assert row counts before writing.** Print
`root_t.num_rows x num_columns` for each; a silent skip on a per-clip
transcript read (e.g. one ASR track failed) reads as a pass. Rule 3:
unmet precondition is a FAIL, unchecked things are named and counted.

**4. Register the RFD** for the emit and cite the URN in the script's
docstring. See the [[register-rfd-under-pen-oid]] skill.

**5. Add the three configs to the HF README** so the dataset viewer
picks them up. See the [[upload-dataset-to-hf-with-citation-cff]] skill —
new-stub parquets go in a subdirectory, and each `data_files:` entry
needs the subdirectory prefix (`speech/maskscore_speech.parquet`, not
just `maskscore_speech.parquet`).

## Rules that catch failure modes

- **No nulls anywhere.** A candidate that has no `candidate_target_text` (a mesh candidate) does not get the column at all; different-shape rows in different stubs are how ETNF handles it. A NULL is a schema mistake.
- **No derivable columns.** `rank` derives from position, `candidate_kind` derives from `rank` in a fixed convention — pick one as source and stop. `AUDIO_KIND_BY_RANK` in `emit_speech_stub.py` hardcodes the convention rather than reading the (partial) manifest.json.
- **Interned vocabularies live in a comment or a schema module**, not as scattered string literals. `task_type` and `input_asset_kind` values from RFD 1173 stay consistent across the eight stubs.
- **Long-form scores.** Adding `normal_l2` is a new `metric_name` value, not a new column. A wide-form scores table freezes the metric set into the schema and fights the next metric.
- **Print row and column counts to stdout** — three lines, one per parquet. A dataset viewer that shows 0 rows is a silent skip that only surfaces on HF, and rule 3 says name and count it locally first.
- **`pa.Table.from_pylist(rows)`** for list-of-dicts. `pa.table(rows)` needs a schema.
- **The `input_asset` path is relative** to the emit script (or the parquet's on-disk location), not absolute. `relpath(p)` in `emit_speech_stub.py` handles the ValueError when a path is outside `HERE`.
