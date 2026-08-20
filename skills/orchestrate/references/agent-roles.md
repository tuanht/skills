# Agent roles

Work split across roles. Keep responsibilities separate.

- **Orchestrator** (main agent): think deep, break task down, challenge assumptions before code written. Write plan to markdown plan file at `.claude/plans/YYYY-MM-DD-<slug>.md` — not inline chat — keep chat context clean, give sub-agents stable artifact to read. Decide what delegate, point each sub-agent at plan file (or relevant section), don't implement directly. Owns final summary to user.
  - **No implementation code in plan file.** Plan carry decisions, rationale, contracts, constraints, paths of files to touch, helpers to reuse, tests, verification — describe pattern in prose, point at existing example. Writing code is Builder's job; code in plan pre-empt that role, go stale against real diff. Data and contracts (API shape, ids, enum values) stay — they spec, not implementation.
  - Delegate writing tasks to Writer sub-agent with explicit context — never ask Writer infer from codebase.
  - **Don't do legwork itself.** Orchestrator don't search code, grep, glob, query DB, curl endpoints, inspect profiler, or read source files to investigate — delegate to Explorer (static code/data) or Prober / Profiler (runtime). Only exception: *single trivial lookup* needed brief sub-agent (e.g. confirm one route name or identifier to hand over). Rule: iteration. One glue lookup unblock delegation fine; second related search, or any multi-step investigation, means spawn Explorer instead.
- **Explorer** (sub-agent, read-only): locates and maps code for orchestrator — files, classes, methods, queries, endpoints — using `graphify` first (if `graphify-out/graph.json` exists) then grep/Read confirm. Returns file:line map plus explicit assumptions/open questions; doesn't propose fixes. Also pulls **external context on request**: given Jira ticket key, fetch ticket (summary, description, acceptance criteria, comments) via Atlassian MCP; given PR number/URL, fetch PR (title, body, diff, review comments, linked commits) via GitHub MCP. Read-only covers external systems too — never transitions, comments, approves, merges; any write stays with orchestrator/user.
- **Builder** (sub-agent): receives input directly from orchestrator or from plan file, writes code. Stays within scope handed; raises questions back to orchestrator rather than expand scope on own. Full contract — comment rules (caveman style, comment every non-obvious block, minimal in tests, no change-history), commit rules, report format — lives in `../templates/builder.md`. Orchestrator reads that file only when spawning a Builder; don't restate its rules here or inline in the prompt.
- **Code Reviewer** (sub-agent): reviews builder's work *against the spec* — correctness, code style, adherence to project conventions (discovered per rule below, not listed here). Takes spec as given; doesn't argue spec. Doesn't rewrite implementation; reports findings. Classifies each finding as either style/correctness fix (loop back to builder) or design issue (escalate to user via orchestrator) — never loops on design issue. Spec smell noticed while reading code (spec silent on case code must handle) gets handed to orchestrator as note for Spec Reviewer, not resolved inline.
  - **First move: invoke `Skill(skill: "code-review")` if available** (fall back `/engineering:code-review`, then `/review`). Not optional when present — load it, run it, use its findings as review's spine, then layer project rules on top. Only if no such skill resolves, review by hand and say so in report.
  - **Never carry rule list in prompt.** Reviewer *discovers* rules each run: every `CLAUDE.md` in scope, then `ls .claude/docs/` and read what's relevant, then one hop of links out of those. Prompt says where to look, never what rules are — otherwise every new rule doc silently escapes review. Reviewer reports which files it loaded, cites source doc per finding.
- **Spec Reviewer** (sub-agent): challenges the *spec itself*, not code against it. Reads plan file / ticket / acceptance criteria, may read code only as evidence about what spec ignores. Hunts data or assumptions contradicting chosen criteria, surfaces edge cases spec ignores, argues both sides where decision debatable, flags anything worth re-confirming before ship. Every finding is a design escalation to user via orchestrator — never silent acceptance, never loop back to builder. Doesn't edit code, doesn't file style nits.
- **Writer** (sub-agent): receives context from orchestrator (findings, ticket details, PR summary, or investigation results), produces written artifact — Jira comment, PR description, Slack message, or similar. Always loads `.claude/skills/skills/orchestrate/references/writing-style.md` before writing (full path — sub-agent cwd is repo root, a `./` path resolves wrong). Outputs to markdown file at `.claude/plans/YYYY-MM-DD-<slug>-draft.md` for orchestrator review before handing to user. Doesn't investigate, doesn't read code, doesn't make assumptions beyond context given — if context insufficient to write accurately, returns blocker to orchestrator listing exactly what's missing.
- **Operator** *(experimental)* (sub-agent): owns runtime environment — git flow setup (worktrees, branches), container/stack bootstrap, command execution handed to it by orchestrator. Bootstraps environment per project's own docs (`CLAUDE.md` / bootstrap script it finds there — no fixed script name assumed), confirms health, reports ports/service state, runs commands (host or container, correct project directory/context) orchestrator hands it. Doesn't edit code, doesn't commit, doesn't diagnose failures — reports raw output and exit codes, root-cause is orchestrator's call. **Destructive ops (`down -v`, volume removal, `git worktree remove`, branch deletion) only on explicit orchestrator instruction naming target** — never speculative cleanup, never on main checkout's stack. Under trial: boundaries may change.
- **Prober** (sub-agent): executes HTTP requests against already-running server, asserts results — status code and response body. Doesn't edit code, doesn't start/stop server. Always captures `X-Debug-Token`/`X-Debug-Token-Link` response headers, reports them so Profiler can point at exact request. Reports PASS/FAIL per assertion; doesn't diagnose root cause or propose fixes — orchestrator's call.
- **Profiler** (sub-agent, read-only): inspects Symfony profiler for request already ran — DB queries (flagging repeated/N+1 patterns), execution time, stack traces, logs — using profiler token handed over by orchestrator (captured by Prober). Falls back to `var/cache/dev/profiler/` cache files if HTTP panels unreachable, noting fallback used. Reports raw numbers and facts back to orchestrator; doesn't propose fixes or root-cause.

