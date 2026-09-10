---
name: release-both-sides
description: Cut a release on both Era repos — agent side (zipwp-credits-saas, this repo) and plugin side (zip-ai) — from their staging-final vs master diff. Writes the agent-side CHANGELOG.md entry, writes the plugin-side readme.txt changelog + runs grunt bump-version, PRs that into staging-final, and immediately (no waiting for that to merge) also opens a second staging-final → master PR titled with the release version, in each repo. Use when the user says "cut a release", "ship staging-final to master", "bump version and changelog both sides", or "release both repos".
---

# Release both sides (agent + plugin)

Runbook for cutting a release across both Era repos. Both repos release the
same way: `staging-final` accumulates work, a changelog entry lands on
`staging-final` via its own PR, then `staging-final` is PR'd into `master`
and merging triggers the automated release (agent side:
`.github/workflows/release.yml` tags off the top `CHANGELOG.md` entry;
plugin side: WP.org deploy off `readme.txt` `Stable tag`).

Repo paths — confirm, don't assume (they move):
- **Agent side**: this repo (`zipwp-credits-saas`). Path: current working directory.
- **Plugin side**: `zip-ai` (memory note `project-era-repo-locations`; last known
  `/Users/harish/Local Sites/era-dev/app/public/wp-content/plugins/zip-ai`).
  Verify it still exists before using it; ask the user if it doesn't.

**Two PRs per repo, in order** — never commit the changelog/version bump
straight onto `staging-final`:
1. A `chore/changelog-<version>-staging-final` branch (matches this repo's
   existing branch-naming convention, e.g. `chore/changelog-0-0-8-staging-final`)
   cut from `staging-final`, carrying the changelog entry (+ plugin version
   bump). PR'd **into `staging-final`**.
2. Immediately after, without waiting for PR #1 to merge, a second PR
   **`staging-final` → `master`**, titled with the release version (e.g.
   `Release 0.3.0`). This is the PR that actually ships the release when
   merged.

Run the two repos independently — a release doesn't have to touch both if
only one side has changes. Check each for an actual diff before doing any
work on it (Step 1).

## Step 0 — Scope

Ask the user only what you can't determine yourself:
- Confirm the plugin repo path if the known one is stale.
- Whether to release both repos or just one (default: whichever has an
  actual `staging-final` vs `master` diff — see Step 1).

## Step 1 — Check each repo for unreleased changes

For each repo:

```
git -C <repo> fetch origin master staging-final
git -C <repo> log origin/master..origin/staging-final --oneline
```

Empty output → nothing to release on that side; skip it (tell the user why).
Non-empty → that's the change set for the changelog entry. Also run
`git -C <repo> diff origin/master...origin/staging-final --stat` to see
which files moved — useful context for writing the entry, not something to
paste in raw.

Check for existing open PRs before doing any work, and stop + tell the user
instead of duplicating:
```
gh pr list --repo <org>/<repo> --base staging-final --head "chore/changelog-*"
gh pr list --repo <org>/<repo> --base master --head staging-final
```

## Step 2 — Determine the next version

Agent side: read the top `## [X.Y.Z]` heading in `CHANGELOG.md` — that's the
current version.

Plugin side: read `Stable tag:` in `readme.txt` (or `version` in
`package.json`, they're kept in sync by `grunt bump-version`) — that's the
current version.

**Ask the user for the bump type (patch/minor/major) or the exact version**
unless the commit log makes it unambiguous (e.g. every commit is `fix:` →
patch). Don't silently pick — semver judgment calls (is this a breaking
change? does a beta plugin even follow strict semver here?) belong to the
user. Each repo can land on a different version; they don't need to match.

## Step 3 — Draft the changelog entries

Summarize the commit log from Step 1 into user-facing entries — not raw
commit messages. Group and word them the way existing entries already do;
read the last few entries in each file first and match tone/format exactly.

**Agent side** (`CHANGELOG.md`): Keep a Changelog format, section headers
`### Added` / `### Changed` / `### Fixed` (only include sections that have
content). New entry goes at the very top of the list, above the previous
`## [X.Y.Z]`:

```
## [<new-version>] - <YYYY-MM-DD>

### Added
- ...

### Changed
- ...

### Fixed
- ...
```

**Plugin side** (`readme.txt`, under `== Changelog ==`): match the existing
`New:` / `Improvement:` / `Fix:` prefix convention exactly (see recent
entries — the repo has history of PRs correcting entries that used the wrong
prefix, e.g. "Fix:" not "Bugfix:"). New entry goes at the top of the list:

```
= <new-version> - <Month DD, YYYY> =
- New: ...
- Improvement: ...
- Fix: ...
```

**Show both drafted entries to the user and get explicit approval before
writing anything to disk.** These are user-facing release notes — wording
matters and the user should see it before it ships.

## Step 4 — Write the changes

Once approved:

