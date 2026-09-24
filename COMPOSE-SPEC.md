# COMPOSE.md

Guidance for creating a new Docker Compose project from scratch.

Follow this document when scaffolding a new compose project so the result is
consistent, predictable, and easy for other agents to work on.

## Deliverable layout

```
docker-compose.yml     single compose file (services, hosts, networks, volumes)
env.example            tracked example configuration, not hidden (source of truth for shape)
.env                   local configuration (secrets) — hidden, never tracked in git, copied from env.example
.gitignore             must ignore `.env`
README.md              project overview, architecture, quickstart, operational notes
CHANGELOG.md           notable changes, kept in sync with development (see below)
AGENTS.md              short, ongoing project rules for agents working in the repo (see below)
<service>/             one directory per service needing a build: Dockerfile, docker/, README
<service>/Dockerfile   build instructions for the service
<service>/docker/      all context files copied into the image (scripts, templates, etc.; entrypoint.sh only where rules 8-9 apply)
<service>/README.md    build details, env vars, reasoning behind security choices
```

* One compose file for the whole project; no per-service compose fragments.
* The tracked example is `env.example` (not hidden, tracked by git); do not
  use a dot-prefixed variant. Copy it to `.env` (hidden, never tracked)
  for local use (e.g. `cp env.example .env`).
* One directory per service holding everything needed to build its image and
  the README explaining it.
* Each service README documents build steps, environment variables, and the
  reasoning behind non-obvious choices (especially security). Keep READMEs in
  sync when behavior changes.

## Changelog

Create a `CHANGELOG.md` at the project root and follow the
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) spec.

* Keep an `## [Unreleased]` section at the top and fill it in **as changes are
  made** — do not batch entries up for a later release.
* The `Unreleased` template carries the standard subsections: `Added`,
  `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`. Add each entry under
  the matching one and leave the unused ones empty.
* Newest entries first within a section; dates are ISO-8601 (`YYYY-MM-DD`).
* **Do not create a release section on your own.** When the user asks for a
  release, move the accumulated `Unreleased` entries into a new
  `## [<version>] - <date>` section (semantic version), leave a fresh empty
  `## [Unreleased]` above it, and keep the sections newest-first.

The header should look like:

```md
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

### Changed

### Deprecated

### Removed

### Fixed

### Security
```

## AGENTS.md

Create an `AGENTS.md` at the project root: a short summary of this spec, scoped
to the rules that stay relevant **while working in the project**.

* Include the ongoing rules an agent must respect on every change: `.env` /
  `env.example` handling, fail-fast `${VAR:?...}` interpolation, reverse-proxy
  integration (no `ports:`) including the **complete** downstream contract
  (every applicable var, `GEN_SELF_SIGNED_CERT` defaulting to `false` via
  `.env`), hostname anchors, named volumes over
  bind mounts, entrypoint seeding + `PUID`/`PGID`, changelog discipline,
  `restart` policy, Homepage labels, and the verification commands.
* Keep it short — a summary, not a copy. Link back to `README.md` and the
  service READMEs for detail instead of duplicating them.
* **Exclude anything that only matters during initial project creation**: the
  deliverable layout, the "Creating a new project" steps, and other one-time
  scaffolding guidance stay in this spec only. If it is not needed to work on
  the finished project, it does not belong in `AGENTS.md`.
* Keep `AGENTS.md` in sync when the project's rules change.

## Non-negotiable rules

