# Compute: App Service, Functions and Containers

Azure offers a dozen ways to run code: virtual machines, App Service, Container Apps, AKS,
Functions, Container Instances, Static Web Apps, Spring Apps, Batch and more. Picking one
isn't a matter of which is "best"; it's a matter of which trade-offs fit your workload, your
team and your budget. Picking wrong is expensive to undo: you'll fight the platform's limits
or pay for operational complexity you didn't need.

This chapter explains the main compute options, how they differ underneath, and how to
choose. Then it deploys Beacon to Azure Container Apps.

---

## 1. The problem: matching workloads to platforms

Beacon has several kinds of workload, with different needs:

| Workload | Characteristics |
|---|---|
| Beacon.Bff (SPA + BFF) | Always-on HTTP, sessions, low latency, scales with users |
| Beacon.Api | Always-on HTTP, WebSockets (SignalR), scales with load |
| Outbox and indexing workers | Background, continuous, should scale with queue depth |
| SLA monitor | Periodic, every minute, one instance |
| Nightly retention job | Scheduled batch, runs minutes, then nothing |
| Export generation | Bursty, CPU-heavy, occasional |
| Migrations | One-shot per deployment |

A single platform can host all of them, but the cost and operational fit differ.

---

## 2. The mental model: the compute spectrum

```text
 More control, more operations                                    Less control, less operations
 ◄──────────────────────────────────────────────────────────────────────────────────────────────►
 Virtual Machines    AKS (Kubernetes)    Container Apps    App Service    Functions
 (you run the OS)    (you run the        (serverless       (managed web    (event-driven,
                      cluster's apps)     containers)       app hosting)    per-execution)
```

| Platform | You deploy | Scaling | Scale to zero | Best for |
|---|---|---|---|---|
| **Virtual Machines** | An OS image + your software | VM scale sets, manual | No | Legacy software, full OS control, special requirements |
| **AKS** | Containers + Kubernetes manifests | Pods and nodes; very flexible | Nodes: limited | Large platforms, many services, teams with Kubernetes expertise |
| **Container Apps** | Container images | HTTP, events, queues (KEDA) | **Yes** | Microservices, APIs, workers, jobs, without managing Kubernetes |
| **App Service** | Code or a container | Instances per plan, autoscale rules | No (plan always runs) | Classic web apps and APIs; simplest path for .NET |
| **Functions** | Functions (code) | Per event, automatic | **Yes** (consumption plans) | Event-driven glue, small APIs, scheduled tasks, bursty work |

### Azure App Service

The long-standing PaaS for web apps. You create an **App Service plan** (a set of VMs of a
given size, which you pay for whether used or not), and deploy one or more apps onto it as
code (`zip deploy`) or containers.

- ✓ Very mature, simple, excellent .NET support, deployment slots (blue-green; Chapter 9),
  built-in authentication, custom domains and managed certificates, VNet integration.
- ✗ You pay for the plan continuously; scaling is per plan instance; less suited to
  event-driven or many small services.

### Azure Functions

Write individual functions triggered by events: HTTP requests, timers, queue messages, blob
uploads, database changes. The platform runs and scales them.

```csharp
public sealed class RetentionJob(BeaconDbContext db, ILogger<RetentionJob> log)
{
    [Function("NightlyRetention")]
    public async Task Run([TimerTrigger("0 30 2 * * *")] TimerInfo timer, CancellationToken ct)
    {
        var anonymized = await db.Tickets
            .Where(t => t.Status == TicketStatus.Closed && t.ResolvedAt < DateTimeOffset.UtcNow.AddYears(-3))
            .Take(1000)
            .ExecuteUpdateAsync(s => s.SetProperty(t => t.Description, (string?)null), ct);
        log.LogInformation("Anonymized {Count} tickets", anonymized);
    }
}
```

- **Hosting plans**: **Flex Consumption** (pay per execution, scale to zero, fast scaling, VNet
  support; the recommended serverless plan), **Premium** (pre-warmed instances, no cold
  starts), **Dedicated** (on an App Service plan), and Container Apps hosting.
- **Isolated worker model**: .NET functions run in a separate process with normal
  dependency injection, middleware and the latest .NET versions.
- **Durable Functions**: orchestrations (workflows with state, retries, fan-out/fan-in, human
  approval steps) written as code.

- ✓ Pay for what you use; great for event-driven work and bursty load; bindings reduce
  boilerplate.
- ✗ **Cold starts** on consumption plans; execution time limits; debugging and local testing
  are more involved; architecture fragments into many small pieces if overused.

### Azure Container Apps

A serverless container platform built on Kubernetes and KEDA, without exposing Kubernetes:

- **Apps**: long-running containers with HTTP ingress (including WebSockets), revisions,
  traffic splitting between revisions (canary; Chapter 9).
- **Scaling rules**: by HTTP concurrency, CPU/memory, or **events** (queue length, Service Bus,
  Kafka, cron) via KEDA scalers; **scale to zero** when idle.
