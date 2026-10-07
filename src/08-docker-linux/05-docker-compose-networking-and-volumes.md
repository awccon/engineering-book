# Docker Compose, Networking and Volumes

A single container is easy. Beacon is several: the API, the BFF, PostgreSQL, an identity
provider, a fake mail server for development, and possibly Redis. They need to find each
other on a network, start in the right order, persist data across restarts, and be
configured consistently. Running them with a long list of `docker run` commands is
error-prone.

**Docker Compose** describes a multi-container application in one file and runs it with
one command. This chapter covers Compose, container networking and storage, and how to
build a local development environment that works the same for every developer.

---

## 1. The problem: many containers, one application

To run Beacon locally you need, at minimum:

- PostgreSQL with a persistent volume and initialization scripts,
- an OIDC identity provider (Keycloak) with test users,
- a mail catcher (Mailpit) to see outgoing emails,
- Beacon.Api, connected to the database and the identity provider,
- Beacon.Bff, connected to the API and the identity provider, serving the SPA.

Each needs ports, environment variables, startup ordering and health checks. Writing that
down **as code** means every developer, CI run and E2E environment gets the same setup.

---

## 2. The mental model: a declarative description of services

```yaml
# compose.yaml (shape)
services:        # containers to run (each from an image or a build)
  db: ...
  api: ...
networks:        # how they connect (a default network is created automatically)
volumes:         # where persistent data lives
```

`docker compose up` reads the file, creates networks and volumes, builds or pulls images,
and starts containers in dependency order. `docker compose down` removes them (keeping
volumes unless you add `-v`).

Compose is **declarative**: you describe the desired state, and Compose makes it so, the
same model as Kubernetes manifests and infrastructure as code (Book IX, Chapter 9) at a
smaller scale.

---

## 3. Container networking

### Bridge networks and service discovery

Compose creates a **user-defined bridge network** for the project. Every service joins it,
and gets a **DNS name equal to its service name**:

```text
 ┌────────────── network: beacon_default ──────────────┐
 │  db (postgres:5432)   api (:8080)   bff (:8080)      │
 │       ▲                  ▲  │           │            │
 │       └──── "db:5432" ───┘  └─"api:8080"┘            │
 └──────────────────────────────────────────────────────┘
         host port 5001 ──► bff:8080 (published)
```

- Inside the network, the API connects to `Host=db;Port=5432`, never `localhost`.
  **`localhost` inside a container means that container itself.**
- Containers talk to each other on **container ports** (`api:8080`), with no publishing
  needed.
- **Publishing** (`ports: ["5001:8080"]`) exposes a container port on the host. Publish only
  what you need to reach from your machine.

> **⚠️ What can go wrong:** `ports: ["5432:5432"]` on a cloud VM publishes PostgreSQL on
> **all host interfaces**, and Docker's port publishing typically bypasses host firewall
> tools like `ufw` (Docker inserts its own packet-filter rules). On servers, publish to
> `127.0.0.1:5432:5432` or don't publish at all. This has exposed many databases to the
> internet.

### Reaching the host

From a container, the host machine is `host.docker.internal` (Docker Desktop; on Linux add
`extra_hosts: ["host.docker.internal:host-gateway"]`). Useful when running some services in
containers and others (like the API under a debugger) on the host.

### Other network modes

- **`host`**: the container shares the host's network stack (no isolation, no port mapping).
  Occasionally useful for performance or tooling on Linux.
