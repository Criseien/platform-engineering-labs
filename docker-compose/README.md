# docker-compose

## Scenario

A three-service stack — nginx + python app + postgres — is deployed for the
first time. The app exits on startup with a connection error. After fixing that,
the stack runs, but the next deploy leaves the database empty. Meanwhile, the
postgres port appears in `docker compose ps` but is unreachable from the host.

## Diagnosis Flow

```
docker compose up → app exits with code 1
→ docker compose logs app           # OperationalError: could not connect to server
→ docker compose ps                 # db shows "Up" — but that means container started, not postgres ready
→ depends_on with no condition      # starts when container starts, not when pg_isready passes
```

After adding healthcheck — stack runs fine. Next deploy:

```
docker compose down && docker compose up
→ postgres is empty — all data gone
→ docker volume ls                  # no named volume listed for this project
→ docker inspect <db_container>     # GraphDriver.Data.UpperDir → destroyed on container remove
```

## Root Cause

Two separate problems surfaced in sequence.

**Startup race:** `depends_on` without `condition: service_healthy` waits for
the container process to start — not for postgres to finish initializing its
data directory and begin accepting connections. The app gets `Connection refused`
because it won the race against postgres.

**Data loss:** Container writes go to the overlay2 writable layer (UpperDir).
`docker compose down` removes containers and destroys UpperDir with them.
Without a named volume, every redeploy starts with a blank database.

## Fix

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - db-data:/var/lib/postgresql/data   # named volume — survives down

  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy         # waits until healthcheck passes

volumes:
  db-data:
```

## The Critical Trap: `down` vs `down -v`

`docker compose down` removes containers but keeps named volumes — data safe.
`docker compose down -v` removes named volumes too — data gone, no undo.

In a production incident, running `down -v` thinking it's a clean restart
is a data-loss event. The volumes flag is destructive and silent about it.

## Additional Traps from This Lab

### Port listed in `ps` but unreachable — `expose` vs `ports`

```
docker compose ps → 5432/tcp   ← no host binding shown
curl localhost:5432             → Connection refused
```

`5432/tcp` in the PORTS column means the port is exposed — documented for
inter-container communication, but not published to the host. The iptables
DNAT rule that maps `localhost:5432` to the container only exists with an
explicit `ports:` entry.

```yaml
ports:
  - "5432:5432"    # creates DNAT rule — host can reach it
expose:
  - "5432"         # documentation only — host cannot reach it
```

In production the DB port should almost never be published. Only add `ports:`
if the host itself needs to reach the container directly.

### Container exits immediately with empty logs

Image has no default `CMD`, or the entrypoint runs once and exits cleanly.
Check with `docker inspect <image> --format '{{json .Config.Cmd}}'`.
Add `command:` in the Compose file. For Python, also set `PYTHONUNBUFFERED=1`
or use `python -u` — buffered stdout silences logs even when the process runs.

### Service name DNS fails — services on different networks

A container only resolves names of containers that share the same network.
If two services can't reach each other by hostname despite being in the same
`docker-compose.yml`, they are likely on explicitly declared networks with
no common bridge between them. See [multi-network](./multi-network/) for
the isolation lab and the fix.

## Decision: named volume vs bind mount

| | Named volume | Bind mount |
|---|---|---|
| Path managed by | Docker (`/var/lib/docker/volumes/`) | You (any host path) |
| Survives `down` | ✅ | ✅ |
| Survives `down -v` | ❌ | ✅ (host path untouched) |
| Portable across hosts | ✅ | ❌ (path must pre-exist) |
| Use case | DB data, any persistent state | Dev: live code reload (`./src:/app/src`) |

Use named volumes for anything the container owns. Use bind mounts in
development for files you want to edit on the host and see immediately
in the container.

## Key Commands

```bash
# Lifecycle
docker compose up -d                        # start detached
docker compose up --build                   # rebuild images first
docker compose down                         # stop + remove containers, keep volumes
docker compose down -v                      # also remove named volumes (destructive)

# Logs and status
docker compose logs -f app                  # follow one service
docker compose ps                           # status and port bindings
docker compose top                          # processes inside each container

# Debug
docker compose exec app bash                # exec into running container
docker compose run --rm app sh              # one-off container (removed after exit)
docker compose exec app nslookup db         # test DNS between services

# Volumes
docker volume ls                            # list all volumes
docker volume inspect <project>_db-data    # see actual path on host
```

## K8s Connection

`depends_on: condition: service_healthy` maps directly to a readiness probe on
the dependency. When the probe passes, the Service starts routing traffic and
the dependent pod can connect — same contract, different mechanism.

Named volumes map to PersistentVolumeClaims. The abstraction is identical:
neither the app nor the pod spec knows where the data actually lives on disk —
both declare what they need, and the runtime (Docker / Kubernetes) provisions it.

`docker compose down -v` is `kubectl delete pvc`: both are permanent and
both will leave you explaining to stakeholders why the data is gone.
