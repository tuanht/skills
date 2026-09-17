# Compose → Quadlet mapping

This document defines how to convert `docker-compose.yml` (or equivalent requirements) into Quadlet units.

## Core heuristic: Pod vs Network

**Use a single `.pod`** when:
- Services are one logical application
- They talk to each other over `localhost` or `127.0.0.1`
- They are sidecars (reverse proxy + app, ui + db sidecar, etc.)
- They share the same data dir or need to share UTS namespace (same hostname)
- Restarting one without the others doesn't make sense

**Use a `.network` + separate containers** when:
- Services are independent (different lifecycles)
- They rely on Compose-style DNS names (`db`, `redis`) from other containers
- Multiple networks or complex network aliases are used
- You want to scale or restart one service without touching others
- Port conflicts would occur inside one netns

**Mixed case**: Ask the user. You can still put tightly-coupled ones in a pod and connect the pod to an external network for the rest.

When in doubt for homelab "stack", default to **pod** for simplicity.

## Service → file mapping

| Compose concept       | Quadlet (pod style)                  | Quadlet (network style)                  |
|-----------------------|--------------------------------------|------------------------------------------|
| `services.foo`        | `stack-foo.container` + `Pod=stack.pod` | `stack-foo.container` + `Network=stack.network` |
| `ports`               | `PublishPort=` in `[Pod]` section    | `PublishPort=` or `PodmanArgs=--publish` on container |
| `volumes`             | `Volume=` in container (or separate `.volume`) | same |
| `environment`         | `Environment=` or `EnvironmentFile=` | same |
| `depends_on`          | implicit via pod or `Wants=`/`After=` | `Wants=` + `After=` on the dependent |
| `restart`             | `[Service] Restart=...`              | same |
| `networks`            | (none needed inside pod)             | `Network=foo.network`                    |

## Volume mapping

Inline is simplest:

```ini
Volume=/path/to/quadlets/data:/app/data:Z
```

For reusable named volumes, create a `.volume` file:

`stack-data.volume`:
```ini
[Volume]
VolumeName=stack-data
Label=app=stack
```

Then reference:
```ini
Volume=stack-data.volume:/data:Z
```

For bind mounts from late FS, add the path to `RequiresMountsFor` and `After`.

## Network mapping

`stack.network`:
```ini
[Network]
NetworkName=stack
Label=app=stack
# Subnet, Gateway, etc. if needed
```

Containers get:
```ini
Network=stack.network
```

Inside a pod you usually don't need an extra network unless you want to expose the pod to other pods/containers.

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
- Secrets → same file, `chmod 600`, never inline.

Do **not** translate `secrets:` top-level with file drivers into quadlet secrets yet (v1 scope); put them in env file.

## Depends / startup order

Compose `depends_on` with `condition: service_healthy` is approximated by:

In the dependent unit:
```ini
Wants=stack-db.service
After=stack-db.service
```

For pod members the pod itself provides ordering.

## Build

Compose `build:` is not directly supported by quadlet units in v1.
Options:
- Pre-build and push image, reference the image in quadlet.
- Or document a manual `podman build` step in INSTALL.md.
- Warn and do not invent a `.build` unit unless user has a very new podman that supports it.

## Other compose fields

| Field            | Action |
|------------------|--------|
| `profiles`       | Ignore for now; document that profiles are not translated |
| `command` / `entrypoint` | Use `Exec=` or `PodmanArgs=--entrypoint` |
| `healthcheck`    | Not native in quadlet; use `ExecStartPost` or external; or rely on app |
| `sysctls`, `cap_add` | `PodmanArgs=--sysctl ... --cap-add ...` |
| `devices`        | `Device=/dev/foo` |
| `tmpfs`          | `Tmpfs=...` |
| `ulimits`        | `PodmanArgs=--ulimit ...` |

Always put complex passthrough under `PodmanArgs=` and document it.

## Ports gotcha

If two services in the same pod try to bind the same container port, it will conflict. In that case split to network model or remap ports.

## Hostname / DNS

- Pod members: hostname = PodName (e.g. `mystack`). Do not set per-container.
- Network model: each container can have `HostName=`, and other containers resolve by container name or network aliases.

## Example translation (simplified)

docker-compose.yml (tightly coupled):
```yaml
services:
  web:
    image: ghcr.io/example/web:main
    ports: ["8080:8080"]
    volumes: ["./data:/app/data"]
    environment:
      BACKEND_URL: http://host.containers.internal:8081
  # (companion service assumed on host or another container)
```

Becomes:
- `mystack.pod` with `PublishPort=8080:8080`, `AddHost=host.containers.internal:host-gateway`
- `mystack-web.container` with `Pod=mystack.pod`, `Volume=...`, `Environment=...`

## Validation after conversion

After generating:
- Run `/quadlet validate`
- Check that no `HostName=` appears in pod-member containers
- Check that all published ports are on the pod file
- Verify volume paths exist or will be created by `ExecStartPre`
- Confirm late-mount `RequiresMountsFor` when paths are on ZFS/NFS

## When to deviate

If the user has a strong reason (e.g. "I need the compose DNS names exactly"), follow network model even if services feel coupled. Note the decision in the generated `INSTALL.md`.
