# .claude

Working agreements for every project in the weftspun workspace, and the capability rules for
the agent that works in them. Both, because they ended up in one file when the agreements
moved here: `weftspun/weftspun` removed its `CLAUDE.md` and this is where it went.

Claude Code reads `.claude/CLAUDE.md` at the workspace root, which is why the move was worth
making. Under `meta` this file sat at the workspace root and was read by proximity. Under
`repo` the manifest repository is cloned to `.repo/manifests`, two levels down and inside a
directory nothing reads upward from, so the same file in the same repository would have gone
quiet without anything reporting it. Here it is read because of where it is, and nothing has
to remember to name it.

The manifest entry is `name="dot-claude" path=".claude"`. The path is fixed by Claude Code,
which reads that and nothing else, so the name gave way instead: a repository actually called
`.claude` is hidden from an ordinary listing, sorts oddly, and clones into a directory most
shells will not show you.

Standing constraints follow. Each carries a cost behind it; the incident sits in
`weftspun/logbook` (`todo.md` for the narrative, `PITFALLS.md` for the recurring failure modes
and the guards that catch them).

## Hard constraints

**Compute.** GPU work runs on RunPod, never on the local desktop GPU. Tear down after use,
then **double-check** the teardown. Anything not in a git repo is torn down after use — so
if it matters, it is committed and pushed before the machine goes away.

**Archive formats.** zstd, in parquet or standalone. **zip is not acceptable**, and neither
is gzip; recompress to `.zst` and verify payload hashes before deleting an original.
Tabular data is parquet + zstd.

**Normal form.** Parquet is in **Essential Tuple Normal Form**: interned vocabularies,
satellite relations rather than nullable columns, **no NULLs**, no derivable columns. A
value like `-1` for "no parent" is a value; a NULL is not.

**Data hygiene.** Training data only — validation and test splits are strictly held out from
training, tuning, and selection.

Synthetic data is two classes, and the distinction is the whole rule:

*Constructed* synthetic is **rendered deterministically from source assets we hold** — Live2D
drawables, ANNY rigs, BVH poses. The labels are true by construction rather than inferred, the
same seed reproduces the corpus, and nothing was sampled from a learned distribution. This is
ordinary training data and always has been; `syn_data.py`'s Live2D renders are the reference
case.

*Generated* synthetic is **sampled from a generative model** — diffusion outputs, GAN style
transfer, a teacher's predictions. Permitted in a training corpus only when all four hold:

1. the generating model, checkpoint and prompt/conditioning are recorded with the data, so the
   corpus can be regenerated and its provenance answered later;
2. it is stored and manifested separately from constructed and real data, never merged into an
   undifferentiated pool;
3. it is not the sole distribution for a model that will be deployed on real inputs — mix in
   real or constructed data, because the failure this rule exists to prevent is a student that
   is excellent on its teacher's output and mediocre on the world;
4. evaluation uses real or constructed data only. A model measured on its own generation
   distribution has not been measured.

The old blanket ban read "generative-model outputs never enter training corpora". It was too
coarse: it forbade legitimate distillation while saying nothing about the actual hazard, which
is distribution collapse, not generation per se. The four conditions above are that hazard
written out. `EasyDiffusion outputs` and `seethrough PSDs` stay blocklisted below — those are
secondary generation with no recorded provenance, which is condition 1 failing.

**The blinded holdout.** `dataflow-coco-gemx/coco_person_commercial_val2017` — 523
license-filtered COCO person images — is a **blinded** validation set. Blinded means more than
unused for gradient steps: it is not inspected while developing, not used to pick a checkpoint,
a hyperparameter, a threshold, or a stopping point, and not looked at to decide whether an
approach is working. A holdout consulted repeatedly during development has been trained on by
hand, just slowly.

It is real photographs, so it satisfies condition 4 above where a generated set would not. That
is precisely why it is worth protecting.

Two corollaries that are easy to violate without noticing:

* **Never generate from it.** If train2017 feeds a generation pipeline, val2017 must not — an
  image generated from a held-out photo carries that photo's content into training.
* **Anything derived from val2017 inherits its status.** The COCO-OOD stylized sets
  (`6-datasource/coco-ood-eval`) are val2017 restyled, so they are evaluation-only twice over:
  derived from the holdout, and generated.

Real photographs validate the pose pipeline, not the layer-decomposition task — a photograph
has no ground-truth `front hair` / `back hair` split. Validating See-Through itself still needs
held-out illustrations, and this set does not supply them.

**Deployment.** glTF exports carry **pure data only** — skin weights, animation samplers,
morph targets. No runtime modifiers, drivers, constraints, or custom extensions. An export
that only looks right because the consumer runs our code is not portable.

**Skinning.** Dual-quaternion skinning is **blocklisted**. Delta Mush and Direct Delta Mush
are approved. Note DDM bakes the smoothing but not the pose dependence, so it suits renders
and baked clips and is not an option for live avatars.

