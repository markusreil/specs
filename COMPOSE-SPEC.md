# COMPOSE.md

Guidance for creating a new Docker Compose project from scratch. 

Follow this document when scaffolding a new compose project so the result is
consistent, predictable, and easy for other agents to work on.

## Deliverable layout

```
docker-compose.yml     single compose file (services, hosts, networks, volumes)
.env                   all configuration (secrets) — tracked in git (source of truth)
README.md              project overview, architecture, quickstart, operational notes
<service>/             one directory per service: Dockerfile, entrypoint.sh, README
<service>/README.md    build details, env vars, reasoning behind security choices
```

* One compose file for the whole project; no per-service compose fragments.
* One directory per service holding everything needed to build its image and
  the README explaining it.
* Each service README documents build steps, environment variables, and the
  reasoning behind non-obvious choices (especially security). Keep READMEs in
  sync when behavior changes.

## Non-negotiable rules

1. **All configuration lives in `.env`.** Secrets, versions, domain, network
   names — everything that varies between deployments goes there.
2. **Variable interpolation is the validation mechanism.** Every required var
   is referenced with `${VAR:?...}` so `docker compose config` fails fast on
   missing or empty values. Never substitute silent defaults for required
   configuration.
3. **Single compose project name and default container names.** Set project
   name once (e.g. `name:` at the top of the file or the project directory).
   Volumes, networks, and containers then share a consistent prefix. Never use
   `container_name` in the compose file; rely on Docker Compose's automatic
   default container naming so containers scale cleanly and avoid name collisions.
4. **Reverse-proxy integration, not host port publishing.** Services attach to
   the shared external proxy network and advertise HTTP via `expose` plus
   `VIRTUAL_HOST` / `VIRTUAL_PORT` / `LETSENCRYPT_HOST` / `LETSENCRYPT_EMAIL`
   env vars. No `ports:` mappings; the proxy owns all public endpoints.
5. **Hostnames anchored once.** Define `x-hosts` anchors at the top of the
   compose file, then reference them everywhere the hostname is needed
   (`VIRTUAL_HOST`, `LETSENCRYPT_HOST`, app-level hostname settings). Changing
   one variable in `.env` moves the whole stack.
6. **`base image alpine`.** Build images from source in the repo; avoid
   third-party/LinuxServer-style images unless there is a real reason not to.
   Extra build steps per service are fine.
7. **Standard named volumes, no bind mounts.** Use standard named docker
   volumes for every stateful path so configuration survives restarts and
   rebuilds. Never use host bind mounts (e.g. `./data:/data`); data and
   configuration must live in named volumes, never on the host filesystem
   directly, to ensure portability and isolation.
8. **Entrypoints are write-once config seeders.** Each `entrypoint.sh` seeds a
   minimal config on FIRST start only (`if [ ! -f ... ]`), then never
   overwrites. Config changes do not apply to existing volumes; document the
   reseed procedure (remove service + volume, then `up -d`).
9. **`PUID`/`PGID` privilege drop.** Entrypoints run as root, create
   user/group with the requested IDs, `chown` correctly, then `su-exec` to the
   unprivileged user. Chown config dirs recursively (small); chown data dirs
   only at the root (potentially huge — new files inherit ownership from the
   running user).
10. **`.env` is tracked in git.** Do not gitignore it and do not scatter
    secrets into other files.

## Creating a new project

1. **Nail down the requirements first** — services, public hostnames, which
   configuration must be per-deployment, and the security posture (LAN-only vs
   exposed). Write the spec down before scaffolding.
2. **Scaffold the layout**: compose file, `.env`, README.md, one service dir per
   service with Dockerfile + entrypoint.sh + README.
3. **Write the compose file bottom-up**:
   `name:` → `x-hosts` anchors → services (each with build context, image tag
   from the version var, env, volumes, expose, proxy network) → external
   `networks:` → `volumes:`.
4. **Every env var gets a `:?...` guard**; optional vars use `${VAR:-default}`
   explicitly and are documented as optional.
5. **Write each entrypoint as a first-run seeder + privilege drop.**
6. **Fill in `.env`** with every variable, a sensible example, and a
   comment for each. Keep it in sync with the compose file and READMEs.
7. **Verify** (see below).

## Build / run / verify

```sh
# edit .env directly (tracked in git as source of truth)
docker compose config  # sanity check; fails fast on missing vars
docker compose build
docker compose up -d
docker compose ps
```

There is usually no test suite; `docker compose config` (variable
interpolation) plus building images and checking the containers start and
serve their UIs is the verification baseline.

## Review checklist

Before finishing a project or change:

1. `docker compose config` passes with `.env`.
2. Required vars still fail fast — no silent defaults introduced.
3. Env vars are documented in `.env` and the service READMEs.
4. Seeding logic only runs on first start; existing configs are untouched.
5. `PUID`/`PGID` chown behavior preserved (no recursive chown on data dirs).
6. No host `ports:` mappings — everything goes through the proxy network.
7. Hostnames come from the anchors, not copy-pasted literals.
8. No hardcoded `container_name:` in services — rely on default container naming.
9. No host bind mounts in volumes (exceptions: docker socket, special devices, etc.)
   — use standard named docker volumes only.

## Common pitfalls

* Replacing `${VAR:?...}` with `${VAR:-default}` for a secret or version —
  breaks fail-fast validation.
* Hardcoding `container_name:` in services — breaks scaling (`replicas > 1`) and
  causes name collision errors on restart or multi-instance setups.
* Using host bind mounts (e.g. `./config:/config`) instead of standard named
  volumes — breaks portability, cross-platform file permissions, and
  container filesystem encapsulation.
* Publishing `ports:` next to the proxy setup — creates bypasses of the
  reverse proxy and surprises on shared hosts.
* Entrypoints that overwrite existing config on every start — destroys user
  configuration on restarts.
* Recursive chown over huge data volumes on every boot — slow and
  unnecessary; ownership of new files inherits from the running user.
* App-generated secrets (API keys) with no seeding path — downstream services
  end up scraping config files or pinning unstable credentials.