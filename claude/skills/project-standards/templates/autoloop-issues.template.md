---
name: autoloop-issues
description: Work the open <REPO_SLUG> issue backlog autonomously, up to <PR_BUDGET> PRs per invocation, with mandatory real-environment `/verify` on the path-mandated list. Resumable across /loop firings via .claude/state/autoloop-state.json. Security/sensitive issues open as draft and wait for human review. Use when the user asks to "burn down the backlog", "work the issues autonomously", or invokes /loop with this skill.
---

<!--
============================================================================
GENERIC AUTOLOOP TEMPLATE — fill EVERY <PLACEHOLDER> before use, then delete
this comment block. Stamped by the `project-standards` skill (catalog #6).

PLACEHOLDER KEY
  <REPO_SLUG>            owner/repo, e.g. mdopp/servicebay
  <MAIN_BRANCH>          default branch, e.g. main
  <PR_BUDGET>            max PRs per invocation, e.g. 8
  <WAKEUP_CAP_SECONDS>   ScheduleWakeup cap in /loop mode, e.g. 480
  <EXCLUDE_LABELS>       labels that drop an issue, e.g. postponed, wontfix, duplicate, autoloop-open
  <SECURITY_SIGNAL>      what marks an issue security/sensitive (label and/or topic list)
  <SELECTION_ORDER>      ordered label buckets, e.g. good first issue > bug > testing > docs > ascending #
  <LOCAL_GATES>          ordered local commands that must pass, e.g. npm run lint && npm run check:arch && npm test && npx tsc --noEmit
  <PATH_MANDATED_VERIFY> glob list whose changes REQUIRE real-environment /verify before merge
  <VERIFY_PROCEDURE>     how to /verify (target host/box/app, access path, what to observe)
  <RELEASE_FLOW>         release-please (give release branch) | tags (give tag pattern) | none
  <UPSTREAM_REPO>        if this repo depends on another, its slug (else "none") — enables cross-repo routing
  <CI_SCOPE>             which paths trigger CI (and which PRs get NO CI), e.g. "all PRs" or "only src paths"
============================================================================
-->

# Autoloop: <REPO_SLUG> backlog burndown

You are working a queue of open GitHub issues on **`<REPO_SLUG>`**, with explicit exit conditions and a resumable state file. Pre-production: the loop may merge changes across the repo, but only after CI green (where CI applies) **and**, on the path-mandated list, only after real-environment **`/verify`** green. Security/sensitive issues open as **draft** PRs and wait for human review.

Any project `CLAUDE.md` or user memory overrides this skill on conflict. Read them before the first iteration of a fresh `/loop` run.

## Per-invocation budget
- **At most <PR_BUDGET> PRs per invocation**, then exit cleanly (`/loop` re-fires you). A draft counts as one PR.
- If you've spent >40 min on one issue without a green PR, stop, comment on the issue with what's blocking, and move on.

## Wakeup cadence (`/loop` dynamic mode)
Every `ScheduleWakeup` uses `delaySeconds: <WAKEUP_CAP_SECONDS>` or less — including "CI running" / "nothing to do" heartbeats. Keep the backlog draining.

## State file
Track progress at `.claude/state/autoloop-state.json` (shape in `state-template.json`). Update at every transition; create with empty arrays if absent. Keys: `started, last_invocation, completed[], in_progress, skipped[], blocked[], notes[]` (+ `upstream_waits[], last_e2e, last_codebase_eval` when `<UPSTREAM_REPO>` ≠ none).

## Step 0 — Preflight (every invocation)
1. **Clean tree?** `git status --porcelain`. If dirty → exit (another session is here). Don't stash/switch.
2. **On `<MAIN_BRANCH>` & current?** `git fetch origin && git checkout <MAIN_BRANCH> && git pull --ff-only`. FF fails → exit + report.
3. **Release preflight** (`<RELEASE_FLOW>`): if release-please — `gh pr list --head <release-branch> --state open`; if open, wait for its CI then `gh pr merge --merge --delete-branch` and `git pull --ff-only` (never edit its contents or bump versions yourself). If tags/none — nothing to merge; never tag/bump versions unless the user asks.
4. **Lock check.** `.claude/scheduled_tasks.lock` mtime within 10 min ⇒ exit.
5. **Read state.** Resume `in_progress` if set; else select per below.

## Step 1 — Issue selection
```bash
gh issue list --repo <REPO_SLUG> --state open --limit 100 --json number,title,labels,body
```
**Exclusion filter** (drop if any): labels include any of `<EXCLUDE_LABELS>`; number is in `state.completed/skipped/blocked`; body is clearly multi-PR ("audit"/"strategy"/"epic") → mark `blocked` reason `"needs scoping"` — *or*, in a refine pass (track b) or when the user asks, **decompose** it into bite-sized child issues (see track b) and keep it open as the tracking umbrella.
**Classification:**
- **Security/sensitive gate** (`<SECURITY_SIGNAL>`) → open the code PR as **draft**, label the issue `autoloop-open`, add to `state.skipped[]` (`"security gate; draft PR #X awaiting review"`). Never merge.
- **Normal flow** → code PR, merged after CI green (where CI applies) + path-mandated `/verify` green.
**Selection order:** `<SELECTION_ORDER>`. Pick the head; set `in_progress = {issue, branch, gate, started_at}`.

### No eligible issues — choose a track
Don't exit or blindly default. Decide:
- **a) Hygiene / lint-sweep** — small, single-file cleanups; where a warning-count tool exists, drive it down (≤2 files, ≤120 LOC net per PR).
- **b) Refine & unblock** — re-check `state.blocked[]` for items a recent merge made actionable; tighten thin issue bodies; then work the head. **Decomposing an epic** is a first-class move here (better than parking it `needs scoping`): break a multi-PR issue into bite-sized child issues — each an independently-shippable PR-unit (foundations first, no dead-code stubs), **filed in dependency order so ascending issue number == dependency order** (the loop works ascending and skips a child whose `Depends on #N` is still open), each body giving deliverable + starting-point files + `Depends on #N`; comment the dependency DAG on the parent and keep the parent open as the tracking epic.
- **c) Codebase evaluation** — run the standing eval prompt (below) against HEAD; file **Pragmatic** findings as issues (symptom-style); record **Academic** ones in `state.notes[]`.
- **d) End-to-end validation** *(only if `<UPSTREAM_REPO>` ≠ none, or the project has a real runtime)* — run the full real-environment smoke; route failures cross-repo (see below).

