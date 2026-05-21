# Azure Workshop — Instructor Reference
## Fundamentals Deep Dive + Student Q&A Guide

---

# PART 1 — CORE FUNDAMENTALS

---

## 1. Azure Hierarchy — Tenant, Subscription, Resource Group

### How Azure is organized (top to bottom):

```
Azure Active Directory Tenant
    └── Subscription (billing + access boundary)
            └── Resource Group (logical container)
                    └── Resources (VMs, VNets, Storage, etc.)
```

### Tenant
- Represents your **organization** in Azure Active Directory (AAD/Entra ID).
- One company = one tenant (usually).
- Manages **identities** — users, service principals, managed identities.
- Think of it as the "owner" of everything.

### Subscription
- A **billing and access boundary** inside a tenant.
- You can have multiple subscriptions under one tenant (e.g., Dev, Prod, Test).
- Free tier = one subscription with $200 credits for 30 days + always-free services for 12 months.
- Resources are always deployed inside a subscription.

### Resource Group
- A **logical container** inside a subscription.
- Groups related resources together for a project or environment.
- Deleting a resource group **deletes everything inside it**.
- Resources in different resource groups can still talk to each other.
- You cannot deploy any Azure resource without a resource group.

### Key rule:
Every resource in Azure belongs to exactly **one resource group**, in exactly **one subscription**, in exactly **one tenant**.

---

## 2. Virtual Network (VNet)

### What it is:
- Your **private network** in Azure. Equivalent to your home router's private network, but in the cloud.
- All resources inside a VNet can talk to each other privately by default.
- Traffic inside a VNet **never leaves Azure's backbone network**.

### Address Space:
- Defined using CIDR notation: `10.0.0.0/16` means addresses from `10.0.0.0` to `10.0.255.255` (65,536 addresses).
- You divide this space into smaller **subnets**.

### What a VNet gives you:
- Private IP addressing
- DNS resolution for resources inside it
- Default outbound internet access
- Isolation from other VNets by default (unless you peer them)

### What a VNet does NOT give you by default:
- Inbound access from the internet (needs public IP + NSG rule)
- Connection to on-premises networks (needs VPN Gateway or ExpressRoute)
- Connection to other VNets (needs VNet Peering)

---

## 3. Subnets

### What it is:
- A **subdivision of the VNet address space**.
- You separate resources into subnets by tier, function, or security boundary.
- Azure routes traffic between subnets **automatically** — no route tables needed for basic setups.

### Reserved IPs per subnet:
Azure reserves **5 IP addresses** in every subnet:
- x.x.x.0 — Network address
- x.x.x.1 — Default gateway (Azure router)
- x.x.x.2 and x.x.x.3 — Azure DNS
- x.x.x.255 — Broadcast

So a `/24` subnet (256 addresses) gives you **251 usable IPs**.

### Why separate subnets per tier:
- Security: apply different NSG rules per subnet
- Isolation: database subnet only accepts traffic from app subnet
- Organisation: clear boundaries between layers

---

## 4. Network Security Groups (NSGs)

### What it is:
- A **firewall at the subnet or NIC level** in Azure.
- Contains a list of **inbound and outbound rules**.
- Each rule has: priority, source, destination, port, protocol, allow/deny.

### Rule evaluation:
- Rules are evaluated **lowest priority number first** (100 is evaluated before 200).
- First matching rule wins. Evaluation stops.
- Default rules at the bottom always exist and cannot be deleted:
  - `AllowVNetInBound` — allow all traffic within VNet
  - `AllowAzureLoadBalancerInBound` — allow Load Balancer health probes
  - `DenyAllInBound` — deny everything else
  - `AllowVNetOutBound` — allow all outbound within VNet
  - `AllowInternetOutBound` — allow all outbound to internet
  - `DenyAllOutBound` — deny everything else

### NSG vs Firewall:
| NSG | Azure Firewall |
|---|---|
| Layer 4 (TCP/UDP) | Layer 7 (application-aware) |
| Free | Expensive managed service |
| Subnet or NIC level | Central hub, all traffic routed through it |
| Simple allow/deny rules | FQDN filtering, threat intelligence, IDPS |

For your workshop, NSGs are sufficient.

