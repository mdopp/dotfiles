# autoloop-issues — how to run (<REPO_SLUG>)

`autoloop-issues` is the **orchestrator** of a multi-agent pipeline. It spawns a fresh sub-agent per stage (Planner → Builder → Verify), coordinated through `.claude/state/work-queue.json`, so the loop session stays clean.

## Self-paced loop (recommended)
```
/loop /autoloop-issues
```
`/loop` re-fires the orchestrator on its own cadence. The work queue persists progress between firings; each stage runs in its own sub-agent context.

## What each stage does
- **Planner** (`stages/planner.md`) — grooms/clusters open issues into queue units, decomposes epics, and **bounces every underspecified issue to `needs_refinement[]` with a specific question.**
- **Builder** (`stages/builder.md`) — implements one unit onto the persistent `batch/<id>` branch with **fast gates**; runs the **full** suite + CI and merges at the batch boundary.
- **Verify** (`stages/verify.md`) — batched real-environment `/verify` for path-mandated changes; gates the release.

## Where human attention goes
Drain `needs_refinement[]` — sharpen ambiguous issues / answer the planner's questions. Everything else runs without you. Secondary: review `review[]` (security drafts).

## Stop / reset
Interrupt the session (the queue persists; mid-batch builds resume on the same branch). Reset: `rm .claude/state/work-queue.json` (recreated from `work-queue-template.json`).

## Tuning models
The orchestrator sets a model per stage (table in `SKILL.md` Step 2). Don't downgrade the builder for real code; lint sweeps and verify can run cheaper.

## When NOT to run
- Another session is editing files here (orchestrator exits on a dirty tree).
- You haven't reviewed any of the first autonomous PRs yet.
- You're mid-incident on the verify target.
