<overview>
Comprehensive guide to Infrastructure as Code on Azure, covering Bicep, Terraform, and ARM templates with decision guidance, current best practices, and real-world patterns for 2024-2025.
</overview>

<tool_comparison>
## Bicep vs Terraform vs ARM Templates

<tool name="Bicep">
**Current Version:** 0.x (2024-2025)
**Status:** Microsoft's recommended IaC language for Azure

**When to use:**
- Azure-only infrastructure
- Team prefers native Azure tooling
- Want newest Azure features immediately (zero lag)
- Don't want to manage state files
- Simpler syntax preferred over JSON

**Strengths:**
- Native Azure support - new features available on day 1
- Cleaner syntax than ARM JSON
- No state file management (Azure manages state)
- Transpiles to ARM templates
- Excellent VS Code integration
- Type safety and intellisense
- Module registry for reusable components

**Weaknesses:**
- Azure-only (no multi-cloud)
- Smaller community compared to Terraform
- Less mature ecosystem
- No advanced state operations

**Learning curve:** Low (especially for Azure-focused teams)

**Example:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-001'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
  }
}
```
</tool>

<tool name="Terraform">
**Current Version:** 1.7+ (2024-2025)
**Status:** Industry standard for multi-cloud IaC

**When to use:**
- Multi-cloud or hybrid environments
- Team has existing Terraform expertise
- Need advanced state management features
- Want mature ecosystem and community modules
- Require fine control over drift detection

**Strengths:**
- Multi-cloud support (Azure, AWS, GCP, etc.)
- Mature ecosystem with thousands of providers
- Advanced state management
- Strong community and modules
- HCL syntax widely understood
- Drift detection and plan previews
- Terraform Cloud for collaboration

**Weaknesses:**
- Slight lag for newest Azure features (days to weeks)
- State file management complexity
- Requires understanding of state locks
- More complex initial setup

**Learning curve:** Medium (HCL syntax, state management concepts)

**Example:**
```hcl
resource "azurerm_linux_web_app" "main" {
  name                = "myapp-prod-001"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  service_plan_id     = azurerm_service_plan.main.id
  https_only          = true
}
```
</tool>

<tool name="ARM Templates">
**Status:** Legacy (still supported, but Bicep is recommended)

**When to use:**
- Only when you already have existing ARM templates
- Need to generate templates programmatically
- Required by third-party tools expecting ARM

**Recommendation:** **Don't start new projects with ARM JSON.** Use Bicep instead. ARM templates are verbose and hard to maintain. Bicep compiles to ARM, giving you all ARM benefits with better syntax.

**Example (for comparison):**
```json
{
  "type": "Microsoft.Web/sites",
  "apiVersion": "2023-12-01",
  "name": "myapp-prod-001",
  "location": "eastus",
  "properties": {
    "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', 'myplan')]",
    "httpsOnly": true
  }
}
```
</tool>
</tool_comparison>

<decision_tree>
## Choosing the Right Tool

**Azure-only and starting fresh?**
→ Use **Bicep**
- Simpler, native, newest features immediately
- Microsoft's recommended approach

**Multi-cloud infrastructure?**
→ Use **Terraform**
- Manages Azure + AWS + GCP from single codebase
- Consistent tooling across clouds

**Existing Terraform investment?**
→ **Stay with Terraform**
- Team already knows it
- Existing modules and patterns
- Migration cost not worth it

**Existing ARM templates?**
→ **Migrate to Bicep** or keep Terraform
- Bicep decompile tool helps migration
- ARM is verbose and hard to maintain

**Complex state management needs?**
→ Use **Terraform**
- Advanced drift detection
- State manipulation operations
- Terraform Cloud for team collaboration

**Frequent out-of-band changes?**
→ Use **Bicep**
- Azure manages state, more forgiving
- Easier to handle manual portal changes (though avoid them!)
</decision_tree>

<best_practices>
## Infrastructure as Code Best Practices (2024-2025)

### 1. Modular Development

Break complex templates into smaller modules:

**Bicep modules:**
```bicep
// modules/app-service.bicep
param location string
param appName string
param planId string

resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: planId
  }
}

