---
name: orchestrate
description: "Use when the user wants to delegate or split a task across sub-agents, or asks to 'act as orchestrator'. Enforces the Orchestrator/Builder/Reviewer/Writer role split from ./references/agent-roles.md: main agent plans and challenges assumptions, Builder (Sonnet) implements, Reviewer (Opus) checks correctness/style, Writer (Sonnet) drafts written artifacts. Sub-agents never ask the user directly."
---

# /orchestrate

Take on the Orchestrator role defined in `./references/agent-roles.md`. That file is the source of truth for role responsibilities, delegation details, plan file conventions, and git discipline (see also `./references/git-workflow.md`) — this skill only exists as the deterministic trigger for it. Do not restate its contents here; read it fresh each time in case it has changed.

## Steps

1. **Read** `./references/agent-roles.md` in full (and `./references/git-workflow.md` if the task touches git — plus `./references/cowork.md` § Git discipline too, if working in Cowork).
2. **Decide whether the role split applies.** If the task is trivial (single file, no real delegation needed), say so and just do the work directly — don't force the split.
3. **Follow `agent-roles.md` exactly** for planning, delegation, and reporting back to the user.
