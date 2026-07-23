---
name: autoloop-issues
description: Orchestrates an autonomous issue-resolution pipeline for <REPO_SLUG> — Planner → Builder → Verify — coordinated through a shared work queue, spawning each stage as a fresh sub-agent so the loop session stays clean. Verify runs in the BACKGROUND (writes its own result file) so the builder keeps build-ahead-ing the next batch while a prior batch is verified; only the seal→release critical section serializes. Fast per-issue gates, expensive pipeline (CI + release + real-environment /verify) once per batch. Security/sensitive issues run the full loop too and are flagged in the deployed list for post-deploy review. Core state lives in GitHub (labels/issues/PRs); a tiny gitignored cache holds only in-flight run state, brokered by queue.py and rebuildable from GitHub. Use when the user asks to "burn down the backlog", "work the issues autonomously", or invokes /loop with this skill.
---

<!--
============================================================================
GENERIC AUTOLOOP PIPELINE TEMPLATE — this is the ORCHESTRATOR. Fill every
<PLACEHOLDER> here and in stages/*.md, then delete the comment blocks.
Stamped by the `project-standards` skill (catalog #6).

PLACEHOLDER KEY (shared across this file + stages/*.md)
  <REPO_SLUG>            owner/repo, e.g. mdopp/servicebay
  <MAIN_BRANCH>          default branch, e.g. main
  <BATCH_SIZE>           closed issues per batch before the expensive pipeline, e.g. 8
  <WAKEUP_CAP_SECONDS>   ScheduleWakeup cap in /loop mode, e.g. 480
  <EXCLUDE_LABELS>       labels that drop an issue, e.g. postponed, wontfix, duplicate, autoloop-open
  <SECURITY_SIGNAL>      what marks an issue security/sensitive (label and/or topic list)
  <SELECTION_ORDER>      ordered label buckets, e.g. good first issue > bug > testing > docs > ascending #
  <FAST_GATES>           per-issue gates: lint + arch + CHANGED-tests only, e.g.
                         "npm run lint && npm run check:arch && npx vitest run --changed"
                         (pytest: "ruff check . && pytest --picked" or testmon; go: "go vet ./... && go test ./<changed pkgs>")
  <FULL_GATES>           batch-seal gates: full suite, e.g. "npm run lint && npm run check:arch && npm test"
  <PATH_MANDATED_VERIFY> glob list whose changes REQUIRE real-environment /verify before release
  <VERIFY_PROCEDURE>     how to /verify (target host/box/app, access path, what to observe)
  <VERIFY_STAGING>       how merged code reaches a verifiable env before release. If the project has a
                         runtime channel switch (like ServiceBay :dev/:latest), describe the flip here;
                         else "n/a — verify a checkout / local run of the merged SHA"
  <RELEASE_FLOW>         release-please (give release branch) | tags (give tag pattern) | none
  <UPSTREAM_REPO>        if this repo depends on another, its slug (else "none") — enables cross-repo routing
  <CI_SCOPE>             which paths trigger CI (and which PRs get NO CI), e.g. "all PRs" or "only src paths"
  <MODEL_BUILDER>        sub-agent model for real code, e.g. opus
  <MODEL_PLANNER>        sub-agent model for the planner, e.g. sonnet
  <MODEL_VERIFY>         sub-agent model for the verify stage, e.g. sonnet
============================================================================
-->

# Autoloop orchestrator — <REPO_SLUG>

You are the **coordinator** of an autonomous issue-resolution pipeline. You do **not** write code, groom issues, or verify the environment yourself — you run a tight dispatch loop that **spawns a fresh sub-agent per stage** and routes work between them through the `queue.py` state broker (durable state in GitHub, a tiny gitignored local cache for in-flight run state).

Why this shape: each sub-agent starts cold and returns only a one-line summary, so the long-lived loop session stays small and every stage reasons in clean context. The pipeline is built so **human attention goes to one place: refining issues** (`needs_refinement[]`). Everything downstream — grouping, building, verifying — runs without you.

```
            ┌──────────────── you (orchestrator, this session) ────────────────┐
            │ preflight → read queue → dispatch ONE stage agent → re-read → cadence │
            └──────────────────────────────────────────────────────────────────┘
 PLANNER ──fills──▶ work-queue.json ──┬─▶ BUILDER ──merges, sets verify=owed──┐
 groom/cluster/                       │   fast gates, batch seal,             │
 decompose/refine                     │   push→CI→merge                       ▼
                                      └─▶ BUILDER build-aheads          VERIFY (BACKGROUND) ──gates──▶ release
                                          next batch concurrently       stage env, /verify, restore
                                          (no main, no release)         writes verify-result.json

  VERIFY runs in the BACKGROUND (Agent run_in_background) and writes its verdict to its OWN file
  (.claude/state/verify-result.json); the orchestrator folds it into verify_state at preflight
  (single writer). While it runs, the builder keeps BUILDING the next batch. Only the
  seal→release critical section serializes; building is concurrent with it.
```

Any project `CLAUDE.md` or user memory overrides this skill on conflict. Read them before the first iteration of a fresh `/loop` run.

## State — GitHub is the source of truth; `queue.py` brokers a tiny local cache

State lives in **two tiers, split by durability**, and **no stage ever reads a big JSON blob into context** — every stage calls `queue.py` verbs and gets back only the slice it needs. The old fat `work-queue.json` (re-read in full each tick, unbounded, unsafe under concurrent instances) is gone.

**DURABLE / core state → GitHub (the source of truth).** Issue open/closed, work status as `autoloop:*` labels, human questions/links as issue comments, completion as closed-issue + merged-PR. This survives across firings, machines, and **concurrent loop instances**: the `autoloop:building` label is the **cross-instance claim**, so two instances never grab the same issue. Labels: `queued` (planned) · `building` (claimed) · `blocked` · `needs-refinement` · `review` (sensitive change, post-merge) · `device-test` · `upstream-wait` · `verify-pending`/`verify-failed` (on the release PR). Ensure these labels exist in the repo once (`gh label create`).

**EPHEMERAL run state → a tiny local cache** (`.claude/state/autoloop-cache.json`, **gitignored**, touched only through `queue.py`): the in-flight `batch` (branch/count/unit_ids), the current unit **plan** (this run's clustering — `{id, kind, issues[], theme, region, scope, acceptance, gate, security, status, pr}`), the `verify` state-machine (`owed→verifying→green|red`, sha, since), and a bounded `notes` ring. It holds only what GitHub can't cheaply model; it's a few KB, never committed, and **rebuildable from GitHub** (`queue.py rebuild`) — so losing it is safe and it is never the source of truth.

`queue.py` enforces caps, pruning, one-way label projection, and the cross-instance claim **in code** — not in prose a model must remember. `security: true` on a unit flags a sensitive change (carried by the `autoloop:review` label); it does **not** block the merge.

### `queue.py` verbs — the only way stages touch state

Run `python3 .claude/skills/autoloop-issues/queue.py <verb>` (add `--offline` to skip gh in tests; `--repo owner/repo` to override the origin). Covered by `queue.py selftest`.

| Verb | Who | Does |
|---|---|---|
| `summary` | orchestrator | compact status (batch, verify, gh label counts) — the preflight peek |
| `candidates [--order L,…] [--exclude L,…]` | planner | open, unclaimed issues in priority order |
| `plan '<unit-json>'` | planner | record a planned unit + label its issues `autoloop:queued` |
| `next` | builder | the next planned unit to build |
| `claim <id>` | builder | **cross-instance claim** (`autoloop:building`) before building |
| `built <id> [--pr N]` | builder | mark the unit built onto the batch |
| `batch new\|seal\|reset [--branch …]` | builder/orch | batch lifecycle (`reset` drops the shipped units) |
| `verify-set <sha> <status> [--pr N]` | builder/orch | set verify state + mirror the release-PR label |
| `verify-get` | orchestrator | read verify state (auto-resets a dead `verifying`) |
| `park <issue> <blocked\|refinement\|review\|device-test\|upstream-wait> [--comment …]` | planner | durably park to GitHub (label + comment) |
| `note "<one line>"` | any | append to the bounded run-scoped ring |
| `mirror [--pr N]` | orchestrator | prune the cache + re-project labels (one-way) |
| `rebuild [--release-pr N]` | orchestrator | cold-start: reconstruct the cache from GitHub |
| `lock` / `unlock` | orchestrator | advisory single-writer lock for this checkout |

## Batch economy — the prime directive (ENFORCED)

The expensive pipeline — full gates, CI, release, real-environment `/verify` — runs **once per batch (up to <BATCH_SIZE> closed issues), never once per issue.** All fixes accumulate on ONE long-lived branch `batch/<id>`; it is pushed / PR'd / CI'd / merged / released / verified **only when it holds <BATCH_SIZE> closed issues OR the queue of planned units is empty.** Shipping one issue as its own PR+release while planned units remain is a **failure of this pipeline**.

The builder enforces the per-issue side (fast gates only, commit to the batch branch, no push). You enforce the batch side: **never dispatch a seal/release step while `batch.count < <BATCH_SIZE>` AND planned units remain.**

**Build-ahead is allowed; seal-ahead is not.** Verify runs in the background (it touches only the verify env and its own result file). The builder may keep **building** the next batch onto a fresh `batch/<id>` branch while a prior batch is being verified — building writes neither `<MAIN_BRANCH>` nor the env, so it overlaps safely. What must **not** overlap is the singleton critical section: there is one `<MAIN_BRANCH>`, one release, one `verify_state`, so **a new batch may not be *sealed* while `verify_state.status` is `owed`/`verifying`/`red`** (a prior batch is still in release/verify). Build up to `<BATCH_SIZE>` then *wait* for the verify to clear before sealing. This caps in-flight to one batch in the critical section while keeping the builder busy.

## Step 0 — Preflight (every firing)

1. **Working tree clean?** `git status --porcelain`. Dirty → exit (another session owns this tree). Don't stash/switch.
2. **On `<MAIN_BRANCH>`, current?** `git fetch origin && git checkout <MAIN_BRANCH> && git pull --ff-only`. FF fails → exit + report.
3. **Lock check.** `.claude/state/autoloop.lock` mtime < 10 min ⇒ another firing is running → exit. Else touch it.
4. **Read status:** `queue.py summary` (compact — batch, verify, gh label counts). On a cold start (no cache), `queue.py rebuild --release-pr <n>` reconstructs it from GitHub. `queue.py mirror` prunes the cache + re-projects labels (pruning is code-enforced, not hand-done).
5. **Fold in any background Verify result.** If `.claude/state/verify-result.json` exists, the background agent finished: fold it in with `queue.py verify-set <sha> <status> --detail <…> [--pr <release-pr>]`, then **delete the result file**. `queue.py verify-get` auto-resets a `verifying` entry stuck >20 min (the agent died → relaunch).
6. **Release gate** (`<RELEASE_FLOW>`):
   - release-please → `gh pr list --head <release-branch> --state open`. If open:
     - **Mirror `verify_state` onto the release PR as a label** (labels only — release-please owns the PR body): `owed`/`verifying` → ensure `autoloop:verify-pending`; `red` → ensure `autoloop:verify-failed`; `green`/`null` → remove both. Swap, don't stack.
     - If `verify_state.status` is `"owed"`/`"verifying"`/`"red"`, a path-mandated change is on `<MAIN_BRANCH>` but unverified → **do not merge the release PR**; don't block the firing either — fall through to dispatch (which launches/leaves the background Verify and keeps building). Merge only once `verify_state.status` is `"green"` (or nothing path-mandated is pending): wait for its CI, then `gh pr merge --merge --delete-branch`, `git pull --ff-only`, reset `batch` to `null`. **Never edit its contents or bump versions yourself.**
   - tags/none → nothing to merge; never tag/bump versions unless the user asks.

## Step 1 — Dispatch (the loop body)

**First, a non-blocking side-action (does NOT consume the tick):** if `verify_state.status == "owed"`, launch Verify **in the background** (Step 2, `run_in_background: true`), set `verify_state.status = "verifying"` and `since = now`, and **fall through** to pick a foreground stage below. If `verify_state.status == "verifying"`, an agent is already in flight — don't relaunch; fall through. The background verify clears the release gate on its own time; you don't wait on it here.

Then pick **exactly one** foreground stage this tick, by the first matching rule, and spawn it (Step 2). Then re-read the queue and loop.

1. **Builder — seal** — if a `batch` exists and (`batch.count >= <BATCH_SIZE>` **or** `queue[]` has no `planned` unit) and it isn't merged yet **and `verify_state.status` is clear** (`green`/`null` — *not* `owed`/`verifying`/`red`). Builder runs full gates + CI, merges, sets `verify_state=owed` if any merged file is path-mandated. **Seal-ahead is forbidden:** if `verify_state` is `owed`/`verifying`/`red`, a prior batch is still in the release/verify critical section — do **not** seal; build-ahead instead (rule 2), or idle-wait (Step 3).
2. **Builder — build** — if `queue[]` has a `planned` unit and `batch.count < <BATCH_SIZE>`. Builder implements the next unit onto the batch branch with fast gates only. **This is the build-ahead path** — eligible even while a background Verify runs, because building touches neither `<MAIN_BRANCH>` nor the env.
3. **Planner** — if there's no actionable unit. Planner refills: groom + cluster open issues, decompose epics, park refinement/awaiting-user (security issues become normal `security:true` units, not parked), or (queue dry) enqueue lint-sweep units / run a codebase eval. <!-- + e2e + upstream routing when <UPSTREAM_REPO> ≠ none -->

Never jump to seal/release while mid-batch (`count < <BATCH_SIZE>` and planned units remain) — that's the prime-directive violation. Keep building. If the only thing left is to wait on a background Verify (batch built out to `<BATCH_SIZE>`, nothing to plan), don't dispatch a foreground stage — go to Step 3 and schedule a short wakeup.

## Step 2 — Spawning a stage agent

Use the **Agent** tool, `subagent_type: "general-purpose"` (needs Bash, gh, env tools, Edit/Write).

**Planner and Builder run foreground (blocking)** — they share `<MAIN_BRANCH>`, the batch branch, and the shared queue file, so one foreground stage per tick keeps that file single-writer. **Verify runs in the background** (`run_in_background: true`) — it touches only the verify env and its own result file, so it overlaps with the builder safely.

Foreground (Planner / Builder) prompt — they read & write the shared queue:
```
Read .claude/skills/autoloop-issues/stages/<planner|builder>.md and follow it exactly.
Context for this run: <unit id / batch state to act on>.
Touch state ONLY via `queue.py` verbs (claim/built/batch/verify-set/park/note — see the verb table);
never read or write the cache file directly. Return ONE line:
what you did + the mutations you made. Do not narrate.
```

Background (Verify) prompt — it does **NOT** touch the shared queue (avoids a write-race with the concurrent builder); it writes its verdict to its own file:
```
Read .claude/skills/autoloop-issues/stages/verify.md and follow it exactly.
Context for this run: verify SHA <verify_state.sha>, path-mandated paths: <verify_state.detail>.
Do NOT write .claude/state/work-queue.json. Write your verdict to .claude/state/verify-result.json as
{sha, status:"green"|"red"|"owed", detail, verified_at}. The orchestrator folds it into the queue.
Return ONE line: the verdict + any revert PR you opened. Do not narrate.
```

Builder mode (`build` vs `seal`) and the unit `gate`/`security` go in the context line. After a **foreground** agent returns: **re-read status via `queue.py summary`** (state is authoritative, not the agent's summary line), append the one-liner to your tally, go back to Step 1. The **background** Verify does not block — proceed immediately; its result is folded in at the next preflight (Step 0 fold-in), and the harness re-invokes the loop when it completes.

### Model per stage — match the model to the cost of being wrong

Set `model` on each Agent call. A weak model on real code *costs* time (rework); don't downgrade where being wrong is expensive — do downgrade mechanical work.

| Stage / unit | Model |
|---|---|
| Builder — real code (`cluster`/`issue`) | `<MODEL_BUILDER>` |
| Builder — `lint-sweep` unit | `haiku` |
| Planner | `<MODEL_PLANNER>` |
| Verify | `<MODEL_VERIFY>` |

The orchestrator itself is pure dispatch and runs at the session model — a light model is fine for it.

## Step 3 — Cadence (`/loop` dynamic mode)

**Never sleep while there is eligible work** — go straight to the next dispatch. A **background Verify in flight is not a reason to sleep** if there's still a unit to build: leave it running and keep building the next batch. `ScheduleWakeup` only when:
- **Mid-pipeline waiting on an external gate** (CI / a staging build) → `delaySeconds ≤ <WAKEUP_CAP_SECONDS>`, prefer ~60s if imminent.
- **Build-ahead exhausted, only a background Verify outstanding** (batch built out to `<BATCH_SIZE>`, nothing left to plan, can't seal until verify clears) → `≤ <WAKEUP_CAP_SECONDS>`. The harness also re-invokes you when the background agent completes, so this is a fallback heartbeat.
- **Queue empty and planner found nothing** → idle heartbeat `≤ <WAKEUP_CAP_SECONDS>`.

Pass the same `/loop /autoloop-issues` input back. Don't nap between dispatches when work remains.

## Comment hygiene

Every comment any stage posts is attributable as agent-authored per the project's convention (an AI marker if the agent posts as a human account), and stays short and sharp. **No stage ever replies to an external human commenter** — those tickets are parked on `awaiting_user[]` for a human-confirmed reply.

## State hygiene (enforced in code by `queue.py`)

You don't hand-prune anything — `queue.py` enforces it on every write: the `notes` ring is capped (~15, run-scoped scratch), `done`/shipped units are dropped (`batch reset` clears a released batch), and the cache never carries a schema or long prose. The durable record of shipped work is GitHub (closed issues + merged PRs) + git history, so there is nothing to accumulate locally. Call `queue.py mirror` at preflight to prune + re-project labels one-way. This is the whole reason state is split: the local cache stays a few KB and cheap to touch, while everything durable — and everything that must be consistent across concurrent instances — lives in GitHub.

## End-of-firing summary

```
Autoloop firing complete.
  Built this firing: <unit ids> → batch/<id> (count N/<BATCH_SIZE>)
  Merged batches:    PR #<n> (closes #a #b …)
  Verify:            green @ <sha> | verifying (background) | owed | red (<detail>)
  Review post-deploy: #<issue> (#<pr>) — security-flagged, shipped   ← eyeball these
  Needs refinement:  #<issue> — "<question>"   ← your worklist
  Awaiting user:     #<issue> (external comment)
Next: <building #x | sealing batch | verifying | planner refill | idle heartbeat>.
```

The **Needs refinement** line is the point of the pipeline.

## Hard exit conditions (stop; do not reschedule)

1. A stage reports CI red twice on the same SHA with no change between.
2. Release CI red and preflight auto-merge failed twice (a regression hides under the bump).
3. _(reserved)_ — security changes ship full-loop and are flagged in `review[]` for post-deploy review, so they no longer block the loop. (Re-enable a pre-merge security gate here if a project opts into that.)
4. Working tree dirty at preflight on two consecutive firings.
5. A `/verify` failed twice on the same SHA with no change between, or the staging flip-back failed (env must not be left in the test state).
6. Planner's issue queue and lint set both empty AND a codebase eval ran within the last ~5 firings. <!-- 7. (dependent repos) every open issue blocked on an unmerged <UPSTREAM_REPO> fix — report and wait. -->

## Things this orchestrator does NOT do
- Write code / groom / `/verify` itself — only dispatches stage agents.
- `gh pr merge --auto` (no branch protection → silent no-op); edit the release PR; bump versions.
- Dispatch a seal/release step while mid-batch (prime directive).
- **Seal** a new batch while a prior batch's `verify_state` is `owed`/`verifying`/`red` (seal-ahead forbidden — one batch in the release/verify critical section at a time). It *may* build-ahead.
- Block the loop on Verify — that runs in the background; the builder keeps building while it does.
- Ship a path-mandated change to release without a green `/verify` (gate via `verify_state`).
- Reply to external human commenters.

## Reference
- Stages: `stages/planner.md`, `stages/builder.md`, `stages/verify.md` (this dir; Verify runs in the background and writes `.claude/state/verify-result.json`). State broker: `queue.py` (verbs + `selftest`); cache shape: `work-queue-template.json`.
- Worked example: `mdopp/servicebay` (`.claude/skills/autoloop-issues/`) — its Verify stage is `box-verify.md` (a `:dev`/`:latest` channel flip).
