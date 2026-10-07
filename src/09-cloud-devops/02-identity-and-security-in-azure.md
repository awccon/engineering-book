# Identity and Security in Azure

In the cloud, **identity is the new perimeter**. On-premises, a firewall around the data
center was the main security boundary. In Azure, every resource is reachable through APIs,
and what protects them is *who* (or *what*) is allowed to call those APIs. A leaked access
key, an over-privileged service principal or an administrator account without MFA can expose
everything, regardless of networks and firewalls.

This chapter covers Microsoft Entra ID, Azure RBAC, managed identities and Key Vault, and
sets Beacon up so that **no application secret exists at all** for Azure resources.

---

## 1. The problem: secrets everywhere

A traditional application deployment is full of credentials:

- the database connection string with a password,
- storage account keys,
- API keys for email and other services,
- certificates for TLS and signing,
- credentials the CI/CD pipeline uses to deploy.

Each is a long-lived secret that must be stored somewhere, rotated, and kept out of logs and
Git. Each leak is an incident. The goal of modern cloud identity is to **eliminate as many of
these secrets as possible**, and protect the rest.

---

## 2. The mental model: principals, permissions, scopes

Every access decision in Azure answers: **can this principal perform this action on this
scope?**

```text
 Principal (who)              Role (what actions)                   Scope (where)
 ───────────────              ───────────────────                   ─────────────
 user maria@contoso.com   ─┐   Reader                                management group
 group beacon-devs         ├─► Contributor                     on    subscription
 managed identity of       │   Key Vault Secrets User                resource group
   app-beacon-api          │   Storage Blob Data Contributor         single resource
 service principal (CI)  ─┘   custom roles...
```

A **role assignment** = principal + role definition + scope. Permissions are inherited down
the hierarchy: Contributor on a subscription applies to every resource group and resource in
it.

---

## 3. Microsoft Entra ID

**Microsoft Entra ID** (formerly Azure Active Directory) is Microsoft's cloud identity
service. It's the identity provider for Azure itself, Microsoft 365, and any application you
register.

### Kinds of identities

| Identity | Represents | Credentials |
|---|---|---|
| **User** | A person | Password + MFA, passkeys, or federated from another IdP |
| **Group** | A set of principals | Used for assignments: assign roles to groups, not individuals |
| **App registration / service principal** | An application (your API, your CI pipeline) | Client secret, certificate, or **federated credential** |
| **Managed identity** | An Azure resource (an app, a VM, a function) | **None you handle**: Azure manages them automatically |

An **app registration** is the global definition of an application (its client ID,
redirect URIs, exposed API scopes, Book III, Chapter 6). A **service principal** is that
application's identity in a specific tenant, which can be granted roles.

### Entra ID as Beacon's OIDC provider

Book III and Book VI used a generic OIDC provider. In Azure:

- **Beacon.Api** has an app registration exposing scopes (`api://beacon-api/tickets`) and app
  roles (`agent`, `lead`). Its JWT bearer configuration uses Entra ID as the authority.
- **Beacon.Bff** has an app registration as a confidential web client that requests those
  scopes.
- **Employees** (agents and leads) sign in with the company's Entra ID tenant, with
  conditional access and MFA.
- **Customers** sign in through **Microsoft Entra External ID** (the customer-identity
  offering), with local accounts, social logins or passkeys, separate from the workforce
  tenant.

> **🔄 Current (as of October 2026):** Microsoft Entra External ID is the successor to Azure
> AD B2C for customer-facing identity. Existing B2C tenants continue to work, but new
> projects use External ID.

### Protecting human accounts

- **MFA for everyone**, phishing-resistant methods (passkeys, FIDO2 keys, Microsoft
  Authenticator with number matching) for administrators.
- **Conditional Access**: policies such as "require MFA and a compliant device for Azure
  management," "block sign-ins from unexpected countries."
- **Privileged Identity Management (PIM)**: administrators are *eligible* for privileged
  roles and activate them **just in time**, for a limited period, with justification and
  approval. No standing Owner access.
- **Break-glass accounts**: one or two emergency accounts excluded from conditional access,
  with strong credentials stored securely and monitored for any use.

---

## 4. Azure RBAC

**Azure role-based access control** governs actions on Azure resources.

### Built-in roles you'll use

| Role | Allows |
|---|---|
| **Owner** | Everything, including assigning roles |
| **Contributor** | Create and manage resources, but not assign roles |
| **Reader** | View resources |
| **User Access Administrator** | Manage role assignments |
| Service-specific roles | e.g. **Key Vault Secrets User**, **Storage Blob Data Contributor**, **AcrPull**, **Website Contributor** |

### Control plane vs data plane

An important distinction:

- **Control plane**: managing the resource (create a storage account, change its settings).
  Covered by roles like Contributor.
- **Data plane**: using the resource's data (read a blob, read a secret, query a database).
  Covered by **data roles** (Storage Blob Data Reader, Key Vault Secrets User).

A Contributor on a Key Vault can configure it but (with RBAC authorization mode) can't read
its secrets. An app that only needs to read secrets gets Key Vault Secrets User on that one
vault, nothing more.