### Attaching NSGs:
- You can attach an NSG to a **subnet** (applies to all VMs in that subnet) — preferred.
- You can also attach to a **NIC** (applies to one VM only).
- If both exist, both are evaluated. Inbound: subnet NSG first, then NIC NSG. Outbound: NIC NSG first, then subnet NSG.

---

## 5. Public IP vs Private IP

### Private IP:
- Automatically assigned by Azure from the subnet's address range.
- Only reachable inside the VNet.
- Used for internal communication between tiers.

### Public IP:
- Globally routable IP address.
- Can be assigned to a VM's NIC or a Load Balancer.
- Without a public IP, a VM cannot receive inbound traffic from the internet.
- Even with a public IP, NSG rules must allow the traffic.

### For your three-tier app:
- Frontend VM → has a Public IP (or exposed via Load Balancer)
- App Tier VM → Private IP only
- Database VM → Private IP only

---

## 6. Load Balancer

### What it is:
- Distributes incoming traffic across **multiple VMs** so no single VM is overwhelmed.
- Operates at **Layer 4** (TCP/UDP).
- Has a **public IP** that users connect to.
- Health probes check if backend VMs are healthy; unhealthy VMs are removed from rotation.

### Components:
- **Frontend IP Config** — the public IP that receives traffic
- **Backend Pool** — the VMs that handle the traffic
- **Health Probe** — pings VMs periodically (HTTP or TCP) to check health
- **Load Balancing Rule** — maps frontend port to backend port

### SKUs:
- **Basic** — free, limited features, no zone redundancy
- **Standard** — paid, zone redundant, production-grade

---

## 7. Managed Identity

### Why it exists:
When your app on a VM needs to talk to Azure services (Key Vault, Storage, SQL), it needs to **authenticate**. Old approach: store a username/password or client secret in config. Bad — secrets can leak. Managed Identity removes the need for credentials entirely.

### How it works:
- Azure assigns an **identity** to your VM (backed by Azure Active Directory).
- The VM can request a **short-lived token** from the Azure metadata endpoint (`169.254.169.254`) automatically.
- Azure services validate this token — no password ever stored.

### System-assigned vs User-assigned:

| | System-assigned | User-assigned |
|---|---|---|
| Created | Automatically when VM is created | Manually as a separate Azure resource |
| Lifecycle | Tied to the VM — deleted when VM is deleted | Independent — exists even if VM is deleted |
| Scope | One VM only | Can be attached to multiple VMs |
| Use case | Simple per-VM identity | Shared identity across multiple resources |

### Role Assignment:
Managed Identity alone does nothing. You must assign it a **role** on a target resource (Key Vault, Storage, etc.) to grant specific permissions.

```
VM Managed Identity
    → assigned role "Storage Blob Data Reader"
    → on Storage Account "storeworkshop01"
    = VM can now read blobs from that storage account
```

---

## 8. Azure Key Vault

### What it is:
- Secure, centralized store for **secrets, keys, and certificates**.
- Access controlled via Azure RBAC or Key Vault access policies.
- All access is logged and auditable.

### Three things it stores:
- **Secrets** — passwords, connection strings, API keys
- **Keys** — cryptographic keys for encryption
- **Certificates** — SSL/TLS certificates

### Why use it instead of environment variables:
- Secrets rotate without redeploying your app
- Access is controlled per identity
- Full audit trail of who accessed what and when
- No secrets in code, config files, or environment

### Access pattern in this workshop:
1. VM's Managed Identity has `Key Vault Secrets User` role on the vault
2. App requests token from metadata endpoint
3. App calls Key Vault API with token
4. Key Vault validates token and returns secret
5. App uses secret to connect to DB — never stored anywhere

---

## 9. Azure Storage Account

### What it is:
- Massively scalable object/file/queue/table storage.
- **Blob Storage** = unstructured object storage (equivalent to S3).

### Access tiers:
- **Hot** — frequently accessed data
- **Cool** — infrequently accessed, cheaper storage
- **Archive** — rarely accessed, cheapest, high retrieval latency

### Authentication methods:
- **Access Key** — like a root password. Full access. Not recommended for apps.
- **SAS Token** — scoped, time-limited token. Better.
- **Managed Identity + RBAC** — best practice. No credentials stored.

