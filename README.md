# dotfiles

Personal configuration, version-controlled so it survives a machine and syncs across them.

## Claude Code — generic skills

`claude/skills/` holds **generic, project-agnostic** Claude Code skills. It is symlinked into place:

```sh
ln -s "$PWD/claude/skills" ~/.claude/skills
```

Generic skills here:

- **project-standards** — set up or incrementally extend a project's engineering standards (CI, pre-commit hooks, release-please, conventional commits, architecture-invariant ratchet, issue hygiene, and a tailored `autoloop-issues` skill). Stack-adaptive and idempotent: `detect → audit → propose → confirm → apply → record`. Includes `templates/autoloop-issues.template.md`, the placeholdered "autoloop with the latest standards" to stamp into any repo.

**Project-specific** skills (e.g. a repo's own `autoloop-issues`) live in that repo's `.claude/skills/`, not here.

## Setup on a new machine

```sh
git clone https://github.com/mdopp/dotfiles ~/dotfiles
ln -s ~/dotfiles/claude/skills ~/.claude/skills
```
