# .claude

The agent configuration for the weftspun workspace, as a repository. Checked out at `.claude/`
at the workspace root, which is where Claude Code looks, so it applies to every project on
every side of the hexagon.

```sh
repo sync .claude
```

| | |
|---|---|
| `CLAUDE.md` | the working agreements, and the rule for adding a permission |
| `settings.json` | the workspace's settings: tracked, shared, reviewed |
| `settings.local.json` | one desk's answers. Gitignored, and stays that way |

## Why a repository

It was going to be a `linkfile` into `0-infrastructure/logbook`, and that was refused: a
symlink is invisible to every check here. `repo status` cannot see drift in it, nothing gates
it, and one repository's permission settings would quietly become every project's.

A repository is ordinary instead. Permissions arrive in a diff somebody approved, a widening
has a commit behind it, and `repo status` reports this like anything else. The manifest entry
is `name="dot-claude" path=".claude"`. The path is fixed by Claude Code, which reads that and
nothing else, so the name gave way: a repository called `.claude` is hidden from an ordinary
listing and clones into a directory most shells will not show you.

## Why not in the manifest repository

`CLAUDE.md` was in `weftspun/weftspun` and was read there, because `meta` put that checkout at
the workspace root. `repo` does not — it clones the manifest to `.repo/manifests`, where
nothing reads upward from. The file would have kept parsing, kept being committed to, and
stopped applying, with no error anywhere — the same shape as the failure the manifest's own
comment records about `path="."`.

`0-infrastructure/logbook` holds the record: what was measured, what was retracted, and the
failure modes behind these rules. This holds what applies going forward. `CLAUDE.md` says why
the two are separate.
