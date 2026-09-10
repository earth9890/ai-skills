---
name: pr-review
description: Review a pull request — or a pair of coordinated PRs across two repos — for correctness, layer-boundary violations, SSOT/reusability/overengineering, regressions and edge cases, security, and end-to-end feature-flow integrity. Use this whenever the user asks to "review a PR", "review this pull request", "check this PR before merge", or describes a change that spans two repos (e.g. the zip-ai plugin plus the zipwp-credits-saas brain/Laravel side) and wants both reviewed together. Trigger even when the user only pastes a GitHub PR URL and asks "what do you think", or when they say a feature "touches both the plugin and the backend" and needs reviewing as one flow.
---

# PR Review (single or cross-repo)

Review a PR the way a careful senior engineer would: understand the change, diff it against the right base, then judge it across several dimensions — with extra weight on **whether a new feature's flow actually holds together end to end**, especially when it spans two repos.

The distinctive value of this skill is the **cross-repo flow trace**. A single change often spans the React/plugin side (`zip-ai`, formerly `zipwp-mcp`) and the backend side (`zipwp-credits-saas`: Laravel billing/auth + Node brain + MCP tools). Reviewing each PR in isolation misses the bugs that live *between* them — a payload field the plugin sends but the brain never reads, an SSE event the client listens for but the brain never emits, a tool contract that drifted. Catching those is the point.

## Why reviews miss bugs (read this first — it shapes everything below)

Approvals that later blow up in production almost always fail the same handful of ways. This skill is built to defeat them; keep them in mind on every PR:

