# skills

My personal skill set for Claude Code and OpenCode.

## Usage

### Claude Code

Git submodule at `.claude/skills`:

```
git submodule add git@github.com:tuanht/skills.git .claude/skills
git submodule update --remote .claude/skills
```

Or plugin marketplace:

```
/plugin marketplace add tuanht/skills
/plugin install tuanht@tuanht-skills
```

Trigger with `/tuanht:orchestrate`.

### OpenCode

Clone this repo, then add the `skills/` directory to `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/path/to/skills/skills"]
  }
}
```

Update: `git pull` in the clone, then quit and restart OpenCode.

## Skills

- **orchestrate** (`/tuanht:orchestrate`) — Orchestrator/Builder/Reviewer/Writer role split for delegating work across sub-agents. See `skills/orchestrate/references/` for the full role definitions.
- **quadlet** (`/tuanht:quadlet`) — Create and manage Podman Quadlet units (.container, .pod, .network, .volume) for homelab stacks. Convert docker-compose, install on Linux (rootful/rootless), late-mount ZFS support. See `skills/quadlet/`.
