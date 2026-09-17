# Quadlet conventions

Hard rules derived from systemd-quadlet behavior. SKILL.md is a router; this file is the source of truth.

## File placement & late mount

- **Source of truth** lives on user-chosen storage: e.g. `/path/to/quadlets/`
- **Generator directory** (read by systemd at boot):
  - Rootful: `/etc/containers/systemd/`
  - Rootless: `$HOME/.config/containers/systemd/`
- **Never symlink** from source-of-truth into the generator dir. The generator runs before late filesystems are mounted.
- Copy with `install -m 644 src dst` (regular file, not symlink).
- After editing source, re-copy + `systemctl daemon-reload` (or `systemctl --user daemon-reload`).

### Late-mount `[Unit]` deps (`quadlet.stack.yaml`)

Fill from `late_mount`, `late_mount_kind` (`zfs` | `nfs` | `local`), and rootful vs rootless. `RequiresMountsFor=` lists **exact data dirs and env file paths**, not only the stack root.

**Rootful + ZFS** (`late_mount: true`, `late_mount_kind: zfs`):

```
Wants=network-online.target zfs-mount.service zfs.target
After=network-online.target local-fs.target zfs-import.target zfs-mount.service zfs.target
RequiresMountsFor=/path/to/quadlets/webdata /path/to/quadlets/mystack.env
```

**Rootful + NFS** (`late_mount_kind: nfs`):

```
Wants=network-online.target remote-fs.target
After=network-online.target remote-fs.target
RequiresMountsFor=/path/to/quadlets/webdata /path/to/quadlets/mystack.env
```

**Rootful + local** (`late_mount: false` or `late_mount_kind: local`): network-online / local-fs only. No `zfs-*`, no extra `RequiresMountsFor` unless a specific path is on another mount.

**Rootless (any late FS):** user units **must not** `Wants=` / `After=` system units such as `zfs-mount.service`. Use:

```
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/path/to/quadlets/webdata /path/to/quadlets/mystack.env
```

Do not emit `zfs-*` dependencies on non-ZFS hosts.

## Unit file headers (every file)

```ini
# <name>.pod
# Source of truth: /path/to/quadlets/<name>.pod
# Generator copy (root):  /etc/containers/systemd/<name>.pod   (rootful)
#                         ~/.config/containers/systemd/<name>.pod (rootless)
# Do not symlink — the Quadlet generator runs before late-mounted storage is available.
#
# Generated unit: <name>-pod.service
```

Same pattern for `.container`, `.network`, `.volume` (generated names: `<name>.service`, `<name>-network.service`, `<name>-volume.service`).

## Pod vs Container

- Pod **members**: never `PublishPort=` on the container. Publish on `[Pod]`.
- Standalone / network-model: `PublishPort=` on `[Container]`.
- Pod members share network + UTS namespace.
- **Never** put `HostName=` on a container that belongs to a pod. The hostname is the `PodName`.
- Use `Pod=foo.pod` inside the container unit.
- `UserNS=keep-id` belongs on `[Pod]` when the container has `Pod=`. On `[Container]` only when standalone / network-model.

## [Container] section

- `Image=` — fully-qualified (`registry/name:tag`) when using `AutoUpdate=registry`
- `ContainerName=`
- `Pod=` (pod model) **or** `Network=` (network model)
- `AutoUpdate=registry`
- `Volume=/absolute/host/path:dst:Z` (use `:Z` on SELinux enforcing hosts)
- `Environment=KEY=val` (repeatable)
- `EnvironmentFile=/absolute/path/to/file`  ← **no leading `-`**; absolute because the generator copy is not next to the source
- `UserNS=keep-id` (rootless standalone only; pod model: set on `[Pod]`)
- `AddHost=` (usually on the pod)
- `PublishPort=` (standalone / network-model only)
- `AddDevice=`, `HealthCmd=`, `Exec=`, `Entrypoint=`, `AddCapability=`, `Sysctl=`, `Ulimit=` — first-class keys; do not invent `Device=` or dump these into `PodmanArgs=` unless there is no key

## [Service] section

```ini
[Service]
Restart=always
TimeoutStartSec=900
```

`ExecStartPre=/usr/bin/mkdir -p /path/to/data` **only** when the same path is in `RequiresMountsFor=` (otherwise mkdir can create a dir on the root FS and mask a late mount). Rootless: create with the mapped uid/gid (`keep-id`), not as root.

## [Install]

