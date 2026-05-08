# multi-network — L2 isolation with explicit bridge networks

## Scenario

A three-tier stack has nginx serving the frontend, an app handling business
logic, and postgres storing data. The requirement: nginx must never be able
to reach postgres directly — only the app can talk to the database. A single
default network puts all three on the same bridge and makes everything
reachable from everything.

## Network Topology

```
┌──────────────────────────────┐    ┌──────────────────────────────┐
│       frontend network       │    │        backend network        │
│    (separate Linux bridge)   │    │    (separate Linux bridge)    │
│                              │    │                              │
│   ┌────────┐   ┌──────────┐  │    │  ┌──────────┐   ┌────────┐  │
│   │ nginx  │   │   app    │  │    │  │   app    │   │   db   │  │
│   │  :80   │   │  :8080   │  │    │  │  :8080   │   │  :5432 │  │
│   └────────┘   └──────────┘  │    │  └──────────┘   └────────┘  │
└──────────────────────────────┘    └──────────────────────────────┘

nginx → app   ✅  same bridge (frontend)
app   → db    ✅  same bridge (backend)
nginx → db    ❌  different bridges — "db" doesn't even resolve from frontend
```

`app` gets two network interfaces — one on each bridge. It is the only service
that can reach both tiers.

## Diagnosis Flow

Run the stack, then verify isolation step by step:

```
docker compose up -d
→ docker compose exec nginx curl http://app:8080     # should succeed
→ docker compose exec app nc -zv db 5432             # should succeed
→ docker compose exec nginx nc -zv db 5432           # should FAIL
→ docker compose exec nginx ping db                  # Name or service not known ← correct behavior
```

The key observation: `nginx` doesn't get a connection refused — it gets a DNS
failure. Docker's embedded DNS only returns addresses for containers that share
a network with the requester. `db` is invisible to the frontend network entirely.

## Root Cause (of the original single-network problem)

When no networks are declared, Compose puts all services on one auto-created
bridge. Every service can resolve every other service by name. There is no
isolation — nginx can reach postgres directly, which defeats the purpose of
having separate tiers.

## Fix

```yaml
services:
  nginx:
    image: nginx:alpine
    networks:
      - frontend          # frontend only — cannot see backend

  app:
    image: python:3.11-slim
    networks:
      - frontend
      - backend           # bridges both tiers

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend           # backend only — invisible to frontend

networks:
  frontend:
  backend:
```

## The Critical Trap: Project Prefix on Network Names

Compose prefixes network names with the project name (directory name by default).
Two separate Compose projects cannot share services by hostname unless they
share an explicitly declared external network.

```bash
docker network ls | grep frontend
# myproject_frontend   ← not just "frontend"
```

If you need cross-project communication (e.g., a shared database used by two
separate stacks), create the network externally first:

```bash
docker network create shared-backend

# In each docker-compose.yml:
networks:
  shared-backend:
    external: true
```

## Additional Traps from This Lab

### Two interfaces on `app` — verify them

```bash
docker compose exec app ip a
# eth0 → frontend subnet (e.g., 172.20.0.x)
# eth1 → backend subnet  (e.g., 172.21.0.x)
```

Two interfaces confirm the container is bridged between both networks.
A single interface means a network declaration was missed in the Compose file.

### Host-level bridge confirmation

Each declared network is a real Linux bridge on the host:

```bash
ip link show type bridge        # two new bridges appear when stack is up
ip link show type veth          # veth pairs connecting containers to bridges
```

## Decision: default network vs explicit networks

Use the **default network** (declare no networks) when all services need to
reach each other and there is no isolation requirement. Simple dev stacks,
single-tier apps.

Use **explicit networks** when you need to prevent certain services from
communicating directly — database not reachable from the public-facing tier,
monitoring sidecar that shouldn't talk to the app. Each declared network
creates a separate Linux bridge with its own subnet.

## Key Commands

```bash
# Run the lab
docker compose up -d
docker compose ps

# Verify topology
docker compose exec nginx curl -s http://app:8080    # ✅ should work
docker compose exec app nc -zv db 5432               # ✅ should work
docker compose exec nginx nc -zv db 5432             # ❌ should fail (Name not known)

# Inspect network membership
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool

# Host-level view
docker network ls | grep $(basename $(pwd))
docker network inspect <project>_frontend

# Cleanup
docker compose down
```

## K8s Connection

This isolation model maps to NetworkPolicy in Kubernetes. In K8s the default
is the opposite of Compose explicit networks: all pods can reach all other
pods by default (flat network). NetworkPolicy restricts that:

```yaml
# Equivalent: db only accepts traffic from app tier, not from nginx tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-backend-only
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
```

Compose uses separate bridges (no shared L2 = DNS doesn't resolve).
K8s uses iptables/eBPF rules on a flat network (DNS resolves, but traffic
is dropped by the kernel before it arrives). Same intent, different mechanism.
