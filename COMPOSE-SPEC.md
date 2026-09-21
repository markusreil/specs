# COMPOSE.md

Guidance for creating a new Docker Compose project from scratch.

Follow this document when scaffolding a new compose project so the result is
consistent, predictable, and easy for other agents to work on.

## Deliverable layout

```
docker-compose.yml     single compose file (services, hosts, networks, volumes)
.env.example           tracked example configuration (source of truth for shape)
.env                   local configuration (secrets) — never tracked in git, copied from .env.example
.gitignore             must ignore `.env`
README.md              project overview, architecture, quickstart, operational notes
<service>/             one directory per service needing a build: Dockerfile, docker/, README
<service>/Dockerfile   build instructions for the service
<service>/docker/      all context files copied into the image (entrypoint.sh, scripts, templates, etc.)
<service>/README.md    build details, env vars, reasoning behind security choices
```

* One compose file for the whole project; no per-service compose fragments.
* The tracked example may be named `.env.example` (preferred) or `env.example`;
  both are accepted. This spec uses `.env.example` throughout — read it as
  `env.example` where the project uses that name (e.g. `cp env.example .env`).
* One directory per service holding everything needed to build its image and
  the README explaining it.
* Each service README documents build steps, environment variables, and the
  reasoning behind non-obvious choices (especially security). Keep READMEs in
  sync when behavior changes.

## Non-negotiable rules

1. **All configuration lives in `.env`.** Secrets, versions, domain, network
   names — everything that varies between deployments goes there.

   Some common variable names used in many different clusters are:
    * **BASE_DOMAIN**: Domain name used for the cluster's vhost names. Use the
        docker hostname to get vhost names like `service1.host.domain` or use an
        additional subdomain `cluster.host.domain` to group services, e.g. 
        `serviceX.cluster.host.domain`).
    * **NGINX_PROXY_NETWORK**: docker network in which nginx talks to downstream services.
    * **BASE_IMAGE**: Base image recorded in the built image as a hardcoded
        `ENV` (e.g. `ENV BASE_IMAGE=alpine:3.20`). No `ARG`, no compose
        wiring — it documents what the image was built `FROM`. Build-time
        only: never put it in `.env` / `.env.example`.
    * **BUILD_DATE**: Build timestamp (UTC ISO-8601) recorded in the built
        image. Dockerfile pattern is `ARG BUILD_DATE=unknown` +
        `ENV BUILD_DATE=${BUILD_DATE}`; compose passes it via
        `build.args` as `${BUILD_DATE:-unknown}` (optional, fails open to
        `unknown`). Stamp it at build time with
        `BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) docker compose build`.
        Build-time only: never put it in `.env` / `.env.example` — it is a
        shell env var at build time, not deployment configuration.

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
   compose file, then reference them everywhere the hostname is needed (`VIRTUAL_HOST`, `LETSENCRYPT_HOST`, app-level
   hostname settings). Changing
   one variable in `.env` moves the whole stack.
6. **`base image alpine` preferred.** Build images from source in the repo where
   it makes sense; upstream images are acceptable on a case-by-case basis
   (e.g. when the upstream image *is* the service, like nginx-proxy or
   acme-companion). Extra build steps per service are fine. Document the
   reasoning when using upstream images.
7. **Standard named volumes preferred, no data bind mounts.** Use standard named docker
   volumes for every stateful path so configuration survives restarts and
   rebuilds. Never use host bind mounts for stateful data (e.g. `./data:/data`);
   bind mounts are acceptable on a case-by-case basis for config overlays
   (e.g. `./vhost.d`, `./conf.d`) and special cases (docker socket, devices).
   Document the reasoning when a bind mount is used.
8. **Entrypoints are write-once config seeders.** Each `entrypoint.sh` seeds a
   minimal config on FIRST start only (`if [ ! -f ... ]`), then never
   overwrites. Config changes do not apply to existing volumes; document the
   reseed procedure (remove service + volume, then `up -d`).
9. **`PUID`/`PGID` privilege drop.** Entrypoints run as root, create
   user/group with the requested IDs, `chown` correctly, then `su-exec` to the
   unprivileged user. Chown config dirs recursively (small); chown data dirs
   only at the root (potentially huge — new files inherit ownership from the
   running user).
10. **`.env` is never tracked in git.** Track `.env.example` instead and
    gitignore `.env`; do not scatter secrets into other files. Standard
    process for the user is to copy the example file and edit it
    accordingly before starting the cluster (`cp .env.example .env`).
11. Set sensible defaults to integrate with [Homepage](https://gethomepage.dev/).
    Set labels like:
    - homepage.group=Download
    - homepage.name=The Website
    - homepage.href=https://nginx.proxy.url
12. `restart` should be `unless-stopped` by default.

## Creating a new project

1. **Nail down the requirements first** — services, public hostnames, which
   configuration must be per-deployment, and the security posture (LAN-only vs
   exposed). Write the spec down before scaffolding.
2. **Scaffold the layout**: compose file, `.env.example` (+ gitignored
   `.env`), `.gitignore`, README.md, one service dir per
   service with Dockerfile + entrypoint.sh + README.
3. **Write the compose file bottom-up**:
   `name:` → `x-hosts` anchors → services (each with build context, image tag
   from the version var, env, volumes, expose, proxy network) → external
   `networks:` → `volumes:`.
4. **Every env var gets a `:?...` guard**; optional vars use `${VAR:-default}`
   explicitly and are documented as optional.
5. **Write each entrypoint as a first-run seeder + privilege drop.**
6. **Fill in `.env.example`** with every variable, a sensible example, and a
   comment for each. Keep it in sync with the compose file and READMEs.
   Ensure `.env` is gitignored.
7. **Verify** (see below).

## Build / run / verify

```sh
# standard start: copy the tracked example and edit it accordingly (.env is never committed)
cp .env.example .env
# edit .env accordingly before starting the cluster
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

1. `docker compose config` passes with `.env` (copied from `.env.example`).
2. Required vars still fail fast — no silent defaults introduced.
3. Env vars are documented in `.env.example` and the service READMEs; `.env` is gitignored.
4. Seeding logic only runs on first start; existing configs are untouched.
5. `PUID`/`PGID` chown behavior preserved (no recursive chown on data dirs).
6. No host `ports:` mappings — everything goes through the proxy network.
7. Hostnames come from the anchors, not copy-pasted literals.
8. No hardcoded `container_name:` in services — rely on default container naming.
9. No host bind mounts for stateful data (exceptions: config overlays like
   `vhost.d`/`conf.d` on a case-by-case basis, docker socket, special devices,
   etc.) — use standard named docker volumes for data.

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