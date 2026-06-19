# PHASE 1 — ARCHITECTURE & DESIGN

🟢 **Can claim:** You operate inside an architecture like this daily — you know how AKS, ingress, observability, and the GitOps flow fit together from the *operating* side. 🟡 **Learning:** *Choosing* the isolation model, *designing* the repo strategy, and *deciding* the promotion flow are architect/lead calls. Frame as "I understand why it's built this way and I work within it," not "I designed it."

---

## 1.1 — Target Architecture (high-level + low-level)

**Why this way / tradeoffs:** The architecture has to satisfy the NFRs from Phase 0 *traceably*: 99.9% availability (NFR-1) → multi-AZ + multi-region DR; data residency (NFR-6) → US-only regions; SOC2 (NFR-7) → private API, audit, separation of duties; legacy coexistence (C-1) → Windows node pool behind the *same* ingress and observability. Alternatives rejected: a single flat cluster with everything (no blast-radius isolation, fails SOC2 separation); fully separate stacks for legacy vs modern (double the operational surface, two observability planes — the opposite of what we want).

Let me show the target architecture.

<img width="1472" height="1240" alt="image" src="https://github.com/user-attachments/assets/11a41470-bd27-4441-80f5-3f2af77e3a66" />

1.2 — Environment Isolation Model (the decision)
Why this way / tradeoffs: This is the question the prompt explicitly asks to decide and justify — multi-cluster vs namespace. The two failure modes are: namespaces-only (cheap, but a cluster-wide misconfig or a noisy-neighbor incident hits dev and prod — unacceptable for SOC2 separation of duties), or cluster-per-environment-per-app (perfect isolation, but operational and cost explosion).
The defensible middle, and the SOC2-compliant one:

```
Decision: CLUSTER-PER-ENVIRONMENT, NAMESPACE-PER-APP within each.

  ┌─ aks-dev      (1 cluster)  → ns: appointments, notifications, records, legacy
  ┌─ aks-staging  (1 cluster)  → ns: appointments, notifications, records, legacy
  └─ aks-prod     (1 cluster)  → ns: appointments, notifications, records, legacy
        (prod also: multi-AZ, DR replica in West US 2)
```

Why three clusters (one per env), not one cluster with dev/staging/prod namespaces:

Blast radius — a bad cluster upgrade, a CRD change, or a runaway workload in dev cannot touch prod. They're separate control planes.
SOC2 separation of duties — prod cluster access is gated to a different group than dev; you can prove "developers cannot touch production" with Azure RBAC at the cluster boundary, which is exactly what an auditor wants to see.
Realistic parity — staging mirrors prod's topology (same node pools, same ingress) so a change that works in staging behaves the same in prod.

Why namespace-per-app within a cluster, not a cluster per app:

Apps within an environment share the same trust boundary and the same observability/ingress, so namespaces give enough isolation (RBAC, network policies, resource quotas) without N clusters to operate.

Assumption: dev can be single-AZ and cheaper (smaller node counts, no DR); only prod gets the full multi-AZ + DR treatment, since NFR-1/4/5 (availability, RTO, RPO) are prod requirements.

1.3 — Branching & Repo Strategy
Why this way / tradeoffs: The core decision is separating application code repos from the GitOps/config repo. If you put deployment manifests in the same repo as app code, every config change triggers an app rebuild, the deploy history is tangled with the code history, and you can't enforce "who can change what's deployed" separately from "who can change code" — which SOC2 separation of duties requires. Alternatives rejected: monorepo-for-everything (coupling, slow CI, no clean audit boundary); manifests-inside-each-app-repo (ArgoCD would watch many repos, no single source of deployed truth).
Branching: trunk-based, not GitFlow. GitFlow's long-lived develop/release branches create merge debt and slow the dev→prod flow. Trunk-based (short-lived feature branches → main, with environment promotion handled by the GitOps repo, not by branches) is the modern default and pairs naturally with GitOps.
App repo (one per microservice — e.g. appointments-api)

```
appointments-api/
├── .github/
│   └── workflows/
│       └── ci.yaml                 # GitHub Actions CI (build/test/scan/sign/push)
├── src/
│   └── ...                         # application code
├── tests/
│   └── ...
├── Dockerfile
├── .gitleaks.toml                  # secret-scan config
├── sonar-project.properties        # SAST quality gate config
└── README.md
```
Legacy app repo (recordsmgr — built via Azure DevOps)
```
recordsmgr/
├── azure-pipelines.yml             # Azure DevOps CI for the legacy Windows app
├── src/                            # .NET Framework source
├── Dockerfile.windows              # Windows container image
└── README.md
```

GitOps / config repo (single source of deployed truth — platform-gitops)

```
platform-gitops/
├── bootstrap/
│   └── root-app.yaml               # ArgoCD app-of-apps entrypoint
├── applicationsets/
│   ├── microservices-appset.yaml   # generates Apps across envs/clusters
│   └── legacy-appset.yaml
├── charts/                         # shared Helm library chart (Phase 5)
│   └── common-lib/
├── apps/
│   ├── appointments-api/
│   │   ├── base/                   # base Helm values
│   │   └── overlays/
│   │       ├── dev/values.yaml
│   │       ├── staging/values.yaml
│   │       └── prod/values.yaml    # image tag updated here by CI PR
│   ├── notifications-svc/
│   ├── records-read-api/
│   └── recordsmgr/                 # legacy, same structure — same GitOps flow
└── platform/                       # platform components as code
    ├── ingress-nginx/
    ├── cert-manager/
    ├── kube-prometheus-stack/      # observability (Phase 8)
    └── kyverno/                    # admission policies (Phase 9)

```

Infra repo (Terraform — platform-infra)

```
platform-infra/
├── modules/                        # reusable modules (Phase 2/3)
│   ├── network/
│   ├── aks/
│   ├── acr/
│   ├── keyvault/
│   └── identity/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf              # remote state, per-env key
│   ├── staging/
│   └── prod/
└── global/
    └── state-backend/              # bootstrap: the storage account for state itself
```
Why dir-per-env (not Terraform workspaces): Separate directories with separate state files and separate .tfvars make it impossible to accidentally apply dev config to prod (the workspace footgun). Each env's state is isolated, and prod's directory can have stricter branch protection. This is the senior call and it directly supports SOC2 (you can prove prod changes go through a separate, reviewed path).

1.4 — Release & Promotion Strategy
Why this way / tradeoffs: Promotion must be the same artifact moving forward, not a rebuild per environment. If you rebuild for staging and again for prod, you can't guarantee the prod image is what you tested in staging. So: build the image once, tag it immutably (git SHA), and promote the tag through environments by updating the GitOps overlay — never rebuild.

```
Promotion flow (one immutable artifact, promoted by GitOps PR):

  commit → CI builds image  ──►  ghcr/ACR: appointments-api:sha-a1b2c3  (immutable)
                                          │
        dev overlay tag = sha-a1b2c3  ◄───┤  (CI auto-PR to gitops repo, dev)
        ArgoCD syncs dev                  │
                                          │
        staging overlay tag = sha-a1b2c3 ◄┤  (PR, requires 1 reviewer approval)
        ArgoCD syncs staging              │
                                          │
        prod overlay tag = sha-a1b2c3   ◄─┘  (PR, requires CODEOWNERS + prod approval)
        ArgoCD syncs prod (Argo Rollouts canary — Phase 7)
```

Key decisions:

Image tagging = immutable git SHA, never latest. You can always trace exactly which commit is running in prod (and Kyverno will enforce no-latest in Phase 9). This is both an operational and a SOC2-audit requirement.
Environment parity — the same Helm chart deploys to all three envs; only the overlay values.yaml differs (replica counts, resource sizes, hostnames). Same chart everywhere means staging genuinely predicts prod.
Promotion = a PR to the GitOps repo that bumps the image tag in the next overlay. Dev is auto-promoted; staging needs one reviewer; prod needs CODEOWNERS + an environment approval (Phase 6 details the gates). This makes every promotion an audited Git event — exactly the change-management trail SOC2 wants.
Feature flags over long branches — risky/incomplete features ship dark behind a flag rather than living on a long-lived branch. This keeps trunk-based development clean and decouples deploy from release.


---
# PHASE 2 — AZURE FOUNDATION (Terraform)

🟡 **Learning (mostly):** This is landing-zone authoring — architect/platform-lead territory. Your real experience is *modifying* Terraform variables and *contributing* during a migration, not writing modules from scratch. Study this to understand *what* the foundation is and *why* each piece exists; the honest interview line is "I work within a Terraform landing zone like this and understand its structure — authoring it from zero is something I'm building toward." 🟢 The one genuinely claimable thing: you understand state, locking, and the no-static-secrets principle, because those concepts came up in your real work.

> **Assumptions for this phase:** Single Azure subscription per environment (dev/staging/prod) under a management group — simpler than multi-sub and sufficient at this scale; I'll note where you'd split. Region `eastus2` primary, `westus2` DR. Org prefix `pp` (PatientPortal). Secrets and org-specific IDs are placeholders.

---

## 2.0 — State backend bootstrap (the chicken-and-egg)

**Why this way / tradeoffs:** Terraform needs a remote backend with locking *before* it can manage anything safely — but the backend (a storage account) is itself Azure infra. You solve the chicken-and-egg by creating the state storage **once, manually or with a tiny local-state bootstrap**, then every other config uses it. Alternatives rejected: local state (no locking, no team safety, secrets on a laptop — covered in your Q&A bank as the #1 Terraform mistake); committing state to Git (plaintext secrets leak).

```hcl
# platform-infra/global/state-backend/main.tf
# Run ONCE with local state, then all other envs use this backend.

terraform {
  required_version = ">= 1.7"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.110" }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "tfstate" {
  name     = "pp-tfstate-rg"
  location = "eastus2"
  tags     = { purpose = "terraform-state", managed_by = "terraform" }
}

resource "azurerm_storage_account" "tfstate" {
  name                            = "pptfstate${var.unique_suffix}" # globally unique
  resource_group_name             = azurerm_resource_group.tfstate.name
  location                        = azurerm_resource_group.tfstate.location
  account_tier                    = "Standard"
  account_replication_type        = "GRS"        # geo-redundant: survives region loss
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  blob_properties {
    versioning_enabled = true                     # state history = recovery from corruption
    delete_retention_policy { days = 30 }
  }
  tags = { purpose = "terraform-state" }
}

resource "azurerm_storage_container" "tfstate" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.tfstate.name
  container_access_type = "private"
}
```

```hcl
# platform-infra/envs/prod/backend.tf
# Every env points at the shared backend with a DIFFERENT key (state isolation).
terraform {
  backend "azurerm" {
    resource_group_name  = "pp-tfstate-rg"
    storage_account_name = "pptfstateXXXXXX"
    container_name       = "tfstate"
    key                  = "prod/foundation.tfstate"  # dev/, staging/ get their own keys
  }
}
```

**Why blob versioning + GRS:** versioning gives you state-file history to recover from a corrupted/bad apply (the recovery answer in your Q&A bank); GRS means the state itself survives a regional outage — you can't run a DR recovery if your state is gone with the region. Locking is automatic — the `azurerm` backend uses a **blob lease**, which is the mechanism that stops two concurrent `apply`s from corrupting state.

---

## 2.1 — Naming, tagging & Azure Policy guardrails

**Why this way / tradeoffs:** A naming convention and mandatory tags aren't bureaucracy — they're what makes cost allocation, ownership, and SOC2 audit *possible*. Azure Policy *enforces* them so a human can't forget. Alternatives rejected: free-form naming (you can't find anything, can't attribute cost); tags-by-convention-only (people forget, audit fails — enforce in code).

```hcl
# platform-infra/modules/naming/main.tf
# Convention: pp-<env>-<workload>-<resourcetype>-<region>
# e.g. pp-prod-aks-eastus2, pp-prod-kv-eastus2

locals {
  prefix = "pp-${var.env}"
  region_short = { eastus2 = "eus2", westus2 = "wus2" }[var.location]

  # Mandatory tags applied to EVERY resource (enforced by Azure Policy below)
  common_tags = {
    environment = var.env
    workload    = "patientportal"
    managed_by  = "terraform"
    cost_center = var.cost_center
    data_class  = var.data_class      # "phi" | "internal" | "public" — drives SOC2 scope
    owner       = var.owner_team
  }
}

output "prefix"      { value = local.prefix }
output "common_tags" { value = local.common_tags }
```

```hcl
# platform-infra/modules/governance/policy.tf
# Azure Policy guardrails — enforced, not advisory.

# 1) Deny resources missing the mandatory "data_class" tag
resource "azurerm_policy_definition" "require_data_class_tag" {
  name         = "require-data-class-tag"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require data_class tag on all resources"

  policy_rule = jsonencode({
    if = {
      field  = "tags['data_class']"
      exists = "false"
    }
    then = { effect = "deny" }
  })
}

# 2) Deny any resource created outside approved US regions (NFR-6 data residency)
resource "azurerm_policy_definition" "allowed_locations" {
  name         = "allowed-locations-us-only"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "Allow only East US 2 and West US 2 (PHI residency)"

  policy_rule = jsonencode({
    if = {
      not = {
        field = "location"
        in    = ["eastus2", "westus2"]
      }
    }
    then = { effect = "deny" }
  })
}

resource "azurerm_subscription_policy_assignment" "residency" {
  name                 = "pp-residency"
  subscription_id      = var.subscription_id
  policy_definition_id = azurerm_policy_definition.allowed_locations.id
}
```

**The senior point:** policy #2 is NFR-6 (PHI data residency) turned into an *enforced control* — no engineer can accidentally provision PHI infra in Europe. That traceability from requirement → enforced policy is exactly what a SOC2 auditor (and a sharp interviewer) wants to see.

---

## 2.2 — Networking module

**Why this way / tradeoffs:** The big decision here is **Azure CNI vs kubenet**, and **private vs public** everything. For a PHI/SOC2 platform, the cluster API and the data services must not be reachable from the public internet — so private endpoints and a private API server. Egress goes through a controlled NAT/Firewall so you can log and restrict outbound (SOC2 wants known egress). Alternatives rejected: kubenet (covered below); public API server (unacceptable attack surface for PHI); default open egress (no control/audit of what the cluster talks to).

```hcl
# platform-infra/modules/network/main.tf

resource "azurerm_resource_group" "net" {
  name     = "${var.prefix}-network-rg"
  location = var.location
  tags     = var.common_tags
}

resource "azurerm_virtual_network" "main" {
  name                = "${var.prefix}-vnet"
  location            = azurerm_resource_group.net.location
  resource_group_name = azurerm_resource_group.net.name
  address_space       = ["10.10.0.0/16"]
  tags                = var.common_tags
}

# AKS nodes + pods (Azure CNI consumes real VNet IPs — size generously)
resource "azurerm_subnet" "aks" {
  name                 = "${var.prefix}-aks-snet"
  resource_group_name  = azurerm_resource_group.net.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.10.0.0/20"]   # /20 = 4096 IPs (Azure CNI is IP-hungry)
}

# Private endpoints for PaaS (SQL, Key Vault, ACR) — keeps PHI traffic off the internet
resource "azurerm_subnet" "pe" {
  name                 = "${var.prefix}-pe-snet"
  resource_group_name  = azurerm_resource_group.net.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.10.16.0/24"]
}

# Dedicated subnet for the NAT gateway / firewall egress path
resource "azurerm_subnet" "egress" {
  name                 = "${var.prefix}-egress-snet"
  resource_group_name  = azurerm_resource_group.net.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.10.17.0/24"]
}

# NSG on the AKS subnet — default-deny inbound, allow only what's needed
resource "azurerm_network_security_group" "aks" {
  name                = "${var.prefix}-aks-nsg"
  location            = azurerm_resource_group.net.location
  resource_group_name = azurerm_resource_group.net.name
  tags                = var.common_tags

  security_rule {
    name                       = "allow-https-from-appgw"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "10.10.18.0/24"  # app gateway subnet
    destination_address_prefix = "*"
  }
  # All other inbound implicitly denied by Azure's default rules.
}

resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks.id
}

# Controlled egress: NAT gateway gives a stable outbound IP + connection scaling
resource "azurerm_public_ip" "nat" {
  name                = "${var.prefix}-nat-pip"
  location            = azurerm_resource_group.net.location
  resource_group_name = azurerm_resource_group.net.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
  tags                = var.common_tags
}

resource "azurerm_nat_gateway" "main" {
  name                = "${var.prefix}-nat"
  location            = azurerm_resource_group.net.location
  resource_group_name = azurerm_resource_group.net.name
  sku_name            = "Standard"
  tags                = var.common_tags
}

resource "azurerm_nat_gateway_public_ip_association" "main" {
  nat_gateway_id       = azurerm_nat_gateway.main.id
  public_ip_address_id = azurerm_public_ip.nat.id
}

resource "azurerm_subnet_nat_gateway_association" "aks" {
  subnet_id      = azurerm_subnet.aks.id
  nat_gateway_id = azurerm_nat_gateway.main.id
}

# Private DNS zones so private endpoints resolve to private IPs inside the VNet
resource "azurerm_private_dns_zone" "sql" {
  name                = "privatelink.database.windows.net"
  resource_group_name = azurerm_resource_group.net.name
  tags                = var.common_tags
}

resource "azurerm_private_dns_zone" "kv" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = azurerm_resource_group.net.name
  tags                = var.common_tags
}

resource "azurerm_private_dns_zone" "acr" {
  name                = "privatelink.azurecr.io"
  resource_group_name = azurerm_resource_group.net.name
  tags                = var.common_tags
}

resource "azurerm_private_dns_zone_virtual_network_link" "sql" {
  name                  = "${var.prefix}-sql-dnslink"
  resource_group_name   = azurerm_resource_group.net.name
  private_dns_zone_name = azurerm_private_dns_zone.sql.name
  virtual_network_id    = azurerm_virtual_network.main.id
}

output "aks_subnet_id"     { value = azurerm_subnet.aks.id }
output "pe_subnet_id"      { value = azurerm_subnet.pe.id }
output "vnet_id"           { value = azurerm_virtual_network.main.id }
output "dns_zone_sql_id"   { value = azurerm_private_dns_zone.sql.id }
output "dns_zone_kv_id"    { value = azurerm_private_dns_zone.kv.id }
output "dns_zone_acr_id"   { value = azurerm_private_dns_zone.acr.id }
```

**The Azure CNI vs kubenet decision (stated explicitly):**

```
Decision: AZURE CNI (overlay mode where IP space is tight).

WHY CNI over kubenet for this platform:
- Pods get real VNet IPs → directly routable, integrates with private endpoints,
  NSGs, and network policy cleanly. Required for the private-PaaS PHI design.
- Better performance, no extra NAT hop for pod traffic.

THE COST (and mitigation):
- Azure CNI is IP-hungry — every pod consumes a VNet IP, nodes pre-allocate blocks.
  This is why the AKS subnet is a /20 (4096 IPs).
- Mitigation: use Azure CNI OVERLAY mode — pods get IPs from an overlay CIDR, not
  the VNet, so you get CNI's integration without exhausting the subnet. Best of both.

kubenet rejected: NAT hop, routing limitations, weaker integration with the
private-endpoint / network-policy model this compliance posture needs.
```

This is straight out of your Q&A bank (the CNI-vs-kubenet answer) — you can speak to this tradeoff credibly because you've operated on Azure CNI.

---

## 2.3 — Identity module (workload identity / OIDC — no static secrets)

**Why this way / tradeoffs:** The entire point is **zero stored secrets**. Pods authenticate to Azure (Key Vault, SQL) via **workload identity** — a federated credential where the Kubernetes ServiceAccount token is exchanged for a short-lived Azure AD token. No service-principal secret to store, leak, or rotate. Alternatives rejected: service principal with a client secret (long-lived secret = a SOC2 finding and a rotation burden — exactly the anti-pattern your Q&A bank calls out); AAD Pod Identity (deprecated).

```hcl
# platform-infra/modules/identity/main.tf

# User-assigned managed identity that pods will federate to
resource "azurerm_user_assigned_identity" "workload" {
  name                = "${var.prefix}-${var.app_name}-wi"
  resource_group_name = var.rg_name
  location            = var.location
  tags                = var.common_tags
}

# Federate the K8s ServiceAccount with this identity (OIDC trust).
# The AKS cluster's OIDC issuer URL comes from the AKS module (Phase 3).
resource "azurerm_federated_identity_credential" "workload" {
  name                = "${var.app_name}-fedcred"
  resource_group_name = var.rg_name
  parent_id           = azurerm_user_assigned_identity.workload.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = var.aks_oidc_issuer_url
  subject             = "system:serviceaccount:${var.namespace}:${var.app_name}-sa"
}

# Grant this identity least-privilege access to Key Vault secrets (read only)
resource "azurerm_role_assignment" "kv_secrets_user" {
  scope                = var.key_vault_id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.workload.principal_id
}

output "client_id"    { value = azurerm_user_assigned_identity.workload.client_id }
output "principal_id" { value = azurerm_user_assigned_identity.workload.principal_id }
```

**The federation chain, in words (worth being able to narrate):** the pod's ServiceAccount → projected OIDC token → exchanged at Azure AD (which trusts the cluster's OIDC issuer via the federated credential) → short-lived Azure AD token → access to Key Vault scoped by RBAC. No secret anywhere in that chain. This is the modern, SOC2-friendly pattern, and it's the same "Workload Identity = no stored secrets, short-lived tokens" point from your Azure Q&A.

