# platform-engineering-labs

Hands-on labs covering the orchestration and platform tooling layer — Docker Compose,
Kubernetes, CI/CD pipelines, and the operational patterns used in production.

This repo picks up where [docker-internals](https://github.com/Criseien/docker-internals-labs)
leaves off. That repo covers what happens at the kernel (cgroups, namespaces, runc).
This one covers how you orchestrate those primitives: how Compose wires multi-service
stacks, how Kubernetes schedules and networks pods, and how platform teams build
the scaffolding that makes all of it operable.

The approach stays the same: start from a real failure scenario, trace it to the root
cause, and connect it explicitly to how the same mechanism works in Kubernetes.

---

## Why this matters

Knowing that a pod is in `CrashLoopBackOff` is not the same as knowing why.
Most production incidents have a clear mechanical explanation — a healthcheck
misconfiguration, a volume that wasn't persisted, a service that can't reach its
dependency because it's on the wrong network. These labs build the mental model to
diagnose those failures in under five minutes.

---

## Labs

### [docker-compose](./docker-compose/) — Multi-service orchestration

**Scenario:** Backend container crashes silently. Port exposed but not reachable.
App starts before the database is ready. Data disappears after a redeploy.
Two services on different networks can't communicate.

Covers: explicit vs default networks (L2 isolation, bridge-per-network), named vs
anonymous volumes (what survives `docker compose down`), `depends_on` with and
without healthcheck conditions, port exposure vs port publishing, and container-to-container
DNS on user-defined networks.

**K8s connection:** Docker Compose networks map directly to Kubernetes NetworkPolicies
and service discovery. Named volumes are the mental model for PersistentVolumeClaims.
`depends_on: condition: service_healthy` is what init containers and readiness probes
solve in Kubernetes.

---

## Prerequisites

- Linux with Docker and Docker Compose v2 (`docker compose`, not `docker-compose`)
- For K8s labs (coming): a local cluster — kind or k3s on AlmaLinux 9

## Recommended order

`docker-compose` → Kubernetes fundamentals (coming)

Docker Compose teaches multi-service networking and volume management in a simple
environment. That mental model transfers directly to Kubernetes — the primitives
are the same, just with more automation and failure modes.
