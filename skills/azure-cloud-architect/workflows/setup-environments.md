<required_reading>
Before setting up environments:
- `references/resource-organization.md` - Naming conventions, tagging, subscriptions
- `references/infrastructure-as-code.md` - Parameterizing IaC for multiple environments
- `references/cost-optimization.md` - Environment-specific cost strategies
</required_reading>

<objective>
Create and configure multiple Azure environments (dev, staging, production) with proper isolation, naming conventions, and environment-specific configurations using Bicep/Terraform parameters.
</objective>

<quick_start>
**5-minute multi-environment setup:**
```bash
# Create resource groups for each environment
az group create --name myapp-dev-eastus-rg --location eastus --tags Environment=dev
az group create --name myapp-staging-eastus-rg --location eastus --tags Environment=staging
az group create --name myapp-prod-eastus-rg --location eastus --tags Environment=prod

# Deploy to dev
az deployment group create \
  --resource-group myapp-dev-eastus-rg \
  --template-file main.bicep \
  --parameters main.parameters.dev.json

# Deploy to staging
az deployment group create \
  --resource-group myapp-staging-eastus-rg \
  --template-file main.bicep \
  --parameters main.parameters.staging.json

# Deploy to production
az deployment group create \
  --resource-group myapp-prod-eastus-rg \
  --template-file main.bicep \
  --parameters main.parameters.prod.json
```
</quick_start>

<process>
## Step 1: Environment Strategy

**1.1 Environment purposes:**

```
Development (dev):
- Rapid iteration
- Developer access (Contributor)
- Minimal cost (auto-shutdown, smaller SKUs)
- Shorter log retention (7 days)
- Relaxed security (for productivity)

Staging (staging/uat):
- Production-like configuration
- Integration testing
- Performance testing
- UAT (User Acceptance Testing)
- Pre-production validation

Production (prod):
- Live traffic
- High availability (multi-instance, multi-region)
- Enhanced security
- Monitoring and alerts
- Longer log retention (90 days)
- Restricted access (Reader for most, Contributor for CI/CD)
```

**1.2 Isolation strategies:**

```
Option 1: Same subscription, different resource groups
✓ Simple
✓ Shared quotas
✗ Less isolation
Use for: Small teams, single product

Option 2: Different subscriptions
✓ Complete isolation
✓ Separate billing
✓ Different RBAC
✗ More complex management
Use for: Enterprises, multiple products

Option 3: Different regions
✓ Geographic isolation
✓ Disaster recovery
✗ Higher cost
Use for: Global products, DR requirements
```

## Step 2: Parameterized Infrastructure

**2.1 Bicep with parameters:**

```bicep
// main.bicep
param environment string  // 'dev', 'staging', 'prod'
param location string = resourceGroup().location

// Environment-specific configurations
var environmentConfig = {
  dev: {
    skuName: 'B1'
    skuTier: 'Basic'
    instanceCount: 1
    autoShutdown: true
    logRetentionDays: 7
  }
  staging: {
    skuName: 'P1v3'
    skuTier: 'PremiumV3'
    instanceCount: 2
    autoShutdown: false
    logRetentionDays: 30
  }
  prod: {
    skuName: 'P2v3'
    skuTier: 'PremiumV3'
    instanceCount: 3
    autoShutdown: false
    logRetentionDays: 90
  }
}

var config = environmentConfig[environment]

// Resource names with environment
var appServicePlanName = 'asp-myapp-${environment}-${location}'
var appServiceName = 'app-myapp-${environment}-${location}'
var sqlServerName = 'sql-myapp-${environment}-${location}'

// Tags
var tags = {
  Environment: environment
  Product: 'myapp'
  CostCenter: 'Engineering'
  Owner: 'team@company.com'
}

resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: appServicePlanName
  location: location
  tags: tags
  sku: {
    name: config.skuName
    tier: config.skuTier
    capacity: config.instanceCount
  }
}

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  tags: tags
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      appSettings: [
        {
          name: 'ENVIRONMENT'
          value: environment
        }
      ]
    }
  }
}

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'la-myapp-${environment}-${location}'
  location: location
  tags: tags
  properties: {
    retentionInDays: config.logRetentionDays
  }
}
```

**2.2 Parameter files:**

```json
// main.parameters.dev.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "dev"
    },
    "location": {
      "value": "eastus"
    }
  }
}

// main.parameters.staging.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "staging"
    },
    "location": {
      "value": "eastus"
    }
  }
}

// main.parameters.prod.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "prod"
    },
    "location": {
      "value": "eastus"
    }
  }
}
```

## Step 3: Environment-Specific Configurations

**3.1 Auto-shutdown for dev:**

```bicep
// Only create auto-shutdown for dev environment
resource autoShutdown 'Microsoft.DevTestLab/schedules@2018-09-15' = if (environment == 'dev') {
  name: 'shutdown-computevm-${vmName}'
  location: location
  properties: {
    status: 'Enabled'
    taskType: 'ComputeVmShutdownTask'
    dailyRecurrence: {
      time: '1900'  // 7 PM
    }
    timeZoneId: 'Eastern Standard Time'
    targetResourceId: vm.id
  }
}
```

**3.2 Deployment slots for staging/prod:**

```bicep
// Staging slot only in production
resource stagingSlot 'Microsoft.Web/sites/slots@2023-12-01' = if (environment == 'prod') {
  parent: appService
  name: 'staging'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```

**3.3 High availability for prod:**

