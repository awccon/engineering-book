# Cloud Fundamentals

> **🔄 Current (as of October 2026):** Service names and features in Book IX reflect Azure as
> of this date. Azure renames and retires services regularly (Azure AD became Microsoft Entra
> ID in 2023, for example). The concepts map closely to AWS and Google Cloud; a comparison
> table appears in section 7.

Book VIII deployed Beacon to a server you manage: you patched the OS, renewed certificates,
configured the firewall, planned backups. The cloud offers to take much of that work off your
hands, in exchange for money, a new set of concepts, and a different set of things that can
go wrong.

This chapter builds the mental model for cloud computing: what the cloud actually is, the
spectrum of service models, how regions and availability work, the shared responsibility
model, and how Azure is organized. The rest of Book IX moves Beacon onto Azure piece by piece.

---

## 1. The problem: running infrastructure is undifferentiated work

Everything in Book VIII was necessary, and none of it is what makes Beacon valuable to its
users. Patching kernels, replacing failed disks, scaling servers before peak season,
configuring backups for PostgreSQL: every company does this, and doing it well requires
specialized, expensive, 24/7 effort.

Cloud providers do this work at enormous scale and rent it out. The question for every
component becomes: **should we operate this ourselves, or rent it as a service?**

---

## 2. The mental model: the responsibility spectrum

```text
 You manage:  ████████████████████████  ████████████████  ██████████  ████
              On-premises               IaaS              PaaS        SaaS
              (your data center)        (VMs)             (managed     (complete
                                                           platforms)   applications)
 Application      you                    you               you          provider
 Data             you                    you               you          you (mostly)
 Runtime          you                    you               provider     provider
 OS / patching    you                    you               provider     provider
 Virtualization   you                    provider          provider     provider
 Hardware         you                    provider          provider     provider
 Facility         you                    provider          provider     provider
```

| Model | Examples | You get | You give up |
|---|---|---|---|
| **IaaS** (Infrastructure as a Service) | Virtual machines, disks, virtual networks | Full control, any software | Patching, scaling, availability are yours |
| **CaaS** (Containers as a Service) | Azure Container Apps, AKS, AWS ECS | Run your container images; scaling and placement managed | Some control over the host |
| **PaaS** (Platform as a Service) | App Service, managed PostgreSQL, Key Vault | Deploy code or use a service; the platform runs it | Limits on configuration and versions |
| **Serverless / FaaS** | Azure Functions, AWS Lambda | Pay per execution; scale to zero | Execution limits, cold starts, platform coupling |
| **SaaS** | Microsoft 365, GitHub, Entra ID | A finished product | Nearly all control |

The further right you go, the less operational work you do, and the more you depend on the
provider's choices, limits and pricing.

> **🧱 Durable:** Default to the most managed option that meets your requirements, and move
> left only for a concrete reason (a capability, a cost at scale, a compliance requirement).
> Your team's time spent operating infrastructure is a cost too, usually the largest one.

---

## 3. Cloud economics

The cloud changes cost structure from **capital expenditure** (buying servers) to
**operational expenditure** (paying for usage):

- **Pay as you go**: per second, per GB, per request, per operation.
- **Elasticity**: scale up for peak hours, down at night; scale to zero for idle
  environments.
- **No upfront investment**, but **no automatic savings** either: idle resources cost money
  every hour, and the bill grows quietly.
- **Commitment discounts** (reservations, savings plans) trade flexibility for 30–60% lower
  prices on steady workloads.

The cloud is not automatically cheaper. A steady workload on a few VMs can cost less in your
own rack or on a simple VPS. The cloud wins on elasticity, managed services that replace
staff time, global reach and speed of experimentation. Chapter 7 covers cost management.

---

## 4. Regions, availability zones and failure domains

Cloud infrastructure is physically organized into nested failure domains:

```text
 Geography (e.g. Europe)
 └─ Region (e.g. West Europe, Netherlands)                 ← a set of nearby data centers
     ├─ Availability Zone 1  (separate power, cooling, network)
     ├─ Availability Zone 2
     └─ Availability Zone 3
 Paired region (e.g. North Europe, Ireland)                ← for regional disaster recovery
```

| Failure | Protection |
|---|---|
| A server or disk fails | Platform redundancy; multiple instances |
| A data center (zone) fails | **Zone-redundant** deployment: instances spread across zones |
| A whole region fails or is unreachable | **Multi-region** deployment with failover (expensive and complex) |