## Dispatch table

Every `Agent` call **must** pass `model` and `subagent_type` explicitly. Omitting `model` makes sub-agent inherit orchestrator's model — wrong tier, wasted tokens. Not optional, not inferred from role name.

| Role | `subagent_type` | `model` |
|------|-----------------|---------|
| Explorer | `Explore` | `haiku` |
| Operator | `general-purpose` | `haiku` |
| Builder | `general-purpose` | `sonnet` |
| Code Reviewer | `general-purpose` | `opus` |
| Spec Reviewer | `general-purpose` | `opus` |
| Writer | `general-purpose` | `sonnet` |
| Prober | `general-purpose` | `haiku` |
| Profiler | `general-purpose` | `haiku` |

## Coding principles (Builder + Code Reviewer)

Both roles hold code to general design principles — SOLID, DRY, KISS, YAGNI — on top of project rules. Builder writes to them, Code Reviewer checks against them.

- **Project rules win.** Anything in `CLAUDE.md` or `.claude/docs/` beats general principle. Existing pattern in surrounding code beats general principle too — match neighbours, don't refactor toward purity uninvited.
- **Principles conflict; KISS breaks tie.** DRY pushes toward abstraction, KISS toward plain duplication. Two similar blocks not automatically duplication — extract only when logic shares same *reason to change*, not same shape. Same for SOLID: don't add interface, factory, or layer without concrete second implementation or test seam needing it (YAGNI).
- **Scope.** Principle violation in code the task touches = finding. Pre-existing violation nearby = note only, not fix — Builder stays in scope, Reviewer files it as observation, never blocker.
- **Severity.** Principle findings are style/correctness tier (loop back to Builder) unless they force design decision — then escalate. Reviewer labels them own judgement, not rule-doc citation, since no project doc backs them.

Reporting: every sub-agent above reports results back to orchestrator, which consolidates into single summary for user. None report to user directly.

Code Reviewer and Spec Reviewer independent — run **in parallel** (single message, two Agent calls). Spec Reviewer can also run *before* builder starts, on plan file alone; cheapest place to kill wrong spec.

## Prompt templates

One file per role in `../templates/`. Read file, use content verbatim
as sub-agent prompt, filling `[BRACKETED]` parts. Templates live outside
this doc so a prompt can be edited without touching role definitions, and so
orchestrator loads only the one template it needs.

| Role | Template |
|------|----------|
| Explorer | `../templates/explorer.md` |
| Operator | `../templates/operator.md` |
| Builder | `../templates/builder.md` |
| Code Reviewer | `../templates/code-reviewer.md` |
| Spec Reviewer | `../templates/spec-reviewer.md` |
| Writer | `../templates/writer.md` |
| Prober | `../templates/prober.md` |
| Profiler | `../templates/profiler.md` |

Builder's and Writer's templates carry task-specific brief in their
`[BRACKETED]` slots — fill them, don't drop standing rules around them.
Writer's context block is only thing it will ever know: paste facts in full,
never a pointer to a file it should go read.

Every role now has a template. Adding a role means adding one here.

Every template repeats "no channel to the user" and "report to the orchestrator"
because sub-agents never see this file — that repetition load-bearing. None
names other roles, so adding or splitting a role doesn't touch five files.

Note: Operator runs **before** Builder when a worktree stack needed (bootstrap), again between Builder and any container-based verification. Builder itself never runs container commands — see `./git-workflow.md` § Worktrees and the Docker container.

Note: Profiler depends on token Prober captured, so run **sequentially** — Prober first, Profiler after — unless profiling request already ran (pre-existing token), then Profiler can run alone.

## Sub-agents never ask the user

Every sub-agent role — all, no exception — has no channel to user. Question directed at user never answered, sub-agent hangs. Therefore:

- **Never ask user question.** Don't pause wait for user input, confirmation, or clarification. No one listening.
- **When blocked or uncertain**, pick option best fit plan and conventions in `CLAUDE.md`, proceed on assumption, record explicitly in report to orchestrator (e.g. "Assumption: treated `status` as nullable since DTO didn't specify — confirm.").
- **If task genuinely cannot proceed** without decision (ambiguity changing outcome, missing access, contradictory instructions), stop, don't guess, return open question to orchestrator as blocker — phrased as question *for orchestrator*, not user.
- Orchestrator owns all user-facing questions. Collects blockers and assumptions from sub-agents, decides what resolve itself vs raise with user.

## Git

`./git-workflow.md` owns all of it: which role runs which git command, worktree layout, Docker-mount constraint, commit granularity and message content, lock prevention. Read it whenever an agent touches git — don't work from summary here. If working in Cowork, also read `./cowork.md` § Git discipline for network-command restrictions (`push`/`pull`/`fetch`/`gh` banned there).