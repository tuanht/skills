# Install & runtime reference

How `/quadlet install`, `update`, and `recreate` work.

Refuse `install`, `update`, and `recreate` unless `uname -s` is Linux and systemd is present.

## Two storage models

### 1. Late-mount (recommended for ZFS/NFS homelab)

```
Source of truth: /path/to/quadlets/<stack>/*.pod *.container ...
Generator dir:   /etc/containers/systemd/   (or ~/.config/containers/systemd)
```

- Real files live on the pool.
- At boot the generator has already run before the source-of-truth storage is imported/mounted.
- Therefore: **copy** (plain regular files) into the generator dir.
- Units declare `RequiresMountsFor` for **data + env paths**. Rootful ZFS may also `After=` `zfs-*`. Rootless must not.

### 2. Direct

Units live directly in the generator dir. No extra copy step. Still safe to keep a git repo next to them. Do not commit `*.env`.

## Rootful paths (default for system homelab)

- Quadlet source: `/path/to/quadlets` (user chosen)
- Generator: `/etc/containers/systemd`
- Runtime generator output: `/run/systemd/generator`
- Commands use `sudo`
- Requires Podman >= 4.4 (Quadlet). `.pod` units need Podman >= 4.5. cgroup v2 required.

## Rootless paths

- Quadlet source: e.g. `~/quadlets`
- Generator: `~/.config/containers/systemd`
- Runtime generator output: `/run/user/$UID/systemd/generator`
- Commands: `systemctl --user`, rootless `podman` (no sudo)
- Data dirs owned by the user
- Once: `loginctl enable-linger "$USER"` so user units start at boot
- Auto-update: `systemctl --user enable --now podman-auto-update.timer`

## The install flow (late-mount rootful)

```bash
SRC=/path/to/quadlets
DST=/etc/containers/systemd

sudo mkdir -p "$DST" "$SRC/mystack"

sudo install -m 644 "$SRC/mystack/mystack.pod"           "$DST/mystack.pod"
sudo install -m 644 "$SRC/mystack/mystack-web.container" "$DST/mystack-web.container"

if [ ! -s "$SRC/mystack/mystack.env" ]; then
  tmp=$(mktemp)
  umask 077
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$tmp"
  sudo install -m 600 "$tmp" "$SRC/mystack/mystack.env"
  rm -f "$tmp"
fi

sudo podman pull ghcr.io/example/web:main

sudo systemctl daemon-reload
sudo systemctl start mystack-pod.service mystack-web.service
```

Do not `systemctl enable` generated units. Do not write systemd drop-ins.

## Rootless equivalent

```bash
loginctl enable-linger "$USER"

SRC="$HOME/quadlets"
DST="$HOME/.config/containers/systemd"

mkdir -p "$DST" "$SRC/mystack"

install -m 644 "$SRC/mystack/mystack.network"        "$DST/mystack.network"
install -m 644 "$SRC/mystack/mystack-web.container"  "$DST/mystack-web.container"

if [ ! -s "$SRC/mystack/mystack.env" ]; then
  umask 077
  install -m 600 /dev/null "$SRC/mystack/mystack.env"
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$SRC/mystack/mystack.env"
fi

podman pull ghcr.io/example/web:main

systemctl --user daemon-reload
systemctl --user start mystack-network.service mystack-web.service
```

## quadlet.stack.yaml (optional but recommended)

Place next to the units so install is not a glob hunt. `/quadlet install` reads this for copy list, `start_order`, images, env files, `late_mount_kind`.

See `templates/quadlet.stack.yaml`.

## Environment files & secrets

- Always absolute path in `EnvironmentFile=`
- Never prefix with `-`
- Create with `install -m 600`; never `tee` secrets
- Do not commit `*.env`

## Image updates

Manual rootful:

```bash
sudo podman pull <image>
sudo systemctl daemon-reload
sudo systemctl restart <svc>.service
```

Auto rootful:

```bash
sudo systemctl enable --now podman-auto-update.timer
```

Auto rootless:

```bash
systemctl --user enable --now podman-auto-update.timer
```

Requires `AutoUpdate=registry` and a fully-qualified `Image=`.

Unit file edits: re-copy + `daemon-reload` + restart. `.pod` / ports / `.network` changes: recreate.

## Recreate (data-safe)

Pod model, rootful:

```bash
sudo systemctl stop mystack-web.service mystack-pod.service
sudo systemctl reset-failed mystack-web.service mystack-pod.service
sudo podman pod rm -f mystack
sudo systemctl daemon-reload
sudo systemctl start mystack-pod.service mystack-web.service
```

Network model, rootful:

```bash
sudo systemctl stop mystack-web.service mystack-db.service
sudo systemctl reset-failed mystack-web.service mystack-db.service
sudo podman rm -f mystack-web mystack-db
sudo systemctl daemon-reload
sudo systemctl start mystack-network.service mystack-web.service mystack-db.service
```

Rootless: `systemctl --user` and rootless `podman`. Leave data dirs alone.

## Common verification commands (put in INSTALL.md)

Rootful:

```bash
/usr/lib/systemd/system-generators/podman-system-generator --dryrun
findmnt /path/to/quadlets
systemctl status mystack-pod.service mystack-web.service --no-pager
systemctl show mystack-web.service -p After -p RequiresMountsFor
ls -l /run/systemd/generator/*.wants/
podman pod ps
podman ps --pod
podman inspect <name> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
ss -tlnp | grep -E '8080|...'
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080
```

Rootless: add `--user` to the generator and `systemctl`; generator output is `/run/user/$UID/systemd/generator`.

## First run notes

Many apps create an admin user on first browser visit. Document that in the stack INSTALL.md.

## Host services that containers talk to

Databases or other processes running on the host:

- Listen on `0.0.0.0` (or the address `host.containers.internal` will use).
- `AddHost=host.containers.internal:host-gateway` (and `host.docker.internal` if the app expects Docker's name) on the pod or container.

Do not emit systemd drop-ins for those host services as part of this skill.

## Darwin / non-Linux agents

Detect with `uname -s`. If not `Linux`:
- `/quadlet create`, `/quadlet docker`, and static `/quadlet validate` are allowed.
- `/quadlet install`, `update`, and `recreate` **must refuse** and tell the user to run them on the Linux host (copy files, then run the INSTALL.md commands there).

## Idempotency

Re-running install should be safe:
- `install -m 644` overwrites
- `daemon-reload` is cheap
- Starting an already-running unit is a no-op

## What not to do

- Do not `systemctl enable` the generated service files.
- Do not write units into `/run/systemd/generator`.
- Do not use relative paths for EnvironmentFile.
- Do not symlink.
- Do not `tee` secrets.
- Do not create `/etc/systemd/system/*.service.d/` drop-ins.
