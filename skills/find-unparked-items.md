---
name: find-unparked-items
description: Survey the workspace for RFDs and tasks that are actively being worked on — not shipped, not shelved, not abandoned — so the operator can tell the queue apart from the backlog. Reads the task list, greps RFD READMEs for the Shelved body paragraph, and cross-references against the merged-PR list. Use when the operator asks "what's unparked", "what's active", "what's left", "zoom out", or "what's the queue look like".
tools: Read, Grep, Glob, Bash
---

# Find unparked items

"Parked" is not a canonical RFD 1000 state (RFD 1000 lists prediscussion,
ideation, discussion, published, committed, abandoned, moved — no
`parked`). The workspace convention is: an RFD stays at `State: discussion`
and carries a `Shelved YYYY-MM-DD:` body paragraph after the metadata
block, naming the unblock condition. Task-list items carry a
`[PARKED · <reason>]` subject prefix and metadata `parked: true` with an
`unpark_when` field, or are deleted from the list when the operator wants
a truly empty queue.

An **unparked** item is one that is either in the task list without the
`[PARKED]` prefix, or an RFD at `State: discussion` with no `Shelved`
paragraph. That is the queue.

## When to invoke

The operator asks about queue state — "what's unparked", "what's left",
"what's active", "what's the task list", "zoom out" (for the queue part
of the answer), or "is anything blocking budget X".

## Where to look

**Task list** — call `TaskList` (or read the injected task-status reminder
if one is already present). Subject lines with `[PARKED · ...]` are
parked; without it, unparked.

**RFD READMEs** — `2-contract/manuals-weftspun/rfd/<N>-<slug>/README.md`.
An unparked RFD has `State: discussion` (or a further-along canonical
state) AND no `Shelved YYYY-MM-DD:` paragraph in the body. A shelved RFD
carries both.

## Pipeline

**1. Task list.** Show the pending tasks not prefixed with `[PARKED`.
Grouped by parent RFD OID when the subject carries one
(`urn:oid:1.3.6.1.4.1.66606.1.2.NNNN`).

**2. RFD survey.** Fast one-liner:

    cd 2-contract/manuals-weftspun
    for rfd in rfd/2*-*/README.md; do
      grep -q "^\*\*State:\*\* discussion" "$rfd" || continue
      grep -q "^Shelved [0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}:" "$rfd" && continue
      echo "$rfd"
    done

That prints RFDs at discussion state with NO shelved paragraph. Filter
further to just the RFDs the current operator opened this session by
cross-referencing PR authorship (`gh pr list --author '@me' --state all`
against the merged-PR list).

**3. Cross-reference with merged PRs.** An RFD in the survey with a
merged PR for it in the last session (or with a `State: committed`) is
delivered, not unparked. Distinguish "unparked and delivered" from
"unparked and open".

**4. Report by tier.** Group unparked items by the tier they occupy in
the current session's dependency chain (foundation → fix → port → ship →
Rung 2). See [[register-rfd-under-pen-oid]] for the OID structure that
makes cross-referencing possible.

## Rules that catch failure modes

- **`parked` is not a canonical state.** If you find an RFD with `State: parked`, that is a drift bug — flag it, do not treat it as a legitimate parked signal. The correct signal is `Shelved YYYY-MM-DD:` in the body with `State: discussion`.
- **Task-list `[PARKED · ...]` prefix trumps the RFD survey.** An operator who wants a truly empty queue deletes parked tasks from the list; the RFD stays shelved in prose. Don't resurrect a deleted task by re-reading its RFD.
- **The task list can drift from the RFDs.** An earlier version of the workspace amend on PR #162 landed only 1 of 5 park edits; the task list flagged all 5 as parked while the RFD READMEs still said `discussion` with no shelved note. That's a real class of drift — always cross-check both directions.
- **Session ownership matters.** An unparked RFD you did not open belongs to whoever opened it. Report it, don't act on it.
- **`gh pr view` shows the truth about a PR that says MERGED.** Auto-merge can fire on an amend-in-flight; the merged commit may not include your last force-push. When reporting on delivered work, check the merged commit's actual diff, not the local branch you thought you pushed. See the RFD 2167 incident in the session log.