Choosing a region involves **latency** (close to users), **data residency** (regulations on
where data may be stored), **service availability** (not every service exists in every
region), **cost** (prices vary by region) and **zone support**.

Most applications should be **zone-redundant within one region**: it protects against the
most common large failures at modest cost. Multi-region active-active is justified only by
strict availability requirements (Chapter 7).

---

## 5. The shared responsibility model

Security and reliability are **shared** between you and the provider, and the split depends
on the service model:

- The provider secures the **physical infrastructure, hypervisors and managed service
  internals**.
- You always own **your data, identities and access, configuration, and application code**.

Most cloud security incidents are not provider failures. They're customer misconfigurations:
a storage container made public, a database open to the internet, an over-privileged
identity, a leaked access key. The cloud gives you powerful tools; it doesn't configure them
for you.

> **⚠️ What can go wrong:** "It's managed, so it's secure" is a dangerous assumption.
> Managed PostgreSQL is patched by Azure, but if you allow public network access with a weak
> password, it's as exposed as a database on your own VM (Book IV, Chapter 8).

---

## 6. How Azure is organized

### The resource hierarchy

```text
 Microsoft Entra tenant (identity boundary: users, groups, app registrations)
 └─ Management groups (optional: apply policy across subscriptions)
     └─ Subscriptions (billing and access boundary; e.g. beacon-prod, beacon-nonprod)
         └─ Resource groups (lifecycle containers; e.g. rg-beacon-prod-weu)
             └─ Resources (an App Service, a database, a Key Vault, ...)
```

- **Tenant**: your organization's identity directory (Chapter 2).
- **Subscription**: a billing account and a security boundary. Separating production and
  non-production into different subscriptions limits blast radius and simplifies access
  control.
- **Resource group**: resources that share a lifecycle (deployed and deleted together). One
  per application per environment is a common pattern.
- **Resources**: everything is a resource with an ID:
  `/subscriptions/<id>/resourceGroups/rg-beacon-prod-weu/providers/Microsoft.Web/sites/app-beacon-api-prod`.

### Azure Resource Manager

Every action (portal click, CLI command, SDK call, infrastructure template) goes through
**Azure Resource Manager (ARM)**, which authenticates, authorizes (RBAC, Chapter 2), applies
policies, and logs the operation. Consequences:

- **Everything is an API call**, so everything can be automated (Chapter 9).
- **Everything is logged** in the activity log: who changed what, when.
- **Policies** can enforce rules org-wide ("no public storage," "only these regions,"
  "resources must have an owner tag").

### Naming and tagging

