# Infrastructure as Code and Deployment Strategies

Beacon's Azure environment now has a VNet, subnets and NSGs, private DNS zones, a Container
Apps environment with several apps and jobs, PostgreSQL, Key Vault, Storage, Redis, SignalR,
Front Door with WAF, managed identities, role assignments, Log Analytics and alerts. Created
by hand in the portal, that's hundreds of settings no one could reproduce exactly, review, or
rebuild after a disaster.

This chapter defines that infrastructure **as code**, so environments are reproducible,
reviewable and versioned like the application. Then it covers **deployment strategies** that
release new versions without downtime, and roll back quickly when something goes wrong.

---

## 1. The problem: snowflake environments

Infrastructure built by clicking:

- **Can't be reproduced**: staging differs from production in ways nobody remembers.
- **Can't be reviewed**: a change to a firewall rule has no pull request, no diff, no second
  pair of eyes.
- **Drifts**: someone "temporarily" opens a port during an incident and forgets it.
- **Can't be recovered**: Book IX, Chapter 7's regional recovery plan ("deploy the
  infrastructure to another region") is impossible if the infrastructure isn't described
  anywhere.

---

## 2. The mental model: declarative desired state

**Infrastructure as Code (IaC)** describes infrastructure in files that tools apply:

```text
 Declarative (what)                        Imperative (how)
 ───────────────────────────────           ───────────────────────────────
 "There is a Key Vault named X with        az keyvault create ...
  RBAC enabled and public access off."     az keyvault update ...
 The tool computes the changes needed.     You script every step and every edge case.
```

Declarative tools (Bicep, Terraform, Pulumi with declarative semantics) compare **desired
state** (your files) with **actual state** and apply the difference. Running the same
deployment twice changes nothing the second time: deployments are **idempotent**.

### The IaC workflow

```text
 edit .bicep/.tf ─► PR ─► CI: validate, lint, security scan, what-if/plan (shows the diff) ─► review
      ─► merge ─► pipeline applies to staging ─► then production
```

The **plan / what-if** step is the key review artifact: "this change will modify the NSG rule
and *delete* the storage account" is exactly what a reviewer needs to see.

---

## 3. Tools

| Tool | Language | State | Scope | Notes |
|---|---|---|---|---|
| **Bicep** | Bicep DSL (compiles to ARM JSON) | Azure is the state (no state file) | Azure only | First-party, day-zero support for new Azure features, simple |
| **Terraform / OpenTofu** | HCL | A state file you store and lock (e.g. in Blob Storage) | Multi-cloud + many providers (GitHub, Datadog, DNS...) | Huge ecosystem; state management is a responsibility |
| **Pulumi** | C#, TypeScript, Python, Go | Pulumi service or self-managed backend | Multi-cloud | Real programming languages; abstraction power and risk |
| **ARM templates** | JSON | Azure | Azure | What Bicep compiles to; verbose to write by hand |
| **Azure Developer CLI (`azd`)** | Wraps Bicep/Terraform + app deployment | — | Azure | Templates and a smooth inner loop for app + infra |

Beacon is Azure-only and uses **Bicep**. Teams spanning multiple clouds or many SaaS
providers often choose Terraform; the principles are the same.

> **🧭 When not to use IaC for something:** Very little. One-off experiments in a sandbox
> subscription can be clicked. Anything in a shared, staging or production environment should
> be code. If you must click during an incident, write it back into code immediately, or the
> next deployment will revert it.

---

## 4. Bicep essentials

```bicep
// infra/modules/keyvault.bicep
@description('Name of the Key Vault (globally unique).')
param name string
param location string = resourceGroup().location
param tags object = {}

@description('Principal IDs that can read secrets.')
param secretReaderPrincipalIds array = []

resource vault 'Microsoft.KeyVault/vaults@2024-11-01' = {
  name: name
  location: location
  tags: tags
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enableSoftDelete: true
    enablePurgeProtection: true
    publicNetworkAccess: 'Disabled'
  }
}

var keyVaultSecretsUser = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')

resource readers 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for principalId in secretReaderPrincipalIds: {
  name: guid(vault.id, principalId, keyVaultSecretsUser)       // deterministic name: idempotent
  scope: vault
  properties: {
    roleDefinitionId: keyVaultSecretsUser
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}]

output id string = vault.id
output uri string = vault.properties.vaultUri
```

Concepts:

- **Parameters** (with decorators for descriptions and validation), **variables**, **outputs**.
- **Resources** declared with a type and API version.
- **Implicit dependencies**: referencing `vault.id` makes the role assignment depend on the
  vault; Bicep orders deployments automatically.
- **Loops** (`for`) and **conditions** (`if`).
- **Modules** compose reusable pieces; the **Azure Verified Modules** registry provides
  well-tested modules for most resources.
- **Parameter files** (`.bicepparam`) per environment.

### Composing an environment

```bicep
// infra/main.bicep (excerpt)
targetScope = 'resourceGroup'

param env string
param location string = resourceGroup().location
param apiImage string
param bffImage string
param postgresSku string
param minReplicas int

var tags = { app: 'beacon', env: env }
var suffix = '${env}-weu'

module identities 'modules/identities.bicep' = {
  name: 'identities'
  params: { suffix: suffix, location: location, tags: tags }
}

module network 'modules/network.bicep' = {
  name: 'network'
  params: { suffix: suffix, location: location, tags: tags }
}

module kv 'modules/keyvault.bicep' = {
  name: 'keyvault'
  params: {
    name: 'kv-beacon-${suffix}'
    tags: tags
    secretReaderPrincipalIds: [identities.outputs.apiPrincipalId, identities.outputs.bffPrincipalId]
  }
}

module db 'modules/postgres.bicep' = {
  name: 'postgres'
  params: { suffix: suffix, tags: tags, sku: postgresSku, subnetId: network.outputs.dataSubnetId,
            zoneRedundantHa: env == 'prod', entraAdminObjectId: identities.outputs.dbAdminGroupId }
}

module apps 'modules/containerapps.bicep' = {
  name: 'apps'
  params: { suffix: suffix, tags: tags, apiImage: apiImage, bffImage: bffImage, minReplicas: minReplicas,
            subnetId: network.outputs.appsSubnetId, keyVaultUri: kv.outputs.uri, /* ... */ }
}
```

```bicep
// infra/prod.bicepparam
using 'main.bicep'
param env = 'prod'
param postgresSku = 'Standard_D4ds_v5'
param minReplicas = 2
param apiImage = readEnvironmentVariable('API_IMAGE')
param bffImage = readEnvironmentVariable('BFF_IMAGE')
```

### Validating and previewing

```bash
az bicep lint --file infra/main.bicep
az deployment group what-if -g rg-beacon-prod-weu -f infra/main.bicep -p infra/prod.bicepparam
```

**What-if** shows creates, modifies and deletes. The CI pipeline posts it on the pull request
for review.

### Drift, deletion and dangerous changes

- **Drift detection**: run what-if on a schedule against each environment; any unexpected diff
  means someone changed something by hand.
- **Deployment modes**: *incremental* (default; resources not in the template are left alone)
  vs *complete* (removes them). **Deployment stacks** manage a set of resources as a unit,
  can delete what's removed from the template, and can deny out-of-band changes.
- **Resource locks** (`CanNotDelete`) on critical resources such as the production database
  and Key Vault, so a template mistake can't delete them.
- **Stateful resources need extra care**: renaming a database server in a template means
  *replace* (delete and create). Always read the what-if for deletes.

---

## 5. Deployment strategies

Infrastructure as code makes environments reproducible; deployment strategies make
**releases** safe. The goal: deploy a new version with no downtime, limit the blast radius of
a bad release, and roll back fast.

### Rolling deployment

Replace instances gradually: start new ones, wait until ready, stop old ones.

- ✓ No extra capacity needed; the default for most platforms.
- ✗ Old and new versions run side by side during the rollout (Book VII, Chapter 1's mixed
  versions); a bad release reaches everyone by the end; rollback is another rollout.

### Blue-green

Run two complete environments: **blue** (current) and **green** (new). Deploy and test green,
then **switch traffic** all at once. Keep blue for instant rollback.

```text
            ┌─► blue (v1.4)   ← 100% traffic        switch ─►   blue (v1.4)   ← 0% (standby)
 router ────┤                                                   
            └─► green (v1.5)  ← 0% (smoke tested)               green (v1.5)  ← 100%
```

- ✓ Instant switch and instant rollback; green tested in place before receiving traffic.
- ✗ Double capacity during the release; shared database must work with both versions.
- In Azure: App Service **deployment slots** (swap), Container Apps **revisions** with traffic
  weights.

### Canary

Send a **small percentage** of traffic to the new version, watch metrics, then increase
gradually (5% → 25% → 50% → 100%), or roll back.

- ✓ Real production traffic validates the release with a small blast radius; automatic
  analysis of SLIs (error rate, latency) can decide promotion.
- ✗ Requires good observability (Chapter 6) and per-version metrics; sticky behavior if users
  bounce between versions.

### Feature flags

**Decouple deployment from release**: ship code dark, then turn features on for internal
users, a percentage of customers, or specific tenants, without deploying.

```csharp
if (await features.IsEnabledAsync("UnreadCommentsBadge"))
{
    // new behavior
}
```

- ✓ Fine-grained, instant on/off; experimentation; kill switches for risky features (Book
  III, Chapter 10's incident was mitigated by one).
- ✗ Flags are technical debt: remove them once a feature is fully released, or the codebase
  fills with dead branches.
- Tools: `Microsoft.FeatureManagement` with Azure App Configuration, or third-party services
  (LaunchDarkly, Unleash, Flagsmith, OpenFeature as the vendor-neutral standard).

### Rollback vs roll forward

- **Rollback**: return to the previous version. Fast if the previous artifact still exists and
  the database is compatible (expand-and-contract makes it so).
- **Roll forward**: ship a fix. Necessary when a rollback isn't possible (a migration that
  can't be undone) and often faster for small issues with a good pipeline.

Rule: **every release must be rollback-safe for at least one version**, which means schema
changes are additive first (Book IV, Chapter 7).

| Strategy | Downtime | Blast radius | Rollback speed | Extra cost |
|---|---|---|---|---|
| Recreate (stop all, start new) | Yes | Everyone | Slow | None |
| Rolling | No | Grows to everyone | Medium | Minimal |
| Blue-green | No | Everyone at switch | Instant | 2× during release |
| Canary | No | Small, then growing | Fast | Small |
| Feature flags | No | Chosen segment | Instant | Code complexity |

---

## 6. In practice: Beacon's infrastructure and releases

### Repository layout

```text
infra/
  main.bicep
  modules/ identities, network, privatedns, keyvault, postgres, storage, redis, signalr,
           containerapps, frontdoor, monitoring (workspace, app insights, alerts, workbooks)
  dev.bicepparam   staging.bicepparam   prod.bicepparam
```

- The same `main.bicep` builds every environment; parameter files hold the differences
  (sizes, HA, replica counts, WAF mode).
- Preview environments for PRs (Book VII, Chapter 3) are a fourth parameter set deployed to
  a temporary resource group and deleted when the PR closes.
- `CanNotDelete` locks on production PostgreSQL, storage and Key Vault.
- Weekly scheduled what-if against production; any diff opens an issue.

### Canary releases with Container Apps revisions

The production deploy workflow (Chapter 8) uses **multiple revision mode**:

```bash
# 1. Deploy the new revision with 0% traffic
az containerapp update -g $RG -n ca-beacon-api --image $API_IMAGE --revision-suffix ${GIT_SHA:0:7}
NEW=ca-beacon-api--${GIT_SHA:0:7}
OLD=$(az containerapp revision list -g $RG -n ca-beacon-api --query "[?properties.trafficWeight>\`0\`].name | [0]" -o tsv)
az containerapp ingress traffic set -g $RG -n ca-beacon-api --revision-weight $OLD=100 $NEW=0

# 2. Smoke-test the new revision directly via its revision-specific URL (internal)

# 3. Canary: 10% for 10 minutes, then 50% for 10 minutes, then 100%
for weight in 10 50 100; do
  az containerapp ingress traffic set -g $RG -n ca-beacon-api --revision-weight $OLD=$((100-weight)) $NEW=$weight
  sleep 600
  ./scripts/check-slo.sh --revision $NEW || { az containerapp ingress traffic set -g $RG -n ca-beacon-api --revision-weight $OLD=100 $NEW=0; exit 1; }
done

# 4. Deactivate the old revision after a cooling-off period (kept for quick rollback until then)
```

`check-slo.sh` queries Application Insights (Chapter 6) for the new revision's error rate and
p95 latency, tagged by `cloud_RoleInstance`/revision, and compares them against the old
revision and the SLO thresholds. A bad release affects 10% of traffic for a few minutes and
rolls back automatically.

### Feature flags with App Configuration

Risky user-facing features (a new ticket list layout, AI-suggested replies in Book XI) ship
behind flags stored in Azure App Configuration, enabled first for the internal "Beacon staff"
tenant, then 10% of tenants, then everyone. Each flag has an owner and a removal date in its
description; a CI check lists flags older than 90 days.

### The full path, end to end

```text
 PR: CI + contract checks + Bicep what-if posted as a comment ─► review ─► merge
 ─► build images once (digests) ─► deploy infra + migrations + app to staging ─► E2E
 ─► approval ─► production: infra (what-if reviewed) → migrations (expand) → canary 10/50/100 with SLO checks
 ─► smoke tests ─► feature flags gradually enabled ─► later release: contract migration, flag removal
```

---

## 7. What can go wrong

- **Click-ops in production**, then IaC deployments reverting or conflicting with manual changes.
- **Unreviewed what-if output**, letting a template delete a stateful resource.
- **Secrets in IaC files or parameter files** (use Key Vault references and managed identities).
- **Environment differences outside parameter files**, so staging isn't really like production.
- **Terraform state** stored unsafely, unlocked, or lost.
- **Canaries without per-version metrics**, so nobody knows if the canary is healthy.
- **Non-rollback-safe migrations**, making every rollback a data problem.
- **Feature flags that never get removed.**

---

## 8. How an experienced engineer thinks about this

- **If it isn't in code, it doesn't exist** (or it's about to drift).
- **Review infrastructure diffs** as carefully as code diffs, especially deletes.
- **Reproducibility is a disaster recovery capability.**
- **Separate deploy from release**: canaries limit blast radius; flags control exposure.
- **Every release must be safely reversible**, which starts with backward-compatible data
  changes.

---

## 9. Check yourself

**Questions**

1. What problems does infrastructure as code solve?
2. What does "declarative" and "idempotent" mean for IaC?
3. Compare Bicep and Terraform. What's Terraform state, and why must it be protected?
4. What is what-if/plan output for, and what should a reviewer look for first?
5. Compare rolling, blue-green and canary deployments.
6. How do feature flags differ from deployment strategies? What's their cost?
7. What makes a release rollback-safe?

**Exercises**

1. Write a Bicep module for Beacon's storage account (ZRS, shared key disabled, containers,
   lifecycle policy, RBAC for the API identity) and deploy it to a dev resource group.
2. Run what-if after renaming a resource and observe the delete/create.
3. Implement a canary release for a Container App with two revisions and manual traffic
   shifting; then automate the SLO check.
4. Add a feature flag with `Microsoft.FeatureManagement` and Azure App Configuration, and
   enable it for one tenant.

**Interview-style questions**

- "What is infrastructure as code? Which tools have you used?"
- "How do you deploy without downtime?"
- "What's the difference between blue-green and canary deployments?"
- "How do you handle a failed deployment?"

---

## 10. Going deeper

- [Bicep documentation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview)
  and [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Terraform documentation](https://developer.hashicorp.com/terraform/docs) /
  [OpenTofu](https://opentofu.org/docs/)
- [Azure Container Apps: Blue-green and traffic splitting](https://learn.microsoft.com/azure/container-apps/traffic-splitting)
- Kief Morris, *Infrastructure as Code*.

---

## Book IX wrap-up

Beacon now runs on Azure: managed services chosen deliberately, identities instead of
secrets, private networking behind Front Door and a WAF, end-to-end observability with SLOs,
measured capacity and explicit cost and availability trade-offs, a secure CI/CD pipeline that
builds once and promotes by digest, infrastructure defined in Bicep, and canary releases with
automatic rollback.

That completes the "classic" full-stack and cloud engineering path. The next books add the
skills that increasingly define modern application engineering, starting with the language
most AI tooling is built on.

**Next:** [Book X — Python](../10-python/README.md).