output appId string = app.id
```

**Using the module:**
```bicep
module appService 'modules/app-service.bicep' = {
  name: 'appServiceDeployment'
  params: {
    location: location
    appName: 'myapp-prod-001'
    planId: appServicePlan.id
  }
}
```

**Terraform modules:**
```hcl
# modules/app-service/main.tf
variable "app_name" { type = string }
variable "location" { type = string }
variable "service_plan_id" { type = string }

resource "azurerm_linux_web_app" "main" {
  name                = var.app_name
  location            = var.location
  resource_group_name = var.resource_group_name
  service_plan_id     = var.service_plan_id
}

output "app_id" {
  value = azurerm_linux_web_app.main.id
}
```

### 2. Version Control Everything

```
repository/
├── infrastructure/
│   ├── main.bicep (or main.tf)
│   ├── modules/
│   ├── environments/
│   │   ├── dev.parameters.json
│   │   ├── staging.parameters.json
│   │   └── prod.parameters.json
│   └── README.md
├── .github/workflows/
│   └── deploy-infrastructure.yml
└── .gitignore
```

**.gitignore:**
```
# Terraform
.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl

# Bicep
*.zip

# Secrets
*.env
secrets.json
```

### 3. Azure Verified Modules

Use tested, Microsoft-approved modules:

**For Bicep:**
```bicep
// Reference Azure Verified Module
module network 'br/public:avm/res/network/virtual-network:0.1.0' = {
  name: 'vnetDeployment'
  params: {
    name: 'myVnet'
    addressPrefixes: ['10.0.0.0/16']
  }
}
```

**For Terraform:**
```hcl
# Use Azure Verified Module from registry
module "network" {
  source  = "Azure/network/azurerm"
  version = "5.0.0"

  resource_group_name = azurerm_resource_group.main.name
  location            = "eastus"
  vnet_name           = "myVnet"
  address_space       = ["10.0.0.0/16"]
}
```

### 4. Configuration Drift Management

**The Golden Rule:** All changes through IaC pipeline. Zero manual portal changes.

**Bicep drift handling:**
```bash
# Azure automatically detects drift
# Redeploy to fix drift
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --mode Complete  # Removes resources not in template
```

**Terraform drift detection:**
```bash
# Detect drift
terraform plan -detailed-exitcode

# If drift detected (exit code 2)
# Review and import manual changes
terraform import azurerm_app_service.main /subscriptions/.../myapp

# Or reapply to fix
terraform apply
```

### 5. Testing & Validation

**Pre-deployment validation:**

Bicep:
```bash
# Build and validate
az bicep build --file main.bicep

# What-if analysis (see changes before deploying)
az deployment group what-if \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --parameters prod.parameters.json
```

Terraform:
```bash
# Validate syntax
terraform validate

# Format code
terraform fmt -recursive

# Security scan
tfsec .

# See plan before applying
terraform plan -out=tfplan
terraform show -json tfplan | jq
```

### 6. State Management (Terraform)

Store state remotely in Azure Storage:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstatestorage001"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

Enable state locking:
```bash
az storage account update \
  --name tfstatestorage001 \
  --resource-group terraform-state-rg \
  --enable-blob-versioning true
```

### 7. Parameterization

**Bicep parameters:**
```bicep
@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Location for resources')
param location string = resourceGroup().location

@minLength(3)
@maxLength(24)
param storageAccountName string

var skuName = environment == 'prod' ? 'P1v3' : 'B1'
```

**Terraform variables:**
```hcl
variable "environment" {
  type        = string
  description = "Environment name"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "location" {
  type    = string
  default = "eastus"
}

locals {
  sku_name = var.environment == "prod" ? "P1v3" : "B1"
}
```

### 8. Secrets Management

**Never hardcode secrets in IaC.**

Use Azure Key Vault references:

Bicep:
```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'shared-prod-kv'
  scope: resourceGroup('shared-prod-rg')
}

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-001'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'DATABASE_URL'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/db-connection/)'
        }
      ]
    }
  }
}
```

Terraform:
```hcl
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = data.azurerm_key_vault.shared.id
}

resource "azurerm_linux_web_app" "main" {
  app_settings = {
    DATABASE_PASSWORD = data.azurerm_key_vault_secret.db_password.value
  }
}
```

### 9. Documentation

Every IaC repository needs `README.md`:

```markdown
# MyApp Infrastructure

