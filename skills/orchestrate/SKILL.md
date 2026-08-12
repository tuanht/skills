---
name: orchestrate
description: "Use when the user wants to delegate or split a task across sub-agents, or asks to 'act as orchestrator'. Enforces the Orchestrator/Builder/Code Reviewer/Spec Reviewer/Writer role split from ./references/agent-roles.md: main agent plans and challenges assumptions, Builder implements, Code Reviewer checks correctness/style against the spec, Spec Reviewer challenges the spec itself, Writer drafts written artifacts. Sub-agents never ask the user directly."
---
# /orchestrate

Take on Orchestrator role, defined in `./references/agent-roles.md`. That file: source of truth for role duties, delegation detail, plan file convention, git discipline (see also `./references/git-workflow.md`) — this skill just deterministic trigger for it. Don't restate contents here; read fresh each time, may have changed.

## Steps

1. **Read** `./references/agent-roles.md` full (and `./references/git-workflow.md` if task touch git — plus `./references/cowork.md` § Git discipline too, if working in Cowork).
2. **Decide role split apply or not.** Task trivial (single file, no real delegation need) — say so, do work direct — don't force split.
3. **Follow `agent-roles.md` exact** for planning, delegation, report back to user.
4. **Before spawning any sub-agent, read that role's template** from `./templates/` (table at end of `agent-roles.md`). Use its content **verbatim** as the prompt, filling only the `[BRACKETED]` slots — don't paraphrase, don't drop the standing rules, don't write a prompt from memory. Load only the template for the role being spawned; that's why they're separate files.
5. **Every `Agent` call passes `model` and `subagent_type` explicitly**, per the dispatch table in `agent-roles.md`. No omission — omitted `model` inherits orchestrator's tier.