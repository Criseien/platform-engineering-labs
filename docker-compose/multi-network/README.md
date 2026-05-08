# multi-network — L2 isolation with explicit bridge networks

This lab demonstrates that Docker Compose networks are real Linux bridge networks,
not just logical groupings. Each declared network gets its own bridge interface
and its own IP subnet. Containers on different bridges can't route to each other
unless they share a common bridge — that's real L2 isolation at the kernel level,
not a firewall rule.

The scenario mirrors a standard three-tier architecture: an `nginx` frontend that
should never reach the database directly, an `app` that bridges both tiers, and
a `db` that is invisible to the frontend network.

---

## Network topology

```
┌─────────────────────────────┐    ┌──────────────────────────────┐
│      frontend network       │    │       backend network         │
│   (172.20.0.0/16 bridge)    │    │   (172.21.0.0/16 bridge)     │
│                             │    │                              │
│  ┌─────────┐  ┌──────────┐  │    │  ┌──────────┐  ┌─────────┐  │
│  │  nginx  │  │   app    │  │    │  │   app    │  │   db    │  │
│  │ :80     │  │ :8080    │  │    │  │ :8080    │  │ :5432   │  │
│  └─────────┘  └──────────┘  │    │  └──────────┘  └─────────┘  │
└─────────────────────────────┘    └──────────────────────────────┘

nginx → app   ✅  (same frontend network)
app   → db    ✅  (same backend network)
nginx → db    ❌  (different networks — no shared bridge)
```

`app` has two network interfaces — one on each bridge. It's the only service
that can reach both sides. `nginx` and `db` each have one interface and are
invisible to each other.

---

## Files

- `docker-compose.yml` — the full stack definition
- `README.md` — this file

---

## Run the lab

```bash
cd multi-network
docker compose up -d
docker compose ps
```

---

## What to verify

### 1. Confirm the bridges were created

```bash
docker network ls | grep multi-network
# multi-network_frontend   bridge
# multi-network_backend    bridge

# Each is a real Linux bridge — inspect the subnet assigned
docker network inspect multi-network_frontend --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
docker network inspect multi-network_backend  --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

### 2. Confirm `app` has two interfaces

```bash
docker compose exec app ip a
# eth0 → frontend network subnet
# eth1 → backend network subnet
# Two interfaces = bridged between both networks
```

### 3. `nginx` can reach `app` — same network

```bash
docker compose exec nginx curl -s http://app:8080/
# 200 OK — nginx resolves "app" via Docker's embedded DNS on frontend network
```

### 4. `app` can reach `db` — same network

```bash
docker compose exec app nc -zv db 5432
# db (172.21.x.x:5432) open — app resolves "db" via backend network DNS
```

### 5. `nginx` cannot reach `db` — different networks (the important one)

```bash
docker compose exec nginx nc -zv db 5432
# nc: getaddrinfo for host "db" port 5432: Name or service not known
# ← DNS for "db" doesn't exist on the frontend network

docker compose exec nginx ping db
# ping: db: Name or service not known
```

This is the point of the lab. `nginx` doesn't just fail to connect — it can't
even resolve the name `db`. Docker's embedded DNS only responds with addresses
of containers that share the same network as the requester. The database is
completely invisible to the frontend tier.

### 6. Observe the host-level bridges

```bash
# Two separate bridges on the host
ip link show type bridge | grep -A1 "multi-network"

# The veth pairs connecting containers to each bridge
ip link show type veth

# No cross-routing between the two subnets
ip route | grep "172.2"
```

---

## Clean up

```bash
docker compose down
docker network ls | grep multi-network   # confirm networks were removed
```

---

## K8s Connection

This topology maps directly to how Kubernetes NetworkPolicies work with a CNI
plugin like Calico or Cilium.

In K8s there's one flat network (all pods can reach all pods by default).
NetworkPolicy is what restricts it:

```yaml
# Equivalent of "nginx cannot reach db" — deny ingress to db from frontend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-frontend
spec:
  podSelector:
    matchLabels:
      app: db
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend   # only backend pods can reach db
```

The mental model is the same: restrict which services can talk to which.
The implementation differs — Compose uses separate bridges (no shared L2),
K8s uses iptables/eBPF rules on a flat network. Same result, different mechanism.
