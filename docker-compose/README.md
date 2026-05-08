# docker-compose — Multi-service orchestration: what breaks in real stacks

Docker Compose is a layer over the Docker API that lets you declare a multi-service
stack in a single file. Under the hood it's the same `docker run` calls, network
creates, and volume mounts — just automated. Understanding that mapping is what
makes failures diagnosable. When a Compose stack breaks, the root cause is almost
always one of five things: wrong default, missed healthcheck, wrong network scope,
volume confusion, or port exposure vs publishing.

---

## The 5 Problems You'll Actually Hit

### 1. Container exits immediately — no logs, no error

**When it happens:** You run `docker compose up`. One service goes up and immediately
stops. `docker compose logs` for that service is empty.

```bash
docker compose up
# myservice exited with code 0

docker compose logs myservice
# (empty)
```

**Root cause:** The image has no default `CMD` (or the entrypoint exits cleanly).
Docker Compose starts the container, the process finishes in milliseconds, and the
container stops. No process running = no logs.

**Diagnose:**

```bash
# Check what command the image actually runs
docker inspect <image> --format '{{json .Config.Cmd}}'
# null ← no default command

docker inspect <image> --format '{{json .Config.Entrypoint}}'
# ["sh", "-c", "echo hello"]  ← runs once and exits
```

**Fix — specify a command in the Compose file:**

```yaml
services:
  worker:
    image: python:3.11-slim
    command: python -u worker.py    # -u disables output buffering → logs appear
```

**Trap:** Output buffering silences logs even when the process is running.
Python buffers stdout by default — add `-u` or set `PYTHONUNBUFFERED=1`:

```yaml
services:
  app:
    image: python:3.11-slim
    environment:
      - PYTHONUNBUFFERED=1
    command: python app.py
```

---

### 2. Port not reachable from host — exposed but not published

**When it happens:** The service is running. `docker compose ps` shows it up.
But `curl http://localhost:6379` or `curl http://localhost:5432` hangs or refuses.

```bash
docker compose ps
# redis   running   6379/tcp

curl http://localhost:6379
# curl: (7) Failed to connect to localhost port 6379
```

**Root cause:** `6379/tcp` in the PORTS column means the port is exposed
(documented for inter-container communication) but not published to the host.
The DNAT iptables rule that makes the host reach the container is only created
with an explicit port mapping.

**The difference:**

```yaml
services:
  redis:
    image: redis:7
    expose:
      - "6379"         # internal only — other containers on same network can reach it
                       # host CANNOT reach it
```

vs:

```yaml
services:
  redis:
    image: redis:7
    ports:
      - "6379:6379"    # publishes to host — creates DNAT rule
                       # reachable from host at localhost:6379
```

**Fix:**

```yaml
ports:
  - "6379:6379"                  # host:container
  - "127.0.0.1:6379:6379"        # localhost only — safer for dev databases
```

**Trap:** In production, the database port should almost never be published.
If only the app container needs Redis, `expose` or no mapping at all is
correct and more secure.

---

### 3. App starts before database is ready — `depends_on` without `condition`

**When it happens:** Your app errors on startup with "connection refused" or
"relation does not exist". Restarting the app manually fixes it.

```bash
docker compose up
# app | sqlalchemy.exc.OperationalError: could not connect to server: Connection refused
# app exited with code 1
```

**Root cause:** `depends_on` without a condition waits for the container to
**start**, not for the service inside to be **ready**. Postgres starts fast but
takes a few seconds to initialize its data directory and accept connections.

**Wrong (default behavior):**

```yaml
services:
  app:
    depends_on:
      - db          # waits for container start, not for postgres to be ready
```

**Fix — add a healthcheck and wait for it:**

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

  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy   # waits until healthcheck passes
```

**Common healthchecks:**

```yaml
# PostgreSQL
test: ["CMD-SHELL", "pg_isready -U postgres"]

# MySQL / MariaDB
test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

# Redis
test: ["CMD", "redis-cli", "ping"]

# Generic HTTP service
test: ["CMD-SHELL", "curl -sf http://localhost:8080/health || exit 1"]
```

**Trap:** `depends_on` only helps at startup. The app still needs retry logic
for when the database restarts during normal operation.

---

### 4. Data lost after `docker compose down` — volume confusion

**When it happens:** You bring a stack down and back up. Database is empty.
Files the app wrote are gone.

```bash
docker compose down
docker compose up
# db is empty — all data gone
```

**Root cause:** Container writes go to the writable layer (UpperDir in overlay2).
That layer is destroyed on `docker rm` — and `docker compose down` removes
containers. Without a named volume, there's nothing persisting the data.

**The three storage options:**

```yaml
services:
  db:
    image: postgres:16
    volumes:
      # 1. Named volume — Docker manages the path. Survives down. Only deleted with down -v.
      - db-data:/var/lib/postgresql/data

      # 2. Bind mount — your host path. Useful for dev (live reload). Not isolated.
      - ./data:/var/lib/postgresql/data

      # 3. Anonymous volume — no name. Deleted on down -v. Don't use for state.
      - /var/lib/postgresql/data

