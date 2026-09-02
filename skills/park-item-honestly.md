---
name: park-item-honestly
description: Shelve an RFD or a task-list item when budget or capacity forces deferral, using the honest convention — RFD stays at canonical State (discussion, per RFD 1000; parked is not on that list) and carries a `Shelved YYYY-MM-DD:` body paragraph naming the unpark condition; task-list items carry a `[PARKED · <reason>]` subject prefix and metadata with an `unpark_when` field, or get deleted for a truly empty queue. Use when the operator says "park it", "shelve it", "no budget today", "defer", or names a specific RFD/task to hold.
tools: Read, Write, Edit, Grep, Bash
---

# Park an item honestly

The workspace convention has two layers:

1. **RFD README** — `State:` stays at a canonical RFD 1000 value
   (`prediscussion, ideation, discussion, published, committed,
   abandoned, moved`), and a body paragraph starting `Shelved
   YYYY-MM-DD:` records the deferral and the unpark condition.
2. **Task list** — subject prefix `[PARKED · <reason>]` with metadata
   `{parked: true, parked_reason, unpark_when}`. If the operator wants
   an empty queue, delete the task instead; the RFD carries the record.

**Do not write `State: parked`.** It is not on RFD 1000's list and the
check-rfd-structure gate rejects it. This bit me: 6 PRs shipped invalid
state values before the gate caught it (the gate had a DIR_RE that
silently skipped the entire 2xxx register — fixed in
weftspun/request-for-discussion PR #164).

## When to invoke

Operator asks to park, shelve, defer, or hold an RFD or task. Or asks to
mark "no budget today" on specific work.

## Pipeline

**1. Identify scope.** Which RFDs, which task-list items. A parent RFD's
sub-rungs are usually shelved together; explicit sub-rung tasks
inherit the parent's shelved state.

**2. Draft the shelved paragraph** for each RFD README:

    Shelved YYYY-MM-DD: <what was delivered, with PR reference if any>.
    <what remains>. <the unpark condition>.

Example (RFD 2162, 2026-09-02):

    Shelved 2026-09-02: 2162.1 (104-joint ANNY skeleton in USDZ) shipped
    in weftspun/anny-render-corpus#34. 2162.2 (mesh 15-edit expansion)
    needs render capacity currently unbudgeted; resume when local GPU
    frees.

**3. Place it AFTER the metadata block**, not inside it. Blank line
separates the preamble from the body. If you put `**Shelved:**` inside
the preamble, check-rfd-structure reads it as an unknown metadata field
and rejects it.

**4. Trim if needed.** The 40-line cap applies. Compress the Decision
section if the header block pushes it over. See
[[draft-rfd-readme-under-gates]] (unwritten as of 2026-09-02, but the
gates are check-rfd-structure and check_tropes).

**5. Update the task list.** For each parked task:

    TaskUpdate({
      taskId: N,
      subject: "[PARKED · <reason>] <original subject>",
      metadata: {parked: true, parked_reason: "<reason>",
                 unpark_when: "<condition>"}
    })

If the operator wants an empty queue, delete parked tasks instead — the
RFD carries the record and the task list stays a queue.

**6. Commit and PR.** One commit per parked-RFD wave, cite each RFD's
OID in the body, and land as one PR. Verify with:

    pixi run -e gate python scripts/check-rfd-structure.py

Should show 0 problems (or list only pre-existing ones out of scope).

## Rules that catch failure modes

- **Never write `State: parked`.** It is invalid. Use `State: discussion` and a body paragraph. See [[find-unparked-items]] for the rule and the reason.
- **Body paragraph, not preamble.** `**Shelved:**` inside the preamble parses as a bogus metadata field and check-rfd-structure rejects the RFD. A blank line before the paragraph moves it out of the preamble.
- **`gh pr merge --auto` races your amend.** If required checks are already green when you enable auto-merge, the merge fires on whatever commit is at the branch tip AT THAT INSTANT — a force-push seconds later doesn't land. When you amend after arming auto-merge, verify the merged commit's actual diff with `gh pr view <n> --json state` and `git show <merged-sha> -- <file>`. See RFD 2167 incident.
- **Squash may not be allowed.** Some repos disallow squash-merges (weftspun/request-for-discussion is one). `gh pr merge --auto --merge` (regular merge commit) is the fallback.
- **`gh pr merge` arg order matters.** `--repo` before the PR number silently no-ops; the PR number comes first: `gh pr merge <n> --repo <owner/repo> --auto --merge`.
- **Reconcile task list and RFD state before reporting.** If a task says PARKED but the RFD README doesn't carry a Shelved paragraph, either the RFD or the task is wrong. Fix the drift before the next zoom-out.
- **The unpark condition must be concrete.** "when time returns" is not concrete; "when local GPU frees for reward training" is. A vague condition means the item will not be picked back up because nothing will ever obviously trigger it.
