# Containers and Images

The previous three chapters deployed Beacon by hand: install the right .NET runtime, create
users, copy files, write systemd units, set permissions. It works, but every server must be
configured identically, upgrades of the runtime are risky, and "it works on my machine"
remains a real problem.

**Containers** package an application with everything it needs (runtime, libraries,
configuration defaults) into an **image** that runs the same way on a laptop, in CI and in
production. They're now the default unit of deployment for most server software. This
chapter explains what a container really is (it's less magical than it seems), how images
are built, and how to write good Dockerfiles for .NET.

---

## 1. The problem: environments drift

Without containers, an application's behavior depends on the machine:

- Which .NET runtime version and patch is installed?
- Which native libraries (ICU for globalization, OpenSSL versions)?
- What time zone, locale, file permissions, environment variables?
- What else is installed that might conflict?

Virtual machines solved this by shipping a whole operating system, which is heavy (GBs,
minutes to boot). Containers solve it by packaging only the application's **user-space
filesystem** and sharing the host's kernel: megabytes, and milliseconds to start.

---

## 2. The mental model: a container is an isolated process

A container is **not** a lightweight VM. It's a **normal Linux process** that the kernel
isolates using two features:

```text
 ┌──────────────────────── Host (Linux kernel) ───────────────────────┐
 │                                                                     │
 │  sshd   nginx   ┌─────────────── container ────────────────┐       │
 │                 │ dotnet Beacon.Api.dll   (PID 1 inside)    │       │
 │                 │ namespaces: own PIDs, network, mounts,    │       │
 │                 │   hostname, users                         │       │
 │                 │ cgroups: ≤ 512 MB memory, ≤ 1 CPU          │       │
 │                 │ root filesystem: from the image layers    │       │
 │                 └───────────────────────────────────────────┘       │
 └─────────────────────────────────────────────────────────────────────┘
```

- **Namespaces** give the process its own view of the system: its own process tree (it
  thinks it's PID 1), network interfaces and ports, filesystem mounts, hostname, and
  optionally user IDs.
- **Control groups (cgroups)** limit resources: memory, CPU, I/O, number of processes.
- **A root filesystem** comes from the image, so the process sees `/usr`, `/app`, etc. from
  the image, not from the host.

Run `ps aux` on the host and you'll see `dotnet Beacon.Api.dll` like any other process.
The isolation is real but thinner than a VM's: all containers share one kernel. (On macOS
and Windows, Docker Desktop runs a small Linux VM to provide that kernel.)

> **🧱 Durable:** "A container is a process with a restricted view of the system and limited
> resources" explains almost every container behavior: why PID 1 and signal handling
> matter, why containers start in milliseconds, why memory limits kill processes, and why
> kernel vulnerabilities can affect all containers on a host.

---

## 3. Images and layers

An **image** is a read-only template: a stack of **layers** (filesystem changes) plus
metadata (default command, environment, exposed ports, user).

```text
 Image: beacon-api:1.4.0
 ┌────────────────────────────────────────┐
 │ layer 5: /app (your published app)      │  ← changes with every build (small)
 │ layer 4: ASP.NET Core runtime           │  ┐
 │ layer 3: .NET runtime                   │  │ shared with every other .NET app
 │ layer 2: OS packages (ICU, CA certs)    │  │ using the same base image
 │ layer 1: Debian/Ubuntu base filesystem  │  ┘
 └────────────────────────────────────────┘
 + metadata: ENTRYPOINT ["dotnet","Beacon.Api.dll"], USER app, ENV ...
```

Properties of layers:

- **Content-addressed** (like Git objects; Book II, Chapter 1): identified by a hash of their
  contents. Identical layers are stored and downloaded once.
- **Cached**: when building, unchanged steps reuse existing layers. Order your Dockerfile so
  rarely changing things come first.
- **Immutable**: a running container adds a thin writable layer on top. When the container
  is removed, that layer is gone.

### Images vs containers

| | Image | Container |
|---|---|---|
| Analogy | A class / an executable file | An instance / a running process |
| State | Read-only | Writable top layer (temporary) |
| Lifecycle | Built, pushed, pulled | Created, started, stopped, removed |

### Registries and tags

Images are stored in **registries**: Docker Hub, GitHub Container Registry (ghcr.io), Azure
Container Registry, Microsoft's (mcr.microsoft.com). An image reference:

```text
ghcr.io/awccon/beacon-api:1.4.0
└──┬──┘ └───┬───┘ └──┬─────┘ └─┬─┘
registry  namespace  repository  tag
```

Tags are **mutable labels**: `latest` or `10.0` can point to different images over time. For
reproducible deployments, deploy by **digest** (`beacon-api@sha256:3f9a…`) or by unique,
never-reused tags (a version or the Git commit SHA).

> **⚠️ What can go wrong:** Deploying `:latest` means you don't know which version is
> running, can't roll back reliably, and different servers may run different images. Tag
> every build uniquely.

---

## 4. Dockerfiles

A **Dockerfile** is the recipe for building an image.

### A first, naïve Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /src
COPY . .
RUN dotnet publish src/Beacon.Api -c Release -o /app
WORKDIR /app
ENTRYPOINT ["dotnet", "Beacon.Api.dll"]
```

It works, and has every common problem: the image contains the whole SDK (~800 MB) and the
source code; any file change invalidates the restore cache; it runs as root.

### A good multi-stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ---- Build stage: SDK, source, compile ----
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# 1. Restore first, copying only project files: cached until dependencies change
COPY Directory.Build.props Directory.Packages.props* ./
COPY src/Beacon.Core/Beacon.Core.csproj src/Beacon.Core/
COPY src/Beacon.Infrastructure/Beacon.Infrastructure.csproj src/Beacon.Infrastructure/
COPY src/Beacon.Api/Beacon.Api.csproj src/Beacon.Api/
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet restore src/Beacon.Api/Beacon.Api.csproj

# 2. Then copy the source and publish
COPY src/ src/
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet publish src/Beacon.Api/Beacon.Api.csproj -c Release -o /app --no-restore

# ---- Runtime stage: only what's needed to run ----
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS runtime
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
USER app
ENTRYPOINT ["dotnet", "Beacon.Api.dll"]
```

What makes it good:

- **Multi-stage build**: the SDK and source exist only in the `build` stage; the final image
  contains only the runtime and the published output.
- **Layer ordering for caching**: restoring packages (slow, rarely changes) happens before
  copying source (changes constantly). A code change rebuilds only from the second `COPY`.
- **BuildKit cache mounts** keep the NuGet cache between builds.
- **Chiseled runtime image**: Microsoft's "distroless"-style Ubuntu images contain only what
  .NET needs: no shell, no package manager, a non-root user built in. Much smaller and with
  far fewer packages to have vulnerabilities (Chapter 6).
- **Non-root user** (`app`, built into .NET images since .NET 8), listening on **port 8080**
  (non-root processes can't bind below 1024).
- **Exec-form `ENTRYPOINT`** (JSON array): the app is PID 1 and receives `SIGTERM` directly,
  so graceful shutdown works (Chapter 1).

### `.dockerignore`

The **build context** is everything sent to the builder. Exclude what doesn't belong:

```text
# .dockerignore
**/bin/
**/obj/
**/node_modules/
.git/
.vs/
.idea/
**/*.user
**/appsettings.*.local.json
.env
web/dist/
tests/
```

This speeds up builds and prevents secrets and local files from leaking into images.

### Alternative: no Dockerfile at all

The .NET SDK can build container images directly:

```bash
dotnet publish src/Beacon.Api -c Release /t:PublishContainer \
  -p:ContainerRepository=beacon-api -p:ContainerImageTag=1.4.0 \
  -p:ContainerFamily=noble-chiseled
```

It produces a well-structured, non-root image with sensible defaults, without Docker
installed. Dockerfiles remain more flexible (multiple projects, custom steps, non-.NET
components) and are universal.

---

## 5. Running containers

```bash
docker build -t beacon-api:dev .
docker run --rm -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Development \
  -e ConnectionStrings__Beacon="Host=host.docker.internal;Database=beacon;Username=beacon;Password=dev" \
  --memory 512m --cpus 1 \
  --name beacon-api beacon-api:dev
```

| Flag | Meaning |
|---|---|
| `-p 8080:8080` | Publish container port 8080 on host port 8080 |
| `-e KEY=value` | Environment variables (configuration, Book III, Chapter 3) |
| `--memory`, `--cpus` | cgroup limits |
| `--rm` | Remove the container when it stops |
| `-d` | Run in the background (detached) |
| `-v` | Mount a volume or host directory (Chapter 5) |

Everyday commands:

```bash
docker ps                          # running containers
docker logs -f beacon-api          # stdout/stderr (your structured logs)
docker exec -it beacon-api sh      # a shell inside (not available in chiseled images: Chapter 6)
docker stop beacon-api             # SIGTERM, then SIGKILL after 10 s
docker inspect beacon-api          # full configuration and state
docker stats                       # live CPU/memory per container
docker image ls; docker system df  # images and disk usage
docker system prune                # clean up stopped containers, dangling images, build cache
```

### Containers are ephemeral

Anything written inside a container's filesystem disappears when the container is
replaced. Applications should be **stateless**: data goes to databases, object storage or
volumes (Chapter 5), logs go to stdout, configuration comes from environment variables and
mounted files. This is the same *twelve-factor* discipline from Book III, Chapter 3, and it's
what makes containers easy to scale, replace and roll back.

### .NET in containers: things to know

- **Memory limits**: .NET reads cgroup limits and sizes the GC heap accordingly (Book I,
  Chapter 11). DATAS (Server GC's dynamic adaptation) keeps memory usage proportional to load.
- **CPU limits**: the runtime reports a processor count based on the CPU quota, sizing the
  thread pool and Server GC heaps appropriately.
- **Globalization**: chiseled images include ICU in the `-extra` variants; the basic ones may
  run in invariant globalization mode. Check culture-dependent behavior (Book I, Chapter 2).
- **Time zones**: containers default to UTC (good for servers; Book VII, Chapter 1). If you
  need named time zones (`TimeZoneInfo.FindSystemTimeZoneById("Europe/Paris")`), make sure
  the image includes tzdata.

---

## 6. Building the SPA and BFF image

Beacon's BFF serves the built React app (Book VI, Chapter 5). A three-stage build:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS web
WORKDIR /web
RUN corepack enable
COPY web/package.json web/pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store pnpm install --frozen-lockfile
COPY web/ .
RUN pnpm build                                   # tsc + vite build → /web/dist

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY src/Beacon.Bff/Beacon.Bff.csproj src/Beacon.Bff/
RUN dotnet restore src/Beacon.Bff/Beacon.Bff.csproj
COPY src/Beacon.Bff/ src/Beacon.Bff/
RUN dotnet publish src/Beacon.Bff/Beacon.Bff.csproj -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled
WORKDIR /app
COPY --from=build /app .
COPY --from=web /web/dist ./wwwroot
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
USER app
ENTRYPOINT ["dotnet", "Beacon.Bff.dll"]
```

Node.js exists only in the first stage; the final image contains no Node, no source, no
`node_modules`, just the compiled BFF and static files.

---

## 7. In practice: containerizing Beacon

1. Add `Dockerfile.api` and `Dockerfile.bff` (sections 4 and 6) and `.dockerignore`.
2. Build and run locally:

```bash
docker build -f Dockerfile.api -t beacon-api:dev .
docker build -f Dockerfile.bff -t beacon-bff:dev .
docker image ls | grep beacon
```

   Compare sizes: the naïve SDK-based image is roughly 800 MB+; the chiseled runtime image
   for Beacon.Api is on the order of 100–150 MB (most of it the shared runtime layers).

3. Check the important properties:

```bash
docker run --rm beacon-api:dev whoami 2>/dev/null || echo "no shell utilities: chiseled"
docker inspect beacon-api:dev --format '{{.Config.User}} {{.Config.Entrypoint}}'
# app [dotnet Beacon.Api.dll]

docker run -d --name t beacon-api:dev && sleep 3 && docker stop t && docker logs t | tail -3
# "Application is shutting down..." : graceful SIGTERM handling
```

4. Push to a registry with a unique tag in CI (Book IX, Chapter 8):

```bash
docker tag beacon-api:dev ghcr.io/awccon/beacon-api:${GIT_SHA}
docker push ghcr.io/awccon/beacon-api:${GIT_SHA}
```

The same image now runs on a laptop, in E2E tests (Book VII, Chapter 3), and in production,
with only configuration differing.

---

## 8. What can go wrong

- **Huge images** with SDKs, source and build caches inside.
- **Poor layer ordering**, so every code change re-downloads all packages.
- **Running as root** in the container.
- **Shell-form `ENTRYPOINT`** (`ENTRYPOINT dotnet app.dll`): a shell becomes PID 1 and doesn't
  forward `SIGTERM`, so the app is killed without graceful shutdown.
- **Secrets baked into images** (`COPY .env`, `ARG PASSWORD`): anyone who can pull the image
  can read them, from any layer.
- **`:latest` in production.**
- **Writing state inside the container** and losing it on redeploy.
- **Missing `.dockerignore`**, sending gigabytes of context and leaking local files.

---

## 9. How an experienced engineer thinks about this

- **A container is a process**: think about PID 1, signals, users, limits.
- **Images are build artifacts**: immutable, uniquely tagged, built once, promoted through
  environments.
- **Small, minimal, non-root images** are faster and safer.
- **Stateless containers**: state lives outside.
- **Order Dockerfiles for the cache**; builds should be fast.

---

## 10. Check yourself

**Questions**

1. What are namespaces and cgroups, and what does each give a container?
2. How is a container different from a virtual machine?
3. What are image layers, and why does Dockerfile instruction order matter?
4. What does a multi-stage build achieve?
5. Why should containers run as non-root, and why does that imply port 8080?
6. Why does exec-form `ENTRYPOINT` matter for graceful shutdown?
7. Why is deploying `:latest` a problem?

**Exercises**

1. Build the naïve and multi-stage images for Beacon.Api and compare size and rebuild time
   after a one-line code change.
2. Run the container with `--memory 256m` and watch .NET's GC adapt (Book I, Chapter 11's
   counters, via `dotnet-counters` in a sidecar or the diagnostics port).
3. Build Beacon.Api's image with `dotnet publish /t:PublishContainer` and compare it to the
   Dockerfile version.
4. On the host, find the container's `dotnet` process with `ps`, and inspect its namespaces
   with `ls -l /proc/<pid>/ns`.

**Interview-style questions**

- "What's the difference between a container and a VM?"
- "How would you write a Dockerfile for a .NET application?"
- "How do you keep container images small and secure?"

---

## 11. Going deeper

- [Docker documentation: Building best practices](https://docs.docker.com/build/building/best-practices/)
- [Microsoft docs: Containerize a .NET app](https://learn.microsoft.com/dotnet/core/docker/build-container)
  and [.NET container images](https://learn.microsoft.com/dotnet/core/docker/container-images)
- Liz Rice, *Container Security* — how namespaces and cgroups work underneath.

**Next:** [Chapter 5 — Docker Compose, Networking and Volumes](05-docker-compose-networking-and-volumes.md)
runs Beacon's whole stack together.