- **Jobs**: run-to-completion containers triggered manually, on a schedule, or by events.
- **Environments**: a shared boundary (network, logging) for a group of apps that can call
  each other by name.
- **Dapr** integration (optional) for service invocation, pub/sub and state.

- ✓ Runs the container images from Book VIII unchanged; scales precisely; scale to zero for
  non-production; no cluster to manage.
- ✗ Less control than AKS; some Kubernetes features unavailable; newer than App Service.

### AKS (Azure Kubernetes Service)

Managed Kubernetes: Azure runs the control plane; you manage node pools, cluster upgrades,
networking choices, ingress, policies, and the Kubernetes manifests for every app.

> **🧭 When not to use Kubernetes:** Kubernetes is a powerful platform for building platforms.
> For a team running a handful of services, it adds a large surface area (cluster upgrades,
> networking, RBAC, manifests, Helm, operators) that someone must master and maintain. Use a
> higher-level service (Container Apps, App Service) until you need Kubernetes' flexibility,
> its ecosystem, or portability across clouds, and have people to run it.

---

## 3. Choosing

A practical decision path:

```text
 Is it a containerized app or several services?   ──yes──► Container Apps
        │ no                                                (AKS if you need full Kubernetes)
        ▼
 Is it a single web app/API in .NET, always on?   ──yes──► App Service
        │ no
        ▼
 Is it event-driven, scheduled or bursty glue?    ──yes──► Functions (Flex Consumption)
        │ no
        ▼
 Does it need OS-level control or legacy software? ─yes──► Virtual Machines
```

Other factors:

- **Team skills**: App Service is the gentlest learning curve for .NET teams; Kubernetes the
  steepest.
- **Cost profile**: always-on steady traffic suits plans with fixed capacity; spiky or idle
  workloads suit scale-to-zero.
- **Cold starts**: scale-to-zero means the first request after idle waits for a container or
  function to start. Native AOT and minimum replicas (≥ 1 in production) mitigate it.
- **Portability**: containers on Container Apps can move to any container platform with
  little change; Functions code is more Azure-specific.

### Static content

The SPA's static files can be served by the BFF (Beacon's choice, for a single origin and
cookie security), or by **Azure Static Web Apps** or **Blob Storage + Front Door** for
purely static sites with a separate API. A CDN in front (Front Door, Chapter 5) caches the
hashed assets close to users either way.

---

## 4. Platform concerns you still own

Whatever you choose:

- **Health probes**: configure liveness/readiness/startup probes to your endpoints (Book III,
  Chapter 10), with timeouts that suit your startup time.
- **Graceful shutdown**: platforms send `SIGTERM` and wait (Container Apps and Kubernetes: 30
  seconds by default). Book VIII, Chapter 6's settings apply.
- **Instance count**: production should run **at least two replicas** across zones, so a
  single instance restart doesn't cause downtime.
- **Statelessness**: no local files or in-memory session state that matters (Book VIII,
  Chapter 4). Data Protection keys in shared storage.
- **Configuration and secrets** from environment and Key Vault references (Chapter 2).
- **Resource requests and limits** sized from real measurements.

---

## 5. In practice: Beacon on Azure Container Apps

Beacon already has container images (Book VIII), several services and background workers,
and benefits from scale-to-zero in non-production. **Azure Container Apps** fits.

### The layout

```text
 Container Apps environment: cae-beacon-prod-weu  (VNet-integrated, zone-redundant)
 ├─ ca-beacon-bff     ingress: external (behind Front Door), min 2 / max 10 replicas, HTTP scaling
 ├─ ca-beacon-api     ingress: internal only (reachable by the BFF), min 2 / max 20, HTTP scaling
 ├─ ca-beacon-worker  no ingress; outbox + indexing workers; min 1 / max 5
 ├─ job: beacon-migrate     manual trigger (run by the pipeline before each deployment)
 ├─ job: beacon-retention   scheduled: 30 2 * * *
 └─ (SLA monitor runs in ca-beacon-worker with a distributed lock: Book III, Chapter 8)
```

Design decisions:

- **The API has internal ingress only**: it's reachable from the BFF within the environment,
  not from the internet. (Partner integrations later get a separate external entry through
  an API gateway.)
- **Workers are split from the API**, so heavy background processing doesn't steal capacity
  from request handling, and they scale independently.
- **Migrations are a job** run by the pipeline before the new revision receives traffic (Book
  IV, Chapter 7).
- **Minimum two replicas** for user-facing apps in production; non-production scales to zero.

### Creating it (CLI preview; Chapter 9 does it with Bicep)

