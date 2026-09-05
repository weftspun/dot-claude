---
name: parquet-rewrite-row-groups
description: Rewrite existing parquet shards to enforce a row-group size cap without re-extracting from source. Use when the HuggingFace viewer errors with `Parquet error: Scan size limit exceeded` (rule 2 of RFD 2196) or when any consumer requires row groups under a specific byte ceiling. Triggers on "row group too big", "scan size limit exceeded", "parquet rewrite", "HF viewer TooBigContentError", "fix parquet row groups". Not for the initial write; the initial write should pass `row_group_size` directly to `pq.write_table`.
tools: Read, Write, Edit, Bash
---

# Rewrite parquet row groups without re-extracting

## When

The HF dataset viewer errors with:

    Parquet error: Scan size limit exceeded:
    attempted to read <N> bytes, limit is 300000000 bytes

or any other consumer that scans one row group at a time hits a byte
ceiling. Cause: a shard written with pyarrow's default (one big row
group per file, often 500 MB+) exceeds the ceiling. Fix: rewrite the
same rows into more, smaller row groups. No re-extraction needed.

## Sizing

The rule is `rows_per_group × avg_row_bytes < 200 MB` (30 % safety
margin under the 300 MB HF viewer ceiling). For image-embedded rows
at ~800 KB–1 MB per row, `row_group_size=100` produces ~80–100 MB
groups.

## The 20 lines

```python
import pyarrow.parquet as pq
import os, sys, time, glob

SRC = sys.argv[1]  # source dir of *.parquet
DST = sys.argv[2]  # dest dir (side-by-side; safer than in-place)

os.makedirs(DST, exist_ok=True)
files = sorted(glob.glob(f"{SRC}/*.parquet"))
t0 = time.time()

for i, f in enumerate(files, 1):
    name = os.path.basename(f)
    out = os.path.join(DST, name)
    if os.path.exists(out) and os.path.getsize(out) > 0:
        continue
    tbl = pq.read_table(f)
    pq.write_table(tbl, out,
                   compression="zstd", compression_level=3,
                   row_group_size=100)   # tune per data density
    md = pq.ParquetFile(out).metadata
    max_group = max(md.row_group(g).total_byte_size
                    for g in range(md.num_row_groups))
    print(f"[{i:2d}/{len(files)}] {name}: {md.num_rows} rows, "
          f"{md.num_row_groups} groups, max {max_group/1e6:.0f} MB")
```

Observed cost: 25 GB / 45 shards in 44 s on an M2 Pro. Fast because
the schema and encoding do not change; only group boundaries move.

## Upload flow

After rewriting, upload with the recipe from `hf-upload-large`. The
in-repo path is decided by the folder structure you upload, not by
any flag on `hf upload-large-folder`:

```sh
mkdir -p upload_stage/data
for f in $DST/*.parquet; do ln -f "$f" upload_stage/data/$(basename "$f"); done

HF_HUB_DISABLE_XET=1 HF_HUB_ENABLE_HF_TRANSFER=1 \
  hf upload-large-folder <repo-id> upload_stage \
    --repo-type=dataset \
    --include="data/*.parquet"
```

Hardlinks (`ln -f`) avoid doubling disk. `hf upload-large-folder` does
NOT follow symlinks — must be hardlinks or real files.

## Empty tail shards

An upstream writer occasionally emits a 0-row / 0-byte tail shard
(observed `train-00038-of-00039.parquet` at 5,238 bytes with 0 rows).
Leave it alone; the auto-indexer accepts it and the viewer ignores it.
Only remove if a downstream consumer explicitly errors on empty groups.

## Not for

- **The initial write.** Pass `row_group_size` to `pq.write_table` when
  writing the shard the first time; this skill is only for shards
  already published in the wrong shape.
- **Schema changes.** This rewrites row groups only; if the shape is
  wrong (e.g. `sequence:` vs `list:` per RFD 2196 rule 6), you need
  a schema rewrite, not this.
- **Cross-shard rebalancing.** This preserves the source's shard
  boundaries. If shards themselves are the wrong size (~1 GB each,
  say), rewrite with `pq.write_dataset` or a manual split step.

## Related

- RFD 2196 rule 2 — row groups ≤ 300 MB (this skill enforces it retroactively).
- `hf-upload-large` — companion skill for the re-upload step.
- `hf-parquet-image-column` — the initial-write skill this one complements.
