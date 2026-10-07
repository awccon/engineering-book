# Networking

Cloud networking can look like a maze of acronyms: VNets, subnets, NSGs, private endpoints,
private DNS zones, NAT gateways, Front Door, WAF. But it's built on the same fundamentals as
Book VIII, Chapter 2: addresses, routes, ports, DNS, firewalls. The cloud adds software-defined
versions of them, plus managed services at the edge.

This chapter explains Azure networking from those fundamentals, and puts Beacon's databases,
storage and internal services on private networks behind a single secure entry point.

---

## 1. The problem: managed services are public by default

Many Azure PaaS services (PostgreSQL, Storage, Key Vault, Container Registry) are created
with **public endpoints**: reachable from the internet, protected only by authentication and
optional IP firewall rules. That's convenient and risky:

- A leaked credential or misconfigured permission is immediately exploitable from anywhere.
- Data exfiltration paths are wide open.
- Compliance standards often require private connectivity.

Defense in depth (Book III, Chapter 9) says: even with strong identity (Chapter 2), make
sensitive services **unreachable** from the internet.

---

## 2. The mental model: virtual networks and boundaries

```text
 Internet
    │
    ▼
 Azure Front Door (global edge: TLS, WAF, CDN, routing)
    │  private link or locked to Front Door
    ▼
 ┌──────────────────── VNet: vnet-beacon-prod-weu  10.20.0.0/16 ─────────────────────┐
 │                                                                                   │
 │  snet-apps 10.20.0.0/23         snet-data 10.20.4.0/24     snet-pe 10.20.5.0/24   │
 │  ┌──────────────────────┐       ┌───────────────────┐      ┌──────────────────┐   │
 │  │ Container Apps env    │──────►│ PostgreSQL         │      │ private endpoints │   │
 │  │  bff, api, worker     │       │ (VNet-integrated)  │      │  Key Vault, Blob, │   │
 │  └──────────────────────┘       └───────────────────┘      │  ACR, Redis       │   │
 │            │ NSG rules                                       └──────────────────┘   │
 │            ▼ outbound via NAT Gateway (fixed public IP)                              │
 └───────────────────────────────────────────────────────────────────────────────────┘
    Private DNS zones: privatelink.vaultcore.azure.net, privatelink.blob.core.windows.net, ...
```

### Virtual networks (VNets) and subnets

A **VNet** is a private, isolated network in a region with an address space you choose
(`10.20.0.0/16`). It's divided into **subnets** (`10.20.0.0/23`) where resources are placed.
Resources in a VNet communicate privately; nothing reaches them from the internet unless you
add a public entry point.

Plan address spaces so they don't overlap with other VNets or on-premises networks you may
ever connect to: renumbering later is painful.

### Network security groups (NSGs)

**NSGs** are stateful firewalls attached to subnets or network interfaces: rules allowing or
denying traffic by source, destination, port and protocol, evaluated by priority. They're
the cloud equivalent of `ufw` (Book VIII, Chapter 2), applied per subnet.

### Routing and outbound traffic

- By default, resources reach the internet outbound through Azure-provided addresses that
  can change.
- A **NAT Gateway** gives a subnet a fixed, known outbound IP, useful when partners or
  third-party APIs allow-list your IP, and to avoid outbound port exhaustion under heavy load.
- **User-defined routes** can force traffic through a central firewall (Azure Firewall) for
  inspection and egress filtering in stricter environments.

---

## 3. Private connectivity to PaaS services

Two mechanisms keep PaaS traffic off the internet:

| Mechanism | How | Notes |
|---|---|---|
| **VNet integration / injection** | The service is deployed *into* your subnet (PostgreSQL Flexible Server private access, Container Apps environments, App Service VNet integration for outbound) | Service-specific |
| **Private endpoint (Private Link)** | A network interface with a private IP in your subnet that maps to one specific PaaS resource | Works for most PaaS services; disables public access when configured |

With a private endpoint for Key Vault, `kv-beacon-prod-weu.vault.azure.net` resolves (inside
the VNet) to `10.20.5.4`, and the vault's public network access can be **disabled** entirely.

### Private DNS: where it usually goes wrong

Clients still use the public hostname (`kv-beacon-prod-weu.vault.azure.net`); TLS
certificates are issued for it. The trick is DNS:

```text
 kv-beacon-prod-weu.vault.azure.net
   └─ CNAME kv-beacon-prod-weu.privatelink.vaultcore.azure.net
        └─ (inside the VNet, via the linked Private DNS zone) A 10.20.5.4
        └─ (outside) resolves to the public endpoint (which then refuses access)
```

A **Private DNS zone** (`privatelink.vaultcore.azure.net`) linked to the VNet holds the
private records. If the zone isn't linked, or a custom DNS server doesn't forward to Azure
DNS, the app resolves the public address and gets "access denied" or timeouts.

