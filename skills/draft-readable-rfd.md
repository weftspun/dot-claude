---
name: draft-readable-rfd
description: When writing or editing an RFD README, apply three read-cold rules so a fresh reader can understand the document without opening any other RFD. Invoke whenever you draft a new RFD, edit an existing one, or supersede one — the rules gate readability, not just length.
tools: Read, Write, Edit
---

# Draft a readable RFD

An RFD is read cold. A reader arrives at `rfd/2182-*/README.md`
with no prior tabs open, no memory of neighbouring RFDs, no
workspace-private vocabulary. Three rules make that reader's first
pass carry the answer.

## The motivating failure

RFD 2182 as it landed opens like this:

> RFD 2171 fixed pipeline and deliverable vocabulary but left three
> L3 gaps: who runs the atelier-workshop, who receives the shuttle,
> and which market.

A fresh reader hits four unknowns in two lines — `RFD 2171`,
`pipeline and deliverable vocabulary`, `L3`, `atelier-workshop`,
`shuttle` — with no gloss for any of them. The Decision is buried
below a State/Feature/Scope block and a five-line Problem
paragraph that spends most of its budget on the meta-history of an
earlier draft. This skill exists because that shape is systemic,
not a 2182-specific slip.

## The three rules

### 1. BLUF first

After the State block (State / Feature / Scope, still permitted),
the first prose line under `## Decision` is **one sentence** that
states the decision. Problem and detail follow. A reader who reads
only that sentence has read the answer.

**Bad** (2182 as shipped):

```
## Problem

RFD 2171 fixed pipeline and deliverable vocabulary but left three
L3 gaps: who runs the atelier-workshop, who receives the shuttle,
and which market. [...]

## Decision

- **Operator**: chibifire.com [...]
```

**Good**:

```
## Decision

chibifire.com runs the pipeline, V-Sekai is its co-founded partner
project, and the market is avatar-first social VR.

## Problem
[...]
```

The one-sentence line goes first. Bullets, lists, and elaboration
follow it, not the other way round.

### 2. Cross-refs carry a three-word gloss on first use

Every `RFD NNNN` reference in prose (or `urn:oid:...`) takes a
short parenthetical the first time it appears in the file. Bare
numbers hide the referent; the reader has to leave the page.

**Bad**: `RFD 2171 fixed ...`, `See RFD 1106 for ...`,
`urn:oid:1.3.6.1.4.1.66606.1.1.2171`.

**Good**: `RFD 2171 (atelier-workshop vocabulary) fixed ...`,
`RFD 1106 (open/proprietary boundary)`, `RFD 2136 (gacha ladder)`.

The gloss is two or three words, verbatim in every occurrence of
the referent across the register.

### 3. Workspace-private words gloss on first use

A term defined by another RFD carries a two- or three-word inline
gloss the first time it appears in each RFD. The gloss goes in
parentheses right after the term.

Terms this covers today (extend as the workspace grows):

| term | first-use gloss |
| --- | --- |
| atelier-workshop | (the pipeline) |
| shuttle | (the portable-character deliverable) |
| L1 / L2 / L3 | (Flight Levels: execution / coordination / strategy) |
| MaskScore | (edit-reward corpus) |
| gacha ladder | (10-rung generation pipeline) |
| ETNF | (Essential Tuple Normal Form: no nulls, satellites) |
| ANNY | (the reference rig) |
| PEN OID | (Private Enterprise Number arc) |

A subsequent RFD may use the bare term after its first-use gloss
in that file. This is per-RFD, not per-register: readers land on
one file cold, not the whole set.

## What this skill does not fix

- The 40-line cap on `README.md` (RFD 1000). Move detail to
  `DETAILS.md` if the rules above push the file over.
- Cross-references *between* RFDs — the linker check
  (`check_rfd_links.py`, if it exists) is a separate concern.
- Retraction narratives inside CLAUDE.md, PITFALLS.md,
  KEYPOINTS.md — those are working-agreement documents where
  meta-history is load-bearing.

## When to invoke

- Drafting a new RFD.
- Editing an existing RFD's `README.md` in a substantive way (not
  a state flip or a one-line fix).
- Superseding an RFD.
- Reviewing an RFD PR — apply the three rules as a checklist.

## Verifying readability by hand

The three-question self-test for a finished draft:

1. Cover the rest of the file. Does the first prose line under
   `## Decision` answer the RFD's question?
2. Count the RFD-number references. Does each first occurrence
   carry a three-word gloss?
3. List the workspace-private terms used. Does each first
   occurrence carry an inline gloss?

Three yeses ship. Any no is a rewrite before merge.