---

## 2.4 — Key Vault, ACR, Storage modules

**Why this way / tradeoffs:** Key Vault holds secrets (accessed via workload identity + the CSI driver in Phase 9), with RBAC authorization and purge protection (a SOC2 control — you can't permanently destroy audit-relevant key material). ACR is **Premium** because Premium is required for geo-replication (DR image availability in West US 2) and for the built-in vulnerability scanning the pipeline relies on. Alternatives rejected: access-policy-based Key Vault (RBAC is the modern, auditable model); Basic/Standard ACR (no geo-replication, no DR image availability — breaks NFR-4).

```hcl
# platform-infra/modules/keyvault/main.tf

resource "azurerm_key_vault" "main" {
  name                          = "${var.prefix}-kv"
  location                      = var.location
  resource_group_name           = var.rg_name
  tenant_id                     = var.tenant_id
  sku_name                      = "standard"
  enable_rbac_authorization     = true     # RBAC, not access policies (auditable)
  purge_protection_enabled      = true     # SOC2: secrets can't be hard-deleted
  soft_delete_retention_days    = 90
  public_network_access_enabled = false    # private endpoint only
  tags                          = var.common_tags

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }
}

# Private endpoint so Key Vault is reachable only inside the VNet
resource "azurerm_private_endpoint" "kv" {
  name                = "${var.prefix}-kv-pe"
  location            = var.location
  resource_group_name = var.rg_name
  subnet_id           = var.pe_subnet_id
  tags                = var.common_tags

  private_service_connection {
    name                           = "${var.prefix}-kv-psc"
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }
  private_dns_zone_group {
    name                 = "kv-dns"
    private_dns_zone_ids = [var.dns_zone_kv_id]
  }
}

output "key_vault_id"  { value = azurerm_key_vault.main.id }
```

```hcl
# platform-infra/modules/acr/main.tf

resource "azurerm_container_registry" "main" {
  name                          = "${var.prefix_compact}acr"   # alphanumeric only
  resource_group_name           = var.rg_name
  location                      = var.location
  sku                           = "Premium"   # required for geo-replication + scanning
  admin_enabled                 = false        # no admin user; pull via managed identity
  public_network_access_enabled = false
  tags                          = var.common_tags

  georeplications {
    location = "westus2"                        # DR image availability (NFR-4)
    tags     = var.common_tags
  }

  retention_policy {
    days    = 30
    enabled = true
  }
  trust_policy {
    enabled = true                              # content trust — supports signed images
  }
}

# Private endpoint for ACR (nodes pull over the private network)
resource "azurerm_private_endpoint" "acr" {
  name                = "${var.prefix}-acr-pe"
  location            = var.location
  resource_group_name = var.rg_name
  subnet_id           = var.pe_subnet_id
  tags                = var.common_tags

  private_service_connection {
    name                           = "${var.prefix}-acr-psc"
    private_connection_resource_id = azurerm_container_registry.main.id
    subresource_names              = ["registry"]
    is_manual_connection           = false
  }
  private_dns_zone_group {
    name                 = "acr-dns"
    private_dns_zone_ids = [var.dns_zone_acr_id]
  }
}

output "acr_id"          { value = azurerm_container_registry.main.id }
output "acr_login_server"{ value = azurerm_container_registry.main.login_server }
```

---

## 2.5 — Env composition (how the modules wire together)

**Why this way / tradeoffs:** The env directory is *thin* — it just calls modules with env-specific variables. All logic lives in modules (DRY); the env files express *what this environment is*, not *how to build it*. This is the module pattern from your Q&A bank made concrete.

```hcl
# platform-infra/envs/prod/main.tf

terraform {
  required_version = ">= 1.7"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.110" }
  }
}
provider "azurerm" { features {} }

module "naming" {
  source      = "../../modules/naming"
  env         = "prod"
  location    = var.location
  cost_center = var.cost_center
  data_class  = "phi"
  owner_team  = "platform"
}

module "network" {
  source       = "../../modules/network"
  prefix       = module.naming.prefix
  location     = var.location
  common_tags  = module.naming.common_tags
}

module "keyvault" {
  source         = "../../modules/keyvault"
  prefix         = module.naming.prefix
  location       = var.location
  rg_name        = module.network.net_rg_name
  tenant_id      = var.tenant_id
  pe_subnet_id   = module.network.pe_subnet_id
  dns_zone_kv_id = module.network.dns_zone_kv_id
  common_tags    = module.naming.common_tags
}

module "acr" {
  source          = "../../modules/acr"
  prefix          = module.naming.prefix
  prefix_compact  = "ppprod"
  location        = var.location
  rg_name         = module.network.net_rg_name
  pe_subnet_id    = module.network.pe_subnet_id
  dns_zone_acr_id = module.network.dns_zone_acr_id
  common_tags     = module.naming.common_tags
}

# module "aks" { ... }  ← Phase 3
```

```hcl
# platform-infra/envs/prod/variables.tf
variable "location"        { type = string,  default = "eastus2" }
variable "tenant_id"       { type = string }
variable "cost_center"     { type = string,  default = "cc-health-1234" }
```

```hcl
# platform-infra/envs/prod/terraform.tfvars
location    = "eastus2"
tenant_id   = "00000000-0000-0000-0000-000000000000"  # placeholder
cost_center = "cc-health-1234"
```

---

# PHASE 3 — AKS CLUSTER (Terraform)

🟡 **Learning (mostly):** AKS cluster authoring is platform-lead work. 🟢 **But this is your closest-to-real phase** — you *operate* a cluster with these exact components (node pools, autoscaling, RBAC, ingress, upgrades). You can speak to *how these behave in production* with real authority, even if you didn't write the Terraform. That operational knowledge is genuinely yours — lean on it hard here.

> **Assumptions:** Private API server (PHI/SOC2). Four node pools: system (Linux), user (Linux microservices), spot (Linux, non-critical batch), Windows (legacy RecordsMgr). Cilium for network policy + eBPF dataplane. ingress-nginx as the controller. cert-manager with Let's Encrypt for TLS. Pod Security Standards enforced at `restricted` for app namespaces.

---

## 3.1 — The AKS module

**Why this way / tradeoffs:** A private API server (no public endpoint) is non-negotiable for PHI — the control plane is reachable only from inside the VNet (or via a jumpbox/private link). Azure RBAC + Entra integration means cluster access is governed by Entra groups, not static kubeconfig certs — that's the SOC2 separation-of-duties control at the cluster boundary. Workload identity + OIDC issuer is enabled so the Phase 2 federated credentials actually work. Alternatives rejected: public API with authorized IP ranges (smaller attack surface than fully open, but still public — rejected for PHI); local accounts / static admin kubeconfig (un-auditable, can't tie actions to a human — disabled).

```hcl
# platform-infra/modules/aks/main.tf

resource "azurerm_resource_group" "aks" {
  name     = "${var.prefix}-aks-rg"
  location = var.location
  tags     = var.common_tags
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "${var.prefix}-aks"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = "${var.prefix}-aks"
  kubernetes_version  = var.k8s_version
  sku_tier            = "Standard"          # Standard = SLA-backed control plane (prod)

  # --- Private, locked-down API server ---
  private_cluster_enabled             = true
  private_cluster_public_fqdn_enabled = false
  local_account_disabled              = true   # no static admin kubeconfig — Entra only

  # --- Entra (Azure AD) + Azure RBAC for cluster access ---
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true             # K8s authz governed by Azure RBAC
    admin_group_object_ids = var.admin_group_object_ids  # Entra group, per-env
  }

  # --- Workload identity + OIDC issuer (powers Phase 2 federated creds) ---
  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  # --- System node pool (runs CoreDNS, metrics-server, ingress, ArgoCD) ---
  default_node_pool {
    name                 = "system"
    vm_size              = "Standard_D4s_v5"
    vnet_subnet_id       = var.aks_subnet_id
    zones                = ["1", "2", "3"]    # spread across AZs (NFR-1)
    only_critical_addons_enabled = true       # taint so app pods don't land here
    auto_scaling_enabled = true
    min_count            = 2
    max_count            = 4
    orchestrator_version = var.k8s_version
    os_sku               = "AzureLinux"
    upgrade_settings { max_surge = "33%" }
  }

  # --- Network: Azure CNI Overlay + Cilium dataplane ---
  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"           # pods from overlay CIDR — conserves VNet IPs
    network_data_plane  = "cilium"            # eBPF dataplane + network policy
    network_policy      = "cilium"
    pod_cidr            = "10.244.0.0/16"
    service_cidr        = "10.0.0.0/16"
    dns_service_ip      = "10.0.0.10"
    load_balancer_sku   = "standard"
    outbound_type       = "userAssignedNATGateway"  # controlled egress (Phase 2 NAT)
    nat_gateway_profile {}
  }

  identity { type = "SystemAssigned" }

  # --- Auto-upgrade: stay on a patch channel, controlled by maintenance window ---
  automatic_channel_upgrade = "patch"
  maintenance_window_auto_upgrade {
    frequency   = "Weekly"
    interval    = 1
    duration    = 4
    day_of_week = "Sunday"
    start_time  = "03:00"
    utc_offset  = "+00:00"
  }

  azure_policy_enabled = true                 # Gatekeeper/Azure Policy add-on
  tags                 = var.common_tags
}
```

---

## 3.2 — Node pools (system / user / spot / Windows)

**Why this way / tradeoffs:** Separating node pools by *purpose* is what lets you place workloads correctly and control cost. System pool is tainted so only critical addons run there (your app pods can't starve CoreDNS). The Windows pool exists *only* for legacy RecordsMgr and is the platform's biggest cost line — it's isolated so you can see and eventually kill that cost (strangler-fig, Phase 4). Spot pool runs interruptible batch cheaply. Alternatives rejected: one big pool for everything (no workload isolation, Windows and Linux can't mix in one pool anyway); putting app workloads on the system pool (resource contention with control-plane-critical addons).

```hcl
# platform-infra/modules/aks/nodepools.tf

# Linux USER pool — modern microservices
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v5"
  vnet_subnet_id        = var.aks_subnet_id
  zones                 = ["1", "2", "3"]
  auto_scaling_enabled  = true
  min_count             = 3
  max_count             = 8
  os_sku                = "AzureLinux"
  orchestrator_version  = var.k8s_version
  upgrade_settings { max_surge = "33%" }
  node_labels = { "workload" = "general" }
  tags        = var.common_tags
}

# Linux SPOT pool — non-critical batch (BatchBiller, cron). Cheap, interruptible.
resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  name                  = "spot"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v5"
  vnet_subnet_id        = var.aks_subnet_id
  zones                 = ["1", "2", "3"]
  auto_scaling_enabled  = true
  min_count             = 0                  # scale to zero when no batch work
  max_count             = 4
  priority              = "Spot"
  eviction_policy       = "Delete"
  spot_max_price        = -1                 # pay up to on-demand price
  os_sku                = "AzureLinux"
  node_labels = {
    "workload"                            = "batch"
    "kubernetes.azure.com/scalesetpriority" = "spot"
  }
  node_taints = ["kubernetes.azure.com/scalesetpriority=spot:NoSchedule"]
  tags        = var.common_tags
}

# WINDOWS pool — legacy RecordsMgr ONLY. Heavy, expensive, isolated.
resource "azurerm_kubernetes_cluster_node_pool" "win" {
  name                  = "win"               # Windows pool names: max 6 chars
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D8s_v5"   # Windows nodes sized larger
  vnet_subnet_id        = var.aks_subnet_id
  zones                 = ["1", "2", "3"]
  auto_scaling_enabled  = true
  min_count             = 2
  max_count             = 4
  os_type               = "Windows"
  os_sku                = "Windows2022"
  orchestrator_version  = var.k8s_version
  node_labels = { "workload" = "legacy-windows" }
  node_taints = ["os=windows:NoSchedule"]     # only tolerating pods land here
  tags        = var.common_tags
}
```

