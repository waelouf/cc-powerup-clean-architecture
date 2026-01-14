# Workflow: Provision Infrastructure

<required_reading>
**Read these reference files NOW before provisioning infrastructure:**
1. references/infrastructure-as-code.md
2. references/compute-services.md
3. references/resource-organization.md
4. references/security-identity.md
</required_reading>

<process>
## Step 1: Gather Requirements

Ask the user:
- What type of application? (Web app, API, microservices, batch processing, etc.)
- Expected traffic/load? (Helps determine sizing)
- Any specific Azure services required? (SQL Database, Redis, Storage, etc.)
- Budget constraints?
- Environment? (dev/staging/prod)

## Step 2: Choose IaC Tool

Based on requirements:
- **Bicep**: Azure-only, newest features, simpler for Azure-native teams
- **Terraform**: Multi-cloud, mature ecosystem, advanced state management

Ask user preference if not specified.

## Step 3: Design Architecture

Based on application type, recommend compute service:

**Web applications / APIs:**
- Small-medium: Azure App Service (PaaS, easiest)
- Large/microservices: Azure Kubernetes Service (AKS)
- Serverless: Azure Functions

**Databases:**
- Relational: Azure SQL Database (DTU or vCore model)
- NoSQL: Cosmos DB
- Cache: Azure Cache for Redis

**Storage:**
- Blobs/files: Azure Storage Account
- Queues: Azure Storage Queue or Service Bus

Create architecture diagram or list of resources.

## Step 4: Setup Resource Organization

Define naming convention based on references/resource-organization.md:

```
<product>-<env>-<region>-<resource-type>-<instance>

Examples:
- myapp-prod-eastus-app-001
- myapp-prod-eastus-sql-001
- shared-prod-eastus-kv-001 (for shared resources)
```

Create resource group:
```bash
az group create \
  --name <product>-<env>-<region>-rg \
  --location <region> \
  --tags Environment=<env> Product=<product> CostCenter=<cc> Owner=<owner>
```

## Step 5: Create IaC Templates

### For Bicep:

Create `main.bicep`:
```bicep
@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Location for all resources')
param location string = resourceGroup().location

@description('Product name for resource naming')
param productName string

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: '${productName}-${environment}-${location}-asp-001'
  location: location
  sku: {
    name: environment == 'prod' ? 'P1v3' : 'B1'
    tier: environment == 'prod' ? 'PremiumV3' : 'Basic'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// App Service
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: '${productName}-${environment}-${location}-app-001'
  location: location
  identity: {
    type: 'SystemAssigned'  // Managed Identity for secure access
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true  // Security best practice
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      alwaysOn: environment == 'prod' ? true : false
      ftpsState: 'Disabled'  // Security best practice
      minTlsVersion: '1.2'  // Security best practice
    }
  }
  tags: {
    Environment: environment
    Product: productName
  }
}

// Output the app URL
output appUrl string = 'https://${appService.properties.defaultHostName}'
output appIdentityPrincipalId string = appService.identity.principalId
```

Create `bicep.parameters.json`:
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "dev"
    },
    "productName": {
      "value": "myapp"
    }
  }
}
```

### For Terraform:

Create `main.tf`:
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {}  # Configure remote state
}

provider "azurerm" {
  features {}
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "product_name" {
  type = string
}

variable "location" {
  type    = string
  default = "eastus"
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "${var.product_name}-${var.environment}-${var.location}-rg"
  location = var.location

  tags = {
    Environment = var.environment
    Product     = var.product_name
  }
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "${var.product_name}-${var.environment}-${var.location}-asp-001"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v3" : "B1"

  tags = azurerm_resource_group.main.tags
}

# App Service
resource "azurerm_linux_web_app" "main" {
  name                = "${var.product_name}-${var.environment}-${var.location}-app-001"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  service_plan_id     = azurerm_service_plan.main.id
  https_only          = true

  identity {
    type = "SystemAssigned"
  }

  site_config {
    always_on        = var.environment == "prod" ? true : false
    ftps_state       = "Disabled"
    minimum_tls_version = "1.2"

    application_stack {
      node_version = "20-lts"
    }
  }

  tags = azurerm_resource_group.main.tags
}

output "app_url" {
  value = "https://${azurerm_linux_web_app.main.default_hostname}"
}

output "app_identity_principal_id" {
  value = azurerm_linux_web_app.main.identity[0].principal_id
}
```

