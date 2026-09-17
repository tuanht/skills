# Quadlet conventions

These are hard rules derived from real homelab usage and systemd-quadlet behavior.

## File placement & late mount

- **Source of truth** lives on your storage (ZFS pool, NFS, etc.): e.g. `/path/to/quadlets/` (user-chosen)
- **Generator directory** (read by systemd at boot):
  - Rootful: `/etc/containers/systemd/`
  - Rootless: `$HOME/.config/containers/systemd/`
- **Never symlink** from source-of-truth into the generator dir. The generator runs before late filesystems are mounted.
- When source-of-truth is late-mounted:
  - Units **must** declare:
    ```
    Wants=network-online.target zfs-mount.service zfs.target
    After=network-online.target local-fs.target zfs-import.target zfs-mount.service zfs.target
    RequiresMountsFor=/path/to/quadlets
    ```
  - Use exact paths in `RequiresMountsFor`.
- Copy with `install -m 644 src dst` (regular file, not symlink).
- After editing source, re-copy + `systemctl daemon-reload`.

## Unit file headers (every file)

```ini
# <name>.pod
# Source of truth: /path/to/quadlets/<name>.pod
# Generator copy (root):  /etc/containers/systemd/<name>.pod
# Do not symlink — the Quadlet generator runs before late-mounted storage is available.
#
# Generated unit: <name>-pod.service
```

Same pattern for `.container`, `.network`, `.volume`.

## Pod vs Container

- Publish ports on the **pod** only (`PublishPort=` in `[Pod]`).
- Pod members share network + UTS namespace.
- **Never** put `HostName=` on a container that belongs to a pod. The hostname is the `PodName`.
- Use `Pod=foo.pod` inside the container unit.

## [Container] section

- `Image=`
- `ContainerName=`
- `Pod=` (when applicable)
- `AutoUpdate=registry`
- `Volume=src:dst:Z` (use `:Z` on SELinux enforcing hosts)
- `Environment=KEY=val` (repeatable)
- `EnvironmentFile=/absolute/path/to/file`  ← **no leading `-`**
- `Network=` (when not using Pod)
- `UserNS=keep-id` (common for rootless)
- `AddHost=...` (usually on the pod)

## [Service] section

```ini
[Service]
Restart=always
TimeoutStartSec=300
ExecStartPre=/usr/bin/mkdir -p /path/to/data
```

## [Install]

```ini
[Install]
WantedBy=multi-user.target default.target
```

**Do not** run `systemctl enable` on the generated units (`foo.service`, `foo-pod.service`). They are synthesized under `/run/systemd/generator`. The WantedBy is applied by the generator on every boot/daemon-reload.

## Secrets & EnvironmentFile

- Never put secrets in the `.container` file.
- Generate:
  ```bash
  printf 'WEBUI_SECRET_KEY=%s\n' "$(openssl rand -hex 32)" | sudo tee /path/to/secret.env
  sudo chmod 600 /path/to/secret.env
  ```
- Reference with `EnvironmentFile=/absolute/path` (no dash).

## Rootful vs Rootless

| Aspect              | Rootful                          | Rootless                              |
|---------------------|----------------------------------|---------------------------------------|
| Unit dir            | `/etc/containers/systemd`        | `~/.config/containers/systemd`        |
| Copy command        | `sudo install -m 644`            | `install -m 644`                      |
| systemctl           | `sudo systemctl ...`             | `systemctl --user ...`                |
| Ports               | any (incl <1024)                 | >=1024 recommended                    |
| Volumes ownership   | root or container uid            | keep-id + subuid/subgid               |
| Host services (e.g. LLM, DB) | often root                   | usually rootless too                  |
| UserNS              | not needed                       | `UserNS=keep-id` common               |

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
Label=component=webui
```

## Updates

- Images: `podman-auto-update.timer` or manual `podman pull` + restart service.
- Unit changes: re-copy file + `daemon-reload` + restart service.

## Recreate (keep data)

```bash
sudo systemctl stop <svc> <pod>
sudo systemctl reset-failed <svc> <pod>
sudo podman pod rm -f <podname>
sudo systemctl daemon-reload
sudo systemctl start <svc>
```

Data directories under your source-of-truth are untouched.

## Validation checklist (run on host)

```bash
findmnt /path/to/quadlets
systemctl is-active zfs-import.target zfs-mount.service zfs.target
systemctl status <name>-pod.service <name>-<svc>.service --no-pager
systemctl show <name>-<svc>.service -p After -p RequiresMountsFor
podman pod ps
podman ps --pod
ss -tlnp | grep -E '8080|...'
curl -sS http://127.0.0.1:...
```

## Do not do

- Symlink quadlet files
- Put secrets in unit files
- `systemctl enable` generated services
- Publish ports on member containers
- Set HostName on pod members
- Prefix EnvironmentFile with `-`
- Assume the generator dir is writable at boot time
