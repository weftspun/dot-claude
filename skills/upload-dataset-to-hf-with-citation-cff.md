---
name: upload-dataset-to-hf-with-citation-cff
description: Publish a MaskScore stub (or any Weftspun dataset staging directory) to a HuggingFace dataset repository with a CITATION.cff that credits every upstream this dataset cites — the license-required ones (CC-BY-4.0, MIT, BSD, and any other attribution license) and the courtesy citations for Apache-2.0 / MPL / permissive dependencies whose paper or model card the dataset consumes. Wire the README `configs` block so the HF dataset viewer picks up parquets that live in subdirectories. Use when a new stub is ready to ship, when a new upstream is added to a dataset already on HF, or when the dataset viewer is not showing new parquets that are already uploaded.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# Upload dataset to HF with CITATION.cff

Every upstream this dataset cites goes in a `CITATION.cff` at the repository
root. Two classes, both listed:

1. **License-required attribution** — CC-BY-4.0, MIT, BSD, and every other
   license whose terms require a credit line. Without these the corpus is
   undistributable. SpeakingFaces (CC-BY-4.0), Parakeet TDT (CC-BY-4.0),
   allosaurus (MIT), WavLM SV (MIT) are the recurring ones.
2. **Courtesy citations** — Apache-2.0, MPL, and other permissive licenses
   whose paper or model card the dataset consumes. No legal obligation, but
   the reader needs to know which model produced which column. Whisper,
   Voxtral, wav2vec2, Gemma, Qwen3-TTS, ANNY are the recurring ones.

The rule is the same either way: if the dataset used it, cite it.

## When to invoke

Publishing a new dataset repo to HF, adding a new stub subdirectory to an
existing one, adding a new upstream of any license to a corpus, or debugging
a dataset viewer that is not surfacing parquets that are on disk in the repo.

## Author line

Weftspun does not sign these `chibifire org`. Author is:

    authors:
      - family-names: Lee
        given-names: "K. S. Ernest (iFire)"
        alias: fire
        website: https://github.com/fire

Copy that verbatim, including the parenthesised nickname in `given-names`.

## Pipeline

Reference: `6-datasource/anny-render-corpus/build/hf_speech_stage/`.