### Least privilege in practice

- **Assign roles to groups**, not individual users.
- **Narrowest scope**: a role on one resource rather than the resource group; on a resource
  group rather than the subscription.
- **Narrowest role**: data roles instead of Contributor; custom roles when built-ins are too
  broad.
- **No standing privileged access** in production (PIM).
- **Review assignments** periodically (Entra access reviews).

---

## 5. Managed identities: no secrets at all

A **managed identity** is an Entra identity automatically created and managed by Azure for a
resource. Code running on that resource can obtain tokens for that identity from a local
endpoint, with **no credentials in configuration**.

```text
 Container App (Beacon.Api)
 └─ managed identity: id-beacon-api-prod
      │ 1. asks the local identity endpoint for a token for "https://ossrdbms-aad.database.windows.net"
      │ 2. Azure returns a short-lived token (no secret involved)
      ▼
 Azure Database for PostgreSQL ── verifies the Entra token, maps it to a database role
```

Two flavors:

- **System-assigned**: tied to one resource's lifecycle (deleted with it).
- **User-assigned**: a standalone identity resource attached to one or more resources;
  survives redeployments and can be granted access before the app exists. Usually preferred
  for applications deployed with infrastructure as code.

### `DefaultAzureCredential`

The Azure SDKs use `DefaultAzureCredential`, which tries a chain of credential sources:
environment variables, workload identity, **managed identity** (in Azure), Visual Studio /
Azure CLI / `azd` sign-in (on a developer machine). The **same code** works locally with your
developer account and in Azure with the managed identity:

```csharp
using Azure.Identity;

var credential = new DefaultAzureCredential();   // in production, prefer ManagedIdentityCredential with an explicit client ID
var blobs = new BlobServiceClient(new Uri("https://stbeaconprodweu.blob.core.windows.net"), credential);
```

