---
name: project-standards
description: Set up — or incrementally extend over a project's lifetime — the engineering standards distilled from ServiceBay: CI pipeline, pre-commit hooks, release-please versioning, conventional-commit conventions, an architecture-invariants ratchet, issue hygiene, and a tailored autoloop-issues skill. Stack-adaptive (Node/TS, Python, mixed) and idempotent (audit → propose → apply → record). Use when the user says "set up project standards", "add CI / hooks / release-please", "bootstrap this repo", "bring this project up to our standards", or "add autoloop to this project".
---

# Project standards: set up & extend

A generic, **stack-adaptive, idempotent** procedure to apply (or upgrade) the engineering standards learned from ServiceBay to any repository. Run it on a fresh repo to bootstrap, or on an existing repo to fill gaps / upgrade to the latest catalog version.

**Core loop every invocation:** `detect → audit → propose → confirm → apply → record`. Never blindly overwrite — only add what's missing or upgrade what's behind the catalog version, and show the diff before applying anything shared/irreversible.

This skill writes config and CI into the target repo. Treat CI files, hooks, and release config as code: open them for review, don't commit unless the user asks (follow their git conventions), and never `--no-verify`.

---

## Step 0 — Detect the project

Gather, in parallel:
- **Stack**: `package.json` (Node/TS — note workspaces/monorepo, the package manager via lockfile, whether `tsc`/eslint/vitest/jest are present), `pyproject.toml`/`setup.cfg`/`requirements*.txt` (Python — note pytest/ruff/alembic), `go.mod`, `Cargo.toml`, etc. A repo can be **mixed** (e.g. TS app + Python service) — apply per-area.
- **Host**: is it a GitHub repo? `gh repo view --json nameWithOwner,defaultBranchRef` and branch-protection `gh api repos/<owner>/<repo>/branches/<main>/protection` (404 = unprotected).
- **Existing standards**: `.github/workflows/*`, `.husky/`/`.pre-commit-config.yaml`/`lint-staged` config, `release-please-config.json`+`.release-please-manifest.json`+a release workflow, `commitlint`, dependency-cruiser/arch scripts, `CLAUDE.md`, `.claude/skills/autoloop-issues/`, and a `.claude/standards.json` (this skill's own record).
- **Dependent-repo relationship**: does this project run *on* or *depend on* another (like OSCAR→ServiceBay)? If so, the autoloop gets cross-repo routing.

## Step 1 — Audit against the catalog

For each standard in the **Catalog** below, classify: `present & current`, `present but behind` (catalog version newer / missing a sub-part), or `missing`. Read `.claude/standards.json` if present to know what was applied and at which `standardsVersion`.

## Step 2 — Propose & confirm

Report the audit as a short table (standard → status → proposed action). Then use `AskUserQuestion` to let the operator pick which to apply/upgrade (multi-select). Don't apply CI/hooks/release config or commit anything without confirmation. For a fresh bootstrap, offer "apply all recommended" as the first option.

## Step 3 — Apply (idempotent, per stack)

Apply only the confirmed items. Each catalog entry below has detect/apply notes and a stack-adaptive template (inline or in `templates/`). Re-applying must be a no-op when already current.

## Step 4 — Record

Write/update `.claude/standards.json`:
```json
{
  "standardsVersion": 3,
  "appliedAt": "<ISO date>",
  "applied": { "conventional-commits": 1, "ci": 1, "pre-commit-hooks": 1, "release-please": 1, "arch-ratchet": 1, "autoloop-issues": 3, "claude-md": 1, "issue-hygiene": 1, "readiness-gates": 3 },
  "readinessGates": {
    "security":    "partial|present|missing|skipped — one-line note",
    "coverage":    "missing — no new-code coverage threshold",
    "performance": "skipped — CLI/library, latency irrelevant",
    "dependency":  "present — committed lockfile + frozen CI install",
    "humanReview": "partial — post-deploy review[]; pre-merge gate off"
  },
  "stack": ["node-ts", "python"],
  "notes": []
}
```
This record is what makes "extend over the lifetime" work: on a later run, compare `applied[*]` versions against the catalog's current version and offer upgrades for anything behind. Bump `CATALOG_VERSION` (below) whenever a standard's template changes, so existing projects get prompted to upgrade.

> Mirror the host project's ignore convention for `.claude/` (ServiceBay gitignores `/.claude/`). If the autoloop/state should stay local, add `/.claude/` to `.gitignore` (or `.git/info/exclude` for a no-commit local ignore); if the team wants the skill tracked, commit `.claude/skills/` and gitignore only `.claude/state/`.

---

## Catalog (CATALOG_VERSION = 3)

> v3: (a) added catalog #9 (**production-readiness gates**) — a cross-cutting readiness scorecard the audit scores on every run (security scan · new-code coverage · performance baseline · dependency/hallucination check · human review of sensitive areas), routing each gate to an existing catalog entry or proposing the missing job. Adapted from the "five gates" checklist. (b) catalog #6 (**autoloop-issues**) bumped 2 → 3: the Verify stage now runs **in the background** (`run_in_background`, writes its own `.claude/state/verify-result.json` the orchestrator folds in at preflight) so the builder **build-aheads** the next batch concurrently; added the `verify_state` `verifying` intermediate state, the **build-ahead-allowed / seal-ahead-forbidden** critical-section rule, and a **label mirror** (`blocked[]`/`needs_refinement[]` → issue labels, `verify_state` → release-PR label). Existing projects on `autoloop-issues: 2` should be offered the upgrade.
> v2: catalog #6 (autoloop-issues) rewritten from a single-file skill into a multi-agent **pipeline bundle** (`templates/autoloop/`) — orchestrator + Planner/Builder/Verify stages + shared work-queue, with the fast-per-issue / full-per-batch gate split and a `needs_refinement[]` human worklist. Existing projects on `autoloop-issues: 1` should be offered the upgrade.

Each standard records the **ServiceBay learning** behind it so future-you can judge edge cases.

### 1. Conventional Commits + no-parens rule
**What:** commits follow `type(scope): subject` (feat/fix/refactor/chore/docs/test). **Subjects contain no parentheses beyond the conventional `(scope)`.**
**ServiceBay learning:** a `(#non-numeric)` token or parens-heavy subject in a merged commit makes release-please run green but cut **no** release PR — silent release breakage. Keep subjects paren-free.
**Apply:** document in `CLAUDE.md` (catalog #7). Optionally add `commitlint` + a `commit-msg` hook (Node) — see `templates/commitlint.md`. This rule is also enforced socially via the autoloop and review.

### 2. CI pipeline
**What:** per-PR checks that must be green before merge. ServiceBay's set: **lint (0 errors), typecheck, unit tests, build, dependency/arch checks, security scan (semgrep)**. Adapt to stack.
**ServiceBay learning:** lint warnings are allowed but must not *increase*; tests verify code correctness, not feature correctness; pin the CI runtime and reproduce flakes locally with the **same** version (Node 24-vs-20 once hid a real race).
**Apply:**
- Node/TS → `templates/ci-node.yml` (jobs: lint, typecheck `tsc --noEmit`, test, build, optional depcruise/invariants, semgrep). Pin `node-version` and cache the lockfile.
- Python → `templates/ci-python.yml` (jobs: ruff/flake8, mypy if configured, pytest, build image if Dockerfile present, semgrep).
- Image-only repos (like OSCAR) → a `build-images` matrix that builds on PR (no push) and pushes on `main`/tags.
- Write the workflow only for jobs the repo can actually run (don't add a `tsc` job to a repo with no TS). Make jobs path-filtered in monorepos.

### 3. Pre-commit hooks
**What:** auto-format + lint **staged** files on commit; block on errors. Never bypass with `--no-verify`.
**ServiceBay learning:** ServiceBay runs `lint-staged` (eslint --fix on staged `*.{js,jsx,ts,tsx}`) via a pre-commit hook; failures stop the commit so broken lint never lands.
**Apply:**
- Node → `husky` + `lint-staged` (see `templates/hooks-node.md`): `eslint --fix` + formatter on staged files; optional `commitlint` on `commit-msg`.
- Python → `pre-commit` framework (`templates/hooks-python.md`): `ruff`/`black`/`ruff format` + `end-of-file-fixer`.
- Hook must be installed (`husky install` / `pre-commit install`) and the install wired into a `prepare`/postinstall step so it survives fresh clones.

### 4. release-please versioning
**What:** automated versioning + CHANGELOG via release-please. **Never manually bump versions** — merging the release PR bumps `package.json`/manifest/CHANGELOG and tags the release.
**ServiceBay learning:** release-please owns `package.json`/`CHANGELOG.md`/`.release-please-manifest.json`; never hand-edit those or the release PR's contents. The parens-in-commit pitfall (catalog #1) silently breaks it. Release PR lands on a predictable branch (`release-please--branches--<main>--components--<name>`).
**Apply:** add `templates/release-please.yml` (workflow), `release-please-config.json`, and `.release-please-manifest.json` seeded at the current version. Pick the release type (`node`, `python`, `simple`). For tag-only repos that don't want release-please (e.g. OSCAR cuts releases by pushing `v*`), **skip** this and document the tag flow instead — don't force release-please where the team uses tags.

### 5. Architecture-invariants ratchet
**What:** a checked-in script/config that fails CI when a structural invariant is violated; **tighten over time, never loosen**.
**ServiceBay learning:** `scripts/check-invariants.ts` + `.dependency-cruiser.cjs` gate cross-layer imports; when an exemption is resolved, ratchet it tighter. Reviews run the rubric first — "all green" is a valid result; don't invent findings beyond the documented invariants.
**Apply:** only where it earns its keep (layered codebases). Node → `dependency-cruiser` config + an `npm run check:arch`. Python → import-linter. Start permissive, tighten as the codebase allows. Skip for small/flat repos.

### 6. autoloop-issues pipeline (latest standard)
**What:** install a **project-tailored multi-agent autoloop pipeline** into the repo's `.claude/skills/autoloop-issues/` from the bundled generic template directory. The skill is an **orchestrator** that spawns a fresh sub-agent per stage (Planner → Builder → Verify) coordinated through a shared `.claude/state/work-queue.json`, so the long-lived loop session stays clean and each stage reasons in fresh context.
**ServiceBay learning (the "latest standards" the template encodes):**
- **Pipeline, not monolith.** Orchestrator dispatches one stage/tick; stages hand off only through `work-queue.json`; the loop session accumulates only one-line summaries. Each stage runs at a model matched to the cost of being wrong (builder=opus for code / haiku for lint sweeps; planner & verify=sonnet).
- **Human attention goes to one queue: `needs_refinement[]`.** The Planner refuses to guess past ambiguity — it bounces each underspecified issue there with a *specific* question. The pipeline runs everything downstream autonomously.
- **Batch economy + the gate split.** The expensive pipeline (full suite, CI, release, `/verify`) runs **once per batch (≤`<BATCH_SIZE>` issues on one persistent branch), never per issue.** Per-issue gates are **fast** — lint + arch + *changed-tests only* (e.g. `vitest --changed` / `pytest --picked`); the **full** suite runs only at the batch seal. (Rationale: arch + transitive changed-tests catch the common regressions cheaply; the full run is the seal safety-net; in-session atomic commits make a red full-run a cheap bisect.)
- **Security/sensitive issues run the full loop** and are flagged (`security:true` → `review[]`) for **post-deploy** review, not blocked pre-merge (a project can opt back into a pre-merge draft gate — see `templates/autoloop/stages/builder.md`); external-commenter tickets parked on `awaiting_user[]`, never auto-replied.
- Real-environment **`/verify`** for path-mandated changes is its own batched stage gating the release, and it runs **in the background**: it writes its verdict to its own `.claude/state/verify-result.json` (never the shared queue, to avoid racing the builder), which the orchestrator folds into `verify_state` at preflight. This lets the builder **build-ahead** the next batch while a prior one verifies — only the seal→release critical section serializes (**build-ahead allowed, seal-ahead forbidden** while `verify_state` is `owed`/`verifying`/`red`). Manual merge gate when `main` is unprotected; release-please preflight merges the open release PR first. A **label mirror** projects `blocked[]`/`needs_refinement[]` → issue labels and `verify_state` → a release-PR label so a human sees the same worklist (file → labels only, one-way).
- **No-eligible-issues tracks** (Planner): (a) lint-sweep units, (b) refine & unblock, (c) codebase evaluation refills the queue, and — for dependent repos — (d) end-to-end validation + **cross-repo issue routing** (file platform bugs upstream, mark local issue blocked-waiting, wait).
**Apply:** copy the whole `templates/autoloop/` directory → `<repo>/.claude/skills/autoloop-issues/` (so it lands as `SKILL.md`, `USAGE.md`, `work-queue-template.json`, and `stages/{planner,builder,verify}.md`), then **fill every `<PLACEHOLDER>`** across all files by interview/detection (key at the top of `autoloop/SKILL.md`): repo slug, batch size, labels, **fast vs full gate commands** (split the test suite into a changed-tests run and a full run), path-mandated `/verify` list, verify procedure + staging mechanism, release flow, upstream repo (if dependent), per-stage models. Uncomment the `<UPSTREAM_REPO>`-gated blocks only for dependent repos. Reference the live ServiceBay copy (`mdopp/servicebay`) as the worked example.

### 7. CLAUDE.md conventions
**What:** a `CLAUDE.md` capturing the house rules so every session (human or agent) follows them.
**Apply:** create/extend `CLAUDE.md` with: conventional commits + the no-parens rule; **scope discipline** (smallest change that works; three similar lines beat a premature abstraction; no speculative error-handling); **comments default-off** (only the non-obvious *why*); **verify in the real app/environment**, not just type-check/tests; **never** manually bump versions / `--no-verify` / loosen invariants; and the issue-hygiene rule (#8). Use `templates/CLAUDE.md.skeleton` as the base; merge, don't clobber an existing one.

### 8. Issue hygiene
**What:** issues capture **symptom + repro + starting-point files** — not fix-plans, acceptance bullets, or "what to fix" sections. The fix is decided in the PR.
**ServiceBay learning:** fix-plan-heavy issue bodies rot; symptom-style issues age well and let the solver choose the approach.
**Apply:** add an issue template (`.github/ISSUE_TEMPLATE/`) with Symptom / Repro / Starting-point files / (optional) Impact. Note the convention in `CLAUDE.md`.

### 9. Production-readiness gates (readiness scorecard)
**What:** a cross-cutting audit of the five gates that decide whether AI-assisted code is fit for real use. On every run, **score each gate `present` / `partial` / `missing`** for the target repo and propose the next concrete step. This standard installs no single artifact — it scores the five and routes each to the relevant catalog entry or a new job. Adapted from the "five gates" production-readiness checklist (Vibe-Coding → Echtbetrieb).
**The five gates & how they map:**
1. **Security scan** — SAST on every change, **0 ERROR-severity findings**, no exception without a documented risk. Maps to catalog #2 (CI semgrep). *Next steps to offer:* add a blocking `semgrep --severity=ERROR` job if missing; enable a broad ruleset (`p/security-audit`) once the custom baseline is clean; turn on **dependency-CVE scanning** (`npm audit` / `pip-audit` / Dependabot). *ServiceBay today:* custom semgrep rules block (ERROR), but no broad ruleset and `npm audit` is disabled (`--no-audit`) → CVE scanning is the open gap.
2. **Test coverage** — **≥80 % on *new* code** ("the right 80 %, not the wrong 90 %"). **Not yet otherwise in the catalog.** *Propose:* a coverage job (`vitest --coverage` / `pytest --cov`) gated on a **diff/patch threshold** (diff-cover, Codecov patch %), *not* a whole-repo number — legacy debt must not block. *ServiceBay today:* no coverage measured or gated → missing.
3. **Performance baseline** — P95 under a target (book: <200 ms) with an **automatic alarm at ~20 % regression**. **Not yet otherwise in the catalog.** *Propose* only where latency matters (services/APIs): a benchmark test or a captured baseline JSON the CI diffs against; **skip for CLIs/libraries** (document the skip). *ServiceBay today:* none → missing.
4. **Dependency / hallucination check** — ~20 % of AI-suggested packages don't exist (Spracklen et al., USENIX Security 2025). **Structurally covered** by a committed lockfile + frozen install in CI (`npm ci` / `--frozen-lockfile` / `pip-sync`) — a hallucinated dep fails install. *Next step:* confirm the lockfile is committed and CI installs frozen; add CVE scanning (overlaps gate 1). *ServiceBay today:* lockfile + `npm ci` → covered for the hallucination failure mode.
5. **Human review of sensitive areas** — auth, payment, personal data get **≥1 experienced review**. Maps to catalog #6's `security:true` → `review[]` — **but the autoloop default is POST-deploy review, the inverse of this gate.** *Decision to surface:* for repos touching money/PII/auth, offer the **pre-merge draft gate** opt-in (`stages/builder.md`) so sensitive changes get human eyes *before* shipping; document whichever the team picks. *ServiceBay today:* deliberately post-deploy (`review[]`) — fine for a self-hosted homelab, **flag explicitly for any repo with external users / payments**.
**ServiceBay learning:** ServiceBay itself scores strong on 1 & 4, deliberately inverts 5 (post-deploy), and is **missing 2 & 3** — proof the scorecard is honest about its own origin project, not aspirational. Always offer, never impose perf/coverage gates where they don't earn their keep (libraries, CLIs, early prototypes); document every skip.
**Worked apply-mechanisms (reusable patterns proven on ServiceBay):**
- **Diff-coverage, not global threshold** (gate 2): a global % fails on legacy debt. Instead a small house-style check script (`scripts/check-diff-coverage.*`) intersects `git diff --unified=0 <merge-base>` with the coverage report (v8 JSON / lcov / `coverage.py`) and fails when *changed* lines fall below a **ratchetable floor** (start ~70%). Run it in the **full/seal gate**, never per-issue. Avoids a new dep if the language already has a coverage provider.
- **Decoupled CVE audit** (gates 1 & 4): keep the install fast and frozen (`npm ci --no-audit` / `pip-sync`) and run vulnerability scanning as its **own CI job** (`npm audit --audit-level=high` / `osv-scanner` / `pip-audit`), **report-only first → ratchet to blocking** on high/critical once the baseline is clean (the same WARNING/ERROR tiering as semgrep). Add Dependabot/Renovate for auto-update PRs; in an autoloop repo, route those PRs as a planner **filler track**.
- **Additive SAST ruleset** (gate 1): land registry rulesets (`p/security-audit`, `p/typescript`) as a **second, non-blocking** semgrep step, triage into the project's `.semgrep.yml` baseline, then promote rules to blocking incrementally. For public repos, add **CodeQL** (free, zero-maintenance, surfaces in the Security tab).
- **Relative perf baseline, not absolute SLO** (gate 3): in CI, micro-benchmark the genuinely hot paths (e.g. a test runner's `bench()`), commit a **baseline file**, and fail on a **relative regression** (>~25% vs baseline, compare medians — CI runners are noisy, never gate on absolute ms). Re-baselining is an explicit committed step (like a snapshot). A real production P95+alarm is a heavier phase-2 build (runtime histogram + dashboard), worth it only for latency-sensitive services.
**Apply:** primarily **audit+propose**. Record the scorecard in `standards.json` under a `readinessGates` object (`{security, coverage, performance, dependency, humanReview}` each `present|partial|missing|skipped` + a one-line note), so later runs show movement.

---

## Bundled templates (`templates/`)

- `autoloop/` — the generic, placeholdered autoloop **pipeline bundle** (catalog #6): `SKILL.md` (orchestrator), `USAGE.md`, `work-queue-template.json`, and `stages/{planner,builder,verify}.md`. **The most important deliverable** — copy the whole dir into a repo's `.claude/skills/autoloop-issues/` and fill the placeholders.
- (Create the smaller snippets — `ci-node.yml`, `ci-python.yml`, `release-please.yml`, `hooks-node.md`, `hooks-python.md`, `commitlint.md`, `CLAUDE.md.skeleton` — on first use from the inline descriptions in the catalog if they don't yet exist, and keep them here so future runs reuse them. Bumping a template ⇒ bump `CATALOG_VERSION` so existing projects get an upgrade prompt.)

## Things this skill does NOT do
- Does not force release-please onto a repo that releases by tags (offer, don't impose).
- Does not add CI jobs the repo can't run, or hooks for tools it doesn't use.
- Does not commit or push without the user's go-ahead, and never `--no-verify`.
- Does not overwrite an existing `CLAUDE.md`/CI/hook config — it merges/upgrades and shows the diff.
- Does not manually edit versions, CHANGELOG, or the release-please manifest (catalog #4).

## Reference
- Worked examples: `mdopp/servicebay` (`.claude/skills/autoloop-issues/`, its CI `.github/workflows/ci.yml`, release-please on `release-please--branches--main--components--servicebay`) and `mdopp/oscar` (the dependent-repo autoloop variant with cross-repo routing + end-to-end track).
