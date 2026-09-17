# Install & runtime reference

This document describes how `/quadlet install`, `update`, and `recreate` work.

## Two storage models

### 1. Late-mount (recommended for ZFS/NFS homelab)

```
Source of truth: /path/to/quadlets/<stack>/*.pod *.container ...
Generator dir:   /etc/containers/systemd/   (or ~/.config/...)
```

- Real files live on the pool.
- At boot the generator has already run before the source-of-truth storage is imported/mounted.
- Therefore: **copy** (plain regular files) into the generator dir.
- Units declare `RequiresMountsFor` + correct `After`.

### 2. Direct

Units live directly in `/etc/containers/systemd` (or user dir). No extra copy step. Still safe to have a git repo or other VCS next to them.

## Rootful paths (default for system homelab)

- Quadlet source: `/path/to/quadlets` (user chosen, e.g. on ZFS)
- Generator: `/etc/containers/systemd`
- Host drop-ins: `/etc/systemd/system/<unit>.d/`
- Commands use `sudo`

## Rootless paths

- Quadlet source: usually `~/homelab/quadlets` or `~/.config/containers/quadlets`
- Generator: `~/.config/containers/systemd`
- Commands use `systemctl --user` and `podman` (no sudo for most)
- Data dirs owned by the user

## The install flow (late-mount rootful)

```bash
SRC=/path/to/quadlets
DST=/etc/containers/systemd

sudo mkdir -p "$DST" "$SRC/mystack"

# copy units
sudo install -m 644 "$SRC/mystack/mystack.pod"          "$DST/mystack.pod"
sudo install -m 644 "$SRC/mystack/mystack-svc.container" "$DST/mystack-svc.container"

# secrets
if [ ! -s "$SRC/mystack/mystack.env" ]; then
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" | sudo tee "$SRC/mystack/mystack.env"
  sudo chmod 600 "$SRC/mystack/mystack.env"
fi

# images
sudo podman pull ghcr.io/...

sudo systemctl daemon-reload

# start (do not enable generated units)
sudo systemctl start mystack-pod.service mystack-svc.service
```

## Rootless equivalent

```bash
SRC="$HOME/homelab/quadlets"
DST="$HOME/.config/containers/systemd"

mkdir -p "$DST" "$SRC/mystack"

install -m 644 "$SRC/..." "$DST/..."

# no sudo for podman pull if using rootless images
podman pull ...

systemctl --user daemon-reload
systemctl --user start mystack-pod.service mystack-svc.service
```

## quadlet.stack.yaml (optional but recommended)

Place next to the units so install is not a glob hunt:

```yaml
stack: mystack
source_of_truth: /path/to/quadlets
late_mount: true
units:
  - mystack.pod
  - mystack-web.container
start_order:
  - mystack-pod.service
  - mystack-web.service
images:
  - ghcr.io/example/web:main
env_files:
  - mystack.env
checks:
  - curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080
```

`/quadlet install` can read this to know exactly what to copy and start.

## Environment files & secrets

- Always absolute path in `EnvironmentFile=`
- Never prefix with `-`
- chmod 600
- Generate with openssl or age/sops if you have a workflow

## Image updates

Two common ways:

1. Manual
   ```bash
   sudo podman pull <image>
   sudo systemctl restart <svc>.service
   ```

2. Auto
   ```bash
   sudo systemctl enable --now podman-auto-update.timer
   ```
   Requires `AutoUpdate=registry` in the container unit.

## Recreate (data-safe)

```bash
sudo systemctl stop mystack-web.service mystack-pod.service
sudo systemctl reset-failed mystack-web.service mystack-pod.service
sudo podman pod rm -f mystack
sudo systemctl daemon-reload
sudo systemctl start mystack-web.service
```

Leave data dirs alone.

## Common verification commands (put in INSTALL.md)

```bash
findmnt /path/to/quadlets
systemctl is-active zfs-import.target zfs-mount.service zfs.target
systemctl status mystack-pod.service mystack-web.service --no-pager
systemctl show mystack-web.service -p After -p RequiresMountsFor
ls -l /run/systemd/generator/*.wants/
podman pod ps
podman ps --pod
podman inspect <name> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
ss -tlnp | grep -E '8080|...'
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080
```

## First run notes

Many apps (Open WebUI, etc.) create admin user on first browser visit.

## Host services that containers talk to

Common pattern: databases, LLM runtimes, etc. running on the host.

- Make them listen on `0.0.0.0` (or appropriate) so `host.containers.internal` or `host.docker.internal` works.
- Add `AddHost=host.containers.internal:host-gateway` (and docker variant) on the pod or container.
- For rootful host service, you may need a drop-in to set bind address.

## Darwin / non-Linux agents

When the agent is on macOS:
- `/quadlet create` and `/quadlet docker` and `/quadlet validate` (static) are allowed.
- `/quadlet install|update|recreate` must refuse and tell the user to run them on the Linux host (or scp the files + run commands manually).

## Idempotency

Re-running install should be safe:
- `install -m 644` overwrites
- `daemon-reload` is cheap
- Starting already-running unit is noop

## What not to do

- Do not `systemctl enable` the generated service files.
- Do not write units directly into `/run/systemd/generator`.
- Do not use relative paths for EnvironmentFile.
- Do not symlink.
