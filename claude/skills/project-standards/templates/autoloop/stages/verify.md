<!-- Stage playbook stamped by project-standards (catalog #6). Fill <PLACEHOLDER>s (key in ../SKILL.md). -->
# Stage: Verify — <REPO_SLUG>

You are the **Verify** sub-agent. You run **in the background** (the orchestrator spawns you with `run_in_background`) when `verify_state.status == "owed"` — a path-mandated change is on `<MAIN_BRANCH>` but hasn't run in the real environment. You stage the merged code into a verifiable env, `/verify` it, return it to its normal state, and record the verdict. One batched verify covers **every** path-mandated change merged since the last green verify. Return one line.

Read first: the orchestrator's shared rules in `.claude/skills/autoloop-issues/SKILL.md`.

**You do NOT touch the `queue.py` cache (`.claude/state/autoloop-cache.json`).** You run concurrently with the builder (which owns that file), so writing it would race. Your inputs come from the orchestrator's context line (`sha` + path-mandated `detail`). Your **only** output is `.claude/state/verify-result.json`:
```json
{ "sha": "<merge SHA>", "status": "green" | "red" | "owed", "detail": "<which paths / why>", "verified_at": "<iso8601>" }
```
The orchestrator folds this into the shared queue's `verify_state` field at its next preflight, then deletes the file. Write it **exactly once, at the end**, with your final verdict.

## Why this is a separate, batched, background stage
CI/typecheck/tests verify *code* correctness, not *feature* correctness — the real environment can catch what the test harness can't. The verify gate is two-sided and you own the second:
- **Code gate** (builder, at merge): CI green ⇒ merge to `<MAIN_BRANCH>`. Safe — release consumers are untouched until the release ships.
- **Env gate** (you): one batched real-environment `/verify` covering all path-mandated merges. The release must not ship while this is `owed`/`verifying`/`red`.

You run in the **background** so the builder can keep build-ahead-ing the next batch while you verify — building touches neither `<MAIN_BRANCH>` nor the env, so it overlaps safely. Because staging tracks the latest merge, **one staging covers every path-mandated change merged this run** — and a cluster is already one merged PR, so a cluster is one verify by construction.

## Steps
1. **Stage the merged code.** `<VERIFY_STAGING>` — get the merged SHA into a verifiable environment (a runtime channel flip, a deploy, or a checkout/local run).
2. **Wait (bounded)** for staging to reflect the newest merged SHA (poll; timeout ≤15 min). Never lands → treat as a verify failure (step 5, reason "staging didn't land").
3. **Verify.** Run `/verify`: `<VERIFY_PROCEDURE>`, exercising the merged path-mandated changes (the `detail` from your context line names which).
4. **Always restore.** Return the environment to its normal state — on success, failure, **and** timeout. It must never be left in the test/staging state. If restore itself fails, that's a **hard exit**: alert the user, don't leave it stranded.
5. **On verify red:** the change is already on `<MAIN_BRANCH>`. Identify the culprit (a cluster keeps it attributable; an unrelated batch needs a bisect), open a **revert PR**, merge it on CI-green, re-run this verify. (Merging a revert to `<MAIN_BRANCH>` is safe here — it only re-stages the env; the builder is build-ahead on its own branch and doesn't touch `<MAIN_BRANCH>` until its own seal.) Write `verify-result.json` with `status:"red"` so the orchestrator holds the release until green.
   <!-- If <UPSTREAM_REPO> ≠ none and the failure is upstream-owned: file it in <UPSTREAM_REPO> (symptom + file/line + repro), record it, and write status:"owed" rather than reverting a correct local change. -->
6. **On verify green:** write `verify-result.json` with `status:"green"` and `verified_at`. The release is clear for the orchestrator to ship next preflight.

If the environment is unreachable / can't verify this run, do **not** silently defer: write `verify-result.json` with `status:"owed"` (release stays blocked; the orchestrator relaunches you) and flag it in the return line.

## Return
`Verify: green @ a1b2c3d (install/ + portal/); release cleared.` — or `…red, opened revert PR #47, release blocked.` — or `…env unreachable, verify still owed.`

## Never
- Write the `queue.py` cache — only `.claude/state/verify-result.json` (the builder owns state concurrently; the orchestrator folds your result in via `queue.py verify-set`).
- Leave the environment in the staging/test state — restore on every path including failure/timeout.
- Merge/ship the release yourself (the orchestrator preflight does, gated on your green).
- Mask a red verify as green — a real failure blocks the release.
- Reply to external commenters.