- **`none`**: no networking.
- Multiple networks per service let you segment (the database only on a `backend` network
  the BFF can't reach).

---

## 4. Storage: volumes and bind mounts

A container's writable layer disappears with the container. For data that must persist:

| Type | Syntax | Managed by | Use for |
|---|---|---|---|
| **Named volume** | `pgdata:/var/lib/postgresql/data` | Docker | Database files and other persistent state |
| **Bind mount** | `./db/init:/docker-entrypoint-initdb.d:ro` | You (a host path) | Config files, init scripts, source code for hot reload |
| **tmpfs** | `tmpfs: /tmp` | Memory | Scratch data that must never touch disk |

```bash
docker volume ls
docker volume inspect beacon_pgdata
docker compose down -v          # removes volumes too: deletes the local database
```

Notes:

- **Bind mounts and permissions**: files on the host keep their host UIDs. A container
  running as a non-root user may not be able to write a bind-mounted directory. Match UIDs
  or use named volumes.
- **Read-only mounts** (`:ro`) for configuration and scripts.
- **Performance**: bind mounts on macOS and Windows go through a file-sharing layer and can
  be slow for heavy I/O (node_modules, databases). Use named volumes for those.
- **Backups**: a named volume on a laptop is fine to lose. In production, databases belong in
  managed services or have real backups (Book IV, Chapter 8).

---

## 5. Health checks and startup order

`depends_on` alone only waits for the dependency's **container to start**, not for the
service inside to be **ready**. PostgreSQL takes a few seconds to accept connections; an
API that starts first and tries to migrate or connect immediately fails.

Combine health checks with `condition: service_healthy`:

```yaml
db:
  image: postgres:18
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U beacon -d beacon"]
    interval: 5s
    timeout: 3s
    retries: 10

api:
  depends_on:
    db:
      condition: service_healthy
```

Applications should **also** tolerate dependencies being temporarily unavailable (retries
with back-off; EF Core's retrying execution strategy, Book IV, Chapter 7), because in
production, dependencies restart independently and no orchestrator guarantees ordering
forever.

For images without a shell (chiseled), health checks can't run `curl`; rely on the
orchestrator's HTTP probes in production (Book IX) or a tiny health-check binary.

---

## 6. Configuration and secrets in Compose

```yaml
api:
  environment:
    ASPNETCORE_ENVIRONMENT: Development
    ConnectionStrings__Beacon: Host=db;Database=beacon;Username=beacon;Password=${POSTGRES_PASSWORD}
  env_file:
    - .env.api                   # local overrides, not committed
```

- `${VAR}` interpolates from the shell or a `.env` file next to `compose.yaml`.
- Commit a **`.env.example`** with placeholder values; never commit real `.env` files.
- Compose also supports **secrets** mounted as files (`/run/secrets/...`), which .NET can read
  via the key-per-file configuration provider. In production, use the platform's secret
  management (Book IX, Chapter 2).

### Profiles and overrides

- **Profiles** let you start optional services only when needed:
  `docker compose --profile observability up` adds a tracing dashboard.
- **Override files**: `compose.yaml` (shared) plus `compose.override.yaml` (applied
  automatically, for local tweaks) or `-f compose.e2e.yaml` for the E2E environment.

---

## 7. Development workflows

Three common setups:

1. **Infrastructure in Compose, apps on the host**: databases and identity provider in
   containers; the API, BFF and Vite run from the IDE with debugging and hot reload. The most
   comfortable day-to-day setup.
2. **Everything in Compose**: one command starts the full system. Great for new team members,
   demos, E2E tests and reproducing production-like issues.
3. **Dev containers**: the whole development environment (SDKs, tools, extensions) defined
   as a container (`.devcontainer/`), used by VS Code, Rider and GitHub Codespaces. Every
   developer gets identical tooling.

Beacon supports the first two with the same file, using profiles.

> **🧭 When not to use Compose:** Compose is excellent for local development, CI and small
> single-host deployments. It doesn't schedule containers across machines, roll out
> updates gradually, or self-heal across hosts. For production at any scale beyond one
> server, use a managed container platform (Book IX, Chapter 3).

---

## 8. In practice: Beacon's `compose.yaml`

```yaml
# compose.yaml
name: beacon

services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: beacon
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-dev}
      POSTGRES_DB: beacon
    volumes:
      - pgdata:/var/lib/postgresql
      - ./db/init:/docker-entrypoint-initdb.d:ro         # roles, extensions (Book IV, Ch. 3)
    ports:
      - "127.0.0.1:5432:5432"                             # local tools only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U beacon -d beacon"]
      interval: 5s
      retries: 10

  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: ["start-dev", "--import-realm"]
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD:-admin}
    volumes:
      - ./dev/keycloak/beacon-realm.json:/opt/keycloak/data/import/beacon-realm.json:ro   # test users and clients
    ports:
      - "127.0.0.1:8081:8080"

  mail:
    image: axllent/mailpit
    ports:
      - "127.0.0.1:8025:8025"                             # web UI to view sent emails

  migrate:
    profiles: ["app"]
    build: { context: ., dockerfile: Dockerfile.migrator }   # runs the EF Core migration bundle
    environment:
      ConnectionStrings__Beacon: Host=db;Database=beacon;Username=beacon;Password=${POSTGRES_PASSWORD:-dev}
    depends_on:
      db: { condition: service_healthy }

  api:
    profiles: ["app"]
    build: { context: ., dockerfile: Dockerfile.api }
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Beacon: Host=db;Database=beacon;Username=beacon;Password=${POSTGRES_PASSWORD:-dev}
      Authentication__Schemes__Bearer__Authority: http://keycloak:8080/realms/beacon
      Notifications__Channel: email
      Notifications__Smtp__Host: mail
      Notifications__Smtp__Port: "1025"
    depends_on:
      migrate: { condition: service_completed_successfully }
      keycloak: { condition: service_started }

  bff:
    profiles: ["app"]
    build: { context: ., dockerfile: Dockerfile.bff }
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      Oidc__Authority: http://keycloak:8080/realms/beacon
      ReverseProxy__Clusters__api__Destinations__d1__Address: http://api:8080
    ports:
      - "127.0.0.1:5001:8080"
    depends_on:
      - api

volumes:
  pgdata:
```

Usage:

```bash
cp .env.example .env
docker compose up -d                       # infrastructure only: db, keycloak, mail
# ...run API, BFF and Vite from your IDE against localhost ports

docker compose --profile app up -d --build # the whole system in containers
docker compose logs -f api
docker compose ps
docker compose down                        # stop; keep the database volume
docker compose down -v                     # stop and reset all data
```

Notable choices:

- **Migrations run as a separate one-shot service** (`service_completed_successfully`),
  mirroring production, where migrations are a deployment step rather than app startup
  (Book IV, Chapter 7).
- **All published ports bind to `127.0.0.1`.**
- **Service names as hostnames** (`db`, `keycloak`, `mail`, `api`).
- **One file serves both workflows** via the `app` profile, and the E2E environment (Book VII,
  Chapter 3) extends it with `-f compose.e2e.yaml`.

> **⚠️ What can go wrong:** OIDC in containers has a classic trap: the browser reaches
> Keycloak at `localhost:8081`, but the BFF container reaches it at `keycloak:8080`, and
> tokens contain an `iss` (issuer) URL that must match what the BFF expects. Configure
> Keycloak's hostname so the issuer is consistent (or route both through the same
> hostname), otherwise token validation fails with an issuer mismatch.

---

## 9. What can go wrong

- **`localhost` confusion** between host and containers.
- **Publishing ports on all interfaces** on servers, bypassing the host firewall.
- **`depends_on` without health checks**, causing startup races.
- **Losing data** with `down -v`, or **keeping stale data** that hides migration problems.
- **Bind-mount permission errors** with non-root containers.
- **Secrets in committed `.env` files** or in `compose.yaml`.
- **Using Compose as a production orchestrator** beyond what it's designed for.

---

## 10. How an experienced engineer thinks about this

- **The development environment is code**: versioned, reviewed, reproducible.
- **Service names are the network's DNS**; `localhost` is per container.
- **Persist only what must persist**, in named volumes; everything else is disposable.
- **Wait for readiness, and tolerate unavailability anyway.**
- **Mirror production's shape** (migrations as a step, same images, same configuration
  keys) so local runs catch real problems.

