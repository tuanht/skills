# Git workflow — worktrees, ownership, locking

Multi-agent git discipline for Builder/Code Reviewer/Spec Reviewer/Orchestrator split. Apply same in Cowork and normal Claude Code sessions — mechanics don't change by environment, only network-command restrictions differ (Cowork-specific, live in `./cowork.md`).

## Ownership (who runs what)

- **Builder** works own worktree (`<project_root>/.worktrees/<slug>`), stages/commits own change there, Sonnet tier — commit stay with agent holding change context.
- **Code Reviewer** runs read-only git only (`status`, `diff`, `log`). **Spec Reviewer** touches git at most for the same read-only trio; usually no git at all.
- **Orchestrator** no routine `add`/`commit`; integrates worktrees at end, hands `push`/`gh` to user.
- Worktree isolation load-bearing: each linked worktree own index, parallel implementers no collide. Two agents share one tree → index writes contend.

## Commit granularity

Commit by **intent** — one cohesive change per commit, with every piece it need to stand alone (code, callers it touch, tests, config, docs — whatever change requires). Don't split single intent across separate commits by file type or layer; partial commits don't build or pass alone. Unit is meaning: count the *why*s, not files.

Split only across genuinely distinct intents (unrelated refactor, second feature). Avoid opposite too — no 500-line "added feature X" dump. Aim middle: small enough Code Reviewer holds in head and bad one reverts clean, whole enough it builds and tests pass. Commit each intent soon as complete.

## Commit message content

Body carries what the diff can't: reasoning — why this shape, what problem fixed, what breaks if wrong. Plain terms, behavior level. Leave out code internals (type signatures, method names, control-flow) — diff already shows those, prose restating them adds nothing and goes stale as code evolves. See `.claude/skills/orchestrate/references/writing-style.md` for phrasing/tone once content right (full path — this file gets read by sub-agents whose cwd is repo root).

**Builder invokes `Skill(skill: "caveman:caveman-commit")` before composing every commit message** — load it, don't recall it. That skill owns subject format, length cap, body rules, and trailers; where it conflicts with `CLAUDE.md` commit conventions, **the skill wins**. Commit via `git commit -F <file>`, not `-m`, so multi-line body survives quoting. Orchestrator reads the finished messages (`git log master..HEAD --format='%s%n%n%b'`) before integrating and loops Builder back if they drift — Builder doesn't self-certify.

## Worktrees and the Docker container

Docker container mounts **main checkout only**, NOT linked worktrees under `<project_root>/.worktrees/`. Any command running inside container executes against **parent** codebase, will **not** see worktree changes.

Consequences for Builder/Orchestrator split:

- **Builder** (in worktree) may run only host-side, worktree-local checks: `composer phpcs`, `./vendor/bin/phpstan analyse <files>`, `composer rector-dry-run`. Must NOT run container-based phpunit/bin/console and claim validated worktree code — output reflects parent checkout, not its changes. If test needs container, Builder writes test, reports exact `--filter` command, hands verification to Orchestrator.
- **Orchestrator** runs container-based phpunit/bin/console **only after** merging worktree branch into working checkout (or feature branch checked out in main tree). Tests post-integration, not during delegation.
- Constraint applies whenever container reachable, Cowork or normal session alike — Cowork just never reaches it in practice, no `docker` at all (see `./cowork.md`).

## Integration & cleanup (orchestrator, automatic)

Sub-agent commits stay on own worktree branch — never auto-merge. Orchestrator integrates without prompting user:

`builder commits -> code reviewer reads diff (worktree must still exist) -> orchestrator loops fixes back to the same implementer via SendMessage until clean -> merges branch into the feature branch, resolving any conflict itself -> cleans up.`

Clean up only after merge — never before review pass, or fix has nowhere to land. Never delegate conflict resolution to sub-agent (can't ask user). Pushing to remote stays with user.

```bash
git worktree remove ./.worktrees/<slug>   # --force if discarding a dirty tree
git branch -d <slug>                       # -D if abandoned
git worktree prune
```

## Lock prevention

- **Scope: one worktree.** Rules below apply *within a single worktree* — that's where the index lock lives (`.git/worktrees/<name>/index.lock`). Two Builders committing in two different worktrees, on two different branches, don't contend — that isolation is the whole point of the worktree split. Main checkout counts as one worktree like any other.
- Never issue two git commands in parallel bash calls **against the same worktree** — wait previous call return before next.
- Read-only ops: pass `--no-optional-locks` to skip index lock entirely:

```bash
git --no-optional-locks status
git --no-optional-locks diff
git --no-optional-locks log --oneline -10
git --no-optional-locks diff --cached
```

- Never parallelize `git add`, `git commit`, `git merge`, `git rebase`, `git stash`, `git checkout`, or `git switch` with any other git call **in that same worktree**. Exception to the isolation above: `git worktree add` / `remove` / `prune` and any branch-ref write (`git branch -d`, `git push`) touch shared state — orchestrator runs those alone, no other git command anywhere in flight.
- Lock error mid-session: stop, report — don't delete lock files yourself. Hand to user, resolve from own terminal (not agent's bash tool):

```bash
# Run locally, not via an agent's bash tool
ps aux | grep '[g]it' | grep -v grep    # confirm no live git process
rm -f .git/index.lock .git/HEAD.lock    # main worktree
rm -f .git/worktrees/*/index.lock       # linked worktrees
```

## Safe write sequence

Staging and committing, follow exact order — one bash call per step:

1. `git --no-optional-locks status`        — confirm working-tree state
2. `git add <paths>`                        — stage only intended files
3. `git --no-optional-locks diff --cached`  — verify what staged
4. `git commit -F <message-file>`           — commit (never `-m`, see § Commit message content)