volumes:
  db-data:   # declares the named volume
```

**What survives what:**

| Action | Named volume | Bind mount | Anonymous volume |
|--------|-------------|------------|-----------------|
| `docker compose down` | ✅ survives | ✅ survives | ✅ survives |
| `docker compose down -v` | ❌ deleted | ✅ survives (host path) | ❌ deleted |
| `docker compose rm` | ✅ survives | ✅ survives | ❌ deleted |

**Trap:** Volume names are prefixed with the Compose project name (the directory
name by default). Renaming the directory or using `-p` changes the prefix and
Compose creates a new empty volume — the old one is orphaned.

```bash
docker volume ls | grep db-data    # find orphaned volumes
docker volume inspect <project>_db-data
# "Mountpoint": "/var/lib/docker/volumes/<project>_db-data/_data"
```

---

### 5. Service can't reach sibling by hostname — wrong network scope

**When it happens:** Two services are in the same `docker-compose.yml` but
DNS resolution between them fails.

```bash
docker compose exec app curl http://cache:6379
# curl: (6) Could not resolve host: cache
```

**Root cause:** The services are on different explicitly declared networks and
share no common network. A container only resolves names of containers on the
same network.

**Diagnose:**

```bash
docker network ls | grep <project>
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
# app is on "frontend" only — cache is on "backend" only — they never share a network
```

**Fix — ensure shared network membership:**

```yaml
services:
  app:
    networks:
      - frontend
      - backend     # app bridges both networks — can reach nginx and cache

  cache:
    networks:
      - backend     # only reachable from backend

  nginx:
    networks:
      - frontend    # only reachable from frontend

networks:
  frontend:
  backend:
```

With this layout `nginx` cannot reach `cache` directly — real L2 isolation via
separate bridge networks. See [multi-network](./multi-network/) for the full lab.

**Default network (no explicit networks declared):**

```yaml
# All services go on <project>_default automatically.
# All can reach each other by service name. Fine for simple stacks.
services:
  app:
    image: myapp
  db:
    image: postgres:16
# app resolves "db" — both on <project>_default
```

**Trap:** The project prefix comes from the directory name. Two separate Compose
projects that need to communicate require an externally-created shared network:

```yaml
networks:
  shared-net:
    external: true   # docker network create shared-net beforehand
```

---

## Decision: named volume vs bind mount; expose vs ports

Use **named volumes** for databases and any persistent state. Docker manages
the path, it survives stack restarts, and it works identically on any host.

Use **bind mounts** in development for live code reload (`./src:/app/src`).
In production, avoid bind mounts — the path must exist on the host and the
container gets full access to that host directory.

Use **expose** (or nothing) when a port is only needed between containers.
Use **ports** only when the host itself needs to reach the container.

Use the **default network** for simple stacks with no isolation requirement.
Use **explicit networks** when you need certain services unable to reach others.

---

## Quick Reference

```bash
# Lifecycle
docker compose up -d             # start in background
docker compose up --build        # rebuild images before starting
docker compose down              # stop and remove containers
docker compose down -v           # also remove named volumes
docker compose restart <service> # restart one service

# Logs
docker compose logs -f           # follow all services
docker compose logs -f app       # follow one service
docker compose logs --tail 50    # last 50 lines

# Exec and run
docker compose exec app bash                    # exec into running container
docker compose run --rm app sh                  # one-off container

# Status and debug
docker compose ps                               # services and port mapping
docker compose top                              # processes inside containers
docker compose exec app nslookup db             # test DNS between services
docker volume inspect <project>_db-data         # inspect named volume path

# Cleanup
docker compose down -v                          # stop + remove volumes
docker volume ls | grep <project>               # find orphaned volumes
docker volume rm <project>_<name>               # manual delete
```

---

## K8s Connection

| Docker Compose | Kubernetes equivalent |
|----------------|----------------------|
| Named volume | PersistentVolumeClaim |
| Explicit network | NetworkPolicy |
| `expose` | ClusterIP Service (internal only) |
| `ports` | NodePort or LoadBalancer Service |
| `depends_on: condition: service_healthy` | Init container or readiness probe |
| Service hostname (`db`, `cache`) | `<svc>.<namespace>.svc.cluster.local` |
| `docker compose down -v` | `kubectl delete pvc` |
| Default network (all services reachable) | No NetworkPolicy = pods can reach everything |

The mental model transfers directly. Named volumes → PVCs: both abstract away
where the data lives from the workload. Network isolation via explicit networks →
NetworkPolicy: both enforce which containers can talk to which. Healthcheck
conditions → readiness probes: both solve "don't route traffic until the backend
is ready".
