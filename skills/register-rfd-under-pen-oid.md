---
name: register-rfd-under-pen-oid
description: Allocate the next RFD serial in SERIALS-vsekai-fabric.usda under Weftspun's PEN arc, create the rfd/<N>-<slug>/ directory with a bounded README carrying the AI-drafted attestation canary and the urn:oid spine, and verify the pair with the anti-entropy check. Use when a new RFD needs a number, when subdividing an existing serial (2164.N.M), or when backfilling a previously ad-hoc numbering.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# Register RFD under PEN OID

Weftspun's Private Enterprise Number is `1.3.6.1.4.1.66606`. Site 2
(`v-sekai-fabric`) is category 1 (`documents`), so an RFD's full arc is
`1.3.6.1.4.1.66606.1.2.<serial>[.N.M]`, and it is quoted in prose in the
RFC 3061 URN form `urn:oid:1.3.6.1.4.1.66606.1.2.<serial>`.

## When to invoke

The user asks for a new RFD, asks to subdivide an existing serial into
`.N` or `.N.M` sub-rungs, or points out an RFD that landed under an
ad-hoc number and needs a real serial. Also invoke when a related
document needs to cite an RFD by identifier and the identifier does not
yet exist.

## Where the register lives

`2-contract/manuals-weftspun/SERIALS-vsekai-fabric.usda`. The `arc`,
`pen`, `site`, `category` fields at the top of the file are the source
of truth for the OID shape. Read them before assuming.

New RFDs at this site get `2NNN` serials, per the reactivated-2026-08-31
note in the file's `customLayerData`.

## Pipeline

**1. Read the register.** Open `SERIALS-vsekai-fabric.usda` and find
`Scope "Allocated" > "Rfd"`. The highest existing `def "S<N>"` is the
last allocated serial. The next one is `S<N+1>`.

**2. Append the allocation.** Add:

    def "S<N+1>"
    {
        custom string slug = "<kebab-case-slug>"
    }

The slug follows the directory name so that a retitle renames the
document without renumbering it. Never reuse or renumber; a serial is
the last arc of an OID and it names one document forever.

**3. Create the RFD directory.** `2-contract/manuals-weftspun/rfd/<N>-<slug>/README.md`.

The README is bounded at 40 lines by
`request-for-discussion/scripts/check-rfd-structure.py`. Structure:

    # RFD <N>: <title>

    **State:** discussion
    **Feature:** <one-sentence>
    **Scope:** <path or paths>

    ## Problem
    ...

    ## Decision
    ...

    ## Related

    Spine: urn:oid:1.3.6.1.4.1.66606.1.1.<parent-serial>
    Satellite of: urn:oid:1.3.6.1.4.1.66606.1.2.<other-serial>   # optional

    This RFD was drafted by an AI and read by a human before it shipped.

The last sentence is the AI-drafted attestation canary from CLAUDE.md
and `check_rfd_canary.py` gates on it. Verbatim. A drafted-by-a-human
RFD uses `This RFD was drafted by a human without AI help.` instead.

**4. Cite by URN.** Use the full `urn:oid:` form in prose. Sub-rungs
extend with dot-arcs (`urn:oid:1.3.6.1.4.1.66606.1.2.2164.3.2` is
"RFD 2164 sub-rung 3.2"). Never abbreviate to bare `.2164` or invent
sitewise shortcuts.

**5. Run the anti-entropy check.**

    pixi run python scripts/check_anti_entropy.py

It enumerates the serial register against the on-disk directories under
`rfd/`. A serial with no directory and a directory with no serial both
fail. Read what it says, not the last line.

**6. Commit both files in the same commit.** The register entry and the
directory land together, so a `git bisect` never lands on a state with
one and not the other.

## Rules that catch failure modes

- **The `arc`, `pen`, `site`, `category` fields in `SERIALS-vsekai-fabric.usda` are the source of truth.** Do not hardcode `1.3.6.1.4.1.66606.1.2` into scripts; read it from the register.
- **Sub-rungs are dot-arcs on the serial**, not slashes and not colons: `urn:oid:1.3.6.1.4.1.66606.1.2.2164.3.2`, not `.../2164/3/2`.
- **PEN category comes before site**, not after: `<pen>.<category>.<site>.<serial>`. An earlier version of this pipeline had `<pen>.<site>.<category>.<serial>` and the user corrected it.
- **The README canary is verbatim**, and a misspelling is rejected by `check_rfd_canary.py`'s self-test control. Copy-paste it.
- **Slug and directory match.** `SERIALS.usda`'s `slug` field must equal the `<slug>` in `rfd/<N>-<slug>/`. The anti-entropy check enumerates both.
- **A README over 40 lines fails the structural gate.** Trim to fit — cite DETAILS.md if the argument needs more room.
- **A commit with only the serial or only the directory** leaves the register in a drift state that neither half of the pair notices on its own; the anti-entropy check does, but only when it is run.
