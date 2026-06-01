# Catalog #3 — Pre-commit hooks (Python), via the `pre-commit` framework.

Auto-format + lint **staged** files on commit; block on errors. Never bypass
with `--no-verify` (ServiceBay learning: a bypassed hook lands broken lint).

## `.pre-commit-config.yaml`

Pin `ruff-pre-commit`'s `rev` to the **same** ruff version CI installs, so the
hook and CI never disagree. Skip `check-yaml` in repos whose YAML contains
unquoted templating placeholders (e.g. ServiceBay `template.yml` with
`{{VAR}}`) — those aren't valid standalone YAML and the hook would false-fail.

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v<RUFF_VERSION>
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
        args: [--markdown-linebreak-ext=md]   # preserve markdown hard breaks
      - id: check-merge-conflict
```

## Install (survives fresh clones)

`pre-commit` has no postinstall equivalent of husky's `prepare`, so the
one-time install must be documented in `CLAUDE.md`/README:

```bash
pip install pre-commit   # or pipx install pre-commit
pre-commit install
```

`pre-commit run --all-files` once after install to format the existing tree (or
let it drift in incrementally — the hook only touches staged files per commit).