### Built-in roles for Storage:
- `Storage Blob Data Reader` — read blobs only
- `Storage Blob Data Contributor` — read, write, delete blobs
- `Storage Blob Data Owner` — full control including ACLs

---

## 10. VNet Peering

### What it is:
- Connects two separate VNets so resources can communicate privately.
- Traffic stays on Azure backbone — low latency, no public internet.
- Peering is **non-transitive by default**.

### Non-transitive means:
- VNet A peered with VNet B ✅
- VNet B peered with VNet C ✅
- VNet A **cannot** talk to VNet C through B ❌
- You need direct peering between A and C, OR a hub firewall/router in B that forwards traffic.

### Hub and Spoke:
- **Hub VNet** — central VNet, contains shared services (firewall, DNS, VPN gateway)
- **Spoke VNets** — individual workloads peer to the hub
- Traffic between spokes goes through the hub's firewall (which enables transitivity)

---

# PART 2 — STUDENT QUESTIONS & ANSWERS

---

## Section A — Networking Questions

**Q: If the VNet is private, why don't we need an Internet Gateway like in AWS?**

In AWS, you explicitly attach an Internet Gateway to a VPC to enable internet connectivity. Azure's VNet model is different — outbound internet access is **built in by default**. Your VMs can reach the internet outbound without any gateway configuration. For inbound, you just need a public IP and an NSG rule. Azure abstracts the gateway concept away from you.

---

**Q: When do I actually need a NAT Gateway in Azure?**

When your VMs have **no public IP** and need **controlled outbound internet access** with a stable, known IP address. By default, private VMs get outbound internet via Azure's shared pool of IPs (called default outbound access), but that is being deprecated for new VNets. A NAT Gateway gives you a dedicated static public IP for outbound — useful when external services whitelist your IP. For this workshop, you don't need it.

---

**Q: Do I need route tables for this three-tier setup?**

No. Azure automatically handles routing between subnets within a VNet using **system routes**. Route tables (User Defined Routes / UDRs) are only needed when you want to override the default routing — for example, sending all traffic through an Azure Firewall appliance, or routing to on-premises via a VPN. For subnet-to-subnet isolation, NSG rules are your tool, not route tables.

---

**Q: What's the difference between NSG at subnet level vs NIC level?**

Subnet-level NSG applies to **all VMs** in that subnet. NIC-level NSG applies to **one specific VM**. If both exist, both are evaluated. For inbound traffic, Azure checks the subnet NSG first, then the NIC NSG. For outbound, NIC NSG first, then subnet NSG. Best practice is to use subnet-level NSGs and avoid NIC-level unless you need per-VM exceptions.

---

**Q: Can two resource groups in the same VNet communicate?**

A VNet is not scoped to a resource group. A VNet spans a subscription. Resources in different resource groups can be in the same VNet and communicate freely — resource groups are just a management and billing boundary, not a network boundary.

---

**Q: What if I want the frontend to be unable to talk to the database at all, even accidentally?**

The database subnet's NSG has an inbound rule allowing traffic only from the App subnet's IP range (e.g., `10.0.2.0/24`). The frontend subnet is `10.0.1.0/24`. Since that source address is not in the database NSG's allow rule, traffic from the frontend to the database is denied by the default `DenyAllInBound` rule. No route table needed — NSG handles this.

---

**Q: What happens to outbound traffic from a private VM with no public IP?**

By default, Azure allows outbound internet traffic from all VMs including private ones with no public IP. Azure uses **default outbound access** (a shared pool of public IPs). The VM can reach the internet — like calling the Azure Storage API or Key Vault — but nobody from the internet can initiate a connection back to it because it has no public IP and no inbound NSG rule.

---

**Q: What is a Private Endpoint and when do I need it?**

A Private Endpoint gives a PaaS service like Storage or Key Vault a **private IP address inside your VNet**. Instead of traffic leaving your VNet to reach the storage public endpoint on the internet, it stays entirely within your VNet. This is the most secure pattern for production. For this workshop we use the public endpoint for simplicity, but in production you would always use Private Endpoints.

---

## Section B — Identity & Security Questions

**Q: If I enable system-assigned Managed Identity on a VM, does it automatically have access to everything?**

