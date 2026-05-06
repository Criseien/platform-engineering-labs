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