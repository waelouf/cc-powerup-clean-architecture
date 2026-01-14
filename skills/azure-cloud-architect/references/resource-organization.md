<overview>
Comprehensive guide to Azure resource organization, naming conventions, tagging strategies, resource groups, subscriptions, and management groups for 2024-2025.
</overview>

<naming_conventions>
## Azure Naming Conventions

**Microsoft recommended pattern:**
```
<resource-type>-<product/app>-<environment>-<region>-<instance>

Examples:
- rg-myapp-prod-eastus
- app-myapp-prod-eastus-001
- sql-shared-prod-eastus-001
- kv-shared-prod-eastus-001
```

### Components

<component name="Resource Type Abbreviation">
**Purpose:** Identify resource at a glance

**Common abbreviations:**
- `rg` - Resource Group
- `app` - App Service
- `asp` - App Service Plan
- `func` - Function App
- `sql` - SQL Server
- `sqldb` - SQL Database
- `kv` - Key Vault
- `st` - Storage Account (no hyphens, lowercase only)
- `aks` - Azure Kubernetes Service
- `vm` - Virtual Machine
- `vnet` - Virtual Network
- `nsg` - Network Security Group
- `pip` - Public IP
- `la` - Log Analytics Workspace
- `ai` - Application Insights

**Full list:** https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations
</component>

<component name="Product/Application Name">
**Purpose:** Identify which product owns the resource

**Guidelines:**
- Lowercase
- No spaces or special characters
- 3-15 characters
- Examples: `myapp`, `orderapi`, `webapp`
</component>

<component name="Environment">
**Purpose:** Separate dev/staging/prod resources

**Standard values:**
- `dev` - Development
- `test` - Testing
- `staging` - Staging/Pre-production
- `prod` - Production

**Benefits:**
- Easy cost allocation
- Clear environment identification
- Prevent accidental production changes
</component>

<component name="Region">
**Purpose:** Identify geographic location

**Use Azure region short names:**
- `eastus` - East US
- `westus2` - West US 2
- `northeurope` - North Europe
- `westeurope` - West Europe

**Why include region:**
- Multi-region deployments
- Disaster recovery planning
- Cost comparison (regions have different pricing)
</component>

<component name="Instance Number">
**Purpose:** Differentiate multiple instances of same resource

**Format:** 3-digit padded number
- `001`, `002`, `003`

**When to use:**
- Multiple App Services
- Multiple VMs
- Multiple storage accounts
</component>

### Resource-Specific Rules

**Storage Accounts (special case):**
- Must be globally unique
- Lowercase letters and numbers only (no hyphens!)
- 3-24 characters
- Pattern: `st<product><env><region><random>`
- Example: `stmyappprodeastus001`

**Key Vault (special case):**
- Must be globally unique
- 3-24 characters
- Hyphens allowed
- Example: `kv-shared-prod-eastus-001`

**DNS names (globally unique):**
- App Service: `<name>.azurewebsites.net`
- Must be globally unique
- Keep product name short

### Complete Examples

**Simple web app (single product):**
```
rg-myapp-prod-eastus              (Resource Group)
├── asp-myapp-prod-eastus-001     (App Service Plan)
├── app-myapp-prod-eastus-001     (App Service - Production)
├── app-myapp-prod-eastus-staging (App Service - Staging Slot)
├── sqldb-myapp-prod-001          (SQL Database)
├── stmyappprodeastus001          (Storage Account)
├── ai-myapp-prod-eastus          (Application Insights)
└── kv-myapp-prod-eastus-001      (Key Vault)
```

**Shared infrastructure:**
```
rg-shared-prod-eastus             (Shared Resource Group)
├── sql-shared-prod-eastus-001    (Shared SQL Server)
│   ├── sqldb-product1-prod       (Product 1 Database)
│   ├── sqldb-product2-prod       (Product 2 Database)
│   └── sqldb-product3-prod       (Product 3 Database)
├── kv-shared-prod-eastus-001     (Shared Key Vault)
├── la-shared-prod-eastus         (Shared Log Analytics)
└── vnet-shared-prod-eastus-001   (Shared Virtual Network)
```

**Multi-region setup:**
```
# East US
rg-myapp-prod-eastus
├── app-myapp-prod-eastus-001
└── sqldb-myapp-prod-eastus

# West US 2
rg-myapp-prod-westus2
├── app-myapp-prod-westus2-001
└── sqldb-myapp-prod-westus2

# Traffic Manager (global)
rg-myapp-prod-global
└── tm-myapp-prod (Traffic Manager)
```
</naming_conventions>

<tagging_strategy>
## Azure Resource Tagging

**Purpose:** Metadata for organization, cost allocation, automation, and compliance.

### Required Tags (Minimum)

```bicep
tags: {
  Environment: 'prod'                    // dev/staging/prod
  Product: 'myapp'                       // Product/application name
  CostCenter: 'Engineering-MyApp'        // For finance chargeback
  Owner: 'team-myapp@company.com'        // Who to contact
}
```

### Recommended Additional Tags