**Operational note you can claim:** the taint-on-Windows-pool + toleration-on-RecordsMgr pattern is exactly the kind of scheduling control you've worked with. You can explain *why* it's there (keep general Linux workloads off the expensive Windows nodes) from real experience.

---

## 3.3 — Cluster add-ons: ingress, cert-manager, network policy

**Why this way / tradeoffs:** These are deployed as **GitOps-managed platform components** (in the `platform-gitops` repo, synced by ArgoCD in Phase 7) rather than baked into Terraform — because they're Kubernetes workloads that change more often than the cluster itself, and GitOps gives them the same audit/rollback as apps. I show them here as the Helm values they'll be deployed with. Alternatives rejected: AKS managed ingress add-on (less control over nginx config); Terraform-managed Helm releases (mixes infra and app lifecycles, and Terraform is a poor fit for things that reconcile continuously — that's ArgoCD's job).

**Ingress controller choice — ingress-nginx:**
```
Decision: ingress-nginx (single ingress controller for ALL apps).

WHY: One ingress controller behind one Azure LB serves modern microservices AND
legacy RecordsMgr — that's the "same ingress for legacy and modern" requirement.
L7 host/path routing, mature, huge ecosystem, works identically across envs.

Alternatives rejected:
- App Gateway Ingress Controller (AGIC): tighter Azure coupling, but ingress-nginx
  is more portable and the team knows it. App Gateway still sits in front (Phase 1)
  for WAF; nginx does in-cluster L7 routing.
- One LoadBalancer Service per app: expensive (one cloud LB each), no shared L7.
  This is the Ingress-vs-LB point from your K8s Q&A.
```

```yaml
# platform-gitops/platform/ingress-nginx/values.yaml
controller:
  replicaCount: 3                      # HA across the 3 AZs
  service:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # internal LB
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
  metrics:
    enabled: true                      # exposes Prometheus metrics (Phase 8 scrapes)
    serviceMonitor:
      enabled: true
  resources:
    requests: { cpu: 100m, memory: 256Mi }
    limits:   { cpu: 500m, memory: 512Mi }
```

```yaml
# platform-gitops/platform/cert-manager/cluster-issuer.yaml
# cert-manager issues + auto-renews TLS certs (cert expiry is a Phase 8 alert too)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com           # placeholder
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          azureDNS:
            subscriptionID: REPLACE_SUB_ID
            resourceGroupName: pp-dns-rg
            hostedZoneName: patientportal.example.com
            managedIdentity:
              clientID: REPLACE_CERTMGR_WI_CLIENT_ID   # workload identity, no secret
```

**Network policy — default-deny per namespace:**
```yaml
# platform-gitops/platform/network-policies/default-deny.yaml
# Once applied, every pod in the namespace is default-deny; you then allow explicitly.
# (This is the "policy selecting a pod flips it to default-deny" point from your Q&A —
#  and it WORKS here because Cilium enforces it, unlike Flannel.)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: appointments
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress-controller
  namespace: appointments
spec:
  podSelector:
    matchLabels: { app: appointments-api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: ingress-nginx }
      ports:
        - protocol: TCP
          port: 8080
```

---

## 3.4 — Pod Security Standards

**Why this way / tradeoffs:** PSS `restricted` enforced via namespace labels means no privileged containers, no host mounts, no running as root, dropped capabilities — the baseline hardening a SOC2 platform needs, enforced by the API server's admission, free. Kyverno (Phase 9) adds the custom policies on top (signed-images-only, no-latest). Alternatives rejected: PodSecurityPolicy (removed in K8s 1.25); no enforcement (relies on developers getting securityContext right — they won't, consistently).

```yaml
# platform-gitops/platform/namespaces/appointments-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: appointments
  labels:
    # Enforce the 'restricted' Pod Security Standard at admission
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    kubernetes.io/metadata.name: appointments
```

> **Assumption / caveat:** the **legacy** namespace can't always meet `restricted` (Windows IIS containers may need looser settings). For that namespace you'd set `baseline` instead of `restricted` and document the exception — a real SOC2 compensating-control situation. Don't pretend legacy meets the same bar; document *why* and what compensates (network isolation, no internet egress, extra monitoring).

---

## 3.5 — Upgrade strategy

**Why this way / tradeoffs:** AKS upgrades have two parts — control plane (Azure-managed) and node pools (you trigger). The safe order is control plane first, then node pools, with `max_surge` so a new node comes up before an old one drains (respecting PDBs). The `automatic_channel_upgrade = "patch"` + a Sunday-3am maintenance window keeps patch versions current automatically without surprise disruption. This is straight operational knowledge you have.

```
Upgrade runbook (also a Phase 10 runbook):
1. Upgrade CONTROL PLANE first (minor version): az aks upgrade --control-plane-only
2. Then node pools, one at a time, with surge:
   - cordon + drain respects PodDisruptionBudgets (Phase 5 charts set these)
   - max_surge=33% → new nodes up before old drain → no capacity dip
3. Windows pool last (slowest image pulls).
4. Validate: nodes Ready, pods rescheduled, SLO dashboards green before next pool.
5. Patch versions: handled automatically by the weekly maintenance window.
```

---

# PHASE 4 — LEGACY MIGRATION TRACK

🟢 **Can claim (the reality):** You've *lived* legacy-and-modern coexistence in a healthcare platform — that's genuinely rare and valuable. You can speak to why migration is incremental and risky in regulated domains. 🟡 **Learning:** Authoring the strangler-fig design, the Windows Dockerfile, and the data-migration plan are lead/architect tasks. Frame as "I supported a migration like this and understand the pattern," not "I designed it."

> **Assumptions:** RecordsMgr is a .NET Framework 4.8 / IIS app on Windows, coupled to SQL Server, with read-heavy traffic (patients viewing records). Decision from Phase 0 was **replatform** (containerize as-is on Windows nodes), then **strangler-fig** read traffic to a new Linux `records-read-api` microservice over time. BatchBiller = rehost (VMSS). LegacyAuth = repurchase (Entra ID).

---

## 4.1 — The strangler-fig pattern (the core strategy)

**Why this way / tradeoffs:** The strangler-fig pattern wraps the legacy app behind the same ingress, then *incrementally* routes specific paths to new microservices until the legacy app is "strangled" and can be retired — without a big-bang rewrite. In healthcare, a big-bang rewrite of a PHI system means re-doing clinical validation, which is months of risk; strangler-fig lets you migrate one capability at a time, each independently validated. Alternatives rejected: big-bang rewrite (unacceptable clinical-validation risk); leave-on-VMs-forever (never modernizes, carries the Windows cost line indefinitely).

Let me show how traffic shifts incrementally.The migration sequence, stage by stage:

<img width="1472" height="800" alt="image" src="https://github.com/user-attachments/assets/4c3e9fbe-ce4c-4558-b751-f98a44f419c5" />


```
Strangler-fig stages (each independently validated before the next):

Stage 1 — Wrap.    Containerize RecordsMgr as-is on Windows nodes, put it behind the
                   ingress. ALL traffic still goes to legacy. Nothing changes for users,
                   but it's now in the platform (same GitOps, same observability).

Stage 2 — Mirror.  Build records-read-api (Linux). Ingress mirrors GET /records traffic
                   to it (shadow traffic) — it serves nothing yet, but you compare its
                   responses against legacy to validate correctness.

Stage 3 — Shift.   Canary the reads: ingress routes 5% → 25% → 100% of GET /records to
                   the new service, watching error rate + latency (Argo Rollouts, Phase 7).
                   Writes (POST /records) still hit legacy.

Stage 4 — Decouple. Once reads are fully on the new service, tackle the shared DB and the
                   write path. Split the DB (new service gets its own store, sync'd), then
                   migrate writes.

Stage 5 — Retire.  When no path routes to RecordsMgr, decommission it — and kill the
                   Windows node pool (the #1 cost driver from Phase 0). The fig has strangled
                   the host tree.
```