```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: appServicePlanName
  sku: {
    name: environment == 'prod' ? 'P2v3' : (environment == 'staging' ? 'P1v3' : 'B1')
    capacity: environment == 'prod' ? 3 : (environment == 'staging' ? 2 : 1)
  }
  properties: {
    zoneRedundant: environment == 'prod'  // Zone redundancy only in prod
  }
}
```

## Step 4: RBAC Per Environment

```bash
# Development - developers have Contributor
az role assignment create \
  --assignee dev-team@company.com \
  --role "Contributor" \
  --resource-group myapp-dev-eastus-rg

# Staging - developers have Reader, CI/CD has Contributor
az role assignment create \
  --assignee dev-team@company.com \
  --role "Reader" \
  --resource-group myapp-staging-eastus-rg

az role assignment create \
  --assignee cicd-pipeline-sp \
  --role "Website Contributor" \
  --resource-group myapp-staging-eastus-rg

# Production - everyone Reader, only CI/CD has deploy access
az role assignment create \
  --assignee dev-team@company.com \
  --role "Reader" \
  --resource-group myapp-prod-eastus-rg

az role assignment create \
  --assignee cicd-pipeline-sp \
  --role "Website Contributor" \
  --resource-group myapp-prod-eastus-rg

# Oncall engineers get JIT access via PIM (Privileged Identity Management)
```

## Step 5: CI/CD for Multi-Environment

**5.1 GitHub Actions:**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main, develop]

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS_DEV }}

      - name: Deploy infrastructure
        run: |
          az deployment group create \
            --resource-group myapp-dev-eastus-rg \
            --template-file main.bicep \
            --parameters main.parameters.dev.json

      - name: Deploy application
        run: |
          az webapp deployment source config-zip \
            --resource-group myapp-dev-eastus-rg \
            --name app-myapp-dev-eastus \
            --src app.zip

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS_STAGING }}

      - name: Deploy to staging
        run: |
          az deployment group create \
            --resource-group myapp-staging-eastus-rg \
            --template-file main.bicep \
            --parameters main.parameters.staging.json

          az webapp deployment source config-zip \
            --resource-group myapp-staging-eastus-rg \
            --name app-myapp-staging-eastus \
            --src app.zip

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS_PROD }}

      - name: Deploy to prod staging slot
        run: |
          az webapp deployment source config-zip \
            --resource-group myapp-prod-eastus-rg \
            --name app-myapp-prod-eastus \
            --slot staging \
            --src app.zip

      - name: Swap to production
        run: |
          az webapp deployment slot swap \
            --resource-group myapp-prod-eastus-rg \
            --name app-myapp-prod-eastus \
            --slot staging \
            --target-slot production
```

## Step 6: Environment Variables

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'ENVIRONMENT'
          value: environment
        }
        {
          name: 'LOG_LEVEL'
          value: environment == 'prod' ? 'INFO' : 'DEBUG'
        }
        {
          name: 'DATABASE_URL'
          value: '@Microsoft.KeyVault(VaultName=kv-myapp-${environment};SecretName=DatabaseUrl)'
        }
        {
          name: 'API_BASE_URL'
          value: environment == 'prod' ? 'https://api.myapp.com' : 'https://api-${environment}.myapp.com'
        }
      ]
    }
  }
}
```

## Step 7: Verify Environments

```bash
# List all environments
az group list --tag Environment --query "[].{Name:name, Environment:tags.Environment}" --output table

# Check dev environment
az resource list --resource-group myapp-dev-eastus-rg --output table

# Check prod environment
az resource list --resource-group myapp-prod-eastus-rg --output table

# Compare configurations
az webapp show --name app-myapp-dev-eastus --resource-group myapp-dev-eastus-rg --query "{Name:name, State:state, SKU:appServicePlanId}" -o json
az webapp show --name app-myapp-prod-eastus --resource-group myapp-prod-eastus-rg --query "{Name:name, State:state, SKU:appServicePlanId}" -o json
```
</process>

<environment_checklist>
## Multi-Environment Checklist

### Development
- [ ] Resource group created with Environment=dev tag
- [ ] Smaller SKUs configured (B1/B2 tier)
- [ ] Single instance (no HA needed)
- [ ] Auto-shutdown enabled for VMs
- [ ] 7-day log retention
- [ ] Developers have Contributor access
- [ ] Separate Key Vault for dev secrets
- [ ] Database with lower tier (Basic/S0)

### Staging
- [ ] Resource group created with Environment=staging tag
- [ ] Production-like SKUs (P1v3 tier)
- [ ] 2 instances for availability testing
- [ ] 30-day log retention
- [ ] Developers have Reader access
- [ ] CI/CD has deployment access
- [ ] Separate Key Vault for staging secrets
- [ ] Database tier matching production

### Production
- [ ] Resource group created with Environment=prod tag
- [ ] Production SKUs (P2v3+ tier)
- [ ] 3+ instances with zone redundancy
- [ ] 90-day log retention
- [ ] Deployment slots configured
- [ ] RBAC with least privilege
- [ ] Separate Key Vault with soft delete + purge protection
- [ ] Private Endpoints for all data services
- [ ] Azure Defender enabled
- [ ] Alerts and monitoring configured
</environment_checklist>

<success_criteria>
Multi-environment setup is complete when:
- All three environments deployed with appropriate configurations
- Resource naming clearly identifies environment
- Tags applied consistently across all resources
- RBAC configured appropriately per environment
- CI/CD pipeline deploys to correct environment
- Environment variables/secrets isolated per environment
- Cost optimizations applied to dev (auto-shutdown, smaller SKUs)
- Production has HA, monitoring, and security hardening
</success_criteria>