**1. Build the staging directory.** Everything the HF repo will hold sits
under one local directory, with the parquets in stub subdirectories
(`speech/`, `mesh/`, ...) rather than at the root, so future stubs do not
collide by filename.

    build/hf_<name>_stage/
      README.md                  frontmatter + configs + human prose
      CITATION.cff               author + every upstream under references
      speech/*.parquet           this stub's 3 parquets
      mesh/*.parquet             next stub's 3 parquets
      ...

**2. Author the CITATION.cff.** Structure follows citation-file-format.github.io:

    cff-version: 1.2.0
    title: <one-line dataset title>
    message: >-
      If you use this dataset, please cite it and the upstream sources
      listed under `references`.
    authors:
      - family-names: Lee
        given-names: "K. S. Ernest (iFire)"
        alias: fire
        website: https://github.com/fire
    license: Apache-2.0
    license-url: https://apache.org/licenses/LICENSE-2.0
    type: dataset
    version: <RFD serial, e.g. RFD-2164.5>
    date-released: "<YYYY-MM-DD>"
    url: https://huggingface.co/datasets/chibifire/<repo>
    identifiers:
      - type: other
        value: urn:oid:1.3.6.1.4.1.66606.1.2.<serial>
        description: RFD <serial> (<title>)
      - type: other
        value: urn:oid:1.3.6.1.4.1.66606.1.1.<parent>
        description: RFD <parent> (<parent title>)

    references:
      # <source name> -- <license, e.g. CC-BY-4.0 (attribution required)>
      - type: data | software
        title: ...
        authors: ...
        url: ...
        doi: ...             # if the source publishes one (SpeakingFaces: 10.3390/s21103465)
        license: <SPDX-ish>

Include every upstream the dataset actually used. The license field is
recorded honestly (`CC-BY-4.0`, `MIT`, `Apache-2.0`, `MPL-2.0`, ...) so the
reader can tell attribution obligation from courtesy citation, but the
inclusion decision is `did the dataset consume it`, not `does its license
require a credit`. See `build/hf_speech_stage/CITATION.cff` for the full
worked example — 11 references spanning CC-BY-4.0 data, MIT tools, and
Apache-2.0 models.

**3. Wire the README configs.**

    ---
    license: apache-2.0
    tags: [maskscore, rung-<N>, ...]
    configs:
      - config_name: <stub>
        data_files: <stub>/maskscore_<stub>.parquet
      - config_name: <stub>_candidates
        data_files: <stub>/maskscore_<stub>_candidates.parquet
      - config_name: <stub>_scores
        data_files: <stub>/maskscore_<stub>_scores.parquet
    ---

**The subdirectory prefix is what makes the dataset viewer render.**
Parquets in a subdirectory are invisible to the viewer unless the
`data_files:` path includes the subdirectory (`speech/maskscore_speech.parquet`,
not `maskscore_speech.parquet`). Missing subdirectory prefix is exactly
the failure mode that made `chibifire/maskscore-rung-1-bootstrap`'s
speech configs not surface until we fixed the README.

**4. Upload, incrementally.** Use `hf` (the HF CLI). The upload must
survive a dropped connection or a killed session: bytes already sent
land as commits, and a rerun resumes rather than restarting. Upload the
staging directory with `upload-large-folder`, which commits per file as
each finishes and resumes on rerun, then follow up with individual
`README.md` / `CITATION.cff` uploads when only those change:

    export HF_XET_HIGH_PERFORMANCE=1
    hf upload-large-folder --repo-type dataset chibifire/<repo> build/hf_<name>_stage
    hf upload --repo-type dataset chibifire/<repo> build/hf_<name>_stage/README.md README.md
    hf upload --repo-type dataset chibifire/<repo> build/hf_<name>_stage/CITATION.cff CITATION.cff

`hf upload <repo> <dir> .` is the atomic form: one commit for the whole
directory, nothing lands until it finishes. Do not use it for a staging
directory over about 50 MB. `HF_XET_HIGH_PERFORMANCE` is the current
name of the fast-transfer knob; `HF_HUB_ENABLE_HF_TRANSFER` is
deprecated (hub warning surfaced 2026-09-04). Verified against
huggingface_hub 0.36.2: `HfApi.upload_folder` has no `multi_commits`
parameter in this version, so the CLI above is the resumable path, not
a Python-side flag.

Record the final commit hash in the README after the last upload, so a
training run or a paper can pin the exact snapshot it consumed.

**5. Verify from the dataset viewer.** Open
`https://huggingface.co/datasets/chibifire/<repo>` and confirm each new
config appears in the "Subset" dropdown. If not, the config's
`data_files` path is wrong — recheck the subdirectory prefix, not the
upload.

## Rules that catch failure modes

- **Author line is `K. S. Ernest (iFire) Lee` (github.com/fire), not `Weftspun (chibifire org)`.** The organisation owns the HF repo namespace; the author line credits the human. First uploads have caught this reflex twice.
- **CITATION.cff at the repo root** — not inside a stub subdirectory. One file per HF dataset, covering every source used by every stub in it.
- **Every upstream the dataset consumed is a separate `references:` entry**, with its real license recorded honestly. Include a DOI for paper-backed sources (SpeakingFaces: 10.3390/s21103465) or the HF model card URL for weights. `did the dataset use it` decides inclusion; the license field decides whether the credit was legally required or a courtesy.
- **The README `configs:` block is the source of truth for the dataset viewer**, not the parquet layout on disk. A parquet uploaded to a subdirectory that no config points at renders as nothing. Silent skip that reads exactly like a pass — rule 3.
- **Version the CITATION.cff by RFD serial** (`version: RFD-2164.5`) not by a semver invented on the spot. The RFD is the release; the file cites it.
- **`hf upload` never rewrites what it did not upload.** A first pass with the full staging directory is fine; a second pass with only `README.md` will not delete a stray file, so the staging directory is the reference — not the state on HF.
- **The upload is incremental or it is a liability.** `hf upload-large-folder` commits per file and resumes; `hf upload <dir> .` is one atomic commit that costs the same bytes twice on any drop. A multi-GB corpus render or a checkpoint goes through the resumable form every time; the atomic form is for a README.
- **Pin the snapshot.** The final commit hash goes in the README. A dataset consumed by a training run whose commit cannot be named afterwards is a training run that cannot be reproduced.
- **Related skills**: [[emit-etnf-maskscore-stub]] for what to put in the parquets, [[register-rfd-under-pen-oid]] for the serial cited in `version:` and `identifiers:`.