**The senior insight:** notice the migration is sequenced by *risk and coupling* (reads before writes, because reads are idempotent and safe to shadow/canary; the shared DB is tackled last because it's the tightest coupling). And retiring the Windows pool is the explicit cost payoff — you can trace it straight back to the Phase 0 cost model.

---

## 4.2 — Containerizing the legacy Windows/IIS app

**Why this way / tradeoffs:** Containerizing the .NET Framework app *as-is* (no code rewrite) on a Windows base image gets it into AKS without touching the application — the whole point of "replatform, not refactor." It's a heavy image and runs on the expensive Windows pool, which is exactly why you strangler-fig off it. Alternatives rejected: rewrite to .NET (Core) on Linux now (that's *refactor* — the clinical-validation risk we deferred); leave on a VM (doesn't get the GitOps/observability benefits).

```dockerfile
# recordsmgr/Dockerfile.windows
# Legacy .NET Framework 4.8 / IIS app — containerized as-is (replatform).
# Runs on the Windows node pool. Large image; this is intentional/temporary.

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022

# IIS site config + app
WORKDIR /inetpub/wwwroot
COPY ./published/ .

# Expose the metrics endpoint via a sidecar-friendly path (see 4.4)
EXPOSE 80

# Health endpoint for readiness/liveness probes (added without touching app logic —
# a static healthcheck.html IIS can serve, or a lightweight handler)
COPY ./healthcheck.html /inetpub/wwwroot/healthz/index.html

# IIS starts via the base image's ServiceMonitor entrypoint
```

```yaml
# platform-gitops/apps/recordsmgr/base/deployment.yaml
# Legacy app as a K8s Deployment — same shape as a modern app, but Windows-targeted.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recordsmgr
  namespace: legacy
spec:
  replicas: 2
  selector:
    matchLabels: { app: recordsmgr }
  template:
    metadata:
      labels: { app: recordsmgr }
    spec:
      nodeSelector:
        kubernetes.io/os: windows          # land on the Windows pool
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"             # tolerate the Windows-pool taint (Phase 3)
      containers:
        - name: recordsmgr
          image: ppprodacr.azurecr.io/recordsmgr:sha-PLACEHOLDER  # immutable tag
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: /healthz, port: 80 }
            initialDelaySeconds: 30          # Windows containers start slowly
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 80 }
            initialDelaySeconds: 60          # generous — Windows/IIS cold start
            periodSeconds: 15
          resources:
            requests: { cpu: "1", memory: 2Gi }
            limits:   { cpu: "2", memory: 4Gi }
        # metrics exporter sidecar — see 4.4
```

**Operational detail you can claim:** the long `initialDelaySeconds` for a slow-starting app is *exactly* your real CrashLoopBackOff incident (probe killing a healthy-but-slow container). You can speak to why Windows/IIS needs generous probe delays from genuine experience.

---

## 4.3 — The new Linux microservice (alongside it)

This is the modern side — a normal, lean Linux container that will take over the read path.

```dockerfile
# records-read-api/Dockerfile
# Modern Linux microservice — multi-stage, slim, non-root (Phase 9 Kyverno enforces this).
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o records-read-api .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/records-read-api /records-read-api
USER nonroot:nonroot                       # non-root — meets PSS 'restricted'
EXPOSE 8080
ENTRYPOINT ["/records-read-api"]
```

The contrast is the whole story: the legacy image is a multi-GB Windows Server Core image running as a service; the new one is a ~10MB distroless static binary running as non-root. Same platform, same ingress, same observability — radically different footprint. That contrast *is* the modernization argument.

---

## 4.4 — How legacy emits metrics (sharing the SAME observability)

**Why this way / tradeoffs:** The legacy app doesn't expose Prometheus metrics natively. Rather than modify the legacy code, you attach an **exporter sidecar** that scrapes IIS/Windows perf counters and exposes them in Prometheus format — so the legacy app shows up on the *same* Grafana dashboards as the modern apps. For things you can't instrument at all, a **blackbox probe** checks it externally (is the endpoint up, how fast). Alternatives rejected: modifying the legacy app to emit metrics (touches code we're trying not to touch); a separate monitoring stack for legacy (defeats the unified-observability goal).

```yaml
# platform-gitops/apps/recordsmgr/base/deployment.yaml (sidecar excerpt)
        - name: metrics-exporter
          image: ppprodacr.azurecr.io/windows-exporter:sha-PLACEHOLDER
          ports:
            - containerPort: 9182             # windows_exporter Prometheus port
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 250m, memory: 256Mi }
```

```yaml
# platform-gitops/apps/recordsmgr/base/servicemonitor.yaml
# Same ServiceMonitor mechanism the modern apps use — legacy joins the same scrape.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: recordsmgr
  namespace: legacy
  labels: { release: kube-prometheus-stack }  # matches Prometheus selector (Phase 8)
spec:
  selector:
    matchLabels: { app: recordsmgr }
  endpoints:
    - port: metrics
      interval: 30s
```

```yaml
# platform-gitops/platform/blackbox/probe.yaml
# For external/synthetic checks on the legacy endpoint (is /records actually serving?)
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: recordsmgr-blackbox
  namespace: legacy
  labels: { release: kube-prometheus-stack }
spec:
  prober:
    url: blackbox-exporter.monitoring.svc:9115
  module: http_2xx
  targets:
    staticConfig:
      static:
        - http://recordsmgr.legacy.svc.cluster.local/healthz
```

**This is the linchpin of the whole "coexistence" requirement:** because the legacy app emits metrics via a ServiceMonitor (the same CRD your modern apps use — and the same one you wrote in Athena), it appears on the same dashboards, fires the same kinds of alerts, and is on-call'd the same way. One observability plane for both. That's the unified-operations win.

---

## 4.5 — The VMSS / App Service bridge (for rehost workloads)

**Why this way / tradeoffs:** Not everything goes into the cluster. BatchBiller (Phase 0 decision: *rehost*) is a nightly batch job — putting it on a VM Scale Set or running it as an Azure Container Instance on a schedule is simpler and cheaper than replatforming it into AKS. The bridge: it still writes to the shared SQL and still gets monitored, but it lives outside the cluster until the DB split makes replatforming worthwhile. Alternatives rejected: forcing BatchBiller into AKS now (effort for no benefit on a low-traffic nightly job); leaving it un-monitored (it touches the shared PHI DB — it must be in scope).

```hcl
# platform-infra/modules/legacy-bridge/batchbiller.tf
# BatchBiller: rehosted on a small VMSS, scaled to zero outside the nightly window.
# (Conceptual — the nightly trigger is a scheduled scale-out or an ACI job.)
resource "azurerm_linux_virtual_machine_scale_set" "batchbiller" {
  name                = "${var.prefix}-batchbiller-vmss"
  resource_group_name = var.rg_name
  location            = var.location
  sku                 = "Standard_D2s_v5"
  instances           = 0                   # scaled out only for the nightly run
  # ... image, network, identity (managed identity to reach SQL — no stored secret)
  tags = var.common_tags
}
```

> **Assumption:** I'm keeping this conceptual rather than a full VMSS config, because the rehost path is deliberately minimal — the interesting engineering is the strangler-fig (4.1–4.4), and over-investing in BatchBiller contradicts the Phase 0 decision to rehost it cheaply.

---

## 4.6 — Data-layer migration (the hardest part)

**Why this way / tradeoffs:** The shared SQL database is the tightest coupling and the riskiest thing to move — so it's done *last and incrementally*, never as a cutover. The pattern: the new read service first reads from the *same* DB (no data move), then you introduce its own store fed by change-data-capture/replication, validate parity, then flip reads, then migrate writes. Alternatives rejected: dual-write from day one (consistency nightmares); big-bang DB cutover (the highest-risk operation in any migration — avoid).

```
Data migration sequence (incremental, reversible at each step):
1. New read service reads the SAME SQL DB (read-only) — zero data movement, zero risk.
2. Stand up the new datastore; replicate via CDC (change data capture) from SQL.
3. Validate parity (shadow reads compare old vs new — Stage 2 of strangler-fig).
4. Flip reads to the new store (canary). Writes still go to SQL.
5. Migrate the write path last; decommission the shared SQL coupling.
Each step is independently reversible — if parity fails at step 3, you haven't moved
any user traffic yet.
```

> **Honest caveat:** data migration in a PHI system is where the *real* risk lives, and it involves DBAs, compliance sign-off, and careful validation — well beyond a DevOps engineer's scope. Don't claim you owned data migration. The defensible position: "I understand the incremental, parity-validated approach and why the DB is migrated last."

---

# PHASE 5 — HELM CHARTS

🟢 **Can claim (partially):** You work with Helm operationally — deploying, overriding values per environment, rolling back. You understand the chart/values/release model. 🟡 **Learning:** *Authoring* a complete chart with a library chart, helpers, and all the templates from scratch is chart-development work. Frame as "I work with Helm charts and understand their structure; authoring a full chart with a shared library is something I've practiced in my own project," — which is true if you built charts in Athena.

> **Assumptions:** I'll show the `appointments-api` chart as the canonical microservice chart. It uses a shared library chart (`common-lib`) for boilerplate, External Secrets Operator for Key Vault integration (configured in Phase 9), workload identity for the ServiceAccount, and env overlays for dev/staging/prod. Same chart pattern applies to every modern microservice.

---

## 5.1 — Why a library chart + per-service chart (the structure decision)

**Why this way / tradeoffs:** Without a shared library, every microservice chart copy-pastes the same labels, selectors, and helper templates — and they drift. A **library chart** (`common-lib`) holds the shared template logic (naming, labels, common helpers); each **application chart** is then thin, just its own values and the resource templates that call the library. Alternatives rejected: one giant umbrella chart for all services (couples unrelated services into one release — a change to one forces a sync of all); fully independent charts with no sharing (boilerplate drift, inconsistent labels break Prometheus selectors and network policies).

```
platform-gitops/
├── charts/
│   └── common-lib/                    # shared LIBRARY chart (type: library)
│       ├── Chart.yaml
│       └── templates/
│           └── _helpers.tpl           # shared naming/label helpers
└── apps/
    └── appointments-api/
        ├── Chart.yaml                 # depends on common-lib
        ├── values.yaml                # base defaults
        ├── templates/
        │   ├── _helpers.tpl
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   ├── ingress.yaml
        │   ├── hpa.yaml
        │   ├── configmap.yaml
        │   ├── externalsecret.yaml
        │   ├── serviceaccount.yaml
        │   ├── pdb.yaml
        │   └── networkpolicy.yaml
        └── overlays/
            ├── values-dev.yaml
            ├── values-staging.yaml
            └── values-prod.yaml
```

---

## 5.2 — The shared library chart

**Why this way / tradeoffs:** Library charts (`type: library`) can't be installed directly — they only export reusable template definitions. Putting the label/selector helpers here guarantees every app uses *identical* labels, which is what makes Prometheus ServiceMonitors, network-policy selectors, and PDB selectors all line up. A label typo in a copy-pasted chart is a classic "why isn't this pod being scraped / why is traffic blocked" incident — the library prevents it.

```yaml
# platform-gitops/charts/common-lib/Chart.yaml
apiVersion: v2
name: common-lib
description: Shared template helpers for PatientPortal microservice charts
type: library          # library charts are NOT installable — only imported
version: 0.1.0
```

```yaml
# platform-gitops/charts/common-lib/templates/_helpers.tpl
{{/* Standard name */}}
{{- define "common.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/* Standard labels — applied everywhere for consistent selection */}}
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ include "common.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: patientportal
{{- end -}}

{{/* Selector labels — the STABLE subset used by Service/Deployment selectors.
     Must never include version/changing labels or selectors break on upgrade. */}}
{{- define "common.selectorLabels" -}}
app: {{ include "common.name" . }}
{{- end -}}
```

**The senior detail:** `selectorLabels` deliberately excludes version/changing labels. A Deployment's selector is immutable after creation — if you put a changing label in it, upgrades fail. This separation (full labels vs stable selector labels) is a real chart-authoring gotcha.

---

## 5.3 — The application chart

```yaml
# platform-gitops/apps/appointments-api/Chart.yaml
apiVersion: v2
name: appointments-api
description: Appointments microservice
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: common-lib
    version: 0.1.0
    repository: "file://../../charts/common-lib"
```

```yaml
# platform-gitops/apps/appointments-api/values.yaml
# Base defaults — overlays override the env-specific bits (replicas, resources, host).
replicaCount: 3

image:
  repository: ppprodacr.azurecr.io/appointments-api
  tag: "sha-PLACEHOLDER"          # CI updates this per env via GitOps PR (immutable SHA)
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  # workload identity client ID (from the Phase 2 identity module output)
  workloadIdentityClientId: "REPLACE_WI_CLIENT_ID"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: appointments.patientportal.example.com
  tlsSecretName: appointments-tls
  path: /appointments
  pathType: Prefix

resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 2

config:
  LOG_LEVEL: "info"
  DB_HOST: "pp-prod-pg.postgres.database.azure.com"

externalSecret:
  enabled: true
  keyVaultName: "pp-prod-kv"
  secrets:
    - secretKey: DB_PASSWORD       # key the app expects
      remoteKey: appointments-db-password   # name in Key Vault

networkPolicy:
  enabled: true
```

Now the templates. Each precedes with its rationale.

### Deployment

**Why this way:** Zero-downtime rollout (`maxUnavailable: 0`), readiness gating traffic, liveness gating restart, workload-identity label so the pod federates to Azure (no secret), topology spread across zones for AZ resilience (NFR-1), and resources from values so overlays can right-size per env.

```yaml
# platform-gitops/apps/appointments-api/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # zero-downtime
  selector:
    matchLabels: {{- include "common.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "common.labels" . | nindent 8 }}
        azure.workload.identity/use: "true"   # opt this pod into workload identity
    spec:
      serviceAccountName: {{ include "common.name" . }}-sa
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: {{- include "common.selectorLabels" . | nindent 14 }}
      containers:
        - name: {{ include "common.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "common.name" . }}-config
          {{- if .Values.externalSecret.enabled }}
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "common.name" . }}-secret
                  key: DB_PASSWORD
          {{- end }}
          readinessProbe:
            httpGet: { path: /healthz, port: {{ .Values.service.targetPort }} }
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: {{ .Values.service.targetPort }} }
            initialDelaySeconds: 10
            periodSeconds: 10
          resources: {{- toYaml .Values.resources | nindent 12 }}
          securityContext:                 # meets PSS 'restricted' (Phase 3)
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

### Service

```yaml
# platform-gitops/apps/appointments-api/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector: {{- include "common.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

### Ingress

**Why this way:** Routes by host/path through the shared ingress-nginx (Phase 3), with TLS from the cert-manager-issued secret. One ingress per app, all behind one LB — the Ingress-vs-LB point from your Q&A.

```yaml
# platform-gitops/apps/appointments-api/templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    - hosts: [{{ .Values.ingress.host | quote }}]
      secretName: {{ .Values.ingress.tlsSecretName }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ include "common.name" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

### HPA

```yaml
# platform-gitops/apps/appointments-api/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "common.name" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

### ConfigMap

```yaml
# platform-gitops/apps/appointments-api/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "common.name" . }}-config
  labels: {{- include "common.labels" . | nindent 4 }}
data:
  {{- range $key, $val := .Values.config }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

### ExternalSecret (Key Vault → K8s secret, no static secret)

**Why this way:** External Secrets Operator (Phase 9) syncs the secret *from Key Vault* into a K8s Secret at runtime, authenticated via workload identity. The secret value never lives in Git, never in the chart — only a *reference* to its Key Vault name does. This is the "secrets come from the environment, not the source" principle made concrete.

```yaml
# platform-gitops/apps/appointments-api/templates/externalsecret.yaml
{{- if .Values.externalSecret.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault          # ClusterSecretStore defined in Phase 9
    kind: ClusterSecretStore
  target:
    name: {{ include "common.name" . }}-secret
    creationPolicy: Owner
  data:
    {{- range .Values.externalSecret.secrets }}
    - secretKey: {{ .secretKey }}
      remoteRef:
        key: {{ .remoteKey }}
    {{- end }}
{{- end }}
```

### ServiceAccount (federated to workload identity)

```yaml
# platform-gitops/apps/appointments-api/templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "common.name" . }}-sa
  labels: {{- include "common.labels" . | nindent 4 }}
  annotations:
    # ties this SA to the Azure managed identity (the federated subject from Phase 2)
    azure.workload.identity/client-id: {{ .Values.serviceAccount.workloadIdentityClientId }}
{{- end }}
```

### PodDisruptionBudget

```yaml
# platform-gitops/apps/appointments-api/templates/pdb.yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  minAvailable: {{ .Values.pdb.minAvailable }}
  selector:
    matchLabels: {{- include "common.selectorLabels" . | nindent 6 }}
{{- end }}
```

### NetworkPolicy

```yaml
# platform-gitops/apps/appointments-api/templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "common.name" . }}
  labels: {{- include "common.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels: {{- include "common.selectorLabels" . | nindent 6 }}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: {{ .Values.service.targetPort }}
{{- end }}
```

---

## 5.4 — Environment overlays

**Why this way / tradeoffs:** The *same chart* deploys to all three environments — only the overlay values differ. This guarantees environment parity (staging genuinely predicts prod) and means a change to the chart automatically applies everywhere consistently. Dev is small and cheap; prod is sized and HA. Alternatives rejected: separate charts per env (drift, no parity); hardcoded env values in the chart (can't reuse).

```yaml
# platform-gitops/apps/appointments-api/overlays/values-dev.yaml
replicaCount: 1
autoscaling:
  enabled: false              # no autoscaling in dev — keep it cheap
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 250m, memory: 256Mi }
pdb:
  enabled: false              # single replica, no PDB needed
ingress:
  host: appointments.dev.patientportal.example.com
config:
  LOG_LEVEL: "debug"
  DB_HOST: "pp-dev-pg.postgres.database.azure.com"
externalSecret:
  keyVaultName: "pp-dev-kv"
```

```yaml
# platform-gitops/apps/appointments-api/overlays/values-staging.yaml
replicaCount: 2
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 6
ingress:
  host: appointments.staging.patientportal.example.com
config:
  LOG_LEVEL: "info"
  DB_HOST: "pp-staging-pg.postgres.database.azure.com"
externalSecret:
  keyVaultName: "pp-staging-kv"
```

```yaml
# platform-gitops/apps/appointments-api/overlays/values-prod.yaml
replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
pdb:
  enabled: true
  minAvailable: 2
ingress:
  host: appointments.patientportal.example.com
config:
  LOG_LEVEL: "info"
  DB_HOST: "pp-prod-pg.postgres.database.azure.com"
externalSecret:
  keyVaultName: "pp-prod-kv"
```

---

## 5.5 — Validating the chart

These are the commands from your Helm Q&A — worth running in CI (Phase 6 wires `helm lint` + `helm template` into the pipeline):

```bash
helm lint apps/appointments-api -f apps/appointments-api/overlays/values-prod.yaml
helm template appointments-api apps/appointments-api \
  -f apps/appointments-api/overlays/values-prod.yaml   # render to inspect output YAML
helm template ... | kubeconform -strict -                # validate against K8s schemas
```

---

# PHASE 6 — CI PIPELINE WITH SECURITY GATES & APPROVALS

🟢 **Can claim:** You *support and operate* CI/CD on GitHub Actions and Azure DevOps — troubleshooting failures, image promotion, rollbacks. You understand where secrets live, OIDC, and the scan-before-deploy flow. 🟡 **Learning:** *Designing* the gated pipeline, *setting* the threshold policies, and *configuring* the SOC2 approval governance are lead/security-engineer tasks. Frame as "I operate pipelines like this and understand every gate; designing the gate policy was owned by senior/security engineers."

> **Assumptions:** GitHub Actions is primary CI for cloud-native apps (per your real platform). The legacy Windows app builds on Azure DevOps. Both end the same way — push a signed image to ACR, then open a PR to the GitOps repo to bump the tag. CI **never** runs `kubectl` — that's the GitOps security boundary. SOC2 drives the approval gates and audit trail.

---

## 6.1 — The gating principle (why each gate exists)

**Why this way / tradeoffs:** Every gate is a *fail-closed* checkpoint — a security or quality problem stops the artifact from ever reaching a registry, let alone prod. The ordering is deliberate: cheap/fast checks first (secret scan, unit tests) so the pipeline fails early and cheap; expensive checks (image scan, signing) last, only on artifacts that passed everything else. The hard rule that makes it GitOps-safe: **CI builds and signs and pushes, then opens a PR to the GitOps repo — it never touches the cluster.** Alternatives rejected: CI with `kubectl apply` to prod (CI holds cluster creds = the push-model risk from your Q&A); scans as warnings-only (a warning nobody blocks on is theater — SOC2 wants *enforced* controls).

Let me show where each gate sits in the flow.The red and blue gates fail the build before anything is built; the amber gates fail before anything is published; only a fully-passed, signed artifact reaches ACR. Then CI hands off to GitOps via a PR — and stops. That handoff is the security boundary.

<img width="1472" height="1520" alt="image" src="https://github.com/user-attachments/assets/755127ab-d3f5-4a6d-8db5-7561d6fce413" />


---

## 6.2 — The full GitHub Actions pipeline (cloud-native apps)

**Why this way / tradeoffs:** OIDC federation for ACR auth (no stored registry password — the short-lived-token principle from your Q&A). `permissions:` set to least-privilege at the top. Each gate is its own step with an explicit failure threshold. The final job opens a PR to the GitOps repo rather than deploying.

```yaml
# appointments-api/.github/workflows/ci.yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read          # least privilege by default
  id-token: write         # required for OIDC federation to Azure
  security-events: write  # to upload SARIF scan results

env:
  ACR: ppprodacr.azurecr.io
  IMAGE: appointments-api

jobs:
  # ---------- GATE 1: secret scanning ----------
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }     # full history so gitleaks scans all commits
      - name: gitleaks
        uses: gitleaks/gitleaks-action@v2   # FAILS the job on any detected secret
        env:
          GITLEAKS_CONFIG: .gitleaks.toml

  # ---------- GATE 2: build + unit tests + coverage ----------
  test:
    needs: secret-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - name: unit tests with coverage
        run: |
          go test ./... -coverprofile=coverage.out
          COV=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
          echo "coverage: $COV%"
          # COVERAGE GATE: fail if below 70%
          if (( $(echo "$COV < 70" | bc -l) )); then
            echo "::error::coverage $COV% is below 70% threshold"; exit 1
          fi
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage.out }

  # ---------- GATE 3: SAST (static application security testing) ----------
  sast:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/default p/security-audit p/secrets
        # FAILS on findings at or above the configured severity (see semgrep policy)

  # ---------- GATE 4: SCA / dependency + license scan ----------
  sca:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: trivy filesystem (deps + licenses)
        uses: aquasecurity/trivy-action@0.20.0
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL    # CVE THRESHOLD: fail on HIGH or CRITICAL
          exit-code: '1'             # non-zero = fail the build
          format: sarif
          output: trivy-deps.sarif
      - uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: trivy-deps.sarif }

  # ---------- GATE 5: IaC + Helm/manifest scan ----------
  iac-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: checkov (Terraform/IaC misconfig)
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          soft_fail: false           # FAILS on misconfigurations
      - name: helm lint + kubeconform (manifest validity)
        run: |
          helm lint apps/appointments-api -f apps/appointments-api/overlays/values-prod.yaml
          helm template appointments-api apps/appointments-api \
            -f apps/appointments-api/overlays/values-prod.yaml \
            | kubeconform -strict -summary -    # FAILS on invalid K8s manifests
      - name: kube-linter (manifest best-practice)
        uses: stackrox/kube-linter-action@v1
        with: { directory: apps/appointments-api }

  # ---------- GATE 6: build, image scan, SIGN, push ----------
  build-sign-push:
    needs: [sast, sca, iac-scan]    # only runs if ALL gates passed
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "tag=sha-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

      # OIDC login to Azure — short-lived token, NO stored credential
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: ACR login
        run: az acr login --name ppprodacr

      - name: build image
        run: docker build -t $ACR/$IMAGE:${{ steps.meta.outputs.tag }} .

      # IMAGE SCAN gate — fail on HIGH/CRITICAL in the built image
      - name: trivy image scan
        uses: aquasecurity/trivy-action@0.20.0
        with:
          image-ref: ${{ env.ACR }}/${{ env.IMAGE }}:${{ steps.meta.outputs.tag }}
          severity: HIGH,CRITICAL
          exit-code: '1'

      - name: push to ACR (immutable tag)
        run: docker push $ACR/$IMAGE:${{ steps.meta.outputs.tag }}

      # SIGN the image with cosign (keyless, via OIDC) — verified at admission (Phase 9)
      - name: cosign sign
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes \
            $ACR/$IMAGE:${{ steps.meta.outputs.tag }}

  # ---------- HANDOFF: PR to GitOps repo (NO kubectl) ----------
  bump-gitops:
    needs: build-sign-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: org/platform-gitops
          token: ${{ secrets.GITOPS_PAT }}   # scoped PAT, can only open PRs
      - name: bump dev image tag + open PR
        run: |
          TAG=${{ needs.build-sign-push.outputs.tag }}
          yq -i ".image.tag = \"$TAG\"" apps/appointments-api/overlays/values-dev.yaml
          git checkout -b bump-appointments-$TAG
          git commit -am "chore: appointments-api -> $TAG (dev)"
          git push origin bump-appointments-$TAG
          gh pr create --fill --base main \
            --title "deploy: appointments-api $TAG to dev"
        env:
          GH_TOKEN: ${{ secrets.GITOPS_PAT }}
```

**The handoff is the whole point:** the pipeline ends by opening a PR that bumps the *dev* overlay tag. It does not deploy. ArgoCD (Phase 7) picks up the merged change. Promotion to staging/prod is separate, gated PRs (6.4).

---

## 6.3 — The Azure DevOps pipeline (legacy Windows app)

**Why this way / tradeoffs:** The legacy RecordsMgr builds on Azure DevOps (matching your real split) on a Windows agent. Same gating philosophy, same endpoint (signed image to ACR → GitOps PR), just expressed in Azure Pipelines YAML and on a Windows build agent. This proves both CI systems feed *one* GitOps flow.

```yaml
# recordsmgr/azure-pipelines.yml
trigger:
  branches: { include: [main] }

variables:
  acr: ppprodacr.azurecr.io
  image: recordsmgr

stages:
  - stage: SecurityGates
    jobs:
      - job: scan
        pool: { vmImage: 'windows-2022' }   # Windows agent for the legacy build
        steps:
          - task: Gitleaks@2                  # GATE 1: secret scan
            displayName: 'secret scan'
          - script: |
              dotnet test --collect:"XPlat Code Coverage"
            displayName: 'GATE 2: unit tests + coverage'
          - task: SonarQubePrepare@5          # GATE 3: SAST quality gate
            inputs: { SonarQube: 'sonarqube-sc', scannerMode: 'MSBuild' }
          - task: SonarQubeAnalyze@5
          - task: SonarQubePublish@5
            inputs: { pollingTimeoutSec: '300' }   # FAILS if quality gate fails

  - stage: BuildSignPush
    dependsOn: SecurityGates
    condition: succeeded()                    # only if all gates passed
    jobs:
      - job: build
        pool: { vmImage: 'windows-2022' }
        steps:
          - task: AzureCLI@2
            displayName: 'OIDC login + ACR'
            inputs:
              azureSubscription: 'pp-prod-oidc'   # workload-identity service connection
              scriptType: 'pscore'
              scriptLocation: 'inlineScript'
              inlineScript: |
                $tag = "sha-$(git rev-parse --short HEAD)"
                echo "##vso[task.setvariable variable=tag]$tag"
                az acr login --name ppprodacr
                docker build -f Dockerfile.windows -t $(acr)/$(image):$tag .
          - task: AzureCLI@2
            displayName: 'image scan + push + sign'
            inputs:
              azureSubscription: 'pp-prod-oidc'
              scriptType: 'pscore'
              scriptLocation: 'inlineScript'
              inlineScript: |
                trivy image --severity HIGH,CRITICAL --exit-code 1 $(acr)/$(image):$(tag)
                docker push $(acr)/$(image):$(tag)
                cosign sign --yes $(acr)/$(image):$(tag)

  - stage: BumpGitOps
    dependsOn: BuildSignPush
    condition: succeeded()
    jobs:
      - job: pr
        pool: { vmImage: 'ubuntu-latest' }
        steps:
          - script: |
              # same PR-based handoff to platform-gitops — no kubectl
              yq -i ".image.tag = \"$(tag)\"" apps/recordsmgr/overlays/values-dev.yaml
              # ... git checkout/commit/push/PR (omitted for brevity, identical pattern)
            displayName: 'bump GitOps tag + open PR'
```

---

## 6.4 — Approvals & governance (the SOC2 layer)

**Why this way / tradeoffs:** SOC2 requires documented change approval, separation of duties, and an audit trail for every production change. This is implemented as: GitHub **environment protection rules** (required reviewers per environment), **branch protection** on the GitOps repo's `main`, **CODEOWNERS** so the right people must approve the right paths, and the fact that every promotion is a Git PR (inherently audited). The separation of duties: the person who *writes* code is not the person who *approves prod deployment*. Alternatives rejected: anyone-merges (no separation of duties — instant SOC2 failure); approvals tracked in a spreadsheet (not enforced, not auditable).

### GitHub environment protection (gates promotion to staging/prod)

```yaml
# Configured in the platform-gitops repo settings (shown as the effective policy):
# Settings → Environments

environment: staging
  required_reviewers: [team:platform-reviewers]   # 1 reviewer to reach staging
  deployment_branch_policy: protected branches only

environment: production
  required_reviewers: [team:sre-leads, team:security]  # 2 groups must approve prod
  wait_timer: 10                                       # 10-min cool-off before prod sync
  deployment_branch_policy: protected branches only
```

### Branch protection on the GitOps repo `main`

```yaml
# Settings → Branches → main (effective ruleset)
require_pull_request: true
required_approving_reviews: 2
require_review_from_codeowners: true        # CODEOWNERS must approve relevant paths
require_status_checks:                       # CI gates must pass before merge
  - manifest-validation
  - policy-check
dismiss_stale_reviews: true
require_linear_history: true
restrict_who_can_push: [team:platform]      # no direct pushes — PR only
```

### CODEOWNERS (who approves what — separation of duties)

```bash
# platform-gitops/.github/CODEOWNERS
# Path-based required reviewers. The author of a change cannot self-approve their own area.

# Production overlays require SRE leads AND security — NOT the app developers
/apps/*/overlays/values-prod.yaml   @org/sre-leads @org/security

# Platform components (ingress, kyverno, observability) require platform team
/platform/                          @org/platform

# Kyverno policies require security sign-off
/platform/kyverno/                  @org/security

# Library chart changes affect everything — require platform + a senior review
/charts/common-lib/                 @org/platform @org/senior-eng
```

**The separation-of-duties story, concretely:** a developer commits code → CI builds/scans/signs → CI opens a PR bumping the *dev* tag → that merges with one reviewer. To promote to *prod*, someone opens a PR editing `values-prod.yaml`, which CODEOWNERS routes to SRE leads + security — and the developer who wrote the code **cannot** approve their own production deployment. Every step is a Git commit with an author, a reviewer, and a timestamp: that *is* the SOC2 audit trail, for free.

### Change-management / CAB integration

```
For production changes, the prod-promotion PR links to a change ticket
(ServiceNow/Jira). The environment approval in GitHub is the CAB-approved gate —
the approval event + the linked ticket + the Git history together form the
auditable change record. No change reaches prod without: a ticket, CODEOWNERS
approval, and the environment reviewer approval.
```

---

## 6.5 — Security Gate Summary table

| Gate | Pipeline stage | Tool | Threshold / policy | Blocking? | What a failing run looks like |
|------|---------------|------|--------------------|-----------|-------------------------------|
| Secret scan | pre-build | gitleaks | Any detected secret | **Block** | Job fails, lists file + line of the leaked secret |
| Unit tests | pre-build | go test | All tests pass | **Block** | Failing test name + assertion |
| Coverage | pre-build | go cover | ≥ 70% | **Block** | `coverage 63% is below 70% threshold` |
| SAST | pre-build | Semgrep / SonarQube | No findings ≥ HIGH | **Block** | Rule ID, file, severity; SonarQube quality gate red |
| SCA / deps | pre-build | Trivy (fs) | No CVE ≥ HIGH | **Block** | CVE ID, package, fixed version, severity |
| License scan | pre-build | Trivy | No disallowed licenses | **Block** | Offending dependency + license |
| IaC scan | pre-build | Checkov | No misconfig (configurable) | **Block** | Check ID, resource, remediation |
| Manifest validity | pre-build | kubeconform | Valid against K8s schema | **Block** | Invalid field / type error |
| Manifest best-practice | pre-build | kube-linter | No high-severity lint | **Block** | e.g. "no resource limits set" |
| Image scan | build | Trivy (image) | No CVE ≥ HIGH | **Block** | CVE in a base-image layer |
| Image signing | build | cosign | Image must be signed | **Block** (verified at admission, Phase 9) | Unsigned image rejected at deploy |
| Staging approval | promote | GitHub env | 1 reviewer | **Block** | PR can't deploy until approved |
| Prod approval | promote | GitHub env + CODEOWNERS | 2 groups + cool-off | **Block** | PR can't merge/deploy without SRE+security |

> **Assumption / real-world note:** thresholds (70% coverage, fail-on-HIGH) are tunable per org risk appetite. A pragmatic real setup often starts SCA/SAST as *warn* on existing code (to avoid blocking on a mountain of pre-existing findings) and flips to *block* for *new* findings — a "ratchet." I've shown the strict end; the senior judgment is knowing when to ratchet vs block-everything-day-one.

---
# PHASE 7 — ARGOCD GITOPS DEPLOYMENT

🟢 **Can claim (partially):** You've used ArgoCD — you understand Application, sync, self-heal, prune, and the Git-as-source-of-truth model (it's in your Q&A bank, and you used it in Athena). 🟡 **Learning:** Installing ArgoCD HA, wiring Entra SSO, authoring ApplicationSets with generators, and configuring Argo Rollouts with Prometheus analysis are platform-engineering tasks. Frame as "I've worked with ArgoCD and understand the GitOps model; the ApplicationSet/Rollouts setup is something I understand and have explored in my own project."

> **Assumptions:** ArgoCD runs in the *prod* cluster's `argocd` namespace in HA mode, managing dev/staging/prod (a hub-style setup — alternatively one ArgoCD per cluster; I note the tradeoff). SSO via Entra. App-of-apps bootstraps everything; ApplicationSet generates the per-env Applications. Argo Rollouts does canary with automated rollback on Prometheus metrics.

---

## 7.1 — ArgoCD install (HA, SSO, RBAC)

**Why this way / tradeoffs:** HA mode (multiple replicas of the core components + Redis HA) because ArgoCD *is* the deployment control plane — if it's down during an incident you can't ship a fix. Entra SSO so access is governed by the same identity provider as everything else (no separate ArgoCD logins to manage — a SOC2 access-control win). RBAC via `policy.csv` so dev users can see dev but only SRE/platform can sync prod (separation of duties at the deploy layer). Alternatives rejected: single-replica ArgoCD (SPOF for deployments); local ArgoCD accounts (un-auditable, separate credential store).

```yaml
# platform-gitops/platform/argocd/values.yaml  (argo-cd Helm chart)
global:
  domain: argocd.patientportal.example.com

redis-ha:
  enabled: true                 # HA Redis — no single point of failure

controller:
  replicas: 2
repoServer:
  replicas: 2
server:
  replicas: 2
  # Entra (Azure AD) SSO via OIDC
  config:
    oidc.config: |
      name: Entra
      issuer: https://login.microsoftonline.com/REPLACE_TENANT_ID/v2.0
      clientID: REPLACE_ARGOCD_APP_CLIENT_ID
      clientSecret: $oidc.azure.clientSecret    # from a K8s secret, not inline
      requestedScopes: ["openid", "profile", "email", "groups"]
      requestedIDTokenClaims: { groups: { essential: true } }

# RBAC: map Entra groups to ArgoCD roles (separation of duties)
configs:
  rbac:
    policy.csv: |
      # platform/SRE leads: full admin
      g, pp-platform-admins, role:admin
      # developers: read everything, sync only dev
      p, role:dev, applications, get, */*, allow
      p, role:dev, applications, sync, dev/*, allow
      p, role:dev, applications, sync, prod/*, deny
      g, pp-developers, role:dev
      # SRE on-call: sync staging + prod (for incident rollback)
      p, role:sre, applications, sync, */*, allow
      g, pp-sre, role:sre
    policy.default: role:readonly      # default deny-ish: read-only unless granted
```

**The RBAC line that matters:** `p, role:dev, applications, sync, prod/*, deny` — developers can *see* prod apps but cannot *sync* them. Combined with the Phase 6 CODEOWNERS rule (devs can't approve prod PRs), that's defense-in-depth on the separation-of-duties control SOC2 requires.

---

## 7.2 — App-of-apps (the bootstrap)

**Why this way / tradeoffs:** The app-of-apps pattern means *one* root Application points to a directory of other Application definitions — so bootstrapping the whole platform is "apply one manifest, ArgoCD pulls everything else." It makes the platform self-assembling and the GitOps repo the complete declarative truth. Alternatives rejected: manually `kubectl apply` each Application (not declarative, not reproducible); a giant single Application (no per-app sync granularity, one bad app blocks all).

```yaml
# platform-gitops/bootstrap/root-app.yaml
# The ONE manifest you apply to bootstrap everything. It points at applicationsets/.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/platform-gitops
    targetRevision: main
    path: applicationsets        # contains the ApplicationSets below
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 7.3 — ApplicationSet (generates per-env Applications)

**Why this way / tradeoffs:** Without ApplicationSet, you'd hand-write an Application manifest for every app × every environment (appointments-dev, appointments-staging, appointments-prod, notifications-dev…) — dozens of near-identical files that drift. An ApplicationSet with a **generator** templates them all from one definition. The git-directory generator discovers each overlay automatically. Alternatives rejected: hand-written Applications (N×M sprawl); a script that generates them (re-inventing ApplicationSet, not declarative).

```yaml
# platform-gitops/applicationsets/microservices-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    # Matrix generator: every app × every environment overlay
    - matrix:
        generators:
          - list:                          # the environments + their target clusters
              elements:
                - env: dev
                  cluster: https://dev-aks-api    # placeholder cluster URLs
                - env: staging
                  cluster: https://staging-aks-api
                - env: prod
                  cluster: https://prod-aks-api
          - git:                           # discover each microservice
              repoURL: https://github.com/org/platform-gitops
              revision: main
              directories:
                - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}-{{env}}'    # e.g. appointments-api-prod
    spec:
      project: default
      source:
        repoURL: https://github.com/org/platform-gitops
        targetRevision: main
        path: '{{path}}'
        helm:
          valueFiles:
            - 'overlays/values-{{env}}.yaml'   # the right overlay per env
      destination:
        server: '{{cluster}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**What this single file does:** for 3 microservices × 3 environments it generates 9 Applications, each pulling the right chart with the right overlay onto the right cluster — and adding a 4th microservice means *creating one directory*, no ArgoCD changes. That's the leverage of ApplicationSet.

---

## 7.4 — Sync policies, waves & hooks (ordering)

**Why this way / tradeoffs:** Auto-sync + self-heal + prune keeps the cluster *exactly* matching Git (drift is auto-corrected — the selfHeal point from your Q&A). But some things must apply *in order* — a CRD before the resource that uses it, a DB migration before the app that expects the new schema. **Sync waves** order resources; **sync hooks** run jobs at phases (PreSync for a migration). Alternatives rejected: no ordering (app starts before its CRD/schema exists → crash); manual ordering (defeats GitOps automation).

```yaml
# Example: a DB migration must run BEFORE the new app version syncs.
# platform-gitops/apps/appointments-api/templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: appointments-db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync        # runs BEFORE the app syncs
    argocd.argoproj.io/hook-delete-policy: HookSucceeded   # clean up after success
spec:
  template:
    spec:
      serviceAccountName: appointments-api-sa   # workload identity for DB access
      restartPolicy: Never
      containers:
        - name: migrate
          image: ppprodacr.azurecr.io/appointments-api:sha-PLACEHOLDER
          command: ["/migrate", "up"]
```

```yaml
# Sync-wave ordering example (annotations on resources):
#   wave -1: CRDs / namespaces
#   wave  0: configmaps, secrets, serviceaccounts
#   wave  1: deployments, services
#   wave  2: ingress, HPA
# metadata.annotations: { argocd.argoproj.io/sync-wave: "1" }
```

**Rollback:** the GitOps way is `git revert` the tag-bump commit — ArgoCD syncs back to the previous state, fully audited (the "revert the commit" answer from your Q&A). For an emergency, ArgoCD's UI/CLI can roll back to a prior synced revision directly. Git-revert is preferred because it keeps Git as truth.

---

## 7.5 — Progressive delivery: Argo Rollouts + Prometheus analysis

**Why this way / tradeoffs:** A plain Deployment rolling update has no automated safety check — if the new version is subtly broken (higher error rate, slower), it still rolls out fully. **Argo Rollouts** replaces the Deployment with a canary that shifts traffic gradually (5% → 25% → 50% → 100%) and, at each step, runs a **Prometheus AnalysisTemplate** that checks the new version's error rate/latency against a threshold — and **auto-rolls-back** if the metrics are bad. This is the SLO-driven safety net. Alternatives rejected: plain rolling update (no metric-based gate); manual canary (someone has to watch dashboards and decide — slow, error-prone, not 3am-friendly).

```yaml
# platform-gitops/apps/appointments-api/templates/rollout.yaml
# Replaces the Deployment for canary progressive delivery.
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: appointments-api
spec:
  replicas: 3
  selector:
    matchLabels: { app: appointments-api }
  template:
    # ... same pod spec as the Deployment in Phase 5 ...
    metadata:
      labels: { app: appointments-api }
    spec:
      containers:
        - name: appointments-api
          image: ppprodacr.azurecr.io/appointments-api:sha-PLACEHOLDER
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 2m }
        - analysis:                         # automated metric check at 5%
            templates: [{ templateName: error-rate-check }]
        - setWeight: 25
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: error-rate-check }]
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

```yaml
# platform-gitops/apps/appointments-api/templates/analysistemplate.yaml
# The metric gate: if the canary's 5xx rate exceeds 1%, the rollout ABORTS and rolls back.
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      count: 3                              # check 3 times
      successCondition: result < 0.01       # < 1% errors = healthy
      failureLimit: 1                        # one bad reading aborts the rollout
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{app="appointments-api",status=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{app="appointments-api"}[2m]))
```

**This is the senior payoff and it ties the whole platform together:** the canary's safety gate is a PromQL query (the exact error-rate query from your Q&A bank — `rate` before `sum`, 5xx over total) against the Prometheus you set up in Phase 8. If the new version's error rate crosses 1%, Argo Rollouts aborts and reverts automatically — no human, no 3am page to make the call. Progressive delivery + observability + SLOs, working as one system.

Let me show how the canary analysis loop works.The loop on the right (yes → next weight → re-check) is the canary climbing; the path down (no → abort + roll back) is the safety net. The decision is made by a PromQL query, not a person — that's what makes it safe to run unattended.

<img width="1472" height="840" alt="image" src="https://github.com/user-attachments/assets/b6f1f146-221f-4184-a3dd-210d29227dc4" />

---

## 7.6 — Hub-vs-per-cluster ArgoCD (the stated tradeoff)

```
Decision shown: HUB ArgoCD in prod cluster, managing all three clusters.
  + One pane of glass, one RBAC config, simpler SSO.
  − The hub is a cross-environment dependency; its blast radius spans envs.

Alternative (often preferred for strict SOC2 isolation): ArgoCD PER cluster.
  + Full environment isolation — prod ArgoCD can't touch dev and vice versa.
  − Three ArgoCD installs to operate, three RBAC configs.

For strict PHI/SOC2 separation, per-cluster ArgoCD is the more defensible choice;
I showed the hub model for clarity. State which you'd pick and why in an interview —
that tradeoff awareness is the senior signal.
```

---

# PHASE 8 — OBSERVABILITY

🟢 **This is your phase — claim it fully.** Everything here maps to what you do daily and what you built in Athena: Prometheus, ServiceMonitors, recording rules, burn-rate alerts, Alertmanager routing, Grafana, SLOs. You can speak to all of it from real experience. 🟡 The only "learning" caveat: the full kube-prometheus-stack *HA install + storage sizing* at production scale may be platform-team-owned where you work — but the scrape configs, rules, alerts, and dashboards are genuinely yours.

> **Assumptions:** kube-prometheus-stack (Prometheus Operator) via Helm in a `monitoring` namespace. Prometheus with persistent storage + HA (2 replicas). Alertmanager routing to Slack + PagerDuty by severity. Grafana with Entra SSO and dashboards-as-code. SLOs defined for `appointments-api` (99.9% availability per NFR-1) with multi-window burn-rate alerts. Legacy RecordsMgr joins the same stack via the exporter sidecar from Phase 4.

---

## 8.1 — kube-prometheus-stack install

**Why this way / tradeoffs:** The kube-prometheus-stack bundles Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, and kube-state-metrics as one Helm release — so the whole observability plane is declarative and version-pinned. HA (2 Prometheus replicas) because you can't lose visibility during an incident. Persistent storage with generous retention because metrics history *is* your incident-investigation timeline. Resource limits because Prometheus is memory-hungry and an unbounded Prometheus is the thing that OOMs and takes out your monitoring exactly when you need it. Alternatives rejected: hand-deploying each component (version drift, no cohesion); ephemeral storage (lose all history on a pod restart — useless for trend analysis); no resource limits (Prometheus OOMs under cardinality load — the #1 failure from your Q&A).

```yaml
# platform-gitops/platform/kube-prometheus-stack/values.yaml
prometheus:
  prometheusSpec:
    replicas: 2                          # HA — don't lose monitoring during an incident
    retention: 30d                       # 30 days of history for trend/incident analysis
    retentionSize: "45GB"                # cap so the PV can't fill unexpectedly
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-premium   # Azure premium SSD
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    resources:                           # bound it — an unbounded Prometheus OOMs
      requests: { cpu: "1",   memory: 4Gi }
      limits:   { cpu: "2",   memory: 8Gi }
    # Discover ServiceMonitors/PodMonitors across ALL namespaces (not just its own),
    # but only those carrying the release label (avoids picking up junk).
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    replicas: 2
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-premium
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 5Gi } }

grafana:
  replicas: 2
  persistence:
    enabled: true
    storageClassName: managed-premium
    size: 10Gi
  grafana.ini:
    server:
      root_url: https://grafana.patientportal.example.com
    auth.azuread:                        # Entra SSO
      enabled: true
      client_id: REPLACE_GRAFANA_CLIENT_ID
      client_secret: $__file{/etc/secrets/grafana_azuread_secret}
      auth_url: https://login.microsoftonline.com/REPLACE_TENANT/oauth2/v2.0/authorize
      token_url: https://login.microsoftonline.com/REPLACE_TENANT/oauth2/v2.0/token

kube-state-metrics:
  enabled: true                          # cluster object state (pod/deploy/node status)
nodeExporter:
  enabled: true                          # node-level metrics (CPU/mem/disk/network)
```

---

## 8.2 — Scrape configuration: ServiceMonitor & PodMonitor

**Why this way / tradeoffs:** With the Prometheus Operator you don't edit a giant `prometheus.yml` — you declare **ServiceMonitor** CRDs and Prometheus auto-discovers matching targets (the mechanism from your Athena project and your Q&A). The `release` label is the selector that ties a ServiceMonitor to this Prometheus. Relabeling drops dangerous high-cardinality labels *at scrape time* — the front-line defense against the cardinality explosion that kills Prometheus. Alternatives rejected: static scrape configs (don't adapt as pods come and go); no relabeling (a high-cardinality label sails straight in and OOMs Prometheus).

```yaml
# platform-gitops/apps/appointments-api/templates/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: appointments-api
  namespace: appointments
  labels:
    release: kube-prometheus-stack       # MUST match Prometheus's selector
spec:
  selector:
    matchLabels: { app: appointments-api }
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      relabelings:
        # Drop a high-cardinality label if the app ever emits one (defense in depth)
        - action: labeldrop
          regex: "request_id|user_id|trace_id"
```

```yaml
# Example PodMonitor for a workload exposing metrics without a Service
# (e.g. a batch job pod)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-jobs
  namespace: legacy
  labels: { release: kube-prometheus-stack }
spec:
  selector:
    matchLabels: { workload: batch }
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

---

## 8.3 — Recording rules

**Why this way / tradeoffs:** Recording rules pre-compute expensive/frequent queries at scrape interval and store the result as a new metric — so dashboards and alerts read a cheap pre-aggregated series instead of recomputing a heavy query every evaluation. Critically, the **SLI** (success ratio) is computed *once* as a recording rule and referenced by both the dashboard and every burn-rate alert — single source of truth, consistent everywhere. Naming follows `level:metric:operation` (from your Q&A). Alternatives rejected: computing the SLI inline in each alert (drift between alerts, expensive); no recording rules (slow dashboards, repeated heavy queries).

```yaml
# platform-gitops/platform/kube-prometheus-stack/rules/recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: appointments-recording-rules
  namespace: monitoring
  labels: { release: kube-prometheus-stack }
spec:
  groups:
    - name: appointments.sli
      interval: 30s
      rules:
        # The SLI: success ratio over 5m, computed ONCE, reused by alerts + dashboard
        - record: sli:appointments_availability:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{app="appointments-api",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{app="appointments-api"}[5m]))
        # Same at 30m / 1h / 6h windows for multi-window burn-rate (8.5 uses these)
        - record: sli:appointments_availability:ratio_rate30m
          expr: |
            sum(rate(http_requests_total{app="appointments-api",status!~"5.."}[30m]))
            /
            sum(rate(http_requests_total{app="appointments-api"}[30m]))
        - record: sli:appointments_availability:ratio_rate1h
          expr: |
            sum(rate(http_requests_total{app="appointments-api",status!~"5.."}[1h]))
            /
            sum(rate(http_requests_total{app="appointments-api"}[1h]))
        - record: sli:appointments_availability:ratio_rate6h
          expr: |
            sum(rate(http_requests_total{app="appointments-api",status!~"5.."}[6h]))
            /
            sum(rate(http_requests_total{app="appointments-api"}[6h]))
        # p99 latency, computed once (aggregate buckets first — can't avg percentiles)
        - record: sli:appointments_latency_p99:rate5m
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{app="appointments-api"}[5m]))
              by (le))
```

---

## 8.4 — SLO / SLI / error-budget definition

**Why this way / tradeoffs:** This is the heart of SRE — turning NFR-1 (99.9% availability) into a *measured, alerted* objective. The SLO is the target, the SLI is the recording rule above, the error budget is `1 − SLO`, and the burn-rate alerts (next section) fire on how fast you're spending it. This is what makes "highly available" a number, not a vibe. This is straight from your reliability Q&A — claim it.

```
SLO definition for appointments-api (from NFR-1):
  SLI:  proportion of non-5xx requests (sli:appointments_availability:ratio_rate*)
  SLO:  99.9% over 30 days
  Error budget: 1 − 0.999 = 0.1%  ≈ 43 minutes of allowed failure per 30 days

  Latency SLO (from NFR-2):
  SLI:  p99 request duration (sli:appointments_latency_p99:rate5m)
  SLO:  p99 < 300ms (read path)

How the budget drives behavior:
  - Budget remaining → ship features, take risks (canaries can be more aggressive).
  - Budget burning fast → freeze risky releases, focus on reliability.
  This is the "shared currency between dev and ops" framing — a data-driven
  decision instead of an argument about how reliable to be.
```

---

## 8.5 — Alert rules (the real ones)

**Why this way / tradeoffs:** Multi-window, multi-burn-rate SLO alerting is the senior pattern (from your Q&A and Athena): a *fast* burn (page now — you'll exhaust the budget in hours) and a *slow* burn (ticket — sustained low-grade erosion). Two windows per alert (short confirms it's happening now, long confirms it's sustained) eliminate flapping. Plus the operational alerts: crashloop, cert expiry, node pressure, saturation. Every alert is *actionable* — the rule from your real alert-cleanup work. Alternatives rejected: static-threshold alerts ("error rate > 1%" — fires on transient spikes, no urgency tiering); single-window burn rate (flaps).

```yaml
# platform-gitops/platform/kube-prometheus-stack/rules/alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: appointments-alerts
  namespace: monitoring
  labels: { release: kube-prometheus-stack }
spec:
  groups:
    - name: appointments.slo.burnrate
      rules:
        # FAST burn: 14.4x burn rate over 1h AND 5m both breaching → PAGE.
        # 14.4x burn exhausts a 30-day budget in ~2 days — urgent.
        - alert: AppointmentsErrorBudgetFastBurn
          expr: |
            (1 - sli:appointments_availability:ratio_rate1h) > (14.4 * 0.001)
            and
            (1 - sli:appointments_availability:ratio_rate5m) > (14.4 * 0.001)
          for: 2m
          labels: { severity: page, app: appointments-api }
          annotations:
            summary: "Appointments fast error-budget burn"
            description: "Burning >14.4x budget over 1h and 5m — page on-call."
            runbook_url: "https://runbooks.example.com/appointments-burn"

        # SLOW burn: 6x over 6h AND 30m → TICKET (not a page).
        - alert: AppointmentsErrorBudgetSlowBurn
          expr: |
            (1 - sli:appointments_availability:ratio_rate6h) > (6 * 0.001)
            and
            (1 - sli:appointments_availability:ratio_rate30m) > (6 * 0.001)
          for: 15m
          labels: { severity: ticket, app: appointments-api }
          annotations:
            summary: "Appointments slow error-budget burn"
            description: "Sustained budget erosion — investigate, not urgent."

    - name: appointments.latency
      rules:
        - alert: AppointmentsLatencyP99High
          expr: sli:appointments_latency_p99:rate5m > 0.3   # 300ms SLO (NFR-2)
          for: 10m
          labels: { severity: ticket, app: appointments-api }
          annotations:
            summary: "Appointments p99 latency above 300ms SLO"

    - name: platform.operational
      rules:
        # Crashloop (your real incident type)
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[15m]) > 3
          for: 5m
          labels: { severity: page }
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash-looping"

        # Cert expiry (cert-manager certs from Phase 3)
        - alert: CertExpiringSoon
          expr: |
            (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 14
          for: 1h
          labels: { severity: ticket }
          annotations:
            summary: "TLS cert {{ $labels.name }} expires in <14 days"

        # Node memory pressure
        - alert: NodeMemoryPressure
          expr: |
            (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.90
          for: 10m
          labels: { severity: page }
          annotations:
            summary: "Node {{ $labels.instance }} memory >90%"

        # Prometheus cardinality guard (protect the monitoring itself)
        - alert: PrometheusHighCardinality
          expr: prometheus_tsdb_head_series > 1500000
          for: 30m
          labels: { severity: ticket }
          annotations:
            summary: "Prometheus head series very high — investigate cardinality"
```

**The cardinality-guard alert is the connoisseur's touch** — you alert on `prometheus_tsdb_head_series` to catch a cardinality explosion *before* it OOMs Prometheus. That's directly from your Q&A ("monitor `prometheus_tsdb_head_series`") and it's the kind of detail that signals you've actually operated Prometheus at scale.

---

## 8.6 — Alertmanager routing tree

**Why this way / tradeoffs:** Routing sends `page` severity to PagerDuty (wakes someone) and `ticket` severity to Slack (handled in business hours) — matching urgency to channel. **Inhibition** suppresses symptom alerts when a root-cause alert is firing (if a node is down, don't also page for every pod on it). Grouping bundles related alerts into one notification. This is the noisy-alert-reduction discipline from your real cleanup work, expressed as config. Alternatives rejected: everything to one channel (alert fatigue — the exact problem you fixed); no inhibition (one node failure = a storm of pod alerts).

```yaml
# platform-gitops/platform/kube-prometheus-stack/alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: pp-routing
  namespace: monitoring
  labels: { release: kube-prometheus-stack }
spec:
  route:
    receiver: slack-default
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    routes:
      - matchers: [{ name: severity, value: page }]
        receiver: pagerduty
        continue: false
      - matchers: [{ name: severity, value: ticket }]
        receiver: slack-default
  inhibitRules:
    # If a node is down (page), suppress the pod-level alerts on it (don't double-alert)
    - sourceMatch: [{ name: alertname, value: NodeMemoryPressure }]
      targetMatch: [{ name: alertname, value: PodCrashLooping }]
      equal: ['instance']
  receivers:
    - name: pagerduty
      pagerdutyConfigs:
        - routingKey:
            name: pagerduty-key
            key: routingKey            # from a K8s secret
    - name: slack-default
      slackConfigs:
        - apiURL:
            name: slack-webhook
            key: url
          channel: '#pp-alerts'
          sendResolved: true
```

---

## 8.7 — Grafana provisioning (datasources + dashboards as code)

**Why this way / tradeoffs:** Dashboards and datasources are provisioned *as code* (ConfigMaps the Grafana sidecar auto-loads) — so a dashboard is version-controlled, reviewed, and reproducible, not click-built and lost. Folders + Entra SSO control who sees what. Alternatives rejected: hand-built dashboards in the UI (not reproducible, lost if Grafana is rebuilt, no review).

```yaml
# Datasource is auto-provisioned by kube-prometheus-stack (points at in-cluster Prometheus).
# Dashboards as code: a ConfigMap labeled grafana_dashboard is auto-discovered.
apiVersion: v1
kind: ConfigMap
metadata:
  name: appointments-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"            # the sidecar auto-loads any CM with this label
data:
  appointments.json: |
    {
      "title": "Appointments API — RED + SLO",
      "uid": "appointments-red",
      "tags": ["patientportal", "slo"],
      "timezone": "utc",
      "schemaVersion": 39,
      "panels": [
        {
          "title": "Request rate (traffic)",
          "type": "timeseries",
          "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
          "targets": [{
            "expr": "sum(rate(http_requests_total{app=\"appointments-api\"}[5m]))",
            "legendFormat": "req/s"
          }]
        },
        {
          "title": "Error rate (errors)",
          "type": "timeseries",
          "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
          "targets": [{
            "expr": "1 - sli:appointments_availability:ratio_rate5m",
            "legendFormat": "error ratio"
          }],
          "fieldConfig": {"defaults": {"unit": "percentunit"}}
        },
        {
          "title": "p99 latency (duration)",
          "type": "timeseries",
          "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
          "targets": [{
            "expr": "sli:appointments_latency_p99:rate5m",
            "legendFormat": "p99"
          }],
          "fieldConfig": {"defaults": {"unit": "s"}}
        },
        {
          "title": "Error budget remaining (30d)",
          "type": "stat",
          "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
          "targets": [{
            "expr": "1 - ((1 - sli:appointments_availability:ratio_rate6h) / 0.001)",
            "legendFormat": "budget left"
          }],
          "fieldConfig": {"defaults": {"unit": "percentunit"}}
        }
      ]
    }
```

That dashboard is deliberately the **RED method** (Rate, Errors, Duration) plus an error-budget stat — the exact structure from your observability Q&A. A reviewer can see it's RED at a glance.

---

## 8.8 — How legacy emits into the SAME stack

This closes the Phase 4 loop. Because RecordsMgr has the exporter sidecar + a ServiceMonitor with the `release: kube-prometheus-stack` label (Phase 4.4), it's scraped by this *same* Prometheus, appears on Grafana, and can have the same kinds of alerts. For what can't be instrumented, the **blackbox probe** (Phase 4.4) gives an external up/latency signal. One observability plane covers modern *and* legacy — the unified-operations win, and the part of this whole guide that maps most directly to real DevOps/SRE work you can claim.

```
Legacy → same stack, three mechanisms:
  1. Exporter sidecar (windows_exporter) + ServiceMonitor → IIS/Windows metrics
  2. Blackbox probe → external "is /healthz returning 2xx, how fast" signal
  3. Pushgateway (only if a short-lived batch job can't be scraped before it exits)
All land in the same Prometheus → same Grafana dashboards → same Alertmanager routing.
```

---

# PHASE 9 — RUNTIME SECURITY & POLICY

🟡 **Learning (mostly):** Admission policy, secret-operator wiring, and runtime threat detection are security-engineering tasks. 🟢 **Claimable threads:** you understand workload identity / no-stored-secrets, the no-`latest`-tag and resource-limits principles (they're in your Q&A and Helm charts), and network policies. Frame as "I understand these controls and why they exist; authoring the Kyverno policies and wiring the secret operator was security-team work."

> **Assumptions:** External Secrets Operator (ESO) syncs Key Vault → K8s secrets via workload identity (no static creds). Kyverno for admission control (lighter and more readable than Gatekeeper/OPA for this team). Microsoft Defender for Containers for runtime threat detection (Azure-native; Falco noted as the open-source alternative). These are the *enforcement* layer that makes the Phase 6 signing and Phase 5 chart hygiene actually mandatory at deploy time.

---

## 9.1 — External Secrets Operator + Key Vault

**Why this way / tradeoffs:** The Phase 5 charts reference an `ExternalSecret` pointing at a `ClusterSecretStore` — this is where that store is defined. ESO authenticates to Key Vault via **workload identity** (no stored credential), pulls the secret, and syncs it into a normal K8s Secret the pod consumes. The secret value lives only in Key Vault; Git holds only a *reference* to its name. Alternatives rejected: secrets in Git (plaintext leak, even base64 isn't encryption — your Q&A gotcha); the CSI Secrets Store driver mounting as files (valid alternative, but ESO syncing to a native Secret is simpler for apps that expect env vars; I note CSI as the other option); static service-principal credential for Key Vault access (long-lived secret = SOC2 finding).

```yaml
# platform-gitops/platform/external-secrets/clustersecretstore.yaml
# The store the Phase 5 ExternalSecret references. Auth = workload identity, no secret.
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity        # federated token, NO stored credential
      vaultUrl: "https://pp-prod-kv.vault.azure.net"
      serviceAccountRef:
        name: external-secrets-sa        # SA federated to the ESO managed identity
        namespace: external-secrets
```

```yaml
# platform-gitops/platform/external-secrets/serviceaccount.yaml
# ESO's ServiceAccount, federated to a managed identity granted "Key Vault Secrets User"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: external-secrets
  annotations:
    azure.workload.identity/client-id: REPLACE_ESO_WI_CLIENT_ID
```

The full secret chain, end to end (worth narrating in an interview): Key Vault holds the secret → ESO's ServiceAccount federates to a managed identity with `Key Vault Secrets User` → ESO reads the secret over the private endpoint → syncs it into a K8s Secret in the app namespace → the pod (via the Phase 5 chart) mounts it as an env var. **No credential is stored anywhere in that chain** — every hop is a short-lived federated token. That's the "secrets come from the environment at runtime, not the source" principle fully realized.

---

## 9.2 — Kyverno admission policies

**Why this way / tradeoffs:** Phase 6 *signs* images and Phase 5 charts *set* resource limits — but nothing *forces* it until admission control rejects anything that doesn't comply. Kyverno is the admission webhook that makes the rules mandatory: only signed images run, no `latest` tag reaches the cluster, every pod must declare resource limits. Kyverno over Gatekeeper/OPA because its policies are plain YAML (no Rego to learn) — more maintainable for most teams. Alternatives rejected: trusting CI alone (someone bypasses CI, or applies a manifest directly — admission is the last line); OPA Gatekeeper (more powerful but Rego is a steeper learning curve); PodSecurityPolicy (removed in 1.25).

```yaml
# platform-gitops/platform/kyverno/policies/require-signed-images.yaml
# Only images signed by our cosign key (Phase 6) may run. Rejects unsigned images.
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce       # Enforce = reject; Audit = warn-only
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-acr-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "ppprodacr.azurecr.io/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      REPLACE_COSIGN_PUBLIC_KEY
                      -----END PUBLIC KEY-----
```

```yaml
# platform-gitops/platform/kyverno/policies/disallow-latest-tag.yaml
# No ':latest' (or untagged) images — enforces immutable tags (your Q&A + Phase 1 decision)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-explicit-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Image tag ':latest' or missing tag is not allowed — use an immutable tag."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

```yaml
# platform-gitops/platform/kyverno/policies/require-resource-limits.yaml
# Every container must declare CPU+memory requests and limits (your QoS/eviction Q&A).
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-resources
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "CPU and memory requests and limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    memory: "?*"     # must be set (non-empty)
                    cpu: "?*"
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

**Why each policy is here, tied back:** signed-images-only makes the Phase 6 cosign signing *meaningful* (an unsigned image — even pushed manually — can't run); no-`latest` enforces the immutable-tag decision from Phase 1; resource-limits-required means no pod can land without requests/limits, which protects QoS/eviction behavior (your OOMKilled and eviction-order Q&A). Each is a *requirement turned into an enforced gate* — the recurring theme of this whole guide.

> **Assumption / real-world note:** you roll these out as `Audit` (warn) first to find existing violations, then flip to `Enforce` — flipping straight to Enforce on a live cluster rejects existing non-compliant workloads and causes an outage. The audit-then-enforce ratchet is the safe operational path (same judgment as the Phase 6 scan-ratchet).

---

## 9.3 — Network policies (default-deny posture)

This builds on Phase 3 (Cilium enforces them) and the per-app NetworkPolicy in the Phase 5 chart. The cluster-wide posture:

**Why this way / tradeoffs:** Default-deny egress as the baseline means a compromised pod can't phone home or exfiltrate PHI to an arbitrary destination — it can only reach what's explicitly allowed (DNS, the database, the API). For a PHI/SOC2 platform, controlled egress is a real control, not paranoia. Alternatives rejected: default-allow (a compromised pod has free rein — unacceptable for PHI); no policies (relies on Phase 3's CNI doing nothing — and recall the Q&A gotcha that a policy silently does nothing if the CNI doesn't enforce it; here Cilium does).

```yaml
# platform-gitops/platform/network-policies/default-deny-egress.yaml
# Baseline: deny all egress, then allow DNS + the specific dependencies per app.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: appointments
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    # Allow DNS only (to CoreDNS) — everything else must be explicitly added
    - to:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: kube-system }
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

```yaml
# Then allow the specific egress the app needs (e.g. to PostgreSQL private endpoint)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-postgres
  namespace: appointments
spec:
  podSelector:
    matchLabels: { app: appointments-api }
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock: { cidr: 10.10.16.0/24 }    # the private-endpoint subnet (Phase 2)
      ports:
        - { protocol: TCP, port: 5432 }
```

---

## 9.4 — Runtime threat detection

**Why this way / tradeoffs:** Everything before this is *preventive* (stop bad things at admission). Runtime detection catches what gets through — a compromised container doing something it shouldn't *at runtime* (spawning a shell, reading `/etc/shadow`, unexpected network connections, crypto-mining). Defender for Containers is Azure-native (integrates with the existing Azure security posture and Defender for Cloud), so it's the natural pick on AKS. Falco is the open-source alternative (eBPF syscall monitoring) if you want cloud-agnostic. Alternatives rejected: no runtime detection (prevention isn't perfect — you need detection for the residual risk, and SOC2 expects detective controls alongside preventive ones).

```
Runtime threat detection — Defender for Containers (AKS-native):
  - Enabled via the AKS Defender add-on (Terraform: microsoft_defender block on the
    cluster, or the subscription-level Defender plan).
  - Detects: shell spawned in a container, reverse shells, crypto-mining patterns,
    access to sensitive mounts, anomalous outbound connections, known-malicious images.
  - Alerts flow into Defender for Cloud → can route to the same Alertmanager/Slack
    path (Phase 8) so security alerts share the incident pipeline.

Falco (open-source alternative):
  - Runs as a DaemonSet, watches syscalls via eBPF, fires on rule matches
    (e.g. "shell in container", "write below /etc"). Cloud-agnostic.
  - Pick Falco if you want portability off Azure; Defender if you want native
    integration with Defender for Cloud and the existing Azure security posture.
```

```hcl
# platform-infra/modules/aks/main.tf (add to the cluster resource)
  microsoft_defender {
    log_analytics_workspace_id = var.log_analytics_workspace_id  # findings land here
  }
```

---

## 9.5 — The security layers, together

The preventive-to-detective progression across the whole platform:

```
Layer (where it sits)              Control                                  Phase
-----------------------------------------------------------------------------------
Pipeline (pre-merge)               scan code/deps/IaC, sign image            6
Admission (deploy-time, preventive) Kyverno: signed-only, no-latest, limits  9
Pod identity (runtime, no secrets)  workload identity + ESO/Key Vault        2/9
Network (runtime, preventive)       default-deny egress, network policies    3/9
Runtime (detective)                 Defender/Falco threat detection          9
Audit (continuous)                  Git history + Defender for Cloud + logs  6/9
```

Every layer assumes the one before it can fail — defense in depth. The image is scanned in CI *and* must be signed *and* the signature is verified at admission *and* the running container is watched for anomalies. That layering is the senior security story.

---
# PHASE 10 — SRE OPERATIONS / DAY-2

🟢 **Strongly claimable — this is your daily reality.** Incident response, on-call, escalation, following runbooks, node operations — this maps directly to your real work. The DR-failover and Velero-backup *design* is more lead-owned, but the *operating* side (responding to alerts, triaging, escalating with context) is genuinely yours. Lead with that.

> **Assumptions:** Runbooks live in the GitOps repo (versioned, reviewed). On-call uses the Phase 8 Alertmanager → PagerDuty path. DR targets from Phase 0: RTO < 1 hour, RPO < 5 min. Velero backs up cluster state to Azure Blob; the data tier relies on Azure SQL/PostgreSQL geo-replication (DB backups are DBA-owned, not Velero's job).

---

## 10.1 — The incident-response flow (tied to the Phase 8 alerts)

**Why this way / tradeoffs:** The flow connects an alert firing to a human action to a postmortem — closing the loop. It's built so the *fast-burn page* (Phase 8) triggers the stabilize-first response, and everything is tied to runbooks so any on-call engineer responds consistently. This is the systematic-beats-fast discipline from your real on-call experience. Alternatives rejected: ad-hoc response (every incident handled differently, tribal knowledge); no postmortem loop (same incident recurs).

```
Incident flow (from alert to postmortem):

1. ALERT FIRES → PagerDuty pages on-call (severity: page, e.g. fast error-budget burn)
2. ACKNOWLEDGE → on-call ack's within the response SLA; opens an incident channel
3. STABILIZE FIRST → is there an immediate mitigation? (roll back the last deploy via
   git-revert → ArgoCD syncs back; scale up; fail over). Stop the bleeding BEFORE
   deep debugging — the error budget burns while you investigate.
4. GATHER SIGNAL → the three pillars: dashboards (which golden signal?), traces (which
   hop?), logs (exact error). Check "what changed" — last deploy/config (Phase 6 Git
   history is the audit trail of what changed).
5. FOLLOW THE RUNBOOK → if a runbook exists for this alert (linked in the alert
   annotation, 8.5), follow it.
6. RESOLVE or ESCALATE → ops-level issue (crashloop, image-pull, node pressure) →
   resolve. App bug / out of scope → escalate to L3 with a full diagnostic trail.
7. POSTMORTEM → blameless: timeline, root cause, impact (budget consumed), action
   items with owners. Update the runbook so next time is faster.
```

This *is* your real on-call process and your real incident stories (OOMKilled, CrashLoopBackOff, alert cleanup) — you can narrate this flow with genuine authority because you've lived every step.

---

## 10.2 — Runbook: cluster / node upgrades

**Why this way / tradeoffs:** AKS upgrades are two-part (control plane Azure-managed, node pools you trigger) and the safe order is control-plane-first, then node pools one at a time with surge, respecting PDBs (Phase 5 set them). Test in non-prod first. This is operational knowledge you have. Alternatives rejected: upgrading everything at once (capacity dip, simultaneous disruption); skipping PDBs (draining too many replicas → outage).

```markdown
# runbooks/cluster-upgrade.md

## Pre-flight
- [ ] Confirm target K8s version is N-2 or newer from current (AKS support window).
- [ ] Run the upgrade in DEV → STAGING first; soak 24h, watch SLO dashboards.
- [ ] Confirm PodDisruptionBudgets exist for all critical apps (Phase 5).
- [ ] Announce maintenance window (the Sunday 03:00 window, Phase 3).

## Execute (prod)
1. Upgrade CONTROL PLANE only first:
   az aks upgrade -g pp-prod-aks-rg -n pp-prod-aks --control-plane-only -k <ver>
2. Upgrade node pools one at a time (NOT all at once):
   az aks nodepool upgrade -g pp-prod-aks-rg --cluster-name pp-prod-aks -n user -k <ver>
   # cordon+drain respects PDBs; max_surge=33% brings new nodes up before draining old
3. Validate after EACH pool: nodes Ready, pods rescheduled, SLO dashboards green.
4. Upgrade the WINDOWS pool LAST (slowest image pulls).

## Rollback
- Control plane can't be downgraded — forward-fix only. This is why staging soak matters.
- If a node pool upgrade misbehaves: pause, investigate drained workloads, do not
  proceed to the next pool until green.
```

---

## 10.3 — Runbook: cert rotation

cert-manager (Phase 3) auto-renews Let's Encrypt certs, so rotation is mostly *monitoring that it happened* — the `CertExpiringSoon` alert (Phase 8) is the safety net if auto-renewal fails.

```markdown
# runbooks/cert-rotation.md

## Normal (automatic)
cert-manager renews ~30 days before expiry. No action needed if healthy.

## If CertExpiringSoon alert fires (<14 days, auto-renewal didn't happen)
1. Check the Certificate + CertificateRequest status:
   kubectl describe certificate <name> -n <ns>
   kubectl get certificaterequest -n <ns>
2. Common causes: DNS-01 solver failing (check the Azure DNS workload identity perms),
   ACME rate limit, cert-manager pod unhealthy.
3. Check cert-manager logs: kubectl logs -n cert-manager deploy/cert-manager
4. Force a renewal if needed:
   kubectl cert-manager renew <cert-name> -n <ns>
5. Verify the new cert's notAfter date and that ingress is serving it.
```

---

## 10.4 — Runbook: scaling

```markdown
# runbooks/scaling.md

## Automatic (normal)
- HPA scales pods on CPU (Phase 5 charts). Cluster Autoscaler adds nodes when pods
  can't schedule. Two layers — HPA for pods, CA for nodes.

## Manual scale (incident — sudden load)
1. Confirm it's load, not a leak: memory tracking traffic = load; memory climbing on
   a flat-traffic slope = leak (do NOT just scale — that masks a leak; see OOM runbook).
2. Temporarily raise HPA max:
   kubectl patch hpa appointments-api -n appointments --patch \
     '{"spec":{"maxReplicas":20}}'
3. If nodes are the limit, confirm Cluster Autoscaler is adding them
   (kubectl get events | grep -i scale); check node pool max_count isn't hit.
4. After the event: return HPA max to baseline (don't leave inflated limits).
```

The leak-vs-load distinction at the top is your real OOMKilled lesson encoded as a runbook step — a nice authentic touch you can speak to.

---

## 10.5 — Runbook: backup / restore (Velero)

**Why this way / tradeoffs:** Velero backs up *Kubernetes object state* (and optionally PVs) to Azure Blob — so you can restore the cluster's workloads after a disaster or accidental deletion. It does *not* replace database backups — the data tier (Azure SQL/PostgreSQL) uses its own geo-replication and point-in-time restore (DBA-owned). Splitting these is deliberate: Velero for cluster state, managed-DB features for data. Alternatives rejected: Velero for everything including DB (wrong tool for live transactional data); no cluster-state backup (a deleted namespace or a bad sync is unrecoverable without it).

```yaml
# platform-gitops/platform/velero/schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-cluster-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"          # 02:00 daily
  template:
    includedNamespaces:
      - appointments
      - notifications
      - records
      - legacy
    storageLocation: azure-blob
    ttl: 720h0m0s                # retain 30 days
    snapshotVolumes: true        # snapshot PVs too
```

```markdown
# runbooks/backup-restore.md

## Restore a namespace (e.g. accidental deletion)
1. List backups: velero backup get
2. Restore: velero restore create --from-backup daily-cluster-backup-<timestamp> \
     --include-namespaces appointments
3. Verify: pods running, ArgoCD shows the app Synced/Healthy.
   NOTE: ArgoCD will also try to reconcile from Git — ensure Git and the restore agree,
   or ArgoCD's selfHeal may fight the restore. Pause auto-sync during a restore if needed.

## Data tier (DB) — NOT Velero
- Azure SQL / PostgreSQL: use point-in-time restore or geo-restore (DBA-owned).
- RPO < 5 min (NFR-5) is met by the DB's geo-replication, not Velero.
```

The "ArgoCD selfHeal may fight the restore" note is a real gotcha — GitOps means Git is truth, so a restore that disagrees with Git gets reverted. Pausing auto-sync during restore is the kind of operational detail that signals real GitOps experience.

---

## 10.6 — Runbook: DR failover (tied to RTO/RPO)

**Why this way / tradeoffs:** The DR design is driven directly by Phase 0's RTO < 1hr / RPO < 5min (NFR-4/5). RPO < 5min → the data tier must geo-replicate continuously (not nightly backups). RTO < 1hr → a warm secondary in West US 2, not a cold rebuild. Front Door (Phase 1) does the traffic failover. Alternatives rejected: cold DR / restore-from-backup (can't hit a 1-hour RTO); active-active multi-region (overkill and far costlier than the requirement demands — match the design to the RTO/RPO, don't gold-plate).

```markdown
# runbooks/dr-failover.md
# Targets (Phase 0): RTO < 1 hour, RPO < 5 min. Primary East US 2 → DR West US 2.

## Pre-conditions (always-on, verified by drills)
- ACR is geo-replicated to West US 2 (Phase 2) → images available in DR.
- Azure SQL / PostgreSQL active geo-replication to West US 2 (RPO < 5 min).
- DR AKS cluster exists (smaller baseline, scales on failover) OR is provisioned
  via the same Terraform with the westus2 env.
- GitOps repo is region-agnostic → ArgoCD in DR syncs the same apps.

## Failover steps
1. DECLARE the incident; confirm primary region is truly down (not a transient blip).
2. PROMOTE the DR database replica to primary (Azure SQL failover group / PostgreSQL
   promote). This is the RPO-critical step — confirm replication lag was < 5 min.
3. SCALE the DR AKS cluster to production capacity (cluster autoscaler + raise HPA maxes).
4. Ensure ArgoCD in DR has synced all apps (Healthy).
5. FAIL OVER TRAFFIC: update Azure Front Door (Phase 1) to route to the West US 2
   backend. DNS/Front Door propagation is the RTO-critical step.
6. VALIDATE: SLO dashboards (Phase 8) green in DR; run smoke tests against key journeys.

## Failback (after primary recovers)
- Re-establish replication primary→DR direction, drain, then fail back during a
  maintenance window. Never rush failback — the DR region is serving fine.

## Drills
- Run a DR failover drill quarterly (SOC2 expects tested DR, not just documented).
  Measure actual RTO/RPO against the targets; the drill IS the evidence for the audit.
```

The closing line is the SOC2 point: a documented-but-untested DR plan fails an audit. The *drill* is the evidence — and measuring actual RTO/RPO against target is exactly the rigor an auditor (and a senior interviewer) wants to hear.

---

## 10.7 — On-call & escalation

```markdown
# runbooks/oncall.md

## Rotation
- Primary on-call (first responder) + secondary (backup). Weekly handoff with a
  documented handoff note (open incidents, known issues, recent changes).

## Severity → response
- PAGE (fast burn, node down, crashloop on critical app): ack within SLA, stabilize first.
- TICKET (slow burn, latency over SLO, cert expiry): business-hours handling.

## Escalation path
1. Primary on-call attempts stabilization + ops-level fixes.
2. If out of scope (app bug) or not resolved within the escalation timer →
   escalate to L3/engineering WITH a full diagnostic trail (metric trend, trace,
   suspect component) — not just "it's broken."
3. Major incident (sustained user impact) → incident commander role, comms owner,
   status updates on a cadence.

## Principle
Systematic beats fast. Escalating with good context is a strength, not a weakness.
A complete handoff cuts the next person's investigation time dramatically.
```

This is *verbatim your real on-call philosophy* — the escalate-with-context principle, the diagnostic-trail handoff, "systematic beats fast." You can speak to this entire runbook from genuine experience. It's the most claimable artifact in the whole guide.

---

## Phase 10 — Honest framing for your interviews

This phase is, alongside Phase 8, your strongest claimable material.

- 🟢 **Claim fully:** "Day-2 operations is my core — I respond to alerts, stabilize first, gather signal across metrics/traces/logs, follow runbooks, resolve ops-level issues, and escalate app bugs to L3 with a complete diagnostic trail. I've handled node operations and upgrades, and I've written and improved runbooks. My on-call philosophy is systematic-beats-fast and escalate-with-context." Every word real.
- 🟡 **Be honest about:** owning the *DR failover design* and the *Velero/backup architecture* — those are more lead/architect-owned. If drilled: *"DR strategy and the backup architecture were designed by our architects to meet the RTO/RPO; I understand the failover steps and I've participated in the operational side. Owning DR design end-to-end is something I'd grow into."*

The interview move: when Day-2/SRE-ops comes up, **be expansive and confident** — this is where your 4 years genuinely live. Incident response, on-call maturity, and runbook discipline are 100% yours.

---

# SECURITY GATE SUMMARY (recap)

| Layer | Control | Tool | Enforcement | Phase |
|-------|---------|------|-------------|-------|
| Pre-merge | Secret scan | gitleaks | Block on any secret | 6 |
| Pre-merge | Unit tests + coverage | go test | Block < 70% | 6 |
| Pre-merge | SAST | Semgrep / SonarQube | Block on ≥ HIGH | 6 |
| Pre-merge | Dependency/license (SCA) | Trivy fs | Block on CVE ≥ HIGH | 6 |
| Pre-merge | IaC misconfig | Checkov | Block on misconfig | 6 |
| Pre-merge | Manifest validity/lint | kubeconform, kube-linter | Block on invalid | 6 |
| Build | Image scan | Trivy image | Block on CVE ≥ HIGH | 6 |
| Build | Image signing | cosign | Block (verified at admission) | 6/9 |
| Promote | Staging approval | GitHub env | 1 reviewer | 6 |
| Promote | Prod approval | GitHub env + CODEOWNERS | 2 groups + cool-off | 6 |
| Admission | Signed images only | Kyverno | Reject unsigned | 9 |
| Admission | No `latest` tag | Kyverno | Reject mutable tag | 9 |
| Admission | Resource limits required | Kyverno | Reject if missing | 9 |
| Admission | Pod Security Standard | PSS (restricted) | Reject privileged | 3 |
| Runtime | No stored secrets | Workload Identity + ESO | Federated tokens only | 2/9 |
| Runtime | Network isolation | Cilium NetworkPolicy | Default-deny | 3/9 |
| Runtime | Threat detection | Defender / Falco | Detect + alert | 9 |
| Continuous | Audit trail | Git history + Defender for Cloud | Every change logged | 6/9 |

---

# BOOTSTRAP ORDER (zero-to-prod checklist)

The exact sequence to build this platform from nothing:

```
PHASE 0–1 — Design (do before touching Azure)
[ ]  1. Requirements doc + NFRs signed off (Phase 0)
[ ]  2. 6 R's legacy assessment + migration plan (Phase 0)
[ ]  3. Architecture, isolation model, repo strategy decided (Phase 1)
[ ]  4. Create the three repos: platform-infra, platform-gitops, app repos (Phase 1)

PHASE 2 — Azure foundation (Terraform)
[ ]  5. Bootstrap Terraform state backend (storage account) — local state ONCE (2.0)
[ ]  6. Apply governance: naming, tagging, Azure Policy (residency, required tags) (2.1)
[ ]  7. Apply networking: VNet, subnets, NSGs, NAT, private DNS zones (2.2)
[ ]  8. Apply identity: managed identities, workload-identity federation setup (2.3)
[ ]  9. Apply Key Vault, ACR (Premium, geo-replicated), storage (2.4)

PHASE 3 — AKS
[ ] 10. Apply AKS cluster: private API, Entra RBAC, OIDC issuer, system pool (3.1)
[ ] 11. Add node pools: user, spot, Windows (3.2)
[ ] 12. Capture the AKS OIDC issuer URL → feed back into identity federation (2.3)

PHASE 7 (partial) — GitOps control plane FIRST (it deploys everything else)
[ ] 13. Install ArgoCD (HA, Entra SSO, RBAC policy.csv) (7.1)
[ ] 14. Apply the root app-of-apps → it pulls the ApplicationSets (7.2/7.3)

PHASE 3/9 — Platform components (via GitOps, in dependency order / sync waves)
[ ] 15. ingress-nginx, cert-manager + ClusterIssuer (3.3)
[ ] 16. Cilium network policies: default-deny baseline (3.3/9.3)
[ ] 17. External Secrets Operator + ClusterSecretStore (9.1)
[ ] 18. Kyverno + policies — as AUDIT first, then flip to ENFORCE (9.2)

PHASE 8 — Observability (before apps, so apps are observed from day one)
[ ] 19. kube-prometheus-stack (Prometheus HA, Alertmanager, Grafana SSO) (8.1)
[ ] 20. Recording rules, SLO burn-rate + operational alert rules (8.3/8.5)
[ ] 21. Alertmanager routing (PagerDuty/Slack), Grafana dashboards as code (8.6/8.7)

PHASE 5/6/7 — Applications
[ ] 22. Author Helm charts + library chart + env overlays (Phase 5)
[ ] 23. Wire CI: GitHub Actions (cloud-native) + Azure DevOps (legacy) with gates (Phase 6)
[ ] 24. Configure branch protection, CODEOWNERS, environment approvals (6.4)
[ ] 25. First deploy to DEV via CI → GitOps PR → ArgoCD sync; validate observability
[ ] 26. Promote dev → staging → prod via gated PRs; enable Argo Rollouts canary (7.5)

PHASE 4 — Legacy migration (in parallel, incremental)
[ ] 27. Replatform RecordsMgr (Windows container) behind same ingress (4.2)
[ ] 28. Wire legacy into observability (exporter sidecar + ServiceMonitor) (4.4)
[ ] 29. Strangler-fig: shadow → canary → shift read traffic to records-read-api (4.1)
[ ] 30. Data-layer migration (DBA-led); eventually retire Windows pool (4.6/4.1)

PHASE 10 — Day-2
[ ] 31. Author runbooks (upgrade, cert, scaling, backup, DR) (Phase 10)
[ ] 32. Velero backup schedule; DB geo-replication for RPO (10.5)
[ ] 33. On-call rotation + Alertmanager → PagerDuty live (10.7)
[ ] 34. Run a DR failover drill; measure RTO/RPO vs target (SOC2 evidence) (10.6)
```

The non-obvious ordering insight: **ArgoCD goes in early (step 13)** because once it's running, everything else — platform components, observability, apps — is deployed *through* GitOps rather than by hand. And **observability goes in before the apps** (step 19) so your first app deploy is observed from the very first request. Those two ordering decisions are what make the bootstrap clean.

---