> **⚠️ What can go wrong:** `DefaultAzureCredential` tries several sources in order, which is
> convenient in development but can be slow (each failed attempt costs time at startup) and
> occasionally surprising in production (picking up an unexpected credential). In production,
> configure the specific credential (`ManagedIdentityCredential` with the user-assigned
> identity's client ID), or exclude sources you don't use.

### Workload identity federation: no secrets in CI/CD either

GitHub Actions (and other CI systems) can authenticate to Azure **without stored secrets**
via **OpenID Connect federation**: GitHub issues a short-lived token for the workflow run,
and Entra ID trusts it for a specific repository, branch or environment:

```yaml
permissions:
  id-token: write          # allow the job to request an OIDC token
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}           # not secrets: identifiers only
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

The federated credential is scoped (e.g. `repo:awccon/beacon:environment:production`), so
only deployments to that environment from that repository can obtain the identity.

---

## 6. Key Vault

Some secrets can't be eliminated: third-party API keys, signing keys, certificates, an SMTP
password. **Azure Key Vault** stores them:

- **Secrets**: strings (API keys, connection strings for non-Entra services).
- **Keys**: cryptographic keys (for signing and encryption), which can be **non-exportable**
  (operations happen inside the vault or an HSM).
- **Certificates**: with lifecycle management and auto-renewal from supported CAs.

Good practice:

- **RBAC authorization mode** (not legacy access policies), with data roles per app.
- **One vault per application per environment** (blast radius; vault-level permissions).
- **Soft delete and purge protection** enabled, so a deleted vault or secret can be recovered.
- **Private endpoint** (Chapter 5) and firewall rules.
- **Access via managed identity**, so reading the secrets requires no secret.
- **Logging**: diagnostic logs of every access, sent to Log Analytics (Chapter 6).

### Loading Key Vault into .NET configuration

```csharp
// Program.cs
var vaultUri = builder.Configuration["KeyVault:Uri"];
if (!string.IsNullOrEmpty(vaultUri))
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(vaultUri),
        new ManagedIdentityCredential(ManagedIdentityId.FromUserAssignedClientId(builder.Configuration["AZURE_CLIENT_ID"]!)));
}
```

Secrets named with `--` map to configuration sections: a secret `Notifications--Smtp--Password`
becomes `Notifications:Smtp:Password`, so the options pattern (Book III, Chapter 3) works
unchanged. Locally, user secrets provide the same keys.

Platforms such as Container Apps and App Service can also reference Key Vault secrets
directly in app settings, injecting them as environment variables, with the platform's
managed identity reading the vault.

---

## 7. Other security services worth knowing

| Service | Purpose |
|---|---|
| **Microsoft Defender for Cloud** | Security posture score, misconfiguration findings, threat protection for VMs, containers, databases, storage |
| **Azure Policy** | Enforce or audit rules: deny public IPs, require private endpoints, allowed regions, required tags |
| **Microsoft Sentinel** | SIEM: collect and correlate security logs, detect and respond to threats |
| **Activity log / Entra sign-in and audit logs** | Who did what, from where |
| **Microsoft Purview** | Data classification and governance |

At minimum: enable Defender for Cloud's free posture management, act on its recommendations,
and apply a small set of Azure Policies that make the most common misconfigurations
impossible.

---

## 8. In practice: Beacon's identity design

**Humans**

| Group | Role | Scope |
|---|---|---|
| `grp-beacon-devs` | Contributor | `sub-beacon-nonprod` |
| `grp-beacon-devs` | Reader | `sub-beacon-prod` |
| `grp-beacon-ops` | Contributor (eligible via PIM, 4 h, approval) | `rg-beacon-prod-weu` |
| `grp-beacon-oncall` | Key Vault Secrets User (eligible via PIM) | `kv-beacon-prod-weu` |

Nobody has standing write access to production. Changes go through the pipeline.

**Workloads**: one user-assigned managed identity per app, with only what it needs:

| Identity | Grants |
|---|---|
| `id-beacon-api-prod` | **AcrPull** on the registry; **Key Vault Secrets User** on `kv-beacon-prod-weu`; **Storage Blob Data Contributor** on the `attachments` container; Entra-authenticated **PostgreSQL** role `beacon_app` |
| `id-beacon-bff-prod` | AcrPull; Key Vault Secrets User (for its OIDC client certificate) |
| `id-beacon-migrator-prod` | PostgreSQL role `beacon_migrator` (used only by the pipeline's migration job) |
| GitHub Actions (federated) | Contributor on `rg-beacon-prod-weu` for deployments, from `environment:production` only |

**Database access without passwords** (Book IV, Chapter 8's roles, now with Entra):

```sql
-- run as the Entra admin of the PostgreSQL server
select * from pgaadauth_create_principal('id-beacon-api-prod', false, false);
grant beacon_app to "id-beacon-api-prod";
```

```csharp
// Beacon.Infrastructure: Npgsql obtains an Entra token as the password, refreshed automatically
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);   // no Password= in the string
dataSourceBuilder.UsePeriodicPasswordProvider(async (_, ct) =>
{
    var token = await credential.GetTokenAsync(
        new TokenRequestContext(["https://ossrdbms-aad.database.windows.net/.default"]), ct);
    return token.Token;
}, TimeSpan.FromMinutes(30), TimeSpan.FromSeconds(10));
```

**Remaining secrets** in Key Vault: the SMTP provider's API key, and the BFF's OIDC client
credential (a certificate, preferably, or a federated credential between the BFF's managed
identity and its app registration, which removes even that secret).

Result: **no passwords or keys in configuration files, environment variables or the
pipeline**. Rotating credentials becomes Azure's job; revoking access is a role assignment
change.

---

## 9. What can go wrong

- **Standing Owner/Contributor access** for many people in production.
- **Service principals with client secrets** that never expire, stored in CI variables.
- **Roles at subscription scope** "to make it work."
- **Contributor where a data role was needed**, or vice versa.
- **Shared Key Vaults** across apps and environments.
- **Storage account keys and SAS tokens** handed out instead of Entra-based access.
- **No MFA or conditional access** for administrators.
- **Ignoring Defender for Cloud recommendations.**

---

## 10. How an experienced engineer thinks about this

- **Identity is the perimeter.** Protect privileged identities above all else.
- **Eliminate secrets** with managed identities and workload identity federation; vault the rest.
- **Least privilege by default**: narrow roles, narrow scopes, just-in-time elevation.
- **Groups, not individuals**, in role assignments.
- **Policy as guardrails**: make the dangerous configurations impossible, not just discouraged.

---

## 11. Check yourself

**Questions**

1. What three things make up an Azure role assignment?
2. What's the difference between the control plane and the data plane? Give an example role
   for each.
3. What's a managed identity, and why is it better than a client secret?
4. How does `DefaultAzureCredential` work locally vs in Azure? What's the production caveat?
5. How can GitHub Actions deploy to Azure without stored secrets?
6. What should Key Vault be used for, and how should apps access it?
7. What is PIM, and why is just-in-time access better than standing access?

**Exercises**

1. Create a user-assigned managed identity, grant it Storage Blob Data Reader on a container,
   and read a blob from a Container App or a VM using `DefaultAzureCredential`.
2. Configure GitHub Actions OIDC federation to a resource group and run `az group show` in a
   workflow with no secrets.
3. Set up Entra authentication for a PostgreSQL Flexible Server and connect from .NET with a
   token as the password.
4. Assign one Azure Policy (e.g. "Storage accounts should disable public network access") and
   observe its effect.

**Interview-style questions**

- "How do you manage secrets for applications running in Azure?"
- "What's a managed identity?"
- "How would you set up access control for a team working on a production system?"

---

## 12. Going deeper

- [Microsoft docs: Managed identities for Azure resources](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)
- [Microsoft docs: Azure RBAC](https://learn.microsoft.com/azure/role-based-access-control/overview)
- [Microsoft docs: Azure Key Vault best practices](https://learn.microsoft.com/azure/key-vault/general/best-practices)
- [Microsoft docs: Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)

**Next:** [Chapter 3 — Compute: App Service, Functions and Containers](03-compute-app-service-functions-and-containers.md)
decides where Beacon's code runs.