> **🔍 Investigation: "the app can't reach Key Vault/Storage after we enabled private
> endpoints."** From inside the app's network (a debug container, Book VIII, Chapter 6):
> 1. `nslookup kv-beacon-prod-weu.vault.azure.net`: does it return a **private** IP
>    (10.x)? If it returns a public IP, the Private DNS zone isn't linked to the VNet, or
>    custom DNS isn't forwarding to Azure DNS (`168.63.129.16`).
> 2. `nc -vz <private-ip> 443`: is the port reachable? If not, check NSG rules on both
>    subnets.
> 3. If DNS and network are fine, the error is identity: RBAC on the vault (Chapter 2).

---

## 4. The edge: Front Door, Application Gateway and WAF

Something must accept traffic from the internet. Azure's main options:

| Service | Scope | Features | Use when |
|---|---|---|---|
| **Azure Front Door** | **Global** (Microsoft's edge network, hundreds of locations) | TLS termination, managed certificates, CDN caching, WAF, global load balancing and failover, private link to origins | Public web apps and APIs, especially with users in many regions or multi-region backends |
| **Application Gateway** | **Regional** (in your VNet) | Layer-7 load balancing, WAF, TLS, path routing | Regional apps needing a VNet-resident gateway; internal apps |
| **Azure Load Balancer** | Regional, layer 4 (TCP/UDP) | Fast, simple | Non-HTTP traffic, VMs |
| **API Management** | Regional/global | API gateway: keys, quotas, transformations, developer portal | Exposing APIs to partners and external developers |

### Web Application Firewall (WAF)

A **WAF** inspects HTTP requests and blocks common attacks (SQL injection patterns, XSS
payloads, known bad bots, protocol violations) using managed rule sets (OWASP-based), plus
custom rules: rate limiting per IP, geo-filtering, blocking specific paths.

A WAF is **defense in depth**, not a substitute for secure code (Book III, Chapter 9). It
catches generic attacks and buys time during an incident ("block this pattern now while we
patch"). Run new rules in **detection mode** first: false positives (a ticket comment
containing a SQL snippet a customer pasted) block legitimate users.

### Locking origins to the edge

If Front Door protects your app but the app also accepts traffic directly, attackers can
bypass the WAF. Lock the origin:

- **Private Link** from Front Door (Premium) to the Container Apps environment or App
  Service: the origin has no public endpoint at all.
- Or restrict inbound traffic to the `AzureFrontDoor.Backend` service tag **and** validate
  the `X-Azure-FDID` header contains your Front Door's ID (since the service tag is shared by
  all Front Door customers).

---

## 5. Connecting networks

- **VNet peering**: connect two VNets (same or different regions) with private,
  low-latency connectivity. Peering isn't transitive by default.
- **Hub-and-spoke**: a central **hub** VNet holds shared services (firewall, VPN gateway,
  DNS resolvers, bastion); application **spokes** peer with the hub. The standard enterprise
  topology (Azure Landing Zones).
- **VPN Gateway / ExpressRoute**: connect on-premises networks to Azure (over the internet
  encrypted, or a private dedicated circuit).