No. Enabling Managed Identity just creates the identity in Azure AD. It has **zero permissions** by default. You must explicitly create a **role assignment** to grant it access to specific resources. No role assignment = no access, even if the identity exists.

---

**Q: What's the difference between Authentication and Authorization in this context?**

- **Authentication**: proving who you are. The Managed Identity token proves the VM's identity to Azure AD.
- **Authorization**: determining what you can do. The role assignment on the storage account or Key Vault determines what that identity is allowed to do.

Managed Identity handles authentication. RBAC role assignments handle authorization.

---

**Q: Can I assign multiple roles to one Managed Identity?**

Yes. You can assign as many roles as needed to a Managed Identity across different resources. For example, the App VM's identity can have `Storage Blob Data Reader` on Storage and `Key Vault Secrets User` on Key Vault simultaneously.

---

**Q: Can I create a custom role in Azure?**

Yes. If the built-in roles are too broad or too narrow, you define a custom role with exactly the permissions you need. A custom role is defined as a JSON object listing allowed and denied actions, the scope it applies to, and its name. For example, a role that allows only reading blobs but denies deleting them — more granular than `Storage Blob Data Contributor`.

---

**Q: How is Azure RBAC different from Key Vault Access Policies?**

Key Vault has two authorization models:
- **Legacy Access Policies**: Key Vault's own permission system, not RBAC. You grant access per-identity directly in Key Vault settings.
- **Azure RBAC** (recommended): Uses the same role assignment model as the rest of Azure. Consistent, auditable, centrally managed.

In this workshop we use `--enable-rbac-authorization true` when creating Key Vault, which means we use Azure RBAC.

---

**Q: What happens to the system-assigned Managed Identity when the VM is deleted?**

It is permanently deleted along with the VM. This is the key difference from user-assigned — with user-assigned you can detach the identity before deleting the VM and reuse it elsewhere.

---

**Q: Where does the VM actually get the token from when using Managed Identity?**

From the **Azure Instance Metadata Service (IMDS)** — a special non-routable IP address `169.254.169.254` that is only accessible from inside an Azure VM. When the VM calls this endpoint asking for a token for a specific resource (e.g., storage.azure.com), Azure returns a short-lived OAuth 2.0 bearer token. This token is what gets sent to the target service for authentication.

---

## Section C — Storage & Key Vault Questions

**Q: Can the frontend VM also access the storage account?**

It can if you assign the appropriate role to its Managed Identity. But in our architecture, only the App VM has Storage Blob Data Reader assigned. The frontend VM's identity has no role on the storage account, so it cannot access it. This enforces separation of concerns.

---

**Q: What if the Key Vault secret changes — does the app need to restart?**

Depends on your code. If you fetch the secret **once at startup** and cache it, a secret rotation requires a restart. Best practice is to fetch the secret **on each request** (or cache with a TTL like 5 minutes) so rotation takes effect automatically without restarting the app.

---

**Q: Why store the DB password in Key Vault if Managed Identity can authenticate to PostgreSQL directly?**

For Azure-native managed databases like **Azure Database for PostgreSQL**, yes — you can use Managed Identity to authenticate directly without a password. But for a self-hosted PostgreSQL running on a VM (as in this workshop), Managed Identity is not supported at the database level. PostgreSQL on a VM uses traditional username/password. Key Vault securely stores that password so it's not hardcoded in config files or environment variables.

---

**Q: Is there a cost for Key Vault?**

Yes, but very small. Azure charges per 10,000 operations. For a workshop, this will be negligible — a few paise at most. It falls within free tier limits for the duration of the workshop.

---

## Section D — VM & Compute Questions

**Q: What size VM should we use in the free tier?**

`Standard_B1s` — 1 vCPU, 1 GB RAM. Free tier gives you 750 hours per month of B1s. Since you have three VMs per attendee, watch the total hours. For nine attendees × 3 VMs = 27 VMs. Best to run them only during the workshop and delete immediately after.

---

**Q: Can we use App Service instead of VMs?**

Yes, and in production that's often better. App Service is a managed compute platform — no VM to maintain, auto-scaling built in, managed Managed Identity support. But for learning fundamentals including networking, VMs are better because they make the networking concepts (subnets, NSGs, private IPs) visible and tangible. App Services abstract most of that away.

