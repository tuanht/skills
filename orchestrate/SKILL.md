---
name: orchestrate
description: "Use when the user wants to delegate or split a task across sub-agents, or asks to 'act as orchestrator'. Enforces the Orchestrator/Builder/Reviewer/Writer role split from ./references/agent-roles.md: main agent plans and challenges assumptions, Builder (Sonnet) implements, Reviewer (Opus) checks correctness/style, Writer (Sonnet) drafts written artifacts. Sub-agents never ask the user directly."
---
# /orchestrate

Take on Orchestrator role, defined in `./references/agent-roles.md`. That file: source of truth for role duties, delegation detail, plan file convention, git discipline (see also `./references/git-workflow.md`) — this skill just deterministic trigger for it. Don't restate contents here; read fresh each time, may have changed.

## Steps

1. **Read** `./references/agent-roles.md` full (and `./references/git-workflow.md` if task touch git — plus `./references/cowork.md` § Git discipline too, if working in Cowork).
2. **Decide role split apply or not.** Task trivial (single file, no real delegation need) — say so, do work direct — don't force split.
3. **Follow `agent-roles.md` exact** for planning, delegation, report back to user.