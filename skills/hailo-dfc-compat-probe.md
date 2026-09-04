---
name: hailo-dfc-compat-probe
description: Check whether an ONNX model parses on the Hailo Dataflow Compiler for a target Hailo hw_arch, before anything else in a compile-to-HEF pipeline runs. Uses the pre-built `weftspun-hailo-dfc:latest` container and the workspace's `scripts/gate_dfc_parse.py` pattern. Reports pass or names the failing op/format via the traceback. Use when adding a new model to a Hailo-target project (rf-detr, YOLO, ViT variant), when a compile pipeline hits a parser error, or when auditing an ONNX export against DFC before HERO burns training compute on a checkpoint DFC will reject.
tools: Bash, Read, Write, Edit, Grep, Glob
---

# Probe an ONNX against Hailo DFC

The Dataflow Compiler runs only on Linux x86-64 under a specific Python +
apt-package set. A local Docker image is the practical way to poke it from
the workspace host. This skill runs one ONNX through DFC's parser inside
the container and reports what happens.

## When to invoke

The user (or a peer agent) needs to answer "does the DFC accept this
graph" before further work depends on that answer. Common shapes:

- An ONNX export from a research model that has not been targeted at
  Hailo before (rf-detr, a new DETR variant, a custom-head model).
- A model an in-flight compile pipeline just rejected, and the pipeline
  stack only says "compile failed" without naming the parser reason.
- A pretrained checkpoint about to be handed off for QAT training (skip
  the training if the compile side already refuses the arch).
- An ONNX from `hailo_model_zoo` used as a positive control alongside a
  new candidate.

## What the probe answers

The DFC parser is a small fraction of what a full compile does. This
skill answers only the parse question. A green parse means the graph
translates to Hailo's internal HN representation and a HAR can be saved;
it does not yet answer whether optimize or compiler will succeed, and
does not name INT4 support. A red parse names the op or format pattern
DFC refuses, so the follow-up work is scoped.

## The container

Ships local as `weftspun-hailo-dfc:latest`. Built from
`3-interactor/rf-detr-cpp/deploy/hailo-dfc/Dockerfile` (Ubuntu 22.04 +
Python 3.10 + DFC 5.3.0 wheel + a small pruning pass on the Jupyter
stack). No accelerator needed; parse is CPU work. If the image is
missing:

    docker images | grep -i hailo-dfc

If nothing lists, `docker build` from the `deploy/hailo-dfc/` directory
after dropping the DFC wheel next to the Dockerfile. Container operator
account: `hailo`.

## The probe recipe

Layout the probe as three files in a scratchpad directory:

    <scratch>/model.onnx                   # the graph under test
    <scratch>/probe.py                     # runs DFC parse, prints result
    <scratch>/probe.log                    # full stdout+stderr capture

The `probe.py` shape:

```python
import onnx, traceback
from hailo_sdk_client import ClientRunner

P = '/work/probe/model.onnx'
m = onnx.load(P)
print(f'graph: {len(m.graph.node)} nodes, {len(m.graph.output)} outputs')
for i in m.graph.input:
    dims = [d.dim_value if d.HasField('dim_value') else d.dim_param for d in i.type.tensor_type.shape.dim]
    print(f'input {i.name}: {dims}')

try:
    r = ClientRunner(hw_arch='hailo10h')
    # If the ONNX input is fully static, drop net_input_shapes entirely.
    # If dynamic, pass concrete dims:
    #   r.translate_onnx_model(P, 'name', net_input_shapes={'input': [1, 3, 432, 432]})
    r.translate_onnx_model(P, 'name')
    r.save_har('/work/probe/name.hailo10h.har')
    print('PARSE OK')
except Exception:
    traceback.print_exc()
```

Then from the workspace host:

```sh
docker run --rm \
  -v "<scratch>:/work/probe" \
  weftspun-hailo-dfc:latest \
  /opt/dfc/bin/python /work/probe/probe.py *> "<scratch>/probe.log"
```

PowerShell needs `docker run ... *> file.log` to capture stderr into the
same file (docker mixes streams).

## Reading the output

Four possible verdicts in a single-file probe:

1. **`PARSE OK`** with a saved `.har`. Graph translates.

2. **`ValueError: channels is not in list`** at
   `onnx_translator/onnx_graph.py:444` in `update_reshape_output_format`.
   A Reshape received a tensor with a format DFC's tracker cannot map to
   NCHW. Usually downstream of an NLD-shaped tensor being rewrapped as
   spatial. Common in ViT graphs where the encoder outputs a flattened
   sequence and a projector reshapes it back.