---

**Q: Why did we use Ubuntu and not Windows Server?**

Ubuntu is lighter, faster to provision, and the commands are simpler for a workshop. Windows Server VMs are larger, slower to start, and cost more compute hours — not ideal for a free tier hands-on lab.

---

**Q: Can the App VM SSH into the DB VM?**

Only if you allow SSH (port 22) in the DB subnet's NSG inbound rules from the App subnet. By default the NSG we configured only allows port 5432 (PostgreSQL). You can add an SSH rule from the App subnet IP range if needed for debugging during the workshop.

---

## Section E — General Architecture Questions

**Q: Why three tiers? Why not put everything on one VM?**

Single VM is simpler but violates separation of concerns. With three tiers:
- **Security**: database is never directly exposed
- **Scalability**: you can scale each tier independently
- **Maintainability**: update the API without touching the database
- **Fault isolation**: a frontend crash doesn't bring down the database

---

**Q: What happens if the Load Balancer's health probe fails on a VM?**

The Load Balancer stops sending traffic to that VM. It continues checking every probe interval (default 5 seconds). Once the VM responds healthy again, it re-enters the rotation. This gives you zero-downtime deployments and automatic failure recovery.

---

**Q: Is our app highly available?**

No — we have one VM per tier. If any VM goes down, the app is down. For high availability you'd deploy multiple VMs per tier in an **Availability Set** or **Availability Zones**, and the Load Balancer distributes across them. That's a follow-up topic beyond this workshop.

---

**Q: What is the difference between VNet Peering and VPN Gateway?**

| VNet Peering | VPN Gateway |
|---|---|
| Connects Azure VNets to each other | Connects Azure VNet to on-premises or another cloud |
| Low latency, high bandwidth | Higher latency, limited bandwidth |
| No encryption (relies on Azure backbone security) | Encrypted tunnel (IPSec) |
| Simple to set up | Complex, more expensive |

---

**Q: Can two VMs in different subnets of the same VNet talk to each other without any NSG rules?**

Yes, by default. Azure's default NSG rule `AllowVNetInBound` allows all traffic within the VNet address space. You only restrict this when you **explicitly add deny rules** or remove the default allow rule — which is what we do by tightening the NSG rules per subnet.

---

## Section F — AWS vs Azure Comparison (for students who know AWS)

| AWS Concept | Azure Equivalent |
|---|---|
| VPC | Virtual Network (VNet) |
| Subnet | Subnet |
| Security Group | NSG (Network Security Group) |
| NACL | NSG with explicit deny rules |
| Internet Gateway | Built into VNet (not explicit) |
| NAT Gateway | Azure NAT Gateway |
| Route Table | User Defined Routes (UDR) |
| IAM Role | Azure Role (RBAC) |
| IAM Policy | Azure Permission / custom role |
| Instance Profile | Managed Identity |
| S3 | Azure Blob Storage |
| Secrets Manager | Azure Key Vault |
| EC2 | Azure Virtual Machine |
| ALB (Application Load Balancer) | Azure Application Gateway |
| NLB (Network Load Balancer) | Azure Load Balancer (Standard) |
| VPC Peering | VNet Peering |
| Transit Gateway | Azure Virtual WAN / Hub VNet |
| EKS | Azure Kubernetes Service (AKS) |

---

# PART 3 — INSTRUCTOR TIPS

- **Start with the architecture diagram** — draw it on the whiteboard before touching the portal. Let them see where everything fits.
- **Explain NSGs with traffic flow** — don't just say "it's a firewall." Walk through a packet from browser to database and show which NSG checks happen at each hop.
- **Do one full demo first**, then let attendees repeat it. Live mistakes are valuable learning moments.
- **Pre-create resource groups** and assign roles before the session — it saves 20 minutes of IAM confusion.
- **Set budget alerts** on the subscription before the workshop so you get notified if anyone accidentally deploys something expensive.
- **Have a cleanup script ready** — a single bash loop to delete all nine resource groups at the end.

```bash
for i in $(seq -w 1 9); do
  az group delete --name rg-workshop-user0$i --yes --no-wait
done
```

---

*Prepared for Azure Fundamentals 3-Hour Hands-On Workshop*