---

## 11. Check yourself

**Questions**

1. How do containers in a Compose project find each other?
2. What does `localhost` mean inside a container?
3. What's the difference between a named volume and a bind mount?
4. Why isn't `depends_on` enough for startup ordering, and what fixes it?
5. Why can publishing a port on a server be dangerous even with `ufw` enabled?
6. What are Compose profiles useful for?
7. When should you not use Compose?

**Exercises**

1. Create Beacon's `compose.yaml`, start the infrastructure, and run the API from your IDE
   against it.
2. Start the full stack with `--profile app` and run the E2E suite against it.
3. Add a `backend` network so only the API (not the BFF) can reach the database, and verify
   with `nc` from each container.
4. Add an optional `observability` profile with a tracing UI (Jaeger or the Aspire dashboard)
   for Book IX, Chapter 6.

**Interview-style questions**

- "How do you set up a local development environment for a multi-service application?"
- "How do containers communicate with each other in Docker?"
- "How do you persist data with containers?"

---

## 12. Going deeper

- [Docker Compose documentation](https://docs.docker.com/compose/)
- [Docker networking overview](https://docs.docker.com/engine/network/)
- [Development Containers specification](https://containers.dev/)

**Next:** [Chapter 6 — Production Containers and Security](06-production-containers-and-security.md)
hardens Beacon's images and shows how to investigate containers that misbehave.
