# Git workflow — worktrees, ownership, locking

Multi-agent git discipline for the Builder/Reviewer/Orchestrator split. This applies in Cowork and in normal Claude Code sessions alike — the mechanics don't change based on environment, only the network-command restrictions do (those are Cowork-specific and live in `./cowork.md`).

## Ownership (who runs what)

- **Builder** works in its own worktree (`<project_root>/.worktrees/<slug>`) and stages/commits its own change there, on the Sonnet tier — commit stays with the agent that holds the change context.
- **Reviewer** runs read-only git only (`status`, `diff`, `log`).
- **Orchestrator** does no routine `add`/`commit`; it integrates worktrees at the end and hands `push`/`gh` to the user.
- Worktree isolation is load-bearing: each linked worktree has its own index, so parallel implementers don't collide. If two agents ever share one tree, their index writes contend.

## Commit granularity

Commit by logical unit, not by session. A Builder that finishes a multi-part task should leave behind several small commits — one per logical change (e.g. add the repository interface, add the handler, wire the route, add the test) — not a single 500-line commit at the end labeled "added feature X". Commit as soon as a logical unit is complete and working; don't batch everything until the whole task is done.

This is what makes the Reviewer's diff actually reviewable, and what lets a bad piece be reverted or bisected without losing the rest. If a commit touches several unrelated concerns, or a reviewer would need more than a few minutes to hold the whole thing in their head, split it before moving on.

## Worktrees and the Docker container

The Docker container mounts the **main checkout only**, NOT linked worktrees under `<project_root>/.worktrees/`. Therefore any command that runs inside the container executes against the **parent** codebase and will **not** see changes made in a worktree.

Consequences for the Builder/Orchestrator split:

- **Builder** (in a worktree) may run only host-side, worktree-local checks: `composer phpcs`, `./vendor/bin/phpstan analyse <files>`, `composer rector-dry-run`. It must NOT run container-based phpunit/bin/console and claim it validated worktree code — that output reflects the parent checkout, not its changes. If a test needs the container, the Builder writes the test, reports the exact `--filter` command, and hands verification to the Orchestrator.
- **Orchestrator** runs container-based phpunit/bin/console **only after** merging the worktree branch into the working checkout (or the feature branch checked out in the main tree). Run tests post-integration, not during delegation.
- This constraint applies whenever the container is reachable, in Cowork or a normal session alike — Cowork just never reaches it in practice since it has no `docker` at all (see `./cowork.md`).

## Integration & cleanup (orchestrator, automatic)

Sub-agent commits stay on their worktree branch — they never auto-merge. The orchestrator integrates without prompting the user:

`builder commits → reviewer reads diff (worktree must still exist) → orchestrator loops fixes back to the same implementer via SendMessage until clean → merges branch into the feature branch, resolving any conflict itself → cleans up.`

Clean up only after the merge — never before review passes, or a fix has nowhere to land. Never delegate conflict resolution to a sub-agent (it can't ask the user). Pushing to the remote stays with the user.

```bash
git worktree remove ./.worktrees/<slug>   # --force if discarding a dirty tree
git branch -d <slug>                       # -D if abandoned
git worktree prune
```

## Lock prevention

- Never issue two git commands in parallel bash calls — wait for the previous call to return before starting the next.
- For read-only operations, pass `--no-optional-locks` to skip the index lock entirely:

```bash
git --no-optional-locks status
git --no-optional-locks diff
git --no-optional-locks log --oneline -10
git --no-optional-locks diff --cached
```

- Never parallelize `git add`, `git commit`, `git merge`, `git rebase`, `git stash`, `git checkout`, or `git switch` with any other git call.
- If a lock error surfaces mid-session, stop and report it — do not delete lock files yourself. Hand it to the user to resolve from their own terminal (not via the agent's bash tool):

```bash
# Run locally, not via an agent's bash tool
ps aux | grep '[g]it' | grep -v grep    # confirm no live git process
rm -f .git/index.lock .git/HEAD.lock    # main worktree
rm -f .git/worktrees/*/index.lock       # linked worktrees
```

## Safe write sequence

When staging and committing, follow this exact order — one bash call per step:

1. `git --no-optional-locks status`        — confirm working-tree state
2. `git add <paths>`                        — stage only the intended files
3. `git --no-optional-locks diff --cached`  — verify what is staged
4. `git commit -m "<message>"`              — commit
