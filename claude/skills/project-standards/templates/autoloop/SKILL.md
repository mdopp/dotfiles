---
name: autoloop-issues
description: Orchestrates an autonomous issue-resolution pipeline for <REPO_SLUG> — Planner → Builder → Verify — coordinated through a shared work queue, spawning each stage as a fresh sub-agent so the loop session stays clean. Verify runs in the BACKGROUND (writes its own result file) so the builder keeps build-ahead-ing the next batch while a prior batch is verified; only the seal→release critical section serializes. Fast per-issue gates, expensive pipeline (CI + release + real-environment /verify) once per batch. Security/sensitive issues run the full loop too and are flagged in the deployed list for post-deploy review. Resumable via .claude/state/work-queue.json. Use when the user asks to "burn down the backlog", "work the issues autonomously", or invokes /loop with this skill.
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

You are the **coordinator** of an autonomous issue-resolution pipeline. You do **not** write code, groom issues, or verify the environment yourself — you run a tight dispatch loop that **spawns a fresh sub-agent per stage** and routes work between them through one shared file, `.claude/state/work-queue.json`.

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

## The shared work queue (the only handoff)

`.claude/state/work-queue.json` is the single source of truth between stages. Stage agents read it, do their work, **write results back into it**, and return one line. You re-read it after every spawn. Schema in `work-queue-template.json` (same dir); create from it if absent.

Key fields:
- `queue[]` — **units** the builder consumes, in selection order. A unit is `{id, kind: "cluster"|"issue"|"lint-sweep", issues[], theme, region, scope, acceptance, gate: "normal"|"verify", security: false, status: "planned"|"in_progress"|"built"|"blocked", pr, notes}`. A cluster is the work-unit; its members never appear standalone. `gate` is the verification level; `security: true` flags a sensitive change for **post-deploy** review — it does **not** block the merge.
- `batch` — the persistent integration branch: `{branch, units[], count, sealed}`. **Survives across firings.** Reset to `null` after its release/merge completes.
- `needs_refinement[]` — **the human's worklist.** `{issue, question, comment_url, since}`. The planner parks anything it can't make actionable without a human decision here, with the *specific* question.
- `awaiting_user[]` — external human comment unanswered; a `/comment-responder`-style reply-with-confirm job, never the pipeline's.
- `review[]` — **the human's post-deploy review list**: `{issue, pr, flag, merged_at}` for shipped `security:true` (and other sensitive) changes. Informational, **not** a merge gate. (A project that prefers *pre-merge* review for security can instead keep these as draft PRs — see the note in `stages/builder.md`.)
- `verify_state` — `{sha, status: "owed"|"verifying"|"red"|"green", detail, since}`. Gates the release. State machine: `owed` (path-mandated change merged, not yet verified) → `verifying` (a background Verify agent is in flight) → `green`|`red`. You set `verifying` when you launch the background agent; the agent writes its verdict to `.claude/state/verify-result.json` (**its own file, not the shared queue** — avoids a write-race with the concurrent builder), and you fold that verdict back into this field at preflight. A `verifying` entry whose `since` is >20 min old with no result file = the agent died → reset to `owed` (it relaunches).
- `blocked[]` — parked work, each `{issue, blocked_by, reason, since}` where `blocked_by` is a **machine-checkable unblock condition** (`"#<N>"` dependency · `"capability:<x>"` · `"decomposition"` · `"epic"`) the planner rechecks every run.
- `completed[]`, `notes[]` — **bounded caches, not ledgers** (see *State hygiene* below). The durable record of shipped work is GitHub (closed issues + merged PRs) + git history; these hold only a short rolling window for the loop's own context, and are re-read in full by every stage — so never let them grow unbounded.
- `lint_sweep[]`, `release_warnings[]`, `last_codebase_eval`. <!-- + upstream_waits[], last_e2e when <UPSTREAM_REPO> ≠ none -->

**Label mirror (one-way projection).** The queue file is the source of truth; three human-facing states are *mirrored* to GitHub labels so a human sees the same worklist: `blocked[]` → `autoloop:blocked`, `needs_refinement[]` → `autoloop:needs-refinement` (both reconciled by the **planner**), and `verify_state` → `autoloop:verify-pending`/`-failed` on the **release PR** (set here in preflight). Labels are derived from the file every run — never the reverse — so drift is cosmetic and self-heals.

## Batch economy — the prime directive (ENFORCED)

