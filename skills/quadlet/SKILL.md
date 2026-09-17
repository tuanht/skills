---
name: quadlet
description: "Use ONLY when user says /quadlet, /tuanht:quadlet, quadlet, or asks to create Podman Quadlet units (.container, .pod, .network, .volume), convert docker-compose, or install homelab stacks under systemd. Implements create, docker, install, update, recreate, validate. Rootful and rootless. Late-mount (ZFS/NFS) copy pattern. Read references/ before acting."
---

# /quadlet — Podman Quadlet skill for homelab stacks

Create, convert, install, and manage systemd Quadlet units for Podman containers and pods.

## When to use
- User invokes `/quadlet ...` or `/tuanht:quadlet ...`
- User wants `.container`, `.pod`, `.network`, `.volume` files for a service or stack
- User has docker-compose.yml and wants quadlet equivalent
- User wants automated install of quadlets on a Linux host (rootful or rootless)

Never use for plain docker run, kubernetes, or non-systemd setups.

Read `references/conventions.md`, `references/compose-mapping.md`, and `references/install.md` fresh before acting. Do not invent keys or install steps from memory.

## Subcommands
All subcommands are invoked as `/quadlet <subcommand> [args]` (plugin: `/tuanht:quadlet ...`)

| Subcommand | Purpose | Runs on |
|------------|---------|---------|
| `create <name> [spec]` | Generate units from requirements, docs, or description. Single service → one `.container`. Coupled group (localhost/sidecars) → `.pod` + members. Independent / DNS-heavy → `.network` + containers. | Anywhere (generates files) |
| `docker <compose.yml\|dir>` | Read docker-compose and emit quadlet files using `references/compose-mapping.md` | Anywhere |
| `install [stack]` | Copy units (late-mount pattern if needed), daemon-reload, start. Rootful/rootless. | Linux only — refuse if `uname -s` is not Linux |
| `update [stack]` | Re-copy, daemon-reload, pull or auto-update, restart. Pod/port/network changes → recreate. | Linux only — refuse if not Linux |
| `recreate [stack]` | Stop, reset-failed, rm pod or containers, reload, start. Keep data volumes. | Linux only — refuse if not Linux |
| `validate [files...]` | Lint units; on Linux also generator `--dryrun` + systemctl/podman checks | Anywhere (static); host extras on Linux |

## Standing rules (never break)

Details live in `references/conventions.md`. One-line never-break:

1. **Copy, never symlink** source-of-truth into `/etc/containers/systemd` (rootful) or `~/.config/containers/systemd` (rootless) when storage is late-mounted.
2. **Late-mount `[Unit]` deps** are filled from `quadlet.stack.yaml` (`late_mount`, `late_mount_kind`, rootful vs rootless). Do not hardcode `zfs-*` on NFS/local or in rootless user units.
3. **Ports:** pod **members** never `PublishPort=`. Publish on `[Pod]`. Standalone / network-model containers **must** use `[Container] PublishPort=`.
4. **Do not set `HostName=`** on pod members (hostname = PodName).
5. **`EnvironmentFile=`** must be an absolute path and **must not** start with `-` (copy pattern: generator dir is not the source dir).
6. **Do not `systemctl enable`** generated `*.service` files. `[Install] WantedBy=default.target` only.
7. **Secrets** live in mode-600 env files, never inline. Do not `tee` secrets to stdout.
8. **`AutoUpdate=registry`** requires a fully-qualified `Image=` (registry/name:tag).
9. **Restart policy:** `Restart=always` + `TimeoutStartSec` (900 if images may still pull).
10. **`ExecStartPre=mkdir`** only together with `RequiresMountsFor=` for that path (else mkdir can mask a late mount). Rootless: set ownership for keep-id.
11. **Rootful vs rootless:** rootful = `/etc/containers/systemd` + `sudo systemctl`; rootless = `~/.config/containers/systemd` + `systemctl --user` + linger. `UserNS=keep-id` on `[Pod]` for pod model, on `[Container]` only when standalone.
12. **SELinux:** `:Z` or `:z` on `Volume=` on enforcing systems.
13. **Naming:** `<stack>.pod` → `<stack>-pod.service`; `<stack>-<svc>.container` → `<stack>-<svc>.service`; `<stack>.network` → `<stack>-network.service`; `<stack>-data.volume` → `<stack>-data-volume.service`.