```bicep
tags: {
  Environment: 'prod'
  Product: 'myapp'
  CostCenter: 'Engineering-MyApp'
  Owner: 'team-myapp@company.com'

  // Optional but useful
  Project: 'ProjectAlpha'              // Project code
  Department: 'Engineering'             // Organizational unit
  ManagedBy: 'Terraform'               // How resource is managed
  CreatedBy: 'john.doe@company.com'    // Who created it
  CreatedDate: '2025-01-11'            // When created
  DataClassification: 'Confidential'    // PII, Public, Internal
  DR: 'Required'                        // Disaster recovery needed?
  Compliance: 'HIPAA,SOC2'             // Compliance requirements
  MaintenanceWindow: 'Sun-02:00-04:00' // When to patch
}
```

### Apply Tags

**Bicep:**
```bicep
resource resourceGroup 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: 'myapp-prod-eastus-rg'
  location: 'eastus'
  tags: {
    Environment: 'prod'
    Product: 'myapp'
    CostCenter: 'Engineering-MyApp'
    Owner: 'team@company.com'
  }
}

// Tags inherit to resources within resource group
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app'
  location: 'eastus'
  tags: resourceGroup.tags  // Inherit from resource group
}
```

**Azure CLI:**
```bash
# Tag resource group
az group update \
  --name myapp-prod-rg \
  --tags Environment=prod Product=myapp CostCenter=Engineering Owner=team@company.com

# Tag individual resource
az resource tag \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --resource-type Microsoft.Web/sites \
  --tags Environment=prod Product=myapp
```

### Enforce Tags with Azure Policy

```bicep
// Require CostCenter tag on all resource groups
resource requireCostCenterTag 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-cost-center-tag'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025'
    displayName: 'Require CostCenter tag on resource groups'
    parameters: {
      tagName: {
        value: 'CostCenter'
      }
    }
  }
}

// Inherit tags from resource group
resource inheritTags 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'inherit-tags-from-rg'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/cd3aa116-8754-49c9-a813-ad46512ece54'
    displayName: 'Inherit tags from resource group'
    parameters: {
      tagName: {
        value: 'Environment'
      }
    }
  }
}
```

### Query by Tags

**Cost by CostCenter:**
```bash
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="TagKey" type="Dimension" \
  --query "properties.rows[?[2]=='CostCenter']"
```

**Find all prod resources:**
```bash
az resource list \
  --tag Environment=prod \
  --query "[].{Name:name, Type:type, RG:resourceGroup}" \
  --output table
```
</tagging_strategy>

<resource_groups>
## Resource Group Organization

**Principle:** Group resources by lifecycle and ownership.

### Organization Patterns

<pattern name="By Environment">
```
myapp-dev-rg       (All dev resources)
myapp-staging-rg   (All staging resources)
myapp-prod-rg      (All prod resources)
```

**Pros:**
- Simple
- Clear cost breakdown by environment
- Easy to delete entire environment

**Cons:**
- Mixing resources with different lifecycles
- Hard to have shared resources across environments

**Use when:** Small apps, single team
</pattern>

<pattern name="By Product + Environment">
```
product1-prod-eastus-rg
product2-prod-eastus-rg
shared-prod-eastus-rg
```

**Pros:**
- Clear ownership
- Shared resources isolated
- Cost allocation per product

**Cons:**
- More resource groups to manage

**Use when:** Multiple products, multiple teams
</pattern>

<pattern name="By Function + Environment">
```
compute-prod-eastus-rg    (App Services, VMs)
data-prod-eastus-rg       (SQL, Storage)
network-prod-eastus-rg    (VNets, NSGs, Gateways)
shared-prod-eastus-rg     (Key Vault, Log Analytics)
```

**Pros:**
- Logical separation
- Different teams can manage (network team, data team)

**Cons:**
- Complex
- Hard to see full app stack

**Use when:** Large enterprises, specialized teams
</pattern>

### Resource Group Best Practices

1. **Same region:** All resources in RG should be in same region (metadata stored in RG region)
2. **Same lifecycle:** Resources deleted together belong together
3. **Same ownership:** Resources managed by same team
4. **RBAC boundary:** Apply permissions at RG level
5. **Don't** **span apps:** Each app gets its own RG (or set of RGs)
</resource_groups>

<subscriptions>
## Subscription Organization

**When to use multiple subscriptions:**
- Billing separation (departments, products)
- Subscription limits (e.g., 980 resource groups per subscription)
- Security/compliance boundaries
- Different Azure AD tenants

### Subscription Patterns

<pattern name="By Environment">
```
MyCompany-Dev subscription
MyCompany-Staging subscription
MyCompany-Prod subscription
```

**Pros:**
- Clear separation
- Different permissions (developers have contributor in dev, reader in prod)
- Prevent accidental production changes

**Use when:** Medium-sized organizations
</pattern>

<pattern name="By Product">
```
Product1-Prod subscription
Product2-Prod subscription
Platform-Services subscription
```

**Pros:**
- Clear cost attribution
- Product teams have full ownership
- Blast radius limited to product

**Use when:** Large organizations, multiple independent products
</pattern>