The expensive pipeline — full gates, CI, release, real-environment `/verify` — runs **once per batch (up to <BATCH_SIZE> closed issues), never once per issue.** All fixes accumulate on ONE long-lived branch `batch/<id>`; it is pushed / PR'd / CI'd / merged / released / verified **only when it holds <BATCH_SIZE> closed issues OR the queue of planned units is empty.** Shipping one issue as its own PR+release while planned units remain is a **failure of this pipeline**.

The builder enforces the per-issue side (fast gates only, commit to the batch branch, no push). You enforce the batch side: **never dispatch a seal/release step while `batch.count < <BATCH_SIZE>` AND planned units remain.**

**Build-ahead is allowed; seal-ahead is not.** Verify runs in the background (it touches only the verify env and its own result file). The builder may keep **building** the next batch onto a fresh `batch/<id>` branch while a prior batch is being verified — building writes neither `<MAIN_BRANCH>` nor the env, so it overlaps safely. What must **not** overlap is the singleton critical section: there is one `<MAIN_BRANCH>`, one release, one `verify_state`, so **a new batch may not be *sealed* while `verify_state.status` is `owed`/`verifying`/`red`** (a prior batch is still in release/verify). Build up to `<BATCH_SIZE>` then *wait* for the verify to clear before sealing. This caps in-flight to one batch in the critical section while keeping the builder busy.

## Step 0 — Preflight (every firing)

1. **Working tree clean?** `git status --porcelain`. Dirty → exit (another session owns this tree). Don't stash/switch.
2. **On `<MAIN_BRANCH>`, current?** `git fetch origin && git checkout <MAIN_BRANCH> && git pull --ff-only`. FF fails → exit + report.
3. **Lock check.** `.claude/state/autoloop.lock` mtime < 10 min ⇒ another firing is running → exit. Else touch it.
4. **Read the work queue.** Create from `work-queue-template.json` if absent. Seed `started`/`last_invocation`. Then **prune bounded state** (see *State hygiene* below — trim `completed[]`/`notes[]`, drop closed-issue notes) so the file stays token-cheap; you are the single writer, so preflight is the place to do it.
5. **Fold in any background Verify result.** If `.claude/state/verify-result.json` exists, the background agent finished: copy its `{sha, status, detail, verified_at}` into `verify_state` (you are the single writer of the shared queue's `verify_state`), then **delete the result file**. If `verify_state.status == "verifying"` but no result file exists and `since` is >20 min old, the agent died — reset `verify_state.status` to `"owed"` so it relaunches.
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
The shared queue is .claude/state/work-queue.json — read it, write results back (unit status,
completed/review/needs_refinement/…, set verify_state=owed at seal), and return ONE line:
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

Builder mode (`build` vs `seal`) and the unit `gate`/`security` go in the context line. After a **foreground** agent returns: **re-read `work-queue.json`** (the file is authoritative, not the summary), append the one-liner to your tally, go back to Step 1. The **background** Verify does not block — proceed immediately; its result is folded in at the next preflight (Step 0 fold-in), and the harness re-invokes the loop when it completes.

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

## State hygiene (bounded queue — keep it token-cheap)

Every stage re-reads `work-queue.json` **in full**, so its size is a per-tick token cost. The queue is a **bounded orchestration cache, never a ledger** — the durable record of the work already lives in GitHub (open/closed issues, labels, merged PRs) and git history. At preflight (single writer), enforce:
- `completed[]` — keep only the **last released batch's** units; drop older. To see everything ever shipped, read closed issues / merged PRs, not this file.
- `notes[]` — **run-scoped scratch**, cap ~15 entries and drop any note whose subject issue is closed. It records *why the loop did X this run*, not project history; keep each note one terse line.
- `release_warnings[]` — clear on release.
- Never embed the schema or long prose (`_comment`) in the live file — the schema lives in `work-queue-template.json`.

This is also **why the queue is a local file and not a GitHub artifact**: it holds rich in-flight orchestration (issue clustering, the batch branch, the verify state-machine) that GitHub issues can't model without many API calls and two-way-sync races. The parts that *are* durable are projected to labels one-way (above) and otherwise left to GitHub — so the file can stay small and cheap to re-read.

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
- Stages: `stages/planner.md`, `stages/builder.md`, `stages/verify.md` (this dir; Verify runs in the background and writes `.claude/state/verify-result.json`). Queue schema: `work-queue-template.json`.
- Worked example: `mdopp/servicebay` (`.claude/skills/autoloop-issues/`) — its Verify stage is `box-verify.md` (a `:dev`/`:latest` channel flip).
