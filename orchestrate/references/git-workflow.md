# Git workflow — worktrees, ownership, locking

Multi-agent git discipline for Builder/Reviewer/Orchestrator split. Apply same in Cowork and normal Claude Code sessions — mechanics don't change by environment, only network-command restrictions differ (Cowork-specific, live in `./cowork.md`).

## Ownership (who runs what)

- **Builder** works own worktree (`<project_root>/.worktrees/<slug>`), stages/commits own change there, Sonnet tier — commit stay with agent holding change context.
- **Reviewer** runs read-only git only (`status`, `diff`, `log`).
- **Orchestrator** no routine `add`/`commit`; integrates worktrees at end, hands `push`/`gh` to user.
- Worktree isolation load-bearing: each linked worktree own index, parallel implementers no collide. Two agents share one tree → index writes contend.

## Commit granularity

Commit by **intent** — one cohesive change per commit, with every piece it need to stand alone (code, callers it touch, tests, config, docs — whatever change requires). Don't split single intent across separate commits by file type or layer; partial commits don't build or pass alone. Unit is meaning: count the *why*s, not files.

Split only across genuinely distinct intents (unrelated refactor, second feature). Avoid opposite too — no 500-line "added feature X" dump. Aim middle: small enough Reviewer holds in head and bad one reverts clean, whole enough it builds and tests pass. Commit each intent soon as complete.

## Commit message content

Commit body describes behavior change and why, plain terms — not code internals (type signatures, method names, control-flow). Reviewers read diff for mechanics.

Diff already shows what changed at code level; restating in prose adds nothing, goes stale as code evolves. What diff can't show: reasoning — why this shape, what problem fixed, what breaks if wrong. That's what body for. See `./writing-style.md` for phrasing/tone once content right.

## Worktrees and the Docker container

Docker container mounts **main checkout only**, NOT linked worktrees under `<project_root>/.worktrees/`. Any command running inside container executes against **parent** codebase, will **not** see worktree changes.

Consequences for Builder/Orchestrator split:

- **Builder** (in worktree) may run only host-side, worktree-local checks: `composer phpcs`, `./vendor/bin/phpstan analyse <files>`, `composer rector-dry-run`. Must NOT run container-based phpunit/bin/console and claim validated worktree code — output reflects parent checkout, not its changes. If test needs container, Builder writes test, reports exact `--filter` command, hands verification to Orchestrator.
- **Orchestrator** runs container-based phpunit/bin/console **only after** merging worktree branch into working checkout (or feature branch checked out in main tree). Tests post-integration, not during delegation.
- Constraint applies whenever container reachable, Cowork or normal session alike — Cowork just never reaches it in practice, no `docker` at all (see `./cowork.md`).

## Integration & cleanup (orchestrator, automatic)

Sub-agent commits stay on own worktree branch — never auto-merge. Orchestrator integrates without prompting user:

`builder commits -> reviewer reads diff (worktree must still exist) -> orchestrator loops fixes back to the same implementer via SendMessage until clean -> merges branch into the feature branch, resolving any conflict itself -> cleans up.`

Clean up only after merge — never before review pass, or fix has nowhere to land. Never delegate conflict resolution to sub-agent (can't ask user). Pushing to remote stays with user.

```bash
git worktree remove ./.worktrees/<slug>   # --force if discarding a dirty tree
git branch -d <slug>                       # -D if abandoned
git worktree prune
```

## Lock prevention

- Never issue two git commands in parallel bash calls — wait previous call return before next.
- Read-only ops: pass `--no-optional-locks` to skip index lock entirely:

```bash
git --no-optional-locks status
git --no-optional-locks diff
git --no-optional-locks log --oneline -10
git --no-optional-locks diff --cached
```

- Never parallelize `git add`, `git commit`, `git merge`, `git rebase`, `git stash`, `git checkout`, or `git switch` with any other git call.
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
4. `git commit -m "<message>"`              — commit