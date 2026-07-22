# `/orchestrate` — Skill Demo

A walkthrough of the **orchestrate** skill on a real backend task: exploring a ticket, fixing an N+1 query storm on a shopping-cart endpoint through a self-correcting delegation loop, then shipping a PR.

The skill splits work across roles so the main agent (Orchestrator) never does the legwork itself — it plans, delegates, judges, and reports. Sub-agents do the searching, building, reviewing, and writing.

---

## The roles

| Role | Model | Does | Never does |
|------|-------|------|------------|
| **Orchestrator** | main | Plans, delegates, judges output, integrates git, reports to user | Search/grep/curl/edit code directly |
| **Explorer** | Haiku | Maps code (file:line), pulls Jira/PR context | Edits, proposes fixes |
| **Builder** | Sonnet | Writes code, commits in its worktree | Expands scope on its own |
| **Reviewer** | Opus | Reviews the diff, classifies findings | Rewrites the code |
| **Prober** | Haiku | Hits a running endpoint, asserts status/body | Edits code, restarts server |
| **Profiler** | Haiku | Reads the profiler (query counts, timing) | Proposes fixes |
| **Writer** | Sonnet | Drafts PR body / Jira comment / message | Investigates or reads code |

Sub-agents have **no channel to the user** — when blocked they proceed on a stated assumption and hand it back to the Orchestrator, who owns every user-facing decision.

---

## The scenario

> **Request:** "A customer-facing shopping-cart endpoint is firing hundreds of DB queries per request. Investigate and fix the N+1 storm. The response must stay identical — pure performance change."

A demo-safe version of the prompts that drove the session, and how the Orchestrator handled each.

---

### 1. "Explore the ticket for context"

> **Prompt:** *"We're on a new enhancement ticket, `TICKET-964`. First explore the ticket for context, then I'll give the next step."*

**Orchestrator** does not read the tracker itself. It spawns an **Explorer** with the ticket key.

```
Explorer → fetches ticket summary, acceptance criteria, comments.
         → returns: goal = reduce total queries + localized-label queries
           on the public cart endpoint. No code proposed.
```

Orchestrator writes a **plan file** (`.claude/plans/<date>-<slug>.md`) — baseline numbers, root cause, workstreams, a quality bar, and the delegation loop. The plan is the stable artifact every sub-agent reads from, keeping chat context clean.

Crucially, the plan **spells out the loop explicitly**, so "execute the plan" later has something concrete to bind to. The delegation section stays short:

> **Prompt:** *"Work autonomously in the loop: Builder implements, Reviewer reviews, Prober + Profiler measure.
If the output doesn't satisfy the quality, ask Builder to enhance and restart review/probe/profile until it does.
Design blocker -> stop and ask the user. Cap at 3 cycles. I review and test the final work."*

Writing the loop and the gate into the file *before* execution is what lets the next prompt stay a single line.

---

### 2. "Execute the plan"

> **Prompt:** *"Looks good. Now execute the plan."*

The prompt is deliberately short — it doesn't spell out a loop. It doesn't need to: the loop is already written into the **plan file** from step 1 (its delegation section defines the Builder→Reviewer→Prober→Profiler cycle, the quality bar to judge against, and a hard cap of 3 enhance-cycles per workstream so it never spins forever). "Execute the plan" means *execute what that artifact specifies* — the Orchestrator reads its own plan back and runs the loop it defined. The contract lives in the file, not the chat prompt.

For each workstream:

```
Builder implements + commits
   ↓
Reviewer reads the diff, classifies each finding
   ↓
Prober hits the endpoint, captures a profiler token
   ↓
Profiler reads that token, reports the query count
   ↓
Orchestrator judges the combined output against the quality bar
   ├─ Fail (fixable)      → brief Builder with the specific gap, loop
   ├─ Design blocker      → stop, surface to user (never loop on design)
   └─ Pass                → advance to next workstream
```

**What the loop caught in practice** (each fixed by looping back to the Builder, not by the Orchestrator editing):

- A default-locale fallback returning the wrong localized value — Reviewer caught it against the framework's own semantics.
- Array ordering drift after switching to fetch-joins — Profiler/Prober evidence proved it was a deterministic regression, not noise; fixed with explicit `ORDER BY`.
- A localized field the batch loader missed — Builder self-caught during its own verify step.

**Headline result:** total queries **354 → 34**; the localized-label queries dropped from ~92 to single digits; response bodies verified **byte-identical** in both locales (value and array order).

One workstream turned out to rest on a false premise. The Orchestrator did **not** force it through — it surfaced the design question to the user and stopped. That restraint is the skill working as intended.

---

### 3. "Draft a PR, then push and create it"

> **Prompt:** *"Draft a pull request title and body for me to review."* — then, after review — *"Push the branch to remote and create the PR."*

**Orchestrator** briefed a **Writer** with the full context (the problem, the fix, measured numbers, verification) — the Writer does not read code, so the brief carries everything. Writer loaded the house writing-style guide and the PR template, and produced a draft to a file for review.

**Orchestrator reviewed the draft** and caught one inaccuracy: a checklist box claimed tests were added, but the diff contained no test files — unchecked it before showing the user.

Only after explicit user approval — push and PR creation are outward-facing — did the Orchestrator push the branch and open the PR.

---

## What the demo shows about the skill

- **The Orchestrator stays out of the weeds.** Every search, edit, HTTP call, and profiler read went to a sub-agent. The main thread held the plan and the judgment.
- **Evidence over assertion.** No workstream was called "done" on the Builder's word — a Prober/Profiler pair produced runtime numbers, and a Reviewer read the diff independently.
- **The loop self-corrects.** Locale bugs, ordering drift, a missed field — all caught and looped back, with a hard cap so it never spins forever.
- **Design issues stop the loop.** A false-premise workstream was surfaced to the user, not forced through.
- **Outward-facing actions wait for the user.** Push and PR happened only on explicit approval; sensitive/irreversible steps are never autonomous.

---

*Generated from a real session. Feature names, endpoint slugs, tokens, identifiers, and internal URLs have been genericized or trimmed; prompts are shown at feature level.*