1. **Happy-path bias.** Reviewers verify the flow the author intended and stop. The bug lives in the path nobody traced: the timeout, the empty result, the second concurrent caller, the retry after a partial write. **You do not understand a change until you have tried to make it fail** — Step 3's failure-mode pass is not optional.
2. **Timing/concurrency can't be simulated in your head.** Code that reads "effectively atomic" stops being atomic the moment the DB pauses, a cache misses, or two turns race. Any read-modify-write, any shared Redis/DB/module state, any "check then act" is guilty until proven serial. Name the interleaving, don't eyeball it.
3. **Static reading hides runtime bugs.** A contract that type-checks can still drift at runtime (wrong field arrives, JSON is a string not an object, an event never fires). When you can't prove it by reading, say so and **run it** (Step 3.6) — a reproduced failure beats ten hypotheticals.
4. **Big diffs get rubber-stamped.** Counterintuitively, large PRs are approved *faster* because thorough feels too expensive. Resist it: chunk the diff, account for every changed file, and slow down where the logic is densest.
4b. **A green suite proves the happy path, not the guard.** Passing tests almost always exercise in-bounds, well-formed inputs — they say nothing about what a `.max()`, an `.enum()`, a length cap, or a required-field check DOES when it fails. "Tests pass" is not "the failure path is safe." Never let a green run stand in for tracing the reject/limit/empty path yourself (Step 3.a2's bounds bullet); the bug that ships is the input the tests never fed.
5. **Style crowds out logic.** Nits are easy to see and easy to type; logic bugs need work. Assume linters/formatters/CI own style and obvious smells — spend your budget on correctness, flow, and regressions, not whitespace.
6. **Unverified findings erode trust.** A confident wrong finding is as costly as a missed bug. Every finding gets adversarially verified (Step 3.5) with a concrete failure scenario before it ships.

## Step 0 — Establish scope (ask first, don't assume)

Before reading any code, confirm the shape of the review. Ask the user:

1. **Single PR or cross-repo?** "Is this one PR in one repo, or a coordinated change across two repos that should be reviewed as one flow?"
2. **PR link(s) — or a branch.** One GitHub PR URL for single; two URLs for cross-repo (label which is which — e.g. "plugin PR" vs "brain/backend PR"). If the user says *"both sides, same branch"* there may be no PR at all: resolve the **same branch name in both repos** (`git -C <path> rev-parse --abbrev-ref HEAD`), confirm both checkouts actually sit on it, and review `origin/<base>...HEAD` in each. Missing PR ≠ missing review — proceed off local branches.
3. **Local repo path(s) — always ask, for every repo involved.** You need the working copy on disk to read full files for context (the diff alone is not enough to judge correctness). For cross-repo, get both paths.

Do not guess paths from memory. The known repos and their roles live in the memory note `project_era_repo_locations` and in `CLAUDE.md`, but still confirm the actual paths with the user — branches and checkouts move.

## Step 1 — Pin the base branch for each repo

A diff is meaningless without the right base. For each repo:

1. Read the PR's declared base: `gh pr view <pr-url> --json baseRefName,headRefName,title,body,additions,deletions,changedFiles`.
2. Tell the user what you found and confirm: *"This PR merges `<head>` into `<base>`. Review the diff against `<base>` — or did you want a different base branch?"* The user explicitly wanted control here, so always offer the override. They may want to diff against `master`, `staging`, or some integration branch even if the PR targets something else.
3. Make sure the local checkout has the base ref. From the repo path:
   ```
   git -C <repo-path> fetch origin <base-branch> --quiet
   ```

## Step 2 — Get the diff and the context

For each repo, work from its local path so you can open whole files, not just hunks.

- **Authoritative diff** (what the PR actually changes):
  ```
  gh pr diff <pr-url>
  ```
  or, if reviewing a local branch not yet pushed as a PR:
  ```
  git -C <repo-path> diff origin/<base-branch>...HEAD
  ```
- **Full-file context:** open the changed files in the local checkout. Reading only the diff hides the function the change sits inside, the caller two files over, and the test that should have been updated.
- **Callers and tests for every changed symbol.** For each changed function/export, grep its callers across both repos and check whether a test covers it. A change is only understood once you know who calls it — a guard added in one caller usually means the sibling callers are still broken.
- If a code-knowledge-graph MCP is connected, use it for impact radius and `tests_for`; otherwise `git`/`rg`/file reads are the baseline and are sufficient.
- **Chunk a large diff — never scan it.** For anything past ~10 files or ~400 lines, list the changed files first and account for each one explicitly (reviewed / trivial / needs-trace). Large PRs are the ones most often waved through; a file you didn't name is a file you didn't review.
- **Reproduce the bug the PR claims to fix.** When the PR says it fixes something, find (or write in your head) the failing case, and confirm the change actually addresses *that* case — not an adjacent one. If a regression test was added, verify it would genuinely fail without the fix (revert the fix mentally and re-run the test in your head, or actually — Step 3.6). A "fix" with a test that passes both with and without the change fixes nothing.
- **Read the change twice: once as the author (does the intended path work?), once as an adversary (how do I break it?).** The second read is where the bugs are.

## Step 3 — Review dimensions

**This is a bug-finding review first.** The job is to find what breaks — correctness, failure paths, and cross-repo contract drift. The other dimensions (SSOT/reuse/overengineering, security) are *brief optional passes*: run them only when the change actually touches that surface, and keep them to a line or two. Do not turn a review into a full-codebase quality audit — spend the budget on bugs.

The core dimensions (always) are **a + a2** (correctness & failure-mode) and **c** (cross-repo flow), plus **b** (layer boundaries) whenever the change touches a boundary. Dimensions **d + e** are optional. State which you covered.

### a. Correctness bugs (core)
Logic errors, off-by-one, wrong operators (`<` vs `<=`), unhandled null/empty, broken edge cases, regressions in existing behavior, broken or missing tests. For each: what breaks, under what input, and the file:line. State the finding as a **failure scenario** — concrete inputs/state → the wrong output or crash — not as "this looks risky".

Enumerate the callers of every changed signature/return shape and confirm each still holds — a guard added in one caller usually means the siblings are still broken. A missing test for a case the change made reachable is itself a finding.

### a2. Failure-mode / "make it fail" pass (core — where bugs actually hide)
Trace the un-happy paths explicitly — this is the single most productive pass and the one reviewers skip:
- **Error propagation.** Follow every `throw`/reject/`WP_Error`/non-2xx to where it's caught. Is it swallowed (empty `catch`, `catch {}`, `.catch(() => {})`), logged-and-ignored, or surfaced? A caught-and-dropped error is a silent failure — the operation "succeeds" having done nothing. Does a `finally`/cleanup run on the error path?
- **Partial failure & idempotency.** If the change does N writes (form post + content patch, design token + snapshot, several MCP calls), what state is left when write 3 of 5 fails? Is a retry safe, or does it duplicate/corrupt? Does it roll back or fail-forward?
- **Concurrency & ordering.** Any check-then-act, read-modify-write, or shared Redis/DB/module/global state: name the two-caller interleaving that breaks it. Two turns for one session, a duplicate submit, a cancel mid-flight, an out-of-order SSE event, a stale cache read after a write. Don't assert "serial" — prove it (a lock, a transaction, an atomic op, a single-writer invariant).
- **Timeouts, retries, cancellation.** What happens when the LLM stalls, the MCP call hangs, the deadline fires, the client aborts? Does the turn hang, leak, or double-charge? Does a retry storm amplify?
- **Boundary values.** 0, 1, empty, null/undefined, negative, off-by-one, max/overflow, unicode/emoji, very large input, empty collection reduced/indexed at `[0]`, a Promise that resolves `undefined`.
- **Return-shape assumptions.** The value that's usually an object but sometimes a string/null (JSON that parsed to a bare string; `find()` that returned undefined; an API that omits a field). Where is it dereferenced without a guard?
- **Recovery/fallback that re-reads live state must split intentional from accidental.** When code recovers a missing/nulled value by re-reading an external source (a store fell back to `getSelectedBlock()`, a cache miss re-fetches, a default fills an empty field), ask: is the emptiness sometimes DELIBERATE? A user dismiss, an explicit clear, an intentional deselect null the value on purpose — and a recovery that re-reads a source which still reports the old value will silently RESURRECT what was deliberately cleared. Reviewing such a change, don't stop at "the recovered value is correct when the null was transient" — find the path where the null was MEANT, and confirm the recovery is suppressed there (a flag, a sentinel, a distinct signal). This is a miss even a careful read makes by reasoning "the live source is the truth" without enumerating who nulls it and why.
- **Input-size / limit bounds as a contract.** Any validation limit — Zod `.max(N)` / `.min()` / `.enum()`, a length or byte cap, a row/array-count ceiling, a truncation — is a guard whose FAILURE path is almost never tested (a green suite only ever feeds it in-bounds inputs). For each, ask: what does the SENDER actually produce, uncapped? An unbounded `serialize()`, user-pasted text, a base64/data-URI, a big list — the payload that's *usually* under the limit and occasionally isn't. Then follow the reject: a rejected body / thrown Zod error / 4xx is only half the bug — trace it to the UI. Does the client's `.catch` turn it into a **silent Retry loop**, a frozen skeleton, a hang, or a double-charge, with no message to the user and no signal to the model? A hard-reject of the whole request when one field is oversize is worse than a graceful degrade (drop the field, truncate, proceed).
For each real one: the exact trigger, the resulting bad state, and file:line.

### b. Layer-boundary violations (core when a boundary is touched)
`CLAUDE.md` defines hard role boundaries — re-read it before judging, it moves. Flag any change that crosses them:
- **React/plugin** (`zip-ai`) renders and relays — must not run logic, own session state, or interpret tool results. It goes **direct-to-brain** for every AI surface; only auth/billing surfaces hit Laravel.
- **Node brain** (`workers/brain/`) owns all AI: orchestration, sessions/memory, import, design, page/website build, site scan, WP I/O over MCP. Must not be bypassed by Laravel for any AI decision.
- **Laravel** is credits/billing + auth + plan limits + non-brain product APIs. **No WordPress I/O at all**, no LLM calls in the agent surface, no turn state, no import/design role. Any resurrected relay (chat stream, session CRUD, import proxy, tool relay, site-scan, site-state, website-apply) is a blocker — see the GRAVEYARD comments in `routes/api.php`.
- **Redis** is a bridge only — never a source of truth.
- **MCP/WordPress** is a tool adapter only — no orchestration, no session, no UI authority. Namespace is `zipai/*`.
- Brain code standards: `application/` imports `domain/`+`ports/` only; `domain/` is pure (no I/O, no `Date.now()`/`Math.random()` — clock/RNG via ports); Zod at every wire boundary; no `any`/non-null `!`/`delete`. `npm run check` + `scripts/check-boundaries.mjs` are the gate.
- No prompt-begged JSON or regex/keyword matching for brain decisions — structured editor signals + `LlmStructuredClient.generateStructured()`.
- Spectra emission rules if the converter/importer is touched: per-block styling in `className` only; never `style`/`styleAttributes`/`backgroundColor`/`boxShadow`/`textColor` on `spectra/*`/`core/*`. Styling/render fixes belong plugin-side — backend compensation CSS, resets, or `!important` from the converter are blockers.
- Theme conditionals only in `classifyTheme`/`deriveChromeDelivery`; `CommitMode` and `ChromeDelivery` stay orthogonal.
- GBS store SSOT: `spectra_blocks_pro_gs_user_css` is the one key (global option + page meta). `wp_global_styles` is a downstream sync surface — never written directly.

### c. End-to-end feature-flow validation (core — the cross-repo heart)
For a **new feature or a behavior change**, don't just read the diff — reconstruct the flow and verify each hop's contract. Trace it explicitly, e.g.:

> User clicks **X** in the plugin → plugin POSTs payload `{…}` to the brain `<endpoint>` (Sanctum verified in-handler) → brain admission/credit check against Laravel `<internal endpoint>` → brain turn consumes it in `<handler>` → brain calls MCP tool `zipai/<tool>` → result interpreted → SSE event `<type>` / response shape `{…}` returned → plugin renders it → usage reported to Laravel.

At every hop, check the **contract matches on both sides**:
- Field the sender emits but the receiver never reads (or vice versa).
- Shape the relay reshapes/renames in transit.
- Tool input/output schema drift between the caller's expectation and the handler's Zod schema.
- **Bounds agreement, not just field names.** A hop where the names line up can still break on size: a receiver's `.max(N)`/length/enum cap versus a sender that emits the field UNCAPPED (a `serialize()`, user text, a data-URI). Confirm the sender bounds what it sends, or the receiver degrades instead of hard-rejecting — and trace the reject to the UI (a batch 400 that becomes a silent Retry loop is a ❌ hop, not a ✅).
- Events/SSE types the client listens for but the brain never emits.
- Error/empty/cancel paths — does the flow degrade safely, or hang?

State the flow as a numbered trace in the report, mark each hop ✅ verified / ⚠️ risk / ❌ broken, and cite the file:line on each side of every boundary. This is where cross-repo review earns its keep — if a hop can't be verified from static reading, say so and name what runtime check would confirm it.

### d. SSOT / reuse / overengineering (optional — brief pass, only when the change touches this)
One quick scan, a line or two each — not a full quality audit:
- **SSOT:** same fact in two places (a constant, threshold, schema, or a contract hardcoded on both sides of a wire that should come from one shared definition — `config/spectra-contract.json`, a shared Zod schema, a branded id parser). Name the authoritative location and which copy to delete. A new duplicate of an existing SSOT is 🔴/🟠, not a nit — drift is a *when*.
- **Reuse:** did it reinvent a helper the repo already has? Grep before accepting a new util/port/adapter/hook/service; cite the existing symbol with file:line.
- **Overengineering:** interface with one impl, factory for one product, config for a constant, abstraction "for later", 200 lines that could be 50 — cut it. Equally flag the opposite: a symptom patch in one caller when the root-cause guard belongs in the shared function.

### e. Security (optional — only if the change touches a trust boundary)
Auth/authz gaps (missing capability check, missing `auth.brain`, IDOR on session/site ids), injection (incl. prompt-injection reaching a tool call), SSRF via user URLs, path traversal on import/upload, secrets in code/logs, SVG sanitization before `sharp`, exchange-token reuse, cross-frame `postMessage` origin trust, nonce/CSRF on plugin REST routes, credit/plan gate bypass, unescaped React output. If a plugin PR is also in scope, treat a widened plugin route + a brain endpoint that trusts it as **one** issue; if only one side is under review, state the assumption about the other. State severity in plain words — who can do what, to whom, and the reachable route/handler. Add a full CVSS 3.1 vector only for a HIGH/CRITICAL or when the user asks. If nothing security-relevant changed, say so explicitly.

## Step 3.5 — Adversarially verify every finding (kill the false positives)

A confident wrong finding costs as much as a missed bug — it burns the author's trust and your credibility. Research on LLM reviewers is blunt about this: *empirical verification, not confidence, is what should change your belief* (80+ agents once endorsed a vulnerability that a single test killed). So before any finding reaches the report, put it through a fresh-context skeptic pass — argue the change is **correct** and see if the finding survives:

For each candidate finding, answer all three or downgrade/drop it:
1. **Guard genuinely absent?** Re-read the surrounding function AND its callers. The validation/guard you think is missing is often one layer up (a middleware, a Zod parse at the boundary, a caller-side check). If any reachable path guards it, the finding is dead.
2. **Reachable?** Name the concrete entry point (route + who can call it) and the exact input that triggers it. If no reachable caller can produce the trigger, it's a 🟡 note with that reason — not a blocker.
3. **Concrete failure scenario?** Write it as `inputs/state → wrong output/crash`. If you can't make it concrete, you don't understand it well enough to report it — keep digging or drop it.

Label what survives: **CONFIRMED** (traced to a definite failure) vs **PLAUSIBLE** (real risk, couldn't fully prove statically — say what's unproven). Never present PLAUSIBLE as CONFIRMED. A short list of verified findings beats a long list of maybes.

## Step 3.6 — Run what you can (a reproduced failure outranks any argument)

Static reading has a ceiling; cross it when a local checkout exists. Cheap, high-signal, do it by default:
- **Run the touched tests / typecheck / build.** For the brain: `npm run check` (or the specific vitest file). Laravel: `php artisan test --filter=<Test>`. Plugin JS: `npx jest <path>`. A green suite you *ran* is worth more than the PR's claim that it's green; a red one is a finding with the receipt attached.
- **Prove the fix.** For a bug-fix PR, run the new regression test against the branch WITH the fix (passes) and, if quick, with the fix reverted (must fail). A test that passes both ways tests nothing — that's a finding.
- **Reproduce a suspected bug** with the smallest possible harness (a scratch test, a REPL line, a `curl`) rather than asserting it. If you truly can't run it, say exactly which command the author should run to confirm.
- Keep it scoped and read-only — never run migrations, deploys, or anything that mutates a shared/live resource. Note in the report what you ran and the result.

## Step 4 — Write the report

Always produce a markdown report file (so the user has an artifact), then show a summary. Save to the **single PR's repo path**, or for cross-repo to the path the user names first, as `pr-review-<short-pr-ref>.md`.

Lead the report with a **big verdict banner** — the first thing the reader sees, mapped 1:1 to how the PR will be marked on GitHub:

```
# ✅ APPROVED — <title(s)>
```
```
# 🔴 CHANGES REQUESTED — <title(s)>   (N blockers, M should-fix)
```
```
# 💬 COMMENTED — <title(s)>   (no blockers, notes only)
```

Pick the banner from the verdict: **APPROVED** (merge, no blockers) · **CHANGES REQUESTED** (any 🔴 blocker or 🟠 should-fix that gates merge) · **COMMENTED** (only 🟡 nits / observations). Any CVSS HIGH or CRITICAL forces CHANGES REQUESTED, and the banner carries its score: `(1 CRITICAL 9.1, N blockers)`. Keep the banner wording identical to the GitHub review state you'll post in Step 5 so the report and the PR header agree.

Then use this structure:

```
# <verdict banner from above>

## Scope
- Repo(s) and PR link(s)
- Base branch(es) reviewed against
- Dimensions covered

## Feature flow  (omit if not a new feature / behavior change)
Numbered end-to-end trace with ✅/⚠️/❌ per hop and file:line on each side.

## Findings
Grouped by severity. Each finding:
- **[severity] short title** — `repo/path:line` · **CONFIRMED** | **PLAUSIBLE**
- **Failure scenario:** concrete inputs/state → wrong output/crash (the thing that makes it real, per Step 3.5)
- What's wrong / what breaks
- Suggested fix

Severity = 🔴 blocker · 🟠 should-fix · 🟡 nit. Every 🔴/🟠 must be CONFIRMED or carry an explicit note of what's unproven; PLAUSIBLE items never outrank a CONFIRMED one.

## What I ran
The tests/typecheck/build/repro you executed in Step 3.6 and their results (or "static review only — no local checkout" and the commands the author should run). Silence here reads as "ran nothing".

## Security  (only if dimension e applied — omit otherwise, or write "no security-relevant surface in this change")
Findings with plain-words impact (who can do what, to whom, reachable route). Full `CVSS:3.1/...` vector only for HIGH/CRITICAL or when asked.

## SSOT / reuse / overengineering  (only if dimension d surfaced something — omit if clean and say so)
Duplicated truths (authoritative location), reinvented helpers (symbol to use), what to delete. One line each.

## Verdict
Merge / merge-after-fixes / needs-work — one line, with the must-fix count.
```

(Regressions, edge cases, and test-coverage gaps are correctness findings — report them under **Findings**, not in separate sections.)

Lead with the highest-severity, cross-repo flow findings. A nit buried above a contract mismatch wastes the reviewer's time.

## Step 4.5 — Earn the APPROVED verdict (don't rubber-stamp)

APPROVED is a claim that you *tried to break it and couldn't* — not that nothing jumped out. Before writing that banner, confirm you actually did the work:
- Traced every error/timeout/partial-write/concurrency path in the change (Step 3.a2), not just the happy path.
- Enumerated the callers of every changed signature and confirmed each still holds (Step 3.a).
- Adversarially verified each finding and killed the false positives (Step 3.5).
- Ran the touched tests / build, or stated plainly you couldn't and why (Step 3.6).
- Confirmed new-reachable cases have tests; a missing test for a case the change made reachable is itself a finding.

If any of these was skipped, the honest verdict is COMMENTED with "review limited to X", not APPROVED. A fast approval on a large or logic-dense diff is a smell — say what you didn't cover rather than implying you covered it all.

## Step 5 — Offer to post (don't post unprompted)

Posting to GitHub is outward-facing and hard to take back. After showing the report, ask: *"Want me to post this — and should I mark the PR `Approve` / `Request changes` / `Comment` to match the verdict banner?"* Only on a yes:
- **Map the verdict banner to the GitHub review state** so the PR header shows it:
  - ✅ APPROVED → `gh pr review <pr-url> --approve --body-file <report>`
  - 🔴 CHANGES REQUESTED → `gh pr review <pr-url> --request-changes --body-file <report>`
  - 💬 COMMENTED → `gh pr review <pr-url> --comment --body-file <report>`
- Prefer the single-review form above (report body + state in one shot). For per-line notes, add inline comments: `gh api repos/{owner}/{repo}/pulls/{n}/comments` with `path`, `line`, `side`, `body`, `commit_id`.
- **Never `--approve` on your own initiative** — approving is a merge gate; only fire it on the user's explicit yes to that specific state.
For cross-repo, post to each PR the findings and the state that belong to it (one PR may be APPROVED while the other is CHANGES REQUESTED). Confirm the state + comment count before firing.

## Notes
- If `gh` isn't authenticated, say so and fall back to local-git-only diffing using the base branch the user names; PR links then serve only as title/context.
- Keep findings evidence-backed: every claim cites file:line. No vibes-based "this feels off" without a location and a reason.
