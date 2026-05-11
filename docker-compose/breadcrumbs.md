## explicit networks vs default
- compose uses directory name as project prefix (docker-compose_frontend, etc.)
- separate networks = separate bridges, real L2 isolation
- container in two networks = two interfaces, sees both sides
- host reaches container only if port is explicitly exposed (DNAT rule)

## named volumes
- docker manages path: /var/lib/docker/volumes/<project>_<name>/_data
- `down` does NOT delete named volumes — `down -v` does
- anonymous volumes: no name, deleted with `down -v`

## depends_on
- without condition: waits for container to start, not to be healthy
- with condition: service_healthy waits for healthcheck to pass

## traps
- image with no default command exits immediately with zero logs
- diagnosis: `docker compose logs` empty = container never executed anything
- `6379/tcp` without `0.0.0.0:` = internal only, host cannot reach it

## docker volumes & storage drivers (7 May 2026)
- default container writes go to writable layer (UpperDir overlay2)
- writable layer destroyed on `docker rm` — data loss by default
- named volume: docker manages path /var/lib/docker/volumes/<name>/_data
- named volume survives `docker rm` — only deleted with `docker volume rm` explicitly
- bind mount: -v /host/path:/container/path — real host path, process has direct host access
- bind mount is NOT isolation — compromised container can access host path directly
- overlay2 copy-on-write: modifying LowerDir file copies it to UpperDir first, others unaffected
- LowerDir = image layers, read-only, shared between containers (no duplication on disk)
- UpperDir = container writes, unique per container, gone on rm
- MergedDir = unified view the process sees (LowerDir + UpperDir combined)
## traps
- named volume different name = different volume = data loss on recreate
- bind mount on Mac: path lives inside Docker Desktop VM, not visible in Finder
- -v db-data: /path (space after colon) = syntax error, volume not mounted

## docker security (11 May 2026)
- 5 isolation layers: namespaces (see), capabilities (do), filesystem (touch),
  user namespaces (who), seccomp (syscalls)
- --privileged removes capabilities + seccomp. -v /:/host removes filesystem isolation
- --cap-drop ALL + --cap-add <specific> = minimum privilege
- CHOWN capability required for chown on files not owned by process
- ping works without NET_RAW on modern kernels (ICMP sockets via ping_group_range)
- user namespaces: --user 1000:1000 maps root inside to unprivileged UID on host
- seccomp blocks unshare syscall by default — CVE-2022-0185 was blocked this way
- seccomp=unconfined = open attack surface, same risk as --privileged
## traps
- --cap-drop ALL does not block ping on Docker Desktop (ICMP sockets bypass NET_RAW)
- cannot add port mapping to running container — must recreate

## docker troubleshooting (11 May 2026)
- logs first: docker logs --tail 50 <id>
- container not running: docker inspect still works — check Mounts, Env, Config
- ENOENT on config file: check bind mount source exists on host with ls -la
- ss -tulnp inside container confirms app listening — separate from host port mapping
- port in docker ps empty = no -p flag on docker run = DNAT rule missing
- cannot add -p to running container — workaround: nginx/socat proxy to container IP
## traps
- docker ps shows ports only if -p was declared at run time
- app listening inside container ≠ accessible from outside