1. **All configuration lives in `.env` (hidden, never tracked).** Secrets, versions, domain, network
   names — everything that varies between deployments goes there.

   Some common variable names used in many different clusters are:
    * **BASE_DOMAIN**: Domain name used for the cluster's vhost names. Use the
        docker hostname to get vhost names like `service1.host.domain` or use an
        additional subdomain `cluster.host.domain` to group services, e.g. 
        `serviceX.cluster.host.domain`).
    * **NGINX_PROXY_NETWORK**: docker network in which nginx talks to downstream services. Defaults to `web-proxy`; downstream services declare it `external: true` and reuse the same name.
    * **BASE_IMAGE**: Base image recorded in the built image as a hardcoded
        `ENV` (e.g. `ENV BASE_IMAGE=alpine:3.20`). No `ARG`, no compose
        wiring — it documents what the image was built `FROM`. Build-time
        only: never put it in `.env` / `env.example`.
    * **BUILD_DATE**: Build timestamp (UTC ISO-8601) recorded in the built
        image. Dockerfile pattern is `ARG BUILD_DATE=unknown` +
        `ENV BUILD_DATE=${BUILD_DATE}`; compose passes it via
        `build.args` as `${BUILD_DATE:-unknown}` (optional, fails open to
        `unknown`). Stamp it at build time with
        `BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) docker compose build`.
        Build-time only: never put it in `.env` / `env.example` — it is a
        shell env var at build time, not deployment configuration.

    Group `.env` / `env.example` with global vars at the top, then one
    `# --- <service> ---` section per service (matching compose service
    names). Keep both files in the same order and shape; every variable
    keeps its own comment.

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
   `VIRTUAL_HOST` / `VIRTUAL_PORT` / `ACME_HOST` env vars (use the `ACME_`
   spelling for every ACME variable — see
   [ACME vs LETSENCRYPT variables](#acme-vs-letsencrypt-variables)).
   Downstream services must be **variant-agnostic**: they declare their **full**
   proxy configuration — every applicable var, including the real-cert and
   self-signed TLS opt-ins when HTTPS is wanted — and must not know whether the
   cluster is LAN-only or internet-facing. In general `GEN_SELF_SIGNED_CERT` is
   `false`; it is exposed as an env var so the default can be overridden for
   local testing. An omitted var does
   not fail loudly; it silently serves the host HTTP-only, skips the
   certificate, or pins the service to one variant. The proxy cluster decides
   which opt-in applies (see [Downstream proxy contract](#downstream-proxy-contract)).
   No `ports:` mappings; the proxy owns all public endpoints. The cluster
   contact email is `ACME_EMAIL` (in the proxy's `.env`, wired to the
   companion's `DEFAULT_EMAIL`); downstream services inherit it, so set
   `ACME_EMAIL` on a service only for an explicit override.
5. **Hostnames anchored once.** Define `x-hosts` anchors at the top of the
   compose file, then reference them everywhere the hostname is needed (`VIRTUAL_HOST`, `ACME_HOST`, app-level
   hostname settings). Derive hostnames from the cluster's `BASE_DOMAIN`
   (e.g. `service.${BASE_DOMAIN}`) rather than a per-service full-hostname
   variable, so changing one variable in `.env` moves the whole stack.
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
8. **Entrypoints are write-once config seeders (custom-built images only).** When
   a service needs first-run seeding, each `entrypoint.sh` seeds a
   minimal config on FIRST start only (`if [ ! -f ... ]`), then never
   overwrites. Config changes do not apply to existing volumes; document the
   reseed procedure (remove service + volume, then `up -d`). Not required for
   3rd-party upstream images where the upstream image *is* the service (see
   rule 6) and no seeding is needed (e.g. config baked at build time);
   document in the service README why no entrypoint exists.
9. **`PUID`/`PGID` privilege drop (where a custom entrypoint exists).** Entrypoints run as root, create
   user/group with the requested IDs, `chown` correctly, then `su-exec` to the
   unprivileged user. Chown config dirs recursively (small); chown data dirs
   only at the root (potentially huge — new files inherit ownership from the
   running user). Not required when there is no custom entrypoint;
   upstream images manage their own user. Document the choice in the service README.
10. **`.env` (hidden) is never tracked in git.** Track `env.example` instead and
    gitignore `.env`; do not scatter secrets into other files. Standard
    process for the user is to copy the example file and edit it
    accordingly before starting the cluster (`cp env.example .env`).
11. Set sensible defaults to integrate with [Homepage](https://gethomepage.dev/).
    Set labels like:
    - homepage.group=Download
    - homepage.name=The Website
    - homepage.icon=the-website
    - homepage.href=https://nginx.proxy.url
    - homepage.description=What the service does

    Without `icon` and `description` the card has nothing to render and appears
    visually empty; Homepage does not fall back to the site favicon.
12. `restart` should be `unless-stopped` by default.

## Downstream proxy contract

A downstream service declares its **complete** proxy configuration and must not
know, or care, whether the cluster it runs in is LAN-only or internet-facing.
The proxy cluster owns that decision and honours only the part that applies to
its variant; the rest must be inert.

**Set every variable that applies.** The proxy fails **silently**, not loudly,
on an incomplete contract: a missing `VIRTUAL_HOST` means the service is not
routed at all; a missing TLS opt-in means it is served over HTTP only (or with
no certificate); declaring only one TLS opt-in pins the service to a single
variant. A service is correctly wired only when its whole contract is present:

* `VIRTUAL_HOST` — always.
* `VIRTUAL_PORT` / `VIRTUAL_PROTO` — whenever the defaults in the table below do
  not match the app (multiple exposed ports, non-HTTP backend, etc.).
* For HTTPS, `ACME_HOST`, plus the `GEN_SELF_SIGNED_CERT` opt-in exposed as an
  optional `.env` variable. In general the self-signed opt-in is `false` and is
  overridden to `true` only for a LAN/self-signed deployment or local testing;
  each variant still honours only one of the two.

Rules for every proxied service:

* Attach to the shared external proxy network; never publish `ports:`. The
  network is created and owned by the proxy cluster (its name comes from the
  cluster's `NGINX_PROXY_NETWORK`, default `web-proxy`); a downstream service
  declares it `external: true`, reuses the same name, and is started **after**
  the proxy cluster is up.
* Advertise the app via `expose:`. The exposed container port must be the one
  `VIRTUAL_PORT` selects (or the app's only exposed port).
* When the service should be reachable over HTTPS, declare both TLS opt-ins;
  the deployment variant decides which one is used:
  * `ACME_HOST` — request a publicly trusted certificate from the cluster's ACME
    companion. Only meaningful in an internet-facing variant (where the
    companion runs); ignored otherwise. Every host listed must also appear in
    `VIRTUAL_HOST`.
  * `GEN_SELF_SIGNED_CERT` — request a self-signed certificate from the
    cluster's local cert generator. Only meaningful in a LAN/self-signed
    variant; ignored otherwise. **In general this is `false`** (most deployments
    are internet-facing and use `ACME_HOST`). Expose it as an optional `.env`
    variable — e.g. `<SERVICE>_GEN_SELF_SIGNED_CERT`, defaulting to `false` —
    and wire it into the container, so a LAN/self-signed deployment (or local
    testing) can override the default to `true` without editing the compose
    file. The host must be an exact, filename-safe name: wildcard
    (`*.example.com`), regexp (`~^...`), port-bearing (`host:port`),
    path-bearing and `..`-containing `VIRTUAL_HOST` values cannot become a
    `<host>.crt` file and are **silently skipped** — no certificate, no error.
  The key is always declared; only its value is overridable. Wiring it from
  `.env` with a `false` default lets the same compose file deploy unchanged
  against every variant, while hardcoding `true` or omitting the key takes that
  choice away.
* Do not set `HTTPS_METHOD`, `HSTS`, `CERT_NAME`, or a per-service contact email
  (unless deliberately overriding the cluster default) — TLS behaviour is
  cluster policy supplied by the proxy.

| Variable | Required | Purpose |
| --- | --- | --- |
| `VIRTUAL_HOST` | always | Routable hostname(s), comma-separated; from an `x-hosts` anchor. |
| `VIRTUAL_PORT` | when the app exposes more than one port, or the proxied port is not the app's only exposed one | Container port the proxy forwards to. Defaults to the single exposed port, else 80. |
| `VIRTUAL_PROTO` | non-HTTP backends only | Backend protocol (e.g. `https`, `uwsgi`, `fastcgi`). |
| `ACME_HOST` | for HTTPS — always set **with** `GEN_SELF_SIGNED_CERT` | Same value as `VIRTUAL_HOST` (must be a subset); real-certificate opt-in, honoured only by the internet variant. |
| `GEN_SELF_SIGNED_CERT` | for HTTPS — declared with `ACME_HOST` | `true` / `1` / `t`; self-signed-certificate opt-in, honoured only by the LAN variant. In general `false`; wire it from an optional `.env` variable (default `false`) so local testing can override it. |
| `ACME_EMAIL` | override only | Optional per-service ACME contact; defaults to the cluster's `ACME_EMAIL`/`DEFAULT_EMAIL`. |
| `LETSENCRYPT_TEST` | staging only | Optional `true` to use the Let's Encrypt **staging** CA while testing; forces staging and ignores `ACME_CA_URI` and any contact email. |

Clusters that name the local opt-in differently must still expose an equivalent
variant-neutral opt-in — no downstream service may be required to know the
variant.

### ACME vs LETSENCRYPT variables

Use the `ACME_*` spelling for **every** ACME variable — `ACME_HOST`,
`ACME_EMAIL`, `ACME_SINGLE_DOMAIN_CERTS`, `ACME_KEYSIZE`, and the rest. This is
the current upstream naming and the only spelling new or existing services
should use.

Upstream `acme-companion` still accepts the older `LETSENCRYPT_*` aliases
(`LETSENCRYPT_HOST`, `LETSENCRYPT_EMAIL`, …) for backward compatibility, but
they are deprecated and must not be used in this project.

The one exception is the staging toggle, which has no `ACME_` spelling: it is
`LETSENCRYPT_TEST=true` (forces the Let's Encrypt **staging** CA). Do not set
both spellings of any variable on the same service.

Variant behaviour:

* **LAN / self-signed variant** — honours `GEN_SELF_SIGNED_CERT`; `ACME_HOST`
  is inert because no ACME companion is present.
* **Internet-facing variant** — honours `ACME_HOST`; `GEN_SELF_SIGNED_CERT` is
  inert because no local cert generator is present.
* **HTTP-only service** — set `VIRTUAL_HOST` (and `VIRTUAL_PORT` if needed) and
  neither TLS opt-in; it is served over plain HTTP in every variant.

Example — a TLS-capable downstream service:

```yaml
x-hosts:
  app1: &host-app1 app1.${BASE_DOMAIN}

services:
  myapp:
    image: myapp:latest
    expose:
      - "3000"            # the port VIRTUAL_PORT points at
    environment:
      # The complete contract: every applicable var is set.
      VIRTUAL_HOST: *host-app1
      VIRTUAL_PORT: "3000"   # omit only if this is the app's single exposed port
      # Declare both TLS paths; the proxy variant decides which one applies.
      ACME_HOST: *host-app1              # must also be in VIRTUAL_HOST
      # Self-signed opt-in: false by default, overridable for local testing.
      GEN_SELF_SIGNED_CERT: ${MYAPP_GEN_SELF_SIGNED_CERT:-false}
    networks:
      - web-proxy

networks:
  # Owned by the proxy cluster: same name, marked external, proxy started first.
  web-proxy:
    name: ${NGINX_PROXY_NETWORK:-web-proxy}
    external: true
```

The proxy side of a cluster must therefore: (a) consume only the opt-in that
matches its variant, (b) treat the other as a no-op rather than an error, and
(c) let a cluster move between LAN and internet without any downstream change.

## Creating a new project

1. **Nail down the requirements first** — services, public hostnames, which
   configuration must be per-deployment, and the security posture (LAN-only vs
   exposed). Write the spec down before scaffolding.
2. **Scaffold the layout**: compose file, `env.example` (+ gitignored
   `.env`), `.gitignore`, README.md, CHANGELOG.md, AGENTS.md, one service dir
   per service with Dockerfile + docker/ + README (+ entrypoint.sh only where
   rules 8-9 apply; omit for 3rd-party upstream images with no seeding needs).
3. **Write the compose file bottom-up**:
   `name:` → `x-hosts` anchors → services (each with build context, image tag
   from the version var, env, volumes, expose, proxy network) → external
   `networks:` → `volumes:`.
4. **Every env var gets a `:?...` guard**; optional vars use `${VAR:-default}`
   explicitly and are documented as optional.
5. **Write each custom entrypoint (where required by rules 8-9) as a first-run seeder + privilege drop.** Skip for 3rd-party upstream images with no seeding needs; document why in the service README.
6. **Fill in `env.example`** with every variable, a sensible example, and a
   comment for each. Group global vars at the top, then one `# --- <service> ---`
   section per service. Keep it in sync with the compose file and READMEs.
   Ensure `.env` is gitignored.
7. **Verify** (see below).

## Build / run / verify

```sh
# standard start: copy the tracked example and edit it accordingly (.env is never committed)
cp env.example .env
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

1. `docker compose config` passes with `.env` (copied from `env.example`).
2. Required vars still fail fast — no silent defaults introduced.
3. Env vars are documented in `env.example` and the service READMEs; `.env` is gitignored.
4. Where a custom entrypoint exists: seeding logic only runs on first start; existing configs are untouched. Skip for 3rd-party images with no entrypoint (reason documented in service README).
5. Where a custom entrypoint exists: `PUID`/`PGID` chown behavior preserved (no recursive chown on data dirs). Skip when there is no custom entrypoint.
6. No host `ports:` mappings — everything goes through the proxy network.
7. Hostnames come from the anchors, not copy-pasted literals.
8. No hardcoded `container_name:` in services — rely on default container naming.
9. No host bind mounts for stateful data (exceptions: config overlays like
   `vhost.d`/`conf.d` on a case-by-case basis, docker socket, special devices,
   etc.) — use standard named docker volumes for data.
10. `CHANGELOG.md` exists, follows Keep a Changelog, and reflects the change in
    its `## [Unreleased]` section.
11. `AGENTS.md` exists and summarizes the ongoing project rules only — no
    one-time scaffolding guidance.
12. Proxied services are variant-agnostic and declare their **complete**
    contract: `VIRTUAL_HOST` always, `VIRTUAL_PORT`/`VIRTUAL_PROTO` where the
    defaults do not fit, and — for HTTPS — `ACME_HOST` plus a
    `GEN_SELF_SIGNED_CERT` opt-in wired from `.env` (default `false`,
    overridable for local testing). They carry no LAN/internet knowledge; the
    proxy consumes only the opt-in matching its variant. No variable is omitted
    on the assumption that it is inert.
13. Downstream services join the proxy's external network (same
    `NGINX_PROXY_NETWORK` name, `external: true`) and are started after the
    proxy cluster; self-signed hosts use an exact, filename-safe `VIRTUAL_HOST`
    (no wildcard/regexp/port/path), since such values are silently skipped.

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
* Downstream services that hardcode variant-specific proxy config (only
  `ACME_HOST`, or hardcoding `GEN_SELF_SIGNED_CERT: "true"` instead of wiring it
  from `.env`) — takes away the deployment's choice of certificate path; declare
  both and let the self-signed opt-in default to `false` (see
  [Downstream proxy contract](#downstream-proxy-contract)).
* Omitting a contract variable because it "does not apply" — the proxy fails
  silently, not loudly: missing `VIRTUAL_HOST` means no routing, a missing TLS
  opt-in means HTTP-only / no certificate. Declare every applicable var.
* Expecting a self-signed cert for a `VIRTUAL_HOST` that cannot be a filename
  (wildcard, regexp `~^...`, `host:port`, path, `..`) — it is silently skipped
  and the host falls back to HTTP.
* Declaring the proxy network without `external: true`, with a different name
  than the cluster's `NGINX_PROXY_NETWORK`, or starting the downstream service
  before the proxy cluster — the network will not exist or will not be shared.