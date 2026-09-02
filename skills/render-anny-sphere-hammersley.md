---
name: render-anny-sphere-hammersley
description: Render an ANNY mesh at a Pixal3D sphere_hammersley_sequence camera distribution on the local Metal GPU via Mitsuba, emitting per-view PNG + AOV depth/normals sidecar + camera JSON + projected 2D keypoints, and score a candidate render against a reference via score_render_pair.py with identity and negative controls. Use when a new MaskScore stub needs multi-view coverage of an ANNY pose or a candidate edit, when adding a new view count to an existing render pipeline, or when a physical-quantity render metric is drifting.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# Render ANNY on sphere_hammersley

Views are picked from the `sphere_hammersley_sequence` camera distribution
(Pixal3D, Apache-2.0). A hand-picked front view showed error of five stacked
soda cans along the travel axis versus three and a half across it — the
CLAUDE.md rule "Views come from the `sphere_hammersley_sequence` camera
sequence" exists to keep that from recurring. Do not swap the camera
distribution for the convenience of a repeat framing.

## When to invoke

The user asks for a multi-view render of an ANNY mesh (rest, perturbed, edited),
adds a new view count to a stub, extends a candidate ladder with more views, or
asks for a physical-quantity comparison between a reference and candidate render.

## Compute

Local desktop GPU only (CLAUDE.md hard constraint — rented GPU is blocklisted).
Mitsuba `metal_ad_rgb` on macOS for speed; `llvm_ad_rgb` when a reproducible
frame is needed across machines (60x slower). Choose per artifact:

- Corpus renders: `metal_ad_rgb` — throughput matters and the seed still ties
  the sample sequence.
- A shipped fixture cited by an RFD or logbook entry: `llvm_ad_rgb` — the
  reader needs to reproduce the number.

## Pipeline

Reference implementation lives in `6-datasource/anny-render-corpus`, under the
`anny-mac` pixi env.

**1. Author the ANNY pose.** Write to an `.npz` with `verts`, `faces`,
`pose_soma` (78x3 float64 rotations), `translation`. `generate_bootstrap_poses.py`
is the reference: rest + `rank1` (identity) + `rank5` (rest + N(0, 0.05 rad)
per bone, seed 0, ~2.86°/axis). Same-seed reproduction is required by the
constructed-synthetic rule.

**2. Render N views.** Pixal3D methodology: 24 views is the default; 64 for a
walking skeleton that will feed keypoint training.

    pixi run -e anny-mac python render_bootstrap.py --views 24 --spp 16

Each view writes:

    view_XXX.png            color (sRGB)
    view_XXX.aov.npz        depth (float32 HxW) + normals (float32 HxWx3), unit cube
    view_XXX.json           camera sidecar: intrinsics, extrinsics, up axis, units
    view_XXX.keypoints.json 23 COCO body keypoints projected (via project_2d_keypoints.py)

The AOV sidecar is a separate `.npz` rather than a channel in the PNG, so
downstream scoring reads real float depth rather than an 8-bit proxy (rule 1
from CLAUDE.md: measure the physical quantity, not the convenient proxy).

**3. Project keypoints.** After each pose renders, run
`project_2d_keypoints.py <pose.npz> <renders_dir>` to write the
`.keypoints.json` sidecars per view. Uses the camera JSON — do not rederive.

**4. Score against a reference.**

    pixi run -e anny-mac python score_render_pair.py \
        --reference build/bootstrap/renders/input \
        --candidate build/bootstrap/renders/rank1 \
        --out build/scores/rank1.json --assert-identity

    pixi run -e anny-mac python score_render_pair.py \
        --reference build/bootstrap/renders/input \
        --candidate build/bootstrap/renders/rank5 \
        --out build/scores/rank5.json

Per-view metrics: `depth_l1` (scene units, unit cube), `normal_l1`,
`normal_dot` (1.0 = identical). Reads the AOV `.npz`, not the PNG.

**5. Assert both controls before publishing scores** (rule 2):

- Identity control: `rank1` is bit-identical to input (symlink at render time
  or same seed + same pose). `--assert-identity` fails a nonzero depth_l1.
- Negative control: `rank5` (or any perturbed candidate) has a mean depth_l1
  strictly greater than rank1's. If it is not, the metric is not detecting the
  perturbation the ladder is supposed to induce.

## Rules that catch failure modes

- **Never rebuild a view schedule.** `sphere_hammersley_sequence` from Pixal3D is the sequence; a hand-picked "front + 3/4 + profile" layout drops error along the travel axis by nothing while adding it across (five soda cans vs three and a half).
- **The AOV sidecar is the ground truth for depth**, not the PNG's alpha or a grayscale reduction. 8-bit depth is the "convenient proxy" rule 1 warns about.
- **Camera JSON travels with each view.** Downstream scoring reads intrinsics/extrinsics/up axis/units from it (rule 6: conventions are data, parse rather than assume). ANNY's `upAxis = "Z"` matches Mitsuba's default; do not assume the next asset does.
- **Both controls or no publish.** An identity check that passes on rank1 but no negative-control assertion certifies only that the metric returns zero on unchanged input, which is decoration (rule 2).
- **Metal for throughput, llvm for reproducibility.** Corpus-scale renders use `metal_ad_rgb`; a shipped fixture cited by an RFD uses `llvm_ad_rgb` so a reader on another machine gets the same numbers.
- **glTF export from an ANNY pose carries pure data only** (CLAUDE.md deployment rule). No runtime modifiers, drivers, constraints, custom extensions. If the render needs behaviour the consumer's runtime does not have, bake it into the mesh or the animation samplers before export.
- **Household-object equivalents in prose about the metric.** `depth_l1 = 1.79` scene units is opaque; against a unit-cube ANNY that is about the height of one adult wrist across the body. CLAUDE.md's "How Measurements Are Reported" rule holds for render numbers too.
