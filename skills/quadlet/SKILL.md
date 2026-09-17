---
name: quadlet
description: "Use ONLY when user says /quadlet, quadlet, or asks to create Podman Quadlet units (.container, .pod, .network, .volume), convert docker-compose, or install homelab stacks under systemd. Implements create, docker, install, update, recreate, validate. Rootful and rootless. Late-mount (ZFS/NFS) copy pattern. Read references/ before acting."
---

# /quadlet — Podman Quadlet skill for homelab stacks

Create, convert, install, and manage systemd Quadlet units for Podman containers and pods.

## When to use
- User invokes `/quadlet ...`
- User wants `.container`, `.pod`, `.network`, `.volume` files for a service or stack
- User has docker-compose.yml and wants quadlet equivalent
- User wants automated install of quadlets on a Linux host (rootful or rootless)

Never use for plain docker run, kubernetes, or non-systemd setups.

## Subcommands
All subcommands are invoked as `/quadlet <subcommand> [args]`

| Subcommand | Purpose | Runs on |
|------------|---------|---------|
| `create <name> [spec]` | Generate units from requirements, docs, or description. Single service → one `.container`. Coupled group → `.pod` + members. Loose → `.network` + containers. | Anywhere (generates files) |
| `docker <compose.yml|dir>` | Read docker-compose and emit quadlet files using heuristic (see references/compose-mapping.md) | Anywhere |
| `install [stack]` | On Linux host: copy units (late-mount pattern if needed), daemon-reload, start services. Supports rootful/rootless. Emits or runs INSTALL steps. | Linux host only (refuse on Darwin) |
| `update [stack]` | Re-copy units if late-mount source, pull images (or auto-update), restart affected units | Linux host |
| `recreate [stack]` | Stop, reset-failed, podman rm -f, reload, start. Keep persistent data volumes. | Linux host |
| `validate [files...]` | Lint quadlet files for common errors + (if on host) run podman checks, systemctl cat, status | Anywhere + host extras |

## Standing rules (never break)
1. **Source of truth separate from generator dir.** When using late-mounted storage (ZFS, NFS, etc.): keep real files on the pool, **copy** (never symlink) into `/etc/containers/systemd` (rootful) or `~/.config/containers/systemd` (rootless). The Quadlet generator runs early, before the pool is imported.
2. **Late-mount units** must include:
   - `Wants=... zfs-mount.service zfs.target` (or equivalent)
   - `After=... zfs-import.target zfs-mount.service zfs.target local-fs.target network-online.target`
    - `RequiresMountsFor=/path/to/quadlets` (exact paths)
3. **Ports** are published on the `[Pod]`, never on individual containers.
4. **Do not set `HostName=`** on containers that are Pod members (they share the pod UTS namespace; hostname = PodName).
5. **`EnvironmentFile=`** must be absolute path and **must not** start with `-`.
6. **Do not `systemctl enable`** the generated `*.service` files. They are under `/run/systemd/generator/`. Use `[Install] WantedBy=multi-user.target default.target` in the quadlet source.
7. **Secrets** live in `chmod 600` env files or secrets, never inline in unit files.
8. **AutoUpdate**: prefer `AutoUpdate=registry` for images that should auto-pull.
9. **Restart policy**: `Restart=always` + reasonable `TimeoutStartSec`.
10. **Data dirs**: use `ExecStartPre=/usr/bin/mkdir -p ...` when needed.
11. **Rootful vs rootless**:
     - Rootful: units in `/etc/containers/systemd`, use `sudo`, host services (e.g. DB/LLM) usually root.
    - Rootless: units in `~/.config/containers/systemd`, no sudo for podman, may need `UserNS=keep-id`, ports >=1024.
12. **SELinux**: append `:Z` (or `:z`) to Volume= lines on enforcing systems (Fedora/RHEL). Skip or use `:z` on others unless requested.
13. **Naming**:
    - Pod: `<stack>.pod`
    - Container in pod: `<stack>-<svc>.container`
    - Standalone: `<stack>-<svc>.container`
    - Generated services: `<stack>-<svc>.service`, `<stack>-pod.service`

## Workflow for each subcommand

### /quadlet create
1. Clarify stack name, services, images, ports, volumes, env, whether they are coupled.
2. Decide pod vs network using heuristic in `references/compose-mapping.md`.
3. Read relevant templates from `templates/`.
4. Generate files with proper headers (source-of-truth comment, generator copy comment, do-not-symlink).
5. Emit a `quadlet.stack.yaml` manifest (files list, start order, late-mount info).
6. Emit an `INSTALL.md` tailored for the stack (copy commands, env setup, checks).
7. For complex: also `.volume`, `.network`, systemd drop-ins.
8. Output the files and the install script snippet. Do not apply unless user explicitly asks for `/quadlet install`.

### /quadlet docker
1. Read the compose file or dir (use glob/read).
2. Parse services, volumes, networks, ports, env, depends_on, restart.
3. Apply the compose→quadlet mapping (see references/compose-mapping.md).
4. Use pod heuristic: tightly-coupled services (sidecars, localhost, single app) → one `.pod`; independent or DNS-heavy → `.network`.
5. Generate same header comments, correct sections.
6. Produce `INSTALL.md` and `quadlet.stack.yaml`.
7. Warn on compose features with no good quadlet map (build:, profiles, secrets with driver, etc.).

### /quadlet install
- **Must be run on the target Linux host.**
- Detect rootful vs rootless (ask user if ambiguous; default rootful for host ports <1024 or system services).
- If late-mount source is configured: copy from source-of-truth to generator dir using `install -m 644`.
- Otherwise write directly (or copy from generated location).
- Create required dirs with correct perms.
- Handle `EnvironmentFile` secrets (generate if missing using openssl, chmod 600).
- `podman pull` images.
- `systemctl daemon-reload`
- Start the units (do **not** enable generated ones).
- For rootless: use `systemctl --user`.
- Print verification commands.

Refuse with clear message if running on non-Linux or no systemd.

### /quadlet update
- Re-copy units if using late-mount source.
- Either `podman pull` + restart, or enable `podman-auto-update.timer`.
- Restart only changed services.

### /quadlet recreate
- Stop services, `systemctl reset-failed`, `podman pod rm -f` (or container rm).
- `daemon-reload`
- Start again.
- Data volumes left in place.

### /quadlet validate
- Static checks: header comments present, no HostName= on pod members, absolute EnvironmentFile without `-`, ports only on pods, correct After/RequiresMountsFor when late mount.
- If on host: `podman quadlet generate ...`, `systemctl cat`, `systemctl status`, `podman ps --pod`, port checks, curl smoke tests.
- Report findings with file:line.

## Templates & references
Always load fresh:
- `references/conventions.md`
- `references/compose-mapping.md`
- `references/install.md`
- Files under `templates/`

Use templates verbatim for structure; fill only the variable parts. Preserve the exact comment block at top of every unit file.

## Output discipline
- Write files under the stack dir the user specifies (or `./<stack>/`).
- Also write `INSTALL.md` and `quadlet.stack.yaml` next to units.
- Show the user the exact `sudo install` or `install` commands.
- Never commit the generated quadlets in this repo unless user asks.
- When user is on Darwin, only generate; install/update/recreate must be performed on the Linux host (user can copy files or run commands remotely).

Read the templates/ and references/ when you need concrete patterns.
