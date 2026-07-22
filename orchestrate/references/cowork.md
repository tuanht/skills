# Claude Cowork — full notes

Hard rules live inline in `CLAUDE.md` (must see before any command run). File hold rationale + longer guidance.

## Why command bans exist

Cowork sandbox separate Linux env from project. No `php`, `docker`, `composer`, `dea`, `bin/console`, `gh`, network/runtime access installed. Attempt them → fail, or worse, act on wrong environment.

Git one exception, nuance: local-only plumbing (`checkout`, `add`, `commit`, `status`, `diff`, `branch`) works fine against checked-out repo, run freely. Only commands reaching network/remote — `git push`, `git pull`, `git fetch`, `gh` — banned, same reason as rest of list: hand those to user.

## How to work in Cowork

- Anything banned above: give exact command(s), tell user run manually in terminal.
- Focus read/write files; leave execution feedback (test output, phpstan results, push/PR status) to user.
- Would normally run check (`composer test`, `composer phpstan`, etc.): instead state precise command user should run + what to look for in output.

## Git discipline

Worktree ownership, Docker-mount constraint, integration/cleanup, lock prevention, safe write sequence — environment-agnostic, live in `./git-workflow.md`. Only thing specific to Cowork: command ban above.