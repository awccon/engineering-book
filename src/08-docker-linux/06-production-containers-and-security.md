# Production Containers and Security

A container that works on your laptop isn't necessarily ready for production. Production
images must be small and patched, run with minimal privileges, behave correctly under
resource limits, shut down gracefully, expose health endpoints, and be traceable back to
the exact source that built them. And when a container misbehaves (restarts in a loop,
gets killed for memory, responds slowly), you need techniques to investigate a minimal
image that may not even contain a shell.

This chapter covers container security and production readiness, then works through an
investigation playbook.

---

## 1. The problem: containers concentrate risk

Containers make deployment easy, which also makes it easy to ship problems widely:

- A base image with a vulnerable OpenSSL is copied into every service built on it.
- A container running as root with a writable filesystem gives an attacker who exploits the
  app a comfortable foothold.
- An image pulled by tag from a public registry can change underneath you, or be a
  typosquatted malicious image.
- Shared kernels mean a container escape vulnerability affects the whole host.

Container security is mostly **supply chain** (what's in the image and where it came from)
and **runtime** (what the running container is allowed to do).

---

## 2. The mental model: layers of defense

```text
 Supply chain                         Runtime
 ───────────────────────────────      ───────────────────────────────────────
 trusted, minimal base image          non-root user
 pinned and scanned dependencies      read-only root filesystem
 reproducible builds in CI            dropped Linux capabilities
 signed images, SBOM, provenance      no privilege escalation
 private registry, deploy by digest   resource limits (memory, CPU, PIDs)
 regular rebuilds for patches         network policies, minimal exposure
                                      secrets from the platform, not the image
```

No single control is sufficient; together they make exploitation hard and limit the damage
when it happens.

---

## 3. Minimal images

The fewer packages in an image, the fewer vulnerabilities and the less an attacker can use.

| Base | Contents | Size (approx.) | Shell? |
|---|---|---|---|
| `aspnet:10.0` (Ubuntu/Debian) | Full OS userland + .NET | ~220 MB | Yes |
| `aspnet:10.0-alpine` | musl-based minimal OS + .NET | ~110 MB | Yes |
| `aspnet:10.0-noble-chiseled` | Only .NET's dependencies; non-root by default | ~110 MB | **No** |
| `runtime-deps:10.0-noble-chiseled` + self-contained or Native AOT app | Only native dependencies | ~10–40 MB + app | **No** |

**Chiseled** (distroless-style) images have no shell, no package manager and no common
utilities. That eliminates whole classes of attack ("download a tool and run it") and most
scanner findings. The trade-off is debuggability (section 7).

Native AOT (Book I, Chapter 1) on `runtime-deps` chiseled images produces the smallest,
fastest-starting images, if your dependencies support AOT.

---

## 4. Vulnerability scanning and patching

### Scanning

Scanners compare the packages in an image against vulnerability databases (CVEs):

```bash
docker scout cves ghcr.io/awccon/beacon-api:${GIT_SHA}
trivy image ghcr.io/awccon/beacon-api:${GIT_SHA}
```

Run scans in CI (fail on high/critical vulnerabilities with available fixes) and
continuously against images in your registry, because new CVEs are published about
yesterday's images every day.

### Patching means rebuilding

A container image never updates itself. When .NET ships its monthly security patch or the
base OS fixes a vulnerability, you must **rebuild and redeploy**. Practical setup:

- Use **floating minor tags** for base images in Dockerfiles (`aspnet:10.0-noble-chiseled`),
  so a rebuild picks up patches; record the resolved digest in build metadata.
- **Rebuild on a schedule** (weekly) and when base images change (Dependabot and Renovate can
  update `FROM` lines and pinned digests).
- **Automate deployment** so a patch rollout is routine, not a project.

> **⚠️ What can go wrong:** Self-contained apps and containers both mean **you** own runtime
> patching (Book I, Chapter 1). Teams that build an image once and run it for a year are
> running a year of unpatched vulnerabilities.

### Software supply chain

- **SBOM** (Software Bill of Materials): a list of everything in the image, generated at
  build time (`docker buildx build --sbom=true`, Syft). Lets you answer "are we affected by
  CVE-X?" in minutes.
- **Provenance and signing**: record how and where an image was built (SLSA provenance) and
  sign it (Sigstore/cosign, Notation). Platforms can refuse to run unsigned images.
- **Trusted sources**: official images from verified publishers, mirrored into your own
  registry; avoid random public images in production.
- **Deploy by digest** so what you tested is exactly what runs.

---

## 5. Runtime hardening

### Non-root and read-only

```bash
docker run \
  --user app \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 512m --cpus 1 --pids-limit 200 \
  ghcr.io/awccon/beacon-api:${GIT_SHA}
```

| Setting | Protects against |
|---|---|
| Non-root user | Many exploits need root; limits damage to the container's files |
| `--read-only` root filesystem | Attackers can't modify binaries or drop tools; mount `tmpfs` for the few writable paths |
| `--cap-drop ALL` | Removes Linux capabilities (raw sockets, changing file ownership, etc.) the app doesn't need |
| `no-new-privileges` | Prevents gaining privileges via setuid binaries |
| Memory, CPU, PID limits | One misbehaving container can't starve the host or fork-bomb it |

Never run production containers with `--privileged`, never mount the Docker socket
(`/var/run/docker.sock`) into an application container (it's equivalent to root on the host),
and avoid host networking and host path mounts.

ASP.NET Core needs a writable location for a few things (Data Protection keys if not stored
externally, temp files for large uploads). Configure them explicitly to a `tmpfs` or volume,
or to external storage (Book IX).

### Secrets

- Never `COPY` secrets into images or pass them as build `ARG`s (they persist in image
  history). For build-time secrets (private NuGet feeds), use BuildKit secret mounts:
  `RUN --mount=type=secret,id=nuget ...`.
- At runtime, get secrets from the platform: environment variables injected by the
  orchestrator, mounted secret files, or better, a managed identity plus Key Vault (Book IX,
  Chapter 2).

---

## 6. Production readiness for .NET containers

A checklist beyond security:

- **Graceful shutdown**: exec-form entrypoint (PID 1 receives `SIGTERM`); `HostOptions.ShutdownTimeout`
  shorter than the orchestrator's grace period (Kubernetes defaults to 30 s).
- **Health endpoints**: `/health/live` and `/health/ready` (Book III, Chapter 10), used by the
  orchestrator's probes.
- **Logs to stdout/stderr** in JSON (Book III, Chapter 3); the platform collects them.
- **Telemetry** via OpenTelemetry (Book IX, Chapter 6).
- **Configuration only from the environment**; the same image in every environment.
- **Resource limits set**, and the app tested under them (GC heap sizing, thread pool).
- **Data Protection keys persisted** to shared storage when running multiple instances, or
  cookies and antiforgery tokens fail across instances.
- **Labels for traceability**:

```dockerfile
ARG GIT_SHA
LABEL org.opencontainers.image.source="https://github.com/awccon/beacon" \
      org.opencontainers.image.revision="${GIT_SHA}"
```

  Any running container can be traced to its exact commit.

- **Version endpoint or header** exposing the build version, to answer "what's deployed?"
  instantly.

---

## 7. Investigating a misbehaving container

> **🔍 Investigation: a container keeps restarting.**
> 1. **What's the state and exit code?**
>    `docker ps -a` / `docker inspect <id> --format '{{.State.Status}} {{.State.ExitCode}} {{.State.OOMKilled}}'`.
>    - Exit **137** with `OOMKilled=true`: hit the memory limit (section 8).
>    - Exit **139**: segmentation fault (native code crash).
>    - Exit **1** (or another non-zero code): the app exited with an error. Read the logs.
>    - Exit **0**, restarting: the process finished (wrong entrypoint, a one-shot command).
> 2. **Read the logs**, including the previous run: `docker logs --tail 200 <id>`
>    (Kubernetes: `kubectl logs --previous`). Configuration validation failures
>    (`ValidateOnStart`, Book III, Chapter 3) show up here immediately.
> 3. **Check the configuration it received**: `docker inspect` shows environment variables
>    (and therefore any secrets passed that way, another reason to prefer mounted secrets).
> 4. **Check health probes**: an orchestrator restarts containers that fail liveness checks.
>    Is the liveness endpoint depending on the database (Book III, Chapter 10)?
> 5. **Reproduce locally** with the same image digest and configuration.

> **🔍 Investigation: inside a chiseled container with no shell.**
> You can't `docker exec -it ... sh` into a chiseled image. Instead:
> - **Debug sidecar sharing the process namespace**: `docker run -it --rm --pid=container:<id>
>   --network=container:<id> mcr.microsoft.com/dotnet/sdk:10.0 bash`. From there, `ps`
>   sees the app's processes, `curl localhost:8080` hits the app, and the .NET diagnostic
>   tools can attach (`dotnet-counters`, `dotnet-dump`), provided they can reach the app's
>   diagnostic socket in `/tmp` (share it via a volume, or set `DOTNET_DiagnosticPorts`).
> - **Kubernetes**: `kubectl debug -it <pod> --image=... --target=<container>` creates an
>   ephemeral debug container for exactly this purpose.
> - **Copy files out**: `docker cp <id>:/app/appsettings.json .`
> - **Run a debug variant**: the same app on a non-chiseled base image in a non-production
>   environment.

> **🔍 Investigation: the container is slow.**
> - `docker stats`: is it at its CPU limit? CPU throttling under a quota makes latency spike
>   even when the host is idle.
> - `dotnet-counters` from a debug sidecar: thread pool queue length, GC % time, allocation
>   rate (Book I, Chapters 9 and 11).
> - Network: DNS resolution inside the container, connection reuse to dependencies.

---

## 8. Memory limits and .NET

When a container exceeds its memory limit, the kernel's OOM killer terminates it
(`SIGKILL`, exit 137), with no exception, no log line from your app, nothing graceful.

How .NET behaves:

- The GC reads the cgroup limit and, by default, limits the heap to **75% of it**
  (`GCHeapHardLimitPercent`), leaving room for native memory, thread stacks and the runtime.
- Server GC with DATAS adapts the number of heaps to load, keeping memory proportional.
- **Native memory** (large buffers from native libraries, unmanaged allocations, many
  threads) isn't covered by the GC limit.

Diagnosing OOM kills:

1. Confirm: `OOMKilled: true`, exit 137, kernel log (`dmesg`) entries.
2. Watch memory over time (`docker stats`, platform metrics): steady growth suggests a leak
   (Book I, Chapter 11's techniques with `dotnet-gcdump`); spikes suggest a large request
   (an export loading everything into memory, an unbounded query).
3. Fix the cause, then right-size the limit with headroom. Raising the limit alone usually
   just delays the next kill.

---

## 9. In practice: hardening Beacon's images

Changes to `Dockerfile.api`:

```dockerfile
# syntax=docker/dockerfile:1
ARG DOTNET_VERSION=10.0

FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
# ... restore and publish as in Chapter 4 ...

FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled AS runtime
ARG GIT_SHA=unknown
LABEL org.opencontainers.image.source="https://github.com/awccon/beacon" \
      org.opencontainers.image.revision="${GIT_SHA}"
WORKDIR /app
COPY --from=build --chown=app:app /app .
ENV ASPNETCORE_URLS=http://+:8080 \
    DOTNET_gcServer=1 \
    BEACON_VERSION=${GIT_SHA}
EXPOSE 8080
USER app
ENTRYPOINT ["dotnet", "Beacon.Api.dll"]
```

Application changes:

- Data Protection keys persisted to the database (or blob storage in Book IX), so all
  instances share them.
- A `/version` endpoint returning `BEACON_VERSION`.
- Liveness checks that don't depend on the database; readiness that does.

CI additions (Book IX, Chapter 8 builds the full pipeline):

```yaml
- name: Build image
  run: docker buildx build -f Dockerfile.api --build-arg GIT_SHA=${{ github.sha }}
         --sbom=true --provenance=true -t ghcr.io/awccon/beacon-api:${{ github.sha }} --push .
- name: Scan image
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: ghcr.io/awccon/beacon-api:${{ github.sha }}
    severity: CRITICAL,HIGH
    ignore-unfixed: true
    exit-code: '1'
```

And a production run configuration (Compose on a single host, or the equivalent settings in
a container platform):

```yaml
api:
  image: ghcr.io/awccon/beacon-api@sha256:<digest>
  read_only: true
  tmpfs: [/tmp]
  cap_drop: [ALL]
  security_opt: ["no-new-privileges:true"]
  mem_limit: 512m
  cpus: 1.0
  pids_limit: 200
  restart: unless-stopped
  env_file: /etc/beacon/api.env          # root-owned, 640
```

Run `docker scout cves` or `trivy` against the result: a chiseled .NET image typically shows
very few findings, versus dozens to hundreds for a full OS base image.

---

## 10. What can go wrong

- **Bloated base images** full of unnecessary packages and vulnerabilities.
- **Never rebuilding**, so patches never arrive.
- **Root, writable, all capabilities**: the default if you don't change it.
- **Secrets in image layers or build args.**
- **Mounting the Docker socket** into containers.
- **No resource limits**, or limits so tight the app is constantly OOM-killed or throttled.
- **Liveness probes that cause restart storms.**
- **Untraceable images**: no labels, no version endpoint, deployed by mutable tag.

---

## 11. How an experienced engineer thinks about this

- **Minimize what's in the image** and what the container can do.
- **Patching is a pipeline**, not an event: scheduled rebuilds, scans, automated deploys.
- **Know your supply chain**: SBOMs, provenance, trusted registries, digests.
- **Design for investigation**: debug sidecars, diagnostic ports, versions and labels.
- **Treat exit codes and OOM kills as data**: they tell you where to look.

---

## 12. Check yourself

**Questions**

1. What are chiseled images, and what are their benefits and trade-offs?
2. Why must container images be rebuilt regularly?
3. What do `--read-only`, `--cap-drop ALL` and `no-new-privileges` protect against?
4. Why is mounting the Docker socket into a container dangerous?
5. What do exit codes 137, 139 and 1 suggest?
6. How do you debug a container that has no shell?
7. How does .NET behave under a container memory limit, and what isn't covered by it?

**Exercises**

1. Scan a full `aspnet:10.0` image and a chiseled one with Trivy and compare the findings.
2. Run Beacon.Api with the hardened settings and fix anything that breaks (writable paths,
   Data Protection keys).
3. Attach a debug sidecar to a running chiseled container and run `dotnet-counters` against
   the app.
4. Trigger an OOM kill with a deliberately memory-hungry endpoint and a 128 MB limit;
   observe the exit code and `OOMKilled` flag.

**Interview-style questions**

- "How do you secure container images and running containers?"
- "A container keeps getting restarted. How do you investigate?"
- "What's a software supply chain attack, and how do you defend against it?"

---

## 13. Going deeper

- [Microsoft docs: .NET chiseled container images](https://learn.microsoft.com/dotnet/core/docker/container-images)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [SLSA framework](https://slsa.dev/) and [Sigstore](https://www.sigstore.dev/)
- [Trivy](https://trivy.dev/) and [Docker Scout](https://docs.docker.com/scout/)

**Next:** [Chapter 7 — Troubleshooting Linux](07-troubleshooting-linux.md) is a playbook for
servers under pressure.