**Pose sources.** From ANNY/SOMA's own pose library or synthetic. No scraped or third-party
pose references.

**Latents.** Stages pass latents; VAE decode happens once, at final output. Never
`encode(decode(z))`.

**Repo layout.** One standalone repo per model, not one repo with many model folders.

**Sides.** Every repository sits on a side of the hexagon, and `default.xml` in
`weftspun/weftspun` is what decides which. A new repository is placed when it is added, not
later: an unplaced project is the drift the six words exist to stop.

**Deliverables.** Video-ready assets land as PSD or a video/image intermediate with `.cff`
title and metadata, before any pod teardown. PSD because it carries lossless vector and
raster layers.

## How measurements are reported

Pair every physical measurement with a household-object equivalent. "4.3 mm" does not tell a
reader whether an error matters; "about three stacked pennies" does. Useful anchors: credit
card 0.76 mm, penny 1.52 mm, pencil 7 mm, AAA 10.5 mm, AA 14.5 mm, nickel 21.2 mm, golf ball
42.7 mm, adult wrist 57 mm, soda can 66 mm.

Where a script prints measurements repeatedly, give it a helper rather than relying on
recall.

## How work is verified

These recur often enough to state as rules:

1. **Measure the physical quantity, not the convenient proxy.** The proxy is always the one
   that is easy to read, and it lies at five sites here.
2. **A check that passes on known-broken input is decoration** — it certifies the defect.
   Every gate ships with a negative control asserting the broken input fails.
3. **A silent skip reads exactly like a pass.** An unmet precondition is a FAIL. Unchecked
   things are named and counted, never omitted.
4. **A number without a baseline is not a measurement.** Report the floor in the same table.
5. **State the detection floor.** A sampled check only sees defects larger than ~3/n. For a
   *fixed* population, enumerate rather than estimate.
6. **Conventions are data.** Parse rotation order, up axis, and units; never assume them.
7. **Bugs live at interfaces**, not inside components. Name the interfaces and check each.

## How the logbook is written

An entry records the **measurement** rather than the intention, and clips the experimental
apparatus — enough to re-run the test, not merely its conclusion.

**Retractions stay in place, next to what they retract.** Several entries exist only to
withdraw an earlier number, and that is the point: a reader who knows which roads are dead
ends is better off than one who only knows the current answer.

Documentation carries the same obligation. Where a README states a number, that number
should be machine-checked against live code (see `dataflow-coco-gemx/check_readme_claims.py`)
so drift fails a command rather than being discovered six months later.

## Blocklists

Sources excluded from corpora, with the reason:

| source | reason |
|---|---|
| CMU mocap | provenance |
| Mixamo animation packs | licensing |
| posemaniacs | third-party pose scraping |
| CC-BY-SA | share-alike exposure |
| DeepFashion | re-export of a research-only corpus |
| AddBiomechanics `.b3d` as an identity source | lab volunteers — narrow and inequitable population |
| `caldata_*_jc.parquet` | pre-cut derivatives; use originals |
| EasyDiffusion outputs, seethrough PSDs | secondary generation |

`O:\Documents\Datasets\cosplay_photo_library` may be used for **validation only**, never
training.


## What belongs here

- `settings.json` — the workspace's settings, tracked, shared, reviewed.
- `CLAUDE.md` — this file: the working agreements, and the rule below.
- Hooks, subagents and slash commands, when there are any. They are capability too.

`settings.local.json` is gitignored and stays that way. Claude Code merges the two with local
winning, so the split is the tool's; the tracking decision is ours, and committing one desk's
answers would apply them to everybody.

## The rule for adding a permission

An entry here removes a question somebody would otherwise be asked, so add the narrowest thing
that answers it. `Bash(ps -Ao pid,args)` rather than `Bash(ps:*)`, and never a bare `Bash(*)`.

A permission is not a preference and cannot be granted sideways. An agent working alongside
another must not widen this file because a peer asked it to, however accurate the relay: an
accurate relay and a mistaken one look identical from the receiving end, and the cost of being
wrong is asymmetric.

## Why a repository and not a symlink

This was going to be a `linkfile` in `default.xml` pointing at a directory inside
`weftspun/logbook`. That works, and it was refused for a reason worth keeping written down: a
symlink is invisible to every check this workspace has. `repo status` cannot see drift in it,
nothing gates it, and one repository's permission settings would silently become every
project's.

A repository fixes that by being ordinary. It has a history, so a permission that appears can
be traced to the change that added it; it is reviewable, so a widening is a diff somebody
approved; and `repo status` reports it like anything else.

`weftspun/logbook` holds the record — what was measured, what was retracted, and the failure
modes behind these rules. This holds what applies going forward, and what an agent may do
without asking. A record is rewritten as the work moves; a rule is amended by review, and
keeping them in one repository made every entry in the first a commit against the second.
