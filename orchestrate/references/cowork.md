# Claude Cowork — full notes

The hard rules live inline in `CLAUDE.md` (they must be seen before any command runs). This file holds the rationale and the longer guidance.

## Why the command bans exist

The Cowork sandbox is a separate Linux environment from the project. It does **not** have `php`, `docker`, `composer`, `dea`, `bin/console`, `gh`, or network/runtime access installed. Attempting them fails or, worse, acts on the wrong environment.

Git is the one exception with nuance: local-only plumbing (`checkout`, `add`, `commit`, `status`, `diff`, `branch`) works fine against the checked-out repo and may be run freely. Only the commands that reach the network or the remote — `git push`, `git pull`, `git fetch`, `gh` — are banned, for the same reason as everything else on this list: hand those to the user.

## How to work in Cowork

- For anything banned above, provide the exact command(s) and instruct the user to run them manually in their terminal.
- Focus on reading and writing files; leave execution feedback (test output, phpstan results, push/PR status) to the user.
- When you would normally run a check (`composer test`, `composer phpstan`, etc.), instead state the precise command the user should run and what to look for in the output.

## Git discipline

Worktree ownership, the Docker-mount constraint, integration/cleanup, lock prevention, and the safe write sequence are environment-agnostic and live in `./git-workflow.md`. The only thing actually specific to Cowork is the command ban above.