<pattern name="Hub-Spoke">
```
Hub subscription (shared networking, security)
Spoke subscription (Product 1)
Spoke subscription (Product 2)
```

**Pros:**
- Centralized networking
- Shared services (firewall, VPN)
- Individual product subscriptions

**Use when:** Enterprises with centralized IT
</pattern>

### Subscription Limits

Be aware of limits:
- 980 resource groups per subscription
- 800 resources per resource group per deployment
- 50 Azure Policy assignments per subscription
- Regional quotas (vCPUs, IPs, etc.)

**Solution:** Use multiple subscriptions when approaching limits.
</subscriptions>

<management_groups>
## Management Groups

**What:** Hierarchy above subscriptions for applying policies and RBAC at scale.

**Hierarchy:**
```
Root Management Group
├── Platform Services
│   └── Platform-Prod subscription
│       ├── Shared networking
│       └── Shared security
├── Product A
│   ├── ProductA-Dev subscription
│   ├── ProductA-Staging subscription
│   └── ProductA-Prod subscription
└── Product B
    ├── ProductB-Dev subscription
    └── ProductB-Prod subscription
```

### Benefits

**Governance at scale:**
```bicep
// Apply policy to management group (affects all child subscriptions)
resource denyPublicIPs 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'deny-public-ips'
  scope: managementGroup('Production')
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/...'
    displayName: 'Deny creation of public IPs in production'
  }
}
```

**RBAC inheritance:**
- Assign Reader role at root → everyone can view all subscriptions
- Assign Contributor at Product A MG → team has access to all Product A subscriptions

### Best Practices

1. **Don't** overuse: 2-4 levels max
2. **Align** with organization structure (departments, products)
3. **Apply** policies at appropriate level (security policies at root, product policies at product MG)
4. **Test** policies in dev before applying to prod
</management_groups>

<complete_organization_example>
## Complete Organization Example

**Company:** Medium-sized SaaS company with 3 products

### Management Group Structure

```
Contoso (Root)
├── Platform (shared services)
└── Products
    ├── ProductA
    ├── ProductB
    └── ProductC
```

### Subscription Structure

```
Platform:
- Platform-Prod subscription
  - rg-shared-prod-eastus (SQL Servers, Key Vaults)
  - rg-network-prod-eastus (VNets, Firewalls)

ProductA:
- ProductA-Dev subscription
  - rg-producta-dev-eastus
- ProductA-Prod subscription
  - rg-producta-prod-eastus
  - rg-producta-prod-westus2 (DR)

ProductB:
- ProductB-Prod subscription
  - rg-productb-prod-eastus

ProductC:
- ProductC-Dev subscription
- ProductC-Prod subscription
```

### Naming Convention

```
Subscription: <Product>-<Environment>
Resource Group: rg-<product>-<env>-<region>
Resources: <type>-<product>-<env>-<region>-<instance>
```

### Tagging Strategy

```bicep
// Applied to all resources
tags: {
  Environment: 'prod'              // Required
  Product: 'producta'              // Required
  CostCenter: 'Engineering-PA'     // Required
  Owner: 'team-a@contoso.com'      // Required
  Department: 'Engineering'        // Optional
  DataClass: 'Confidential'        // Optional
}
```

### RBAC

```
Root MG:
- Finance team: Reader (view costs)
- Security team: Security Reader

Platform MG:
- Platform team: Contributor

ProductA MG:
- ProductA team: Contributor (dev + prod)
- ProductA oncall: Reader (prod only, via PIM)
```

### Policies

```
Root MG:
- Require tags (Environment, Product, CostCenter, Owner)
- Enable Azure Defender
- Deny deployment to non-approved regions

Platform MG:
- Allow only specific resource types

ProductA Prod subscription:
- Deny public IPs
- Require private endpoints for storage/SQL
- Enable diagnostic logs
```
</complete_organization_example>

<anti_patterns>
**Avoid:**

<anti_pattern name="Inconsistent Naming">
**Problem:** Some resources use `prod`, others `production`, `prd`, `p`

**Why bad:** Hard to query, filter, automate

**Instead:** Agree on standard, document it, enforce with policy
</anti_pattern>

<anti_pattern name="No Tagging">
**Problem:** Resources created without tags

**Why bad:** Can't allocate costs, identify owners, apply automation

**Instead:** Require tags via Azure Policy, inherit from resource group
</anti_pattern>

<anti_pattern name="Everything in One Resource Group">
**Problem:** All resources for all apps in `default-rg`

**Why bad:** Can't manage permissions, costs, or lifecycles independently

**Instead:** One resource group per app per environment at minimum
</anti_pattern>

<anti_pattern name="Generic Names">
**Problem:** Resources named `app1`, `db1`, `kv1`

**Why bad:** Can't tell what it does or who owns it

**Instead:** Use descriptive, pattern-based names
</anti_pattern>

<anti_pattern name="No Documentation">
**Problem:** Naming convention exists only in one person's head

**Why bad:** Inconsistency, onboarding difficulty

**Instead:** Document in README.md, enforce with policy
</anti_pattern>
</anti_patterns>