```ini
[Install]
WantedBy=default.target
```

**Do not** run `systemctl enable` on the generated units. They are synthesized under `/run/systemd/generator` (rootful) or `/run/user/$UID/systemd/generator` (rootless). Do not use `multi-user.target` (dangling for rootless user systemd).

## Secrets & EnvironmentFile

- Never put secrets in the `.container` file. Do not commit `*.env`.
- Create mode 600 **without** printing the secret (`tee` leaks to the agent transcript):

Rootful:

```bash
tmp=$(mktemp)
umask 077
printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$tmp"
sudo install -m 600 "$tmp" /path/to/quadlets/mystack.env
rm -f "$tmp"
```

Rootless (no sudo):

```bash
umask 077
install -m 600 /dev/null "$HOME/quadlets/mystack.env"
printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$HOME/quadlets/mystack.env"
```

- Reference with `EnvironmentFile=/absolute/path` (no dash).

## Rootful vs Rootless

| Aspect | Rootful | Rootless |
|--------|---------|----------|
| Unit dir | `/etc/containers/systemd` | `~/.config/containers/systemd` |
| Copy command | `sudo install -m 644` | `install -m 644` |
| systemctl | `sudo systemctl ...` | `systemctl --user ...` |
| Generator (runtime) | `/run/systemd/generator` | `/run/user/$UID/systemd/generator` |
| Ports | any (incl <1024) | >=1024 recommended |
| Volumes ownership | root or container uid | keep-id + subuid/subgid |
| Host services | often root | often also rootless |
| UserNS | not needed | `UserNS=keep-id` on `[Pod]` or standalone `[Container]` |
| Linger | n/a | `loginctl enable-linger "$USER"` |
| Auto-update | `sudo systemctl enable --now podman-auto-update.timer` | `systemctl --user enable --now podman-auto-update.timer` |
| Late-mount deps | may `After=` `zfs-*.service` | `RequiresMountsFor=` only |

Choose based on:
- Need privileged ports or host network services → rootful.
- Multi-user desktop or security preference → rootless.

## SELinux volume labels

- Enforcing (Fedora, RHEL, Rocky, Alma): `:Z` (private) or `:z` (shared)
- Permissive or Debian/Ubuntu without enforcing: omit or use `:z`
- When in doubt, start with `:Z` on app data volumes.

## Common labels

```ini
Label=app=mystack
Label=component=web
```

## Updates

- Images: `podman-auto-update.timer` or manual `podman pull` + restart service.
- Unit changes: re-copy file + `daemon-reload` + restart service.
- `.pod` / `PublishPort=` / `.network` changes: **recreate**, not restart.

## Recreate (keep data)

Pod model (rootful):

```bash
sudo systemctl stop mystack-web.service mystack-pod.service
sudo systemctl reset-failed mystack-web.service mystack-pod.service
sudo podman pod rm -f mystack
sudo systemctl daemon-reload
sudo systemctl start mystack-pod.service mystack-web.service
```

Network model (rootful):

```bash
sudo systemctl stop mystack-web.service mystack-db.service
sudo systemctl reset-failed mystack-web.service mystack-db.service
sudo podman rm -f mystack-web mystack-db
sudo systemctl daemon-reload
sudo systemctl start mystack-network.service mystack-web.service mystack-db.service
```

Rootless: drop `sudo`, use `systemctl --user` and rootless `podman`. Data directories under source-of-truth are untouched.

## Validation on host

```bash
/usr/lib/systemd/system-generators/podman-system-generator --dryrun
# rootless: add --user
# files not in generator dir yet: QUADLET_UNIT_DIRS=/path/to/units

findmnt /path/to/quadlets
systemctl status mystack-pod.service mystack-web.service --no-pager
systemctl show mystack-web.service -p After -p RequiresMountsFor
podman pod ps
podman ps --pod
ss -tlnp | grep -E '8080|...'
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080
```

Rootless: `systemctl --user` and skip `systemctl is-active zfs-*`.

## Do not do

- Symlink quadlet files
- Put secrets in unit files or `tee` them
- `systemctl enable` generated services
- `WantedBy=multi-user.target`
- Publish ports on pod member containers
- Set HostName on pod members
- Prefix EnvironmentFile with `-`
- `UserNS=keep-id` on a container that has `Pod=`
- `Wants=`/`After=` `zfs-*.service` in rootless user units
- Assume the generator dir is writable at boot time
- Emit systemd drop-ins as part of this skill