3. **`IndexError: list index out of range`** at
   `onnx_graph.py:2463` in `_convert_axes_to_nhwc` under
   `_create_layer_normalization_layer`. A LayerNormalization node's
   `axes` attribute points beyond what the tracker knows. Often surfaces
   under the simplifier-retry pass, after the first attempt fails with
   `channels is not in list` — both errors name the same LN
   mishandling.

4. **Shape-mismatch broadcast error** at
   `element_wise_ops.h:560` in `BroadcastIterator::Append`. The
   `net_input_shapes` overrode a static ONNX input to a resolution the
   position embeddings cannot span. Fix by dropping the override or
   passing the ONNX's declared shape. Check via
   `[d.dim_value for d in m.graph.input[0].type.tensor_type.shape.dim]`.

## The retry pass changes what passes

DFC's parser tries a first parse; on failure it runs `onnxsim` and
retries, logging `[info] Simplified ONNX model for a parsing retry
attempt (completion time: MM:SS.ss)` and saving `<model>.sim.onnx`
next to the input. The retry often converts a `Transpose → LN →
Transpose` sequence into an atomic LN DFC accepts. A minimal PyTorch
repro of the pattern therefore passes on the retry, while the same
pattern in a whole graph stays red because the surrounding
Concat/Split/residual context blocks the simplification.

To isolate a failure component, use `onnx.utils.extract_model(src, dst,
input_tensors, output_tensors)` to write a standalone ONNX of just the
component, then probe the extract. Prefer this over
`ClientRunner.translate_onnx_model(..., start_node_names=[...],
end_node_names=[...])`, because DFC's `start_node_names` still walks
the ancestors of the requested end nodes and can fail during output-
layer construction on intermediate cuts (raw NLD tensors trigger
`IndexError: list index out of range` at the output boundary, which
looks like a real failure but is a probe-methodology artefact).

## Norm-layer node counts when a recipe asserts a specific placement

Some target recipes assert that a specific normalization op lives in a
specific part of the graph. Hailo's vit_base_bn recipe asserts every
encoder Norm is BatchNormalization; the recipe's DFC-parseability rests
on that assertion. A plain `PARSE OK` on such an incoming ONNX is not
enough — a BN-vs-LN node count says whether the recipe held.

Count both op types and slice by name-prefix for the region under the
assertion:

```python
import onnx, collections
m = onnx.load('/work/probe/model.onnx')
prefix = '/backbone/'   # or whatever the recipe scopes
ops = collections.Counter(
    n.op_type for n in m.graph.node
    if n.name and n.name.startswith(prefix)
)
bn = ops.get('BatchNormalization', 0)
ln = ops.get('LayerNormalization', 0)
print(f'{prefix}: BatchNormalization={bn}, LayerNormalization={ln}')
```

Reads for a vit_base_bn-recipe incoming ONNX:

- `BN > 0, LN == 0` in the encoder: recipe held, arch amendment
  landed. Compat probe verdict is a full pass.
- `BN > 0, LN > 0` in the encoder: recipe partial. Retrain kept
  some old LN shapes somewhere (a subgraph the amendment missed).
  Reads as a **recipe failure, not a probe failure**. Report the
  specific LN node names so the retrain can be corrected.
- `BN == 0, LN > 0` in the encoder: recipe not applied at all.
  Probe should still run the DFC parse, and the parse will surface
  the earlier `channels is not in list` / `IndexError` at
  `_convert_axes_to_nhwc` — that error's presence on a recipe-asserted
  BN ONNX is the same recipe-failure signal by a different path.

The same shape applies to any recipe that swaps one norm for another:
run the count, name the region, report which nodes escaped the swap.

## Gotchas

- `end_node_names` takes **node** names (`NodeProto.name`), not tensor
  names (`_output_0`). Passing a tensor name returns
  `MisspellNodeError: Unable to find end node name`, silently.
- Windows PowerShell's `Select-Object -Last N` on a docker pipe drops
  the earlier prints from the Python script; capture the full log with
  `*> file.log` and Read the head separately.
- Older DFC container versions may have API renames; `hw_arch` argument
  has been stable across 5.0-5.4, `translate_onnx_model` too.
- The DFC install lands in `/opt/dfc/` inside the container; the
  container's system Python is 3.10, and DFC's venv is at `/opt/dfc/bin/python`.

## Reference

Extended discussion in
`logbook-dfc-bisect-methodology-rf-detr.md`
(`weftspun/request-for-discussion`), which records the pattern
converged on during the RFD 2199 rf-detr investigation.
`weftspun/rf-detr-cpp` `scripts/gate_dfc_parse.py` is the reference
script that adds a macOS-gate cross-check on top of the plain parse.