Create `terraform.tfvars`:
```hcl
environment  = "dev"
product_name = "myapp"
location     = "eastus"
```

## Step 6: Validate Templates

### Bicep:
```bash
az bicep build --file main.bicep
```

### Terraform:
```bash
terraform init
terraform validate
terraform plan
```

Review the plan output with user before deploying.

## Step 7: Deploy Infrastructure

### Bicep:
```bash
az deployment group create \
  --name "deployment-$(date +%Y%m%d-%H%M%S)" \
  --resource-group <rg-name> \
  --template-file main.bicep \
  --parameters bicep.parameters.json
```

### Terraform:
```bash
terraform apply
```

## Step 8: Verify Deployment

```bash
# Check deployment status
az deployment group list --resource-group <rg-name> --output table

# List created resources
az resource list --resource-group <rg-name> --output table

# Test web app (if applicable)
curl https://<app-name>.azurewebsites.net
```

## Step 9: Configure Post-Deployment

Set up essential configurations:

```bash
# Enable Application Insights (if not in template)
az monitor app-insights component create \
  --app <app-name>-insights \
  --location <region> \
  --resource-group <rg-name> \
  --application-type web

# Get instrumentation key and configure app
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app <app-name>-insights \
  --resource-group <rg-name> \
  --query instrumentationKey -o tsv)

az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY=$INSTRUMENTATION_KEY
```

## Step 10: Document and Save

Create README.md in the IaC repository:
```markdown
# <Product Name> Infrastructure

## Resources
- Resource Group: <rg-name>
- App Service: <app-name>
- App Service Plan: <plan-name>

## Deployment
\`\`\`bash
az deployment group create \
  --resource-group <rg-name> \
  --template-file main.bicep \
  --parameters bicep.parameters.json
\`\`\`

## Outputs
- App URL: https://<app-name>.azurewebsites.net
- Managed Identity Principal ID: <principal-id>
```

Commit all IaC files to version control (Git).
</process>

<anti_patterns>
**Avoid:**

- **Creating resources manually in portal**: Always use IaC. Manual resources can't be replicated or version controlled.

- **Hardcoding values**: Use parameters/variables for environment-specific values (names, SKUs, locations).

- **Skipping resource tagging**: Tags are critical for cost allocation and organization.

- **Using weak SKUs in production**: Don't use Free/Basic tiers for production workloads.

- **Ignoring security**: Always enable HTTPS only, disable FTP, use Managed Identity, set minimum TLS version.

- **Not using Managed Identity**: Never store credentials in app settings. Use Managed Identity for Azure service access.

- **Deploying without validation**: Always run `bicep build` or `terraform plan` before deploying.

- **No naming convention**: Inconsistent names make resources hard to manage and organize.

- **Forgetting outputs**: Output important values (URLs, principal IDs) for downstream use.

- **Not version controlling IaC**: All templates must be in Git.
</anti_patterns>

<success_criteria>
A well-provisioned infrastructure has:

- All resources defined in IaC (Bicep or Terraform)
- Consistent naming convention following pattern
- Proper resource tagging (Environment, Product, CostCenter, Owner)
- Security enabled (HTTPS only, TLS 1.2+, Managed Identity)
- Appropriate SKUs for environment (cheaper for dev, production-grade for prod)
- Deployment validated before executing
- All resources deployed successfully
- Application accessible at expected URL
- Monitoring configured (Application Insights)
- IaC files committed to version control
- Documentation created (README.md)
- No manual resources created in portal
</success_criteria>
