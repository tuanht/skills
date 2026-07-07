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

## Skills

- **orchestrate** — Orchestrator/Builder/Reviewer/Writer role split for delegating work across sub-agents. See `orchestrate/references/` for the full role definitions.
- **trading-analysis** — TradingView MCP-driven price-action analysis and trade planning workflow (H4 bias / H1 execution / Daily swing validation), persisted to a trading wiki.