## Prerequisites
- Azure CLI 2.55+
- Bicep CLI 0.24+ (or Terraform 1.7+)
- Access to subscription <subscription-id>

## Deployment

### Development
\`\`\`bash
az deployment group create \
  --resource-group myapp-dev-rg \
  --template-file main.bicep \
  --parameters environments/dev.parameters.json
\`\`\`

### Production
Requires approval. Deploy via GitHub Actions workflow.

## Resources Created
- App Service Plan (P1v3)
- App Service
- Azure SQL Database
- Application Insights
- Key Vault

## Outputs
- App URL: https://myapp-prod-001.azurewebsites.net
```

### 10. CI/CD Integration

Infrastructure changes must flow through pipeline:

```yaml
# .github/workflows/infrastructure.yml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Validate Bicep
        run: az bicep build --file infrastructure/main.bicep

      - name: What-If Analysis
        run: |
          az deployment group what-if \
            --resource-group myapp-prod-rg \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/prod.parameters.json

      - name: Deploy Infrastructure
        run: |
          az deployment group create \
            --name "deploy-$(date +%Y%m%d-%H%M%S)" \
            --resource-group myapp-prod-rg \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/prod.parameters.json
```
</best_practices>

<anti_patterns>
**What NOT to do:**

<anti_pattern name="Manual Portal Changes">
**Problem:** Creating or modifying resources manually in Azure Portal

**Why it's bad:**
- Creates configuration drift
- Can't reproduce in other environments
- No audit trail
- Lost when resources recreated

**Instead:** Always update IaC templates, deploy through pipeline
</anti_pattern>

<anti_pattern name="Hardcoded Values">
**Problem:** Hardcoding resource names, locations, SKUs in templates

**Why it's bad:**
- Can't reuse for multiple environments
- Difficult to change configuration
- No flexibility

**Instead:** Use parameters and variables for all configurable values
</anti_pattern>

<anti_pattern name="No State Management (Terraform)">
**Problem:** Using local state file for team projects

**Why it's bad:**
- Conflicts when multiple people deploy
- Lost if machine fails
- No locking mechanism

**Instead:** Use remote backend (Azure Storage) with state locking
</anti_pattern>

<anti_pattern name="God Templates">
**Problem:** Single massive template defining entire infrastructure

**Why it's bad:**
- Hard to understand and maintain
- Difficult to reuse components
- Long deployment times
- Blast radius too large

**Instead:** Break into logical modules (networking, compute, data, etc.)
</anti_pattern>

<anti_pattern name="No Validation">
**Problem:** Deploying without testing templates first

**Why it's bad:**
- Syntax errors caught only at deployment
- Unknown changes applied to production
- No preview of what will change

**Instead:** Always run `bicep build` or `terraform plan` before deploying
</anti_pattern>

<anti_pattern name="Secrets in Code">
**Problem:** Storing passwords, connection strings in templates or parameters

**Why it's bad:**
- Secrets exposed in Git history
- Security vulnerability
- Compliance violations

**Instead:** Use Key Vault references or data sources
</anti_pattern>
</anti_patterns>

<performance_considerations>
## Deployment Performance

**Bicep parallel deployment:**
Bicep automatically deploys independent resources in parallel. Dependencies managed via resource references.

**Terraform parallelism:**
```bash
terraform apply -parallelism=10  # Deploy 10 resources concurrently (default)
```

**Large deployments:**
- Split into multiple resource groups to avoid ARM limits
- Use modules to organize and parallelize
- Deploy networking infrastructure first, then dependent resources

**ARM template limits:**
- Max 800 resources per deployment
- Max 256 parameters
- Max 64 outputs
- 4MB template size limit
</performance_considerations>

<current_trends_2024_2025>
## What's New and Recommended

1. **Azure Deployment Stacks** (Preview) - Managed groups of resources with lifecycle management
2. **Bicep Registry** - Share and reuse modules across organization
3. **Terraform Cloud Integration** - Better team collaboration for Terraform
4. **Policy as Code** - Azure Policy defined in Bicep/Terraform
5. **Testing Frameworks** - Bicep test framework, Terraform test command (1.6+)
6. **AI-Assisted IaC** - GitHub Copilot for generating templates
7. **FinOps Integration** - Cost estimation in CI/CD before deployment
</current_trends_2024_2025>