1. Agent side: insert the new `## [X.Y.Z]` block into `CHANGELOG.md` (Edit
   tool, don't touch anything else in the file).
2. Plugin side:
   - Insert the new `= X.Y.Z - ... =` block into `readme.txt`'s
     `== Changelog ==` section (Edit tool).
   - Run the version bump: `cd <plugin-repo> && grunt bump-version
     --ver=<X.Y.Z>`. This rewrites `package.json` version, the
     `ZIPAI_MCP_VERSION` constant in `loader.php`, the `Version:` header in
     `zip-ai.php`, `version.json`, and the `Stable tag:` line in
     `readme.txt` — all from the single `--ver` flag, so don't hand-edit
     those.
3. Show the user `git -C <repo> diff` for each repo (or the relevant files)
   so they can see exactly what's about to be committed.

## Step 5 — PR #1: changelog branch → staging-final

Per this user's standing rule: **never `git add`/commit without explicit
approval**, and pushing + opening a PR are visible, hard-to-reverse actions
— confirm before each.

For each repo, once the user approves the drafted entry:

```
git -C <repo> checkout -b chore/changelog-<version-dashed>-staging-final origin/staging-final
git -C <repo> add CHANGELOG.md                    # agent side
git -C <repo> add readme.txt package.json loader.php zip-ai.php version.json   # plugin side
git -C <repo> commit -m "chore: changelog + version bump for <X.Y.Z>"
git -C <repo> push -u origin chore/changelog-<version-dashed>-staging-final
gh pr create --repo <org>/<repo-name> --base staging-final \
  --head chore/changelog-<version-dashed>-staging-final \
  --title "chore: changelog for <X.Y.Z>" \
  --body "Changelog entry (+ version bump on the plugin side) for the <X.Y.Z> release."
```

`<version-dashed>` = the version with dots as dashes, matching this repo's
existing convention (`chore/changelog-0-0-8-staging-final`).

Use the actual GitHub org/repo slugs (`git -C <repo> remote get-url origin`
if unsure — don't guess).

## Step 6 — PR #2: staging-final → master, titled with the release version

Open this right after PR #1, without waiting for PR #1 to merge — GitHub PR
diffs are live comparisons, so PR #2 (`staging-final...master`) picks up the
changelog commit automatically once PR #1 merges into `staging-final`; no
need to gate one on the other.

```
gh pr create --repo <org>/<repo-name> --base master --head staging-final \
  --title "Release <X.Y.Z>" \
  --body "<plain-English summary of what's shipping, per CLAUDE.md PR-description convention: (1) what was happening, (2) what this release changes>"
```

This is the release PR — merging it fires the automated release pipeline
(tag + GitHub Release on the agent side, WP.org deploy on the plugin side),
so confirm with the user before creating it, same as any other push/PR.

## Step 7 — Add reviewers to all four PRs

Once every PR exists (both repos, both PRs each), add the standing release
reviewers to each one:

```
gh pr edit <pr-number> --repo <org>/<repo-name> \
  --add-reviewer rohit-patil-lifecode,patilvikasj
```

- `rohit-patil-lifecode` = Rohit Patil
- `patilvikasj` = Vikas Patil

Do this **after** all PRs are raised, not as each one is created — reviewers
get a notification per PR, so batching keeps it to one pass. If a PR was
reused rather than newly created (e.g. an existing `staging-final → master`
PR that was retitled), it still needs the reviewers added.

`--add-reviewer` fails if a listed user is already a reviewer on that PR;
that's harmless — check the current reviewers with `gh pr view <n> --repo
<repo> --json reviewRequests` and only add the missing ones.

## Step 8 — Report

End with a **paste-ready block the user can drop straight into Slack** —
this is the deliverable, not a prose summary. Exact shape, per side: a
`<Side> Side : -` label, then both PR URLs together in one code block
(changelog PR first, release PR second), then the changelog entry that side
is shipping in its own code block.

````
Agent Side : -

```
https://github.com/brainstormforce/zipwp-credits-saas/pull/<changelog-pr>
https://github.com/brainstormforce/zipwp-credits-saas/pull/<release-pr>
```

```
## [X.Y.Z] - YYYY-MM-DD

### Added
- ...

### Fixed
- ...
```

Plugin Side : -

```
https://github.com/brainstormforce/zip-ai/pull/<changelog-pr>
https://github.com/brainstormforce/zip-ai/pull/<release-pr>
```

```
= X.Y.Z - Month DD, YYYY =
- New: ...
- Improvement: ...
- Fix: ...
```
````

Bare URLs on their own lines inside the code block — no markdown link
syntax, no bullets, no titles. The changelog blocks are the entries
verbatim as written to disk.

After the block, in plain prose: confirm reviewers were added, note which
side(s) had nothing to release, and remind the user PR #2 needs PR #1
merged first for its diff to include the changelog. Don't merge any PR —
that's the user's call, not something this skill does automatically.
