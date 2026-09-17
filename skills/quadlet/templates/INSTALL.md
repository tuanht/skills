# <stack> — Quadlet installation

## Summary
- Pod / services: ...
- UI / endpoints: ...
- Images: ...
- Data location on host: ...

## Layout
| Path | Role |
|------|------|
| `source_of_truth/` | Your ZFS/NFS or git location |
| `/etc/containers/systemd/` (or `~/.config/...`) | Generator copies (never symlinks) |

## Prerequisites
- Podman >= 4.0 with quadlet support
- systemd
- (rootful) sudo access
- ZFS/NFS pool mounted before starting units (or use direct layout)

## Install (late-mount rootful example)

```bash
SRC=<source_of_truth>
DST=/etc/containers/systemd

sudo mkdir -p "$DST" "$SRC/data" \
  /etc/systemd/system/some.service.d

sudo install -m 644 "$SRC/<stack>.pod" "$DST/<stack>.pod"
sudo install -m 644 "$SRC/<stack>-svc.container" "$DST/<stack>-svc.container"

# secrets (example)
if [ ! -s "$SRC/<stack>.env" ]; then
  printf 'SECRET=%s\n' "$(openssl rand -hex 32)" | sudo tee "$SRC/<stack>.env"
  sudo chmod 600 "$SRC/<stack>.env"
fi

sudo podman pull <image>

sudo systemctl daemon-reload
sudo systemctl start <stack>-pod.service <stack>-svc.service
```

## Rootless variant

Replace sudo + paths with user equivalents, use `systemctl --user`.

## Checks

```bash
podman pod ps
podman ps --pod
systemctl status <stack>-pod.service <stack>-svc.service --no-pager
curl -sS http://127.0.0.1:<port>
```

## Updates

See references/install.md

## Recreate (data preserved)

See references/install.md
