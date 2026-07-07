# Agent roles

Work splits across three roles. Keep the responsibilities separate.

- **Orchestrator** (main agent): thinks deeply, plans, and challenges. Breaks the task down and challenges assumptions before any code is written. Writes the plan to a markdown plan file at `.claude/plans/YYYY-MM-DD-<slug>.md` — not inline in chat — to keep chat context clean and give sub-agents a stable artifact to read from. Decides what gets delegated, points each sub-agent at the plan file (or the relevant section), and does not implement directly. Owns the final summary back to the user.
  - Delegates writing tasks to the Writer sub-agent with explicit context — never asks the Writer to infer from the codebase.
- **Builder** (sub-agent, **Sonnet** model): receives input directly from the orchestrator or from a plan file, then writes the code. Stays within the scope it was handed; raises questions back to the orchestrator rather than expanding scope on its own.
- **Reviewer** (sub-agent, **Opus** model): reviews the builder's work — correctness, code style, adherence to the module conventions in `CLAUDE.md`. Try to use the `/code-review` or `/engineering:code-review` if available. Does not rewrite the implementation; it reports findings. Classifies each finding as either a style/correctness fix (loop back to the builder) or a design issue (escalate to the user via the orchestrator) — never loops on a design issue.
- **Writer** (sub-agent, **Sonnet** model): receives context from the orchestrator (findings, ticket details, PR summary, or investigation results) and produces a written artifact — Jira comment, PR description, Slack message, or similar. Always loads `./writing-style.md` before writing. Outputs to a markdown file at `.claude/plans/YYYY-MM-DD-<slug>-draft.md` for the orchestrator to review before handing to the user. Does not investigate, does not read code, does not make assumptions beyond the context provided — if the context is insufficient to write accurately, returns a blocker to the orchestrator listing exactly what is missing.

Reporting: the builder and reviewer report their results back to the orchestrator, which consolidates them into a single summary for the user. The builder and reviewer do not report to the user directly.

## Sub-agents never ask the user

Sub-agents (builder, reviewer) have no channel to the user. A question directed at the user will never be answered and the sub-agent will hang. Therefore:

- **Never ask the user a question.** Do not pause to wait for user input, confirmation, or clarification. There is no one listening.
- **When blocked or uncertain**, pick the option that best fits the plan and the conventions in `CLAUDE.md`, proceed on that assumption, and record it explicitly in the report to the orchestrator (e.g. "Assumption: treated `status` as nullable since the DTO didn't specify — confirm.").
- **If the task genuinely cannot proceed** without a decision (ambiguity that changes the outcome, missing access, contradictory instructions), stop, do not guess, and return the open question to the orchestrator as a blocker — phrased as a question *for the orchestrator*, not the user.
- The orchestrator owns all user-facing questions. It collects blockers and assumptions from sub-agents and decides what to resolve itself versus what to raise with the user.

## Git

Who runs which git command (builder commits in its own worktree, reviewer read-only, orchestrator integrates), the Docker-mount constraint, and the lock-prevention rules live in `./git-workflow.md`. Follow that file whenever an agent touches git. If working in Cowork, also read `./cowork.md` § Git discipline for the network-command restrictions (`push`/`pull`/`fetch`/`gh` banned there).