**Choose:** interactive → `AskUserQuestion`. Autonomous → (d) if runtime artifacts merged since `last_e2e`; else (b) if `blocked[]` non-empty; else (c) if no eval in ~5 invocations (`last_codebase_eval`); else (a). Record the choice in `state.notes[]`.

#### Codebase-evaluation prompt (track c)
```
Evaluate the <REPO_SLUG> codebase across its core areas. Assume a solid, production-ready baseline; skip generic style complaints unless they have measurable impact on bugs or velocity.
CRITICAL: focus ONLY on active, unresolved bugs / logical flaws / security exploits / UX dead-ends live at the current HEAD. Do NOT cite historical/resolved issues or already-merged fixes. Inspect actual source files to confirm each is live.
Group findings into: (1) Academic / Theoretical (near-zero ROI) and (2) Pragmatic / Real-World (load-bearing flaws compromising security, data integrity, runtime/deploy stability, or actively blocking users). For each Category 2 item: (a) exact file(s)+line range, (b) real-world consequence of ignoring, (c) brief patch outline.
```

## Step 2 — Implementation
Branch `fix/issue-<N>-<kebab>`. Read the issue + referenced files fully (+~50 lines around any line ref). Ambiguous → comment the specific question and move on; don't guess. Smallest change that closes the ticket; no drive-by refactors (a `[Refactor]` ticket stays within its named module). Tighten any invariant ratchet you resolve; never loosen it.

## Step 3 — Local verification (each must pass before the next)
Run `<LOCAL_GATES>`. Lint warnings allowed only if the count didn't increase. Diagnose test failures at the root — don't mock around or skip them.
**Mandatory real-environment `/verify`** if the diff touches any of `<PATH_MANDATED_VERIFY>`: `<VERIFY_PROCEDURE>`. CI/type-check/tests verify code correctness, not feature correctness. If `/verify` fails → treat like CI-red (stop, post summary on the PR, leave open, move on) AND triage owner if `<UPSTREAM_REPO>` ≠ none (cross-repo routing below).

**Narrow, deliberately-logged exception to the pre-merge `/verify`:** you MAY merge a path-mandated change on CI-green + a strong unit test and *defer* the real-env `/verify` ONLY when ALL hold — (1) the user explicitly prioritized it, or a dependent repo is blocked waiting on it; (2) the change adds **no new runtime logic** (reuses an already-tested, pre-existing path — removing a coercion, threading an existing-contract value, docs); (3) a test covers the new behaviour; (4) you document the deferral in the PR + `state`, with the real check to happen at the next natural opportunity. Absent that signal, default: path-mandated ⇒ `/verify` before merge. Never for a security-gated change. Logged judgement call, not a general loosening.

<!-- Include this block only when <UPSTREAM_REPO> ≠ none -->
### Cross-repo issue routing (this repo vs <UPSTREAM_REPO>)
A failure may be owned by `<UPSTREAM_REPO>`, not this repo. Local-owned → fix here / file here. Upstream-owned → file in `<UPSTREAM_REPO>` (symptom + upstream file/line + the repro that exposed it), mark the local issue `blocked "waiting on <UPSTREAM_REPO>#N"`, record in `state.upstream_waits[]`, and **wait** (re-check later firings; unblock when it merges). Don't open a PR against `<UPSTREAM_REPO>` from here — filing the issue is the handoff.

## Step 4 — Open the PR
**Commit:** Conventional Commits; scope mirrors the path. **No parens in the subject beyond `(scope)`** (parens break release-please). Body: brief summary + `Closes #<N>`.
**Push:** `git push -u origin fix/issue-<N>-<slug>`. **PR body:** real body (no `--fill`) with What / Why (`Closes #N`) / Risk / Rollback / Verification checklist (local gates + `/verify` if path-mandated).
**Security gate:** `--draft`, label `autoloop-open`, record in `state.skipped[]`, move on.
**Normal flow merge gate:** `main` likely unprotected (check `gh api .../branches/<MAIN_BRANCH>/protection` → 404) so `--auto` no-ops — use a manual gate: wait for CI where it applies (`<CI_SCOPE>`); run `/verify` if path-mandated; if both green `gh pr merge <PR#> --merge --delete-branch`. CI red twice on the same SHA or `/verify` red → stop, comment the failing link, leave open, move on. Then `git checkout <MAIN_BRANCH> && git pull --ff-only`; for release-please, leave any new release PR for the next preflight.

## Step 5 — End of invocation
After <PR_BUDGET> PRs, stop and print a summary: Merged / Security drafts / Skipped / Blocked / Next eligible issue.

## Hard exit conditions (stop entirely)
1. CI red twice on the same PR with no code change between.
2. >3 security-gate drafts pending review.
3. Tree dirty at preflight on two consecutive invocations.
4. `/verify` red twice on the same PR with no code change between.
5. Issue queue empty AND track-(a/c) exhausted — but prefer (c)/(d) over exiting.
6. *(dependent repos)* every open issue is blocked on an unmerged `<UPSTREAM_REPO>` fix — report and wait.

## Things this skill does NOT do
- Bump versions / push tags / edit the release PR (release tooling owns those).
- `gh pr merge --auto` (no branch protection); `--fill`-only PR bodies; `--no-verify`.
- Refactor beyond ticket scope; loosen invariants; skip path-mandated `/verify`.
- File new issues except in track (c); otherwise comment on the existing issue.
