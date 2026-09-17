# <stack> — Quadlet installation

Run these commands on the Linux host. Do not run install/update/recreate on Darwin.

## Summary
- Model: pod | network
- Mode: rootful | rootless
- Endpoints: ...
- Images: fully-qualified, e.g. ghcr.io/example/app:main
- Data location on host: ...

## Layout
| Path | Role |
|------|------|
| `source_of_truth/` | User-chosen ZFS/NFS/git location |
| `/etc/containers/systemd/` or `~/.config/containers/systemd/` | Generator copies (never symlinks) |

## Prerequisites
- Podman >= 4.4 (Quadlet). `.pod` units need Podman >= 4.5.
- systemd, cgroup v2
- (rootful) sudo
- Late-mount: pool/share mounted before starting units, and `RequiresMountsFor=` set on data + env paths

## Install (late-mount rootful)

```bash
SRC=<source_of_truth>
DST=/etc/containers/systemd

sudo mkdir -p "$DST"

sudo install -m 644 "$SRC/<stack>.pod" "$DST/<stack>.pod"
sudo install -m 644 "$SRC/<stack>-web.container" "$DST/<stack>-web.container"

if [ ! -s "$SRC/<stack>.env" ]; then
  tmp=$(mktemp)
  umask 077
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$tmp"
  sudo install -m 600 "$tmp" "$SRC/<stack>.env"
  rm -f "$tmp"
fi

sudo podman pull <image>

sudo systemctl daemon-reload
sudo systemctl start <stack>-pod.service <stack>-web.service
```

Do not `systemctl enable` generated units.

## Install (rootless)

```bash
loginctl enable-linger "$USER"

SRC=<source_of_truth>
DST="$HOME/.config/containers/systemd"

mkdir -p "$DST"

install -m 644 "$SRC/<stack>.network" "$DST/<stack>.network"
install -m 644 "$SRC/<stack>-web.container" "$DST/<stack>-web.container"

if [ ! -s "$SRC/<stack>.env" ]; then
  umask 077
  install -m 600 /dev/null "$SRC/<stack>.env"
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" > "$SRC/<stack>.env"
fi

podman pull <image>

systemctl --user daemon-reload
systemctl --user start <stack>-network.service <stack>-web.service
```

## Checks

```bash
/usr/lib/systemd/system-generators/podman-system-generator --dryrun
# rootless: add --user

podman pod ps
podman ps --pod
systemctl status <stack>-pod.service <stack>-web.service --no-pager
# rootless: systemctl --user status ...
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<port>
```

## Update

Re-copy units, then:

```bash
sudo systemctl daemon-reload
sudo podman pull <image>
sudo systemctl restart <stack>-web.service
```

Rootless: `systemctl --user daemon-reload` / `systemctl --user restart ...` and `systemctl --user enable --now podman-auto-update.timer`.

If `.pod`, `PublishPort=`, or `.network` changed, use recreate instead of restart.

## Recreate (data preserved)

Pod model:

```bash
sudo systemctl stop <stack>-web.service <stack>-pod.service
sudo systemctl reset-failed <stack>-web.service <stack>-pod.service
sudo podman pod rm -f <stack>
sudo systemctl daemon-reload
sudo systemctl start <stack>-pod.service <stack>-web.service
```

Network model:

```bash
sudo systemctl stop <stack>-web.service
sudo systemctl reset-failed <stack>-web.service
sudo podman rm -f <stack>-web
sudo systemctl daemon-reload
sudo systemctl start <stack>-network.service <stack>-web.service
```

Rootless: drop sudo, use `systemctl --user` and rootless `podman`. Leave data dirs in place.