```bash
az containerapp env create -g rg-beacon-prod-weu -n cae-beacon-prod-weu -l westeurope \
  --infrastructure-subnet-resource-id $SUBNET_ID --zone-redundant \
  --logs-workspace-id $LAW_ID --logs-workspace-key $LAW_KEY

az containerapp create -g rg-beacon-prod-weu -n ca-beacon-api \
  --environment cae-beacon-prod-weu \
  --image crbeaconprod.azurecr.io/beacon-api:$GIT_SHA \
  --user-assigned $API_IDENTITY_ID --registry-server crbeaconprod.azurecr.io --registry-identity $API_IDENTITY_ID \
  --ingress internal --target-port 8080 --transport auto \
  --min-replicas 2 --max-replicas 20 \
  --scale-rule-name http --scale-rule-type http --scale-rule-http-concurrency 50 \
  --cpu 1.0 --memory 2Gi \
  --env-vars ASPNETCORE_ENVIRONMENT=Production AZURE_CLIENT_ID=$API_CLIENT_ID \
             KeyVault__Uri=https://kv-beacon-prod-weu.vault.azure.net/ \
             ConnectionStrings__Beacon="Host=psql-beacon-prod-weu.postgres.database.azure.com;Database=beacon;Username=id-beacon-api-prod;Ssl Mode=VerifyFull"
```

The image is pulled with the managed identity (AcrPull); configuration comes from environment
variables and Key Vault; the database connection string contains **no password** (Chapter 2).

### Probes

```yaml
# (excerpt of the container app's template, as in Bicep/YAML)
probes:
  - type: Startup
    httpGet: { path: /health/live, port: 8080 }
    periodSeconds: 5
    failureThreshold: 30          # up to 150 s to start (cold JIT, Key Vault load)
  - type: Liveness
    httpGet: { path: /health/live, port: 8080 }
    periodSeconds: 10
  - type: Readiness
    httpGet: { path: /health/ready, port: 8080 }
    periodSeconds: 5
```

### SignalR at scale

With several API replicas, SignalR needs a backplane (Book III, Chapter 8). Options on Azure:
**Azure SignalR Service** (offloads connection handling entirely; the API becomes stateless
for WebSockets) or a Redis backplane. Beacon uses Azure SignalR Service in production with
one line (`builder.Services.AddSignalR().AddAzureSignalR()`), authenticated via managed
identity.

### A function for exports

Ticket exports are bursty and CPU-heavy. Rather than sizing the API for them, the API
enqueues an export request (Book III, Chapter 8's async request-reply), and a **Container
Apps job triggered by queue messages** (or an Azure Function on Flex Consumption) generates
the file into Blob Storage. It scales from zero when exports are requested and back to zero
afterwards.

---

## 6. What can go wrong

- **Choosing Kubernetes by default** and spending months operating it.
- **One instance in production**: every deployment or platform maintenance becomes downtime.
- **Probes misconfigured**: liveness too aggressive (restart loops during slow starts) or
  checking the database (restart storms during a database blip).
- **Cold starts** surprising users after idle periods; set minimum replicas for latency-sensitive
  production apps.
- **Running background work in the API's request path** instead of workers or jobs.
- **Function sprawl**: dozens of tiny functions with no clear structure.
- **App Service plans sized for peak, running 24/7** in non-production.

---

## 7. How an experienced engineer thinks about this

- **Match the platform to the workload's shape**: always-on, event-driven, scheduled, batch.
- **Prefer the highest-level platform that fits**; Kubernetes is a choice with a team cost.
- **Separate workloads with different scaling needs** (API vs workers vs jobs).
- **At least two replicas across zones** for anything user-facing in production.
- **Containers keep options open**: the same image runs locally, on Container Apps, on AKS or
  elsewhere.

---

## 8. Check yourself

**Questions**

1. Compare App Service, Functions, Container Apps and AKS. When is each the best fit?
2. What's a cold start, and how do you mitigate it?
3. What does "scale to zero" save, and what does it cost?
4. Why should the API have internal-only ingress in Beacon's design?
5. Why separate workers from the API?
6. What's the difference between liveness, readiness and startup probes?
7. When is Kubernetes worth its complexity?

**Exercises**

1. Deploy Beacon.Api to Azure Container Apps from your registry with a managed identity and
   internal ingress; call it from a second app in the same environment.
2. Create a scheduled Container Apps job for the retention task and check its execution history.
3. Load-test the API (Azure Load Testing or k6) and watch replicas scale out and in.
4. Deploy the same API to App Service and compare the setup, scaling and cost.

**Interview-style questions**

- "How would you choose between App Service, Container Apps, Functions and AKS?"
- "How do you run scheduled and background jobs in the cloud?"
- "How would you scale a WebSocket-based application across multiple instances?"

---

## 9. Going deeper

- [Microsoft docs: Choose an Azure compute service](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Azure Container Apps documentation](https://learn.microsoft.com/azure/container-apps/)
- [Azure Functions documentation](https://learn.microsoft.com/azure/azure-functions/)
- [Azure App Service documentation](https://learn.microsoft.com/azure/app-service/)

**Next:** [Chapter 4 — Data and Storage](04-data-and-storage.md) moves Beacon's data to
managed services.
