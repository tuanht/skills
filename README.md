# skills

My personal skill set for customizing Claude Code across projects.

## Usage

This repo is meant to be consumed as a git submodule at `.claude/skills` in any project:

```
git submodule add git@github.com:tuanht/skills.git .claude/skills
```

To pick up updates in a project that already has the submodule:

```
git submodule update --remote .claude/skills
```

Or install as a Claude Code plugin marketplace:

```
/plugin marketplace add tuanht/skills
/plugin install orchestrate@tuanht-skills
```

## Skills

- **orchestrate** — Orchestrator/Builder/Reviewer/Writer role split for delegating work across sub-agents. See `orchestrate/references/` for the full role definitions.
