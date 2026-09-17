# Compose → Quadlet mapping

How to convert `docker-compose.yml` (or equivalent requirements) into Quadlet units.

## Core heuristic: Pod vs Network

**Default to a `.network` + separate containers.** Compose stacks almost always talk via service DNS names (`db`, `redis`). A pod shares one netns; those names do not resolve unless rewritten.

**Use a single `.pod`** only when:
- Services actually talk over `localhost` / `127.0.0.1`
- They are sidecars that must share one netns (reverse proxy + app on loopback)
- They must share UTS namespace (same hostname)
- Restarting one without the others does not make sense

**If you choose pod**, you **must rewrite** compose service hostnames:
- `postgres://db:5432` → `postgres://127.0.0.1:5432`
- or `AddHost=db:127.0.0.1` on the `[Pod]`
- If two services bind the same container port, split to network model or remap ports

**Use a `.network` + separate containers** when:
- Services are independent (different lifecycles)
- They rely on Compose-style DNS names (`db`, `redis`)
- Multiple networks or network aliases
- You want to restart one service without touching others
- Port conflicts would occur inside one netns

**Mixed case**: Ask the user. Tightly-coupled services can sit in a pod; connect that pod to a network for the rest.

When in doubt, **network**. Do not default to pod.

## Service → file mapping

| Compose concept | Quadlet (pod style) | Quadlet (network style) |
|-----------------|---------------------|-------------------------|
| `services.foo` | `stack-foo.container` + `Pod=stack.pod` | `stack-foo.container` + `Network=stack.network` |
| `ports` | `PublishPort=` in `[Pod]` | `PublishPort=` in `[Container]` |
| `volumes` | `Volume=` (absolute host path) or `.volume` | same |
| `environment` | `Environment=` or `EnvironmentFile=` | same |
| `depends_on` | `[Unit] After=`/`Wants=` other `.container` | same |
| `restart` | `[Service] Restart=...` | same |
| `networks` | usually none inside the pod | `Network=foo.network` |
| `healthcheck` | `HealthCmd=` / `HealthInterval=` / `HealthOnFailure=` | same |
| `command` | `Exec=` | same |
| `entrypoint` | `Entrypoint=` | same |

## Volume mapping

Resolve **every** compose volume to an **absolute host path** before emitting `Volume=`. Quadlet resolves relative `Volume=` against Podman's cwd (systemd: `/`), **not** the compose file directory.

Compose `./data:/app/data` next to `/path/to/stack/docker-compose.yml` becomes:

```ini
Volume=/path/to/stack/data:/app/data:Z
```

Named volumes: create a `.volume` file:

`stack-data.volume`:
```ini
[Volume]
VolumeName=stack-data
Label=app=stack
```

Then:
```ini
Volume=stack-data.volume:/data:Z
```

For bind mounts from late FS, add **that path** (and env files) to `RequiresMountsFor`.

## Network mapping

`stack.network`:
```ini
[Network]
NetworkName=stack
Label=app=stack
```

Containers:
```ini
Network=stack.network
```

Inside a pod you usually do not need an extra network unless exposing the pod to other units.

## Environment & secrets

Compose:
```yaml
environment:
  - FOO=bar
  - SECRET_KEY
env_file:
  - .env
```

Quadlet:
- Simple vars → repeated `Environment=FOO=bar`
- File → `EnvironmentFile=/absolute/path/to/stack.env`
- Secrets → same file, mode 600, never inline, never `tee` (see conventions.md)

Do **not** translate top-level compose `secrets:` file drivers into quadlet secrets in v1; put them in the env file.

## Depends / startup order

Put these under **`[Unit]`**, never under `[Container]`. Quadlet accepts the sibling quadlet name:

```ini
[Unit]
Wants=stack-db.container
After=stack-db.container
```

The pod does **not** order members. `StartWithPod=true` starts them together. Translate `depends_on` into `[Unit] After=` / `Wants=` between `.container` files even inside a pod.

`depends_on` + `condition: service_healthy`: also map the healthcheck to `HealthCmd=` / `HealthOnFailure=kill` on the dependency.

## Build

Compose `build:` is not supported by quadlet units in v1.
- Pre-build and push an image, reference it in quadlet.
- Or document a manual `podman build` step in INSTALL.md.
- Warn; do not invent a `.build` unit unless the user has a Podman that supports it.

## Other compose fields

| Field | Quadlet key |
|-------|-------------|
| `profiles` | Ignore; document that profiles are not translated |
| `command` | `Exec=` |
| `entrypoint` | `Entrypoint=` |
| `healthcheck` | `HealthCmd=`, `HealthInterval=`, `HealthTimeout=`, `HealthRetries=`, `HealthOnFailure=kill` |
| `cap_add` | `AddCapability=` |
| `cap_drop` | `DropCapability=` |
| `sysctls` | `Sysctl=` |
| `devices` | `AddDevice=/dev/foo` (not `Device=`; that is a `[Volume]` option) |
| `tmpfs` | `Tmpfs=` |
| `ulimits` | `Ulimit=` |
| `user` | `User=` |
| `privileged` | `PodmanArgs=--privileged` (no first-class key; document it) |

Prefer first-class keys. Use `PodmanArgs=` only when there is no Quadlet key.

## Ports gotcha

If two services in the same pod bind the same container port, it conflicts. Split to network model or remap.

Validate after conversion:
- Pod members: all `PublishPort=` on the `.pod` file, none on members
- Network / standalone: `PublishPort=` on each `.container` that had compose `ports`

## Hostname / DNS

- Pod members: hostname = PodName. Do not set per-container `HostName=`. Rewrite compose service DNS to `127.0.0.1` or `AddHost=`.
- Network model: containers resolve by `ContainerName=` / network aliases. `HostName=` is allowed.

## Example: network (default)

```yaml
services:
  web:
    image: ghcr.io/example/web:main
    ports: ["8080:8080"]
    volumes: ["./data:/app/data"]
    environment:
      DATABASE_URL: postgres://db:5432/app
    depends_on: [db]
  db:
    image: docker.io/library/postgres:16
```

Becomes:
- `mystack.network`
- `mystack-web.container` with `Network=mystack.network`, `PublishPort=8080:8080`, absolute `Volume=`, `[Unit] After=mystack-db.container`
- `mystack-db.container` with `Network=mystack.network`
- `DATABASE_URL` kept as `postgres://db:5432/app` (network DNS)

## Example: pod (localhost/sidecar only)

Tightly coupled, talks over loopback. Rewrite hostnames:

- `mystack.pod` with `PublishPort=8080:8080`
- `mystack-web.container` with `Pod=mystack.pod`, **no** `PublishPort=`
- Compose `http://backend:8081` → `http://127.0.0.1:8081` (or `AddHost=backend:127.0.0.1` on the pod)

## Validation after conversion

- Run `/quadlet validate`
- No `HostName=` on pod-member containers
- Ports on the pod XOR on standalone/network containers
- Every `Volume=` host path is absolute
- `depends_on` became `[Unit]` deps
- Late-mount `RequiresMountsFor` when paths are on ZFS/NFS
- If pod model: sibling DNS rewritten

## When to deviate

If the user needs compose DNS names exactly, use network model even if services feel coupled. Note the decision in INSTALL.md.