Adopt a convention early (Microsoft's Cloud Adoption Framework suggests one):

```text
<resource-type>-<workload>-<environment>-<region>[-<instance>]
rg-beacon-prod-weu          resource group
app-beacon-api-prod-weu     app
psql-beacon-prod-weu        PostgreSQL server
kv-beacon-prod-weu          Key Vault (globally unique: may need a suffix)
```

And tags for cost allocation and ownership: `env=prod`, `app=beacon`, `owner=support-platform`,
`costCenter=...`.

### Tools

```bash
az login
az account set --subscription beacon-nonprod
az group create -n rg-beacon-dev-weu -l westeurope --tags app=beacon env=dev
az resource list -g rg-beacon-dev-weu -o table
```

The **portal** is good for exploring and learning; the **CLI** (`az`) and **infrastructure as
code** (Bicep, Terraform; Chapter 9) are how real environments are built, so they're
reproducible and reviewable.

---

## 7. Mapping concepts across clouds

| Concept | Azure | AWS | Google Cloud |
|---|---|---|---|
| Identity directory | Microsoft Entra ID | IAM Identity Center | Cloud Identity |
| Permissions | Azure RBAC | IAM policies | IAM |
| Billing/isolation unit | Subscription | Account | Project |
| Grouping | Resource group | (tags, CloudFormation stacks) | (labels, projects) |
| VMs | Virtual Machines | EC2 | Compute Engine |
| Managed web apps | App Service | Elastic Beanstalk / App Runner | App Engine / Cloud Run |
| Serverless containers | Container Apps | App Runner / ECS Fargate | Cloud Run |
| Kubernetes | AKS | EKS | GKE |
| Functions | Azure Functions | Lambda | Cloud Run functions |
| Object storage | Blob Storage | S3 | Cloud Storage |
| Managed PostgreSQL | Azure Database for PostgreSQL | RDS / Aurora | Cloud SQL / AlloyDB |
| Secrets | Key Vault | Secrets Manager / KMS | Secret Manager / KMS |
| Monitoring | Azure Monitor / Application Insights | CloudWatch / X-Ray | Cloud Monitoring / Trace |
| Infrastructure as code | Bicep / ARM | CloudFormation / CDK | Deployment Manager / (Terraform) |

Terraform and OpenTelemetry work across all of them.

---

## 8. In practice: planning Beacon's Azure footprint

Before creating anything, decide the shape:

**Subscriptions and environments**

- `sub-beacon-nonprod`: `dev` and `staging` resource groups, plus ephemeral preview
  environments (Book VII, Chapter 3).
- `sub-beacon-prod`: `prod` only, with stricter access.

**Region**: West Europe (customers are mostly in the EU; data residency requirement), with
zone redundancy for the database and app tier.

**Service choices** (each justified in the following chapters):

| Component | Service | Model |
|---|---|---|
| Identity for users and services | Microsoft Entra ID (+ External ID for customers) | SaaS |
| Beacon.Api, Beacon.Bff, workers | **Azure Container Apps** (the images from Book VIII) | CaaS |
| PostgreSQL | **Azure Database for PostgreSQL Flexible Server** | PaaS |
| Secrets, keys | **Key Vault** with managed identities | PaaS |
| Attachments, exports | **Blob Storage** | PaaS |
| Edge: TLS, WAF, global routing | **Azure Front Door** | PaaS |
| Monitoring | **Application Insights / Azure Monitor** via OpenTelemetry | PaaS |
| Container images | **Azure Container Registry** (or GitHub Container Registry) | PaaS |
| CI/CD | **GitHub Actions** (Azure DevOps Pipelines is equivalent) | SaaS |
| Infrastructure as code | **Bicep** | — |

What Beacon no longer manages compared with Book VIII: OS patching, Nginx, certificate
renewal, database backups and failover, log storage, load balancing. What it still owns:
its code, data, identities and permissions, network rules, configuration, and the cost.

---

## 9. What can go wrong

- **Lift-and-shift without rethinking**: VMs configured exactly like on-premises servers,
  paying cloud prices for none of the cloud benefits.
- **Everything in one subscription and resource group**, with everyone as Owner.
- **Click-ops**: production built by hand in the portal, impossible to reproduce or review.
- **Misconfiguration exposure**: public endpoints, broad permissions, leaked keys.
- **Cost surprises**: forgotten resources, over-provisioned tiers, data egress charges.
- **Choosing a region** without checking service availability, zones or data residency.

---

## 10. How an experienced engineer thinks about this

- **Rent the undifferentiated, build the differentiating.**
- **Most managed option that fits**, with an exit path in mind.
- **You always own identity, data, configuration and cost.**
- **Design for zone failure** by default; multi-region only when the business requires it.
- **Everything as code**, from the first resource.

---

## 11. Check yourself

**Questions**

1. Compare IaaS, CaaS, PaaS, serverless and SaaS. What do you give up moving right?
2. Is the cloud always cheaper? When might it not be?
3. What's the difference between a region and an availability zone? What does each protect
   against?
4. What does the shared responsibility model say you always own?
5. Describe Azure's hierarchy from tenant to resource.
6. Why separate production and non-production subscriptions?
7. Why is Azure Resource Manager important for automation and auditing?

**Exercises**

1. Create a free Azure account (or use a sandbox), create a resource group with tags via the
   CLI, and find the operation in the activity log.
2. Write a naming convention for Beacon's resources across three environments.
3. For each component of Beacon, write one sentence on why you'd choose the managed service
   over running it yourself, or the reverse.
4. Use the Azure pricing calculator to estimate Beacon's monthly cost for a small production
   setup.

**Interview-style questions**

- "What are the differences between IaaS, PaaS and serverless? When would you use each?"
- "How would you design a cloud application to survive a data center failure?"
- "What's the shared responsibility model?"

---

## 12. Going deeper

- [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- Microsoft Learn: AZ-900 (Azure Fundamentals) learning path.

**Next:** [Chapter 2 — Identity and Security in Azure](02-identity-and-security-in-azure.md)
covers the foundation everything else depends on: who and what can access which resources.