- **Azure Bastion**: browser-based or native-client SSH/RDP to VMs without public IPs on them
  (Book VIII, Chapter 2's "no public SSH," done right).

> **🧭 When not to over-network:** A small team with one application doesn't need a hub-and-spoke
> topology, central firewalls and ExpressRoute. Start with one VNet per environment, private
> endpoints for data services, NSGs, and a WAF at the edge. Grow into landing zones when the
> organization's scale and compliance needs require it.

---

## 6. DNS and custom domains

- **Public DNS**: `beacon.example.com` → CNAME to the Front Door endpoint. Azure DNS can host
  the zone, or keep it at your registrar.
- **Domain validation and managed certificates**: Front Door issues and renews certificates
  automatically once you prove domain ownership with a TXT record. No Certbot (Book VIII,
  Chapter 3) required.
- **Apex domains** (`example.com`, no subdomain) can't be a CNAME; use Azure DNS alias records
  or your DNS provider's ALIAS/ANAME feature.
- **Private DNS zones** for internal names and private endpoints (section 3).

---

## 7. In practice: Beacon's production network

**Topology** (one VNet per environment; production shown):

| Subnet | Contents | NSG highlights |
|---|---|---|
| `snet-apps` (/23) | Container Apps environment (workload profiles, internal) | Inbound only from Front Door private link; outbound to `snet-data`, `snet-pe`, and internet via NAT Gateway |
| `snet-data` (/24) | PostgreSQL Flexible Server (delegated subnet) | Inbound 5432 only from `snet-apps` |
| `snet-pe` (/24) | Private endpoints: Key Vault, Storage, ACR, Redis, SignalR | Inbound 443/6380 only from `snet-apps` |

**Private DNS zones** linked to the VNet: `privatelink.postgres.database.azure.com`,
`privatelink.vaultcore.azure.net`, `privatelink.blob.core.windows.net`, `privatelink.azurecr.io`,
`privatelink.redis.azure.net`, `privatelink.service.signalr.net`.

**Public network access disabled** on PostgreSQL, Key Vault, Storage (except as noted below),
ACR, Redis and SignalR.

**Edge**: Azure Front Door Premium:

- Custom domain `beacon.example.com` with a managed certificate.
- Origin: the BFF's Container App via **Private Link** (no public ingress on the app).
- WAF policy: managed default rule set + bot protection in **prevention** mode after two weeks
  in detection mode; a custom rate-limit rule (300 requests/minute per IP on `/bff/login`,
  Book III, Chapter 9).
- Caching rule for `/assets/*` (hashed, immutable), no caching for everything else.
- Health probe on `/health/ready`.

**Outbound**: NAT Gateway on `snet-apps` with a static IP, given to the email provider for
allow-listing.

**Attachment downloads** (Chapter 4) use user delegation SAS URLs, which browsers must reach
directly, so Blob Storage keeps a **public endpoint restricted to SAS access**, with shared
key access disabled and RBAC for the app via the private endpoint. (Alternative: serve
downloads through Front Door with a private-link origin to storage.) Trade-offs like this
should be written down, not left implicit.

**Operator access**: no public IPs anywhere; on-call engineers use Azure Bastion to a small
jump VM in the hub (or `az containerapp exec` and debug containers), with PIM-elevated access
(Chapter 2).

**Verification checklist** after deployment:

```bash
# From the internet: only Front Door answers
curl -sSI https://beacon.example.com/health/ready             # 200 via Front Door
curl -sSI https://ca-beacon-bff.<env>.westeurope.azurecontainerapps.io   # not reachable / 403
nslookup kv-beacon-prod-weu.vault.azure.net                   # public resolution; access refused from outside

# From inside (debug container in the environment)
nslookup kv-beacon-prod-weu.vault.azure.net                   # → 10.20.5.x
nc -vz psql-beacon-prod-weu.postgres.database.azure.com 5432  # open
```

---

## 8. What can go wrong

- **Public endpoints left enabled** on data services after adding private endpoints.
- **Private DNS zones not linked** (or custom DNS not forwarding), so apps resolve public IPs.
- **Overlapping address spaces** that block future peering or VPNs.
- **Origins reachable directly**, bypassing Front Door and the WAF.
- **WAF in prevention mode on day one**, blocking legitimate traffic.
- **Unpredictable outbound IPs** breaking partner allow-lists (fix: NAT Gateway).
- **Over-engineered topologies** for small workloads; or no network controls at all.

---

## 9. How an experienced engineer thinks about this

- **Identity first, network as a second layer**: private by default for data services.
- **One public entry point** with TLS and WAF; everything else private.
- **DNS is part of networking design**; most private-endpoint problems are DNS problems.
- **Plan address space for the future.**
- **Write down exceptions** (like public blob access for SAS) and the reasoning.

---

## 10. Check yourself

**Questions**

1. What are VNets, subnets and NSGs, and what are their equivalents on a single Linux server?
2. What's the difference between VNet integration and a private endpoint?
3. Why do private endpoints depend on Private DNS zones? What happens without the zone link?
4. Compare Front Door and Application Gateway.
5. What does a WAF protect against, and why isn't it a substitute for secure code?
6. How do you prevent attackers from bypassing Front Door to reach your origin directly?
7. Why use a NAT Gateway?

**Exercises**

1. Create a VNet with a private endpoint for a Key Vault, disable public access, and read a
   secret from a Container App in the VNet. Then unlink the Private DNS zone and observe the
   failure.
2. Put Front Door in front of a Container App with Private Link and confirm the app's default
   hostname no longer answers from the internet.
3. Add a WAF custom rate-limit rule and test it with a load tool.
4. Draw Beacon's network diagram for staging, using cheaper choices where appropriate, and
   justify each difference from production.

**Interview-style questions**

- "How would you secure network access to a cloud database?"
- "What's the difference between a CDN, a load balancer and a WAF?"
- "Explain how private endpoints work in Azure."

---

## 11. Going deeper

- [Microsoft docs: Azure Virtual Network](https://learn.microsoft.com/azure/virtual-network/virtual-networks-overview)
- [Microsoft docs: Private endpoint DNS configuration](https://learn.microsoft.com/azure/private-link/private-endpoint-dns)
- [Microsoft docs: Azure Front Door](https://learn.microsoft.com/azure/frontdoor/front-door-overview)
- [Azure Landing Zones](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)

**Next:** [Chapter 6 — Monitoring and Observability](06-monitoring-and-observability.md)
makes Beacon's behavior in production visible.