## Workflow for each subcommand

### /quadlet create
1. Clarify stack name, services, images (fully-qualified), ports, volumes, env, coupling, rootful vs rootless, late_mount kind.
2. Decide pod vs network using `references/compose-mapping.md` (default **network** unless localhost/sidecars).
3. Read templates from `templates/`.
4. Generate files with the header block from conventions (source-of-truth, both generator paths, do-not-symlink).
5. If `late_mount: true`, uncomment `RequiresMountsFor=` with **data + env file** paths; add ZFS unit deps only for rootful ZFS.
6. Emit `quadlet.stack.yaml` (units, start_order matching the model, late_mount, late_mount_kind).
7. Emit stack `INSTALL.md` from `templates/INSTALL.md` (inline commands; do not link this skill repo).
8. For extra persistence/isolation: also `.volume` / `.network`. Do **not** emit systemd drop-ins.
9. Output files and install snippets. Do not apply unless user asks `/quadlet install`.

### /quadlet docker
1. Read the compose file or dir.
2. Parse services, volumes, networks, ports, env, depends_on, restart, healthcheck, devices.
3. Apply `references/compose-mapping.md`. Default to **network**. Use **pod** only for localhost/sidecars; then rewrite sibling hostnames to `127.0.0.1` (or `AddHost=svc:127.0.0.1`).
4. Resolve every volume to an **absolute** host path before `Volume=`.
5. Produce `INSTALL.md` and `quadlet.stack.yaml`.
6. Warn on `build:`, `profiles:`, compose `secrets:` drivers.

### /quadlet install
- Refuse unless `uname -s` is Linux (and systemd is present).
- Detect rootful vs rootless (ask if ambiguous; default rootful for ports <1024).
- Late-mount: `install -m 644` from source-of-truth to generator dir.
- Create data dirs only after the mount is present (`RequiresMountsFor`).
- Env files: create mode 600 without printing secrets (see conventions).
- `podman pull` images.
- `systemctl daemon-reload` (or `--user`).
- Start units listed in `start_order`. Do **not** enable generated units.
- Rootless: `loginctl enable-linger "$USER"` if needed; `systemctl --user`.
- Print verification commands from `references/install.md`.

### /quadlet update
- Refuse unless Linux.
- Re-copy units if late-mount.
- `systemctl daemon-reload` (or `--user`).
- `podman pull` + restart, or enable `podman-auto-update.timer` (`--user` if rootless).
- Restart only changed services.
- If `.pod` / `PublishPort=` / `.network` changed: treat as **recreate**, not restart.

### /quadlet recreate
- Refuse unless Linux.
- Stop units, `reset-failed`.
- Pod model: `podman pod rm -f <pod>`. Network model: `podman rm -f` each container (not `pod rm`).
- `daemon-reload`, then start `start_order` (include `*-network.service` / `*-volume.service` when present).
- Rootless: `systemctl --user` / rootless `podman`.
- Leave data dirs in place.

### /quadlet validate
- Static: headers present; no `HostName=` on pod members; `EnvironmentFile=` absolute without `-`; ports on `[Pod]` XOR standalone `[Container]` (never on pod members); late-mount `RequiresMountsFor` on data+env; no `zfs-*` in rootless units; fully-qualified `Image=` when `AutoUpdate=registry`; `WantedBy=default.target` only.
- On Linux host:
  - `/usr/lib/systemd/system-generators/podman-system-generator --dryrun` (add `--user` if rootless; `QUADLET_UNIT_DIRS=` if files are not in the generator dir yet)
  - `systemctl cat` / `status` (or `--user`)
  - `podman ps --pod`, port and curl checks
- Report findings with file:line.

## Templates & references
Always load fresh:
- `references/conventions.md`
- `references/compose-mapping.md`
- `references/install.md`
- Files under `templates/`

Use templates for structure; fill only the variable parts. Preserve the header comment block.

## Output discipline
- Write files under the stack dir the user specifies (or `./<stack>/`).
- Also write `INSTALL.md` and `quadlet.stack.yaml` next to units. Do not commit `*.env`.
- Show exact `sudo install` or `install` commands.
- Never commit generated quadlets in this repo unless the user asks.
- Darwin: generate + static validate only. install/update/recreate refuse.

Read templates/ and references/ when you need concrete patterns.
