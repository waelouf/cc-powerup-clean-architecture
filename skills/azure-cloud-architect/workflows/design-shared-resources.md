# Workflow: Design Shared Resources

<required_reading>
**Read these reference files NOW before designing shared resources:**
1. references/multi-tenancy-patterns.md
2. references/resource-organization.md
3. references/security-identity.md
4. references/cost-optimization.md
5. references/storage-data.md
</required_reading>

<process>
## Step 1: Identify Sharing Scope

Ask the user:
- How many products/applications will share resources?
- What type of resources need to be shared? (SQL Server, Key Vault, App Service Plan, etc.)
- Are the products owned by the same team or different teams?
- What's the isolation requirement? (Complete isolation, logical isolation, or shared with Row-Level Security)
- Is this single-tenant or multi-tenant SaaS?

## Step 2: Choose Sharing Pattern

Based on requirements, recommend pattern:

### Pattern A: Shared Infrastructure, Isolated Data
**Use when:** Multiple products from same organization, cost optimization priority

**Example:**
- **Shared**: SQL Server, App Service Plan, Key Vault, Application Insights
- **Isolated**: Databases (one per product), App Services, Storage Accounts

**Benefits:**
- Lower cost (one SQL Server instead of N servers)
- Centralized management
- Easier to maintain and patch

**Considerations:**
- Resource contention (one product can impact others)
- Need quota/resource limits per product
- Blast radius (server issue affects all products)

### Pattern B: Shared Resource Groups, Isolated Resources
**Use when:** Different teams, cost allocation important, moderate isolation needed

**Example:**
- Shared resource group: `shared-prod-eastus-rg`
- Each product has own: SQL Server, App Service, Key Vault
- Shared: Virtual Network, Application Gateway (routing)

**Benefits:**
- Clear cost allocation per product
- Better isolation
- Less blast radius

**Considerations:**
- Higher cost
- More resources to manage

### Pattern C: Fully Isolated
**Use when:** Strong isolation required, compliance needs, different security boundaries

**Example:**
- Each product: Own resource group, subscription, or management group
- Nothing shared

**Benefits:**
- Maximum isolation
- Clear security boundaries
- Independent scaling and lifecycle

**Considerations:**
- Highest cost
- Most complex to manage
- Duplicated effort

## Step 3: Design Shared SQL Server Architecture

If sharing SQL Servers (most common scenario):

### Recommended Pattern: Shared Server, Isolated Databases

```
SQL Server: shared-prod-eastus-sql-001
├── Database: product1-prod-db
├── Database: product2-prod-db
└── Database: product3-prod-db
```

**Naming Convention:**
- Server: `shared-<env>-<region>-sql-<instance>`
- Database: `<product>-<env>-db`

### Security Architecture:

1. **Azure AD Authentication Only** (no SQL auth)
2. **Managed Identity** for each app to access its database
3. **RBAC on database level**:
   - Product1 app identity → db_datareader + db_datawriter on product1-prod-db only
   - Product2 app identity → db_datareader + db_datawriter on product2-prod-db only

### Implementation:

Create Bicep template for shared SQL Server:

```bicep
@description('Environment name')
param environment string

@description('Location for resources')
param location string = resourceGroup().location

@description('Azure AD admin object ID')
param azureAdAdminObjectId string

@description('Azure AD admin email')
param azureAdAdminEmail string

// Shared SQL Server
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'shared-${environment}-${location}-sql-001'
  location: location
  properties: {
    administratorLogin: null  // Azure AD only
    administratorLoginPassword: null  // Azure AD only
    administrators: {
      administratorType: 'ActiveDirectory'
      login: azureAdAdminEmail
      sid: azureAdAdminObjectId
      tenantId: subscription().tenantId
      azureADOnlyAuthentication: true  // Disable SQL auth
    }
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'  // Use Private Endpoint
  }
  tags: {
    Environment: environment
    Purpose: 'Shared Database Server'
  }
}

// Private Endpoint for SQL Server
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${sqlServer.name}-pe'
  location: location
  properties: {
    subnet: {
      id: <subnet-id>  // Reference to VNet subnet
    }
    privateLinkServiceConnections: [
      {
        name: '${sqlServer.name}-connection'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: ['sqlServer']
        }
      }
    ]
  }
}

// Firewall: Allow Azure Services (for deployment only)
resource firewallRule 'Microsoft.Sql/servers/firewallRules@2023-08-01-preview' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

output sqlServerName string = sqlServer.name
output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
```

Create database template (called per product):

```bicep
@description('SQL Server name')
param sqlServerName string

@description('Database name')
param databaseName string

@description('Environment')
param environment string

@description('App Managed Identity Principal ID')
param appIdentityPrincipalId string

resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' existing = {
  name: sqlServerName
}

// Create database for product
resource database 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: databaseName
  location: sqlServer.location
  sku: {
    name: environment == 'prod' ? 'S3' : 'S0'  // Standard tier
    tier: 'Standard'
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: environment == 'prod' ? 268435456000 : 2147483648  // 250GB prod, 2GB dev
  }
  tags: {
    Environment: environment
    Product: databaseName
  }
}

// Grant Managed Identity access to database
// NOTE: This requires running SQL commands post-deployment
output postDeploymentSql string = '''
-- Connect to database: ${databaseName}
CREATE USER [${appIdentityPrincipalId}] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [${appIdentityPrincipalId}];
ALTER ROLE db_datawriter ADD MEMBER [${appIdentityPrincipalId}];
GO
'''
```

## Step 4: Design Shared Key Vault Architecture

### Pattern: Shared Key Vault with RBAC

```
Key Vault: shared-prod-eastus-kv-001
├── Secret: product1-db-connection
├── Secret: product2-db-connection
├── Secret: product3-api-key
└── Secret: shared-storage-key
```

**Access Control:**
- Use RBAC (not access policies)
- Each app's Managed Identity gets "Key Vault Secrets User" role scoped to specific secrets

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'shared-${environment}-${location}-kv-001'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // Use RBAC instead of access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'  // Private endpoint only
    }
  }
}

// Grant app access to specific secret
resource secretRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, appIdentityPrincipalId, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')  // Key Vault Secrets User
    principalId: appIdentityPrincipalId
    principalType: 'ServicePrincipal'
  }
}
```

## Step 5: Design Shared App Service Plan

### When to Share App Service Plans:

**Share when:**
- Products have similar traffic patterns
- Cost optimization is priority
- Products are owned by same team

**Don't share when:**
- Products have different scaling needs
- One product has unpredictable spikes
- Strong isolation required

```bicep
resource sharedAppServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'shared-${environment}-${location}-asp-001'
  location: location
  sku: {
    name: 'P1v3'
    capacity: 2  // Scale based on total load
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// Each product gets own App Service on shared plan
resource product1App 'Microsoft.Web/sites@2023-12-01' = {
  name: 'product1-${environment}-${location}-app-001'
  location: location
  properties: {
    serverFarmId: sharedAppServicePlan.id
    // Product 1 config...
  }
}

resource product2App 'Microsoft.Web/sites@2023-12-01' = {
  name: 'product2-${environment}-${location}-app-001'
  location: location
  properties: {
    serverFarmId: sharedAppServicePlan.id
    // Product 2 config...
  }
}
```

## Step 6: Implement Resource Organization

### Naming Convention for Shared Resources:

```
shared-<env>-<region>-<resource-type>-<instance>

Examples:
- shared-prod-eastus-sql-001
- shared-prod-eastus-kv-001
- shared-prod-eastus-asp-001
```

### Tagging Strategy:

```bicep
tags: {
  Environment: 'prod'
  Purpose: 'Shared Infrastructure'
  SharedBy: 'product1,product2,product3'
  CostCenter: 'IT-Infrastructure'
  ManagedBy: 'Platform Team'
}
```

### Resource Group Organization:

**Option A: Single shared resource group**
```
shared-prod-eastus-rg
├── SQL Server (shared)
├── Key Vault (shared)
├── App Service Plan (shared)
└── Application Insights (shared)
```

**Option B: Shared resource group + product resource groups**
```
shared-prod-eastus-rg
├── SQL Server (shared)
└── Key Vault (shared)

product1-prod-eastus-rg
├── App Service (uses shared ASP)
└── Database (on shared SQL Server)

product2-prod-eastus-rg
├── App Service (uses shared ASP)
└── Database (on shared SQL Server)
```

Option B is recommended for better cost visibility.

## Step 7: Implement Cost Allocation

### Tagging for Chargeback:

Tag each database and app with product owner:

```bicep
tags: {
  Environment: 'prod'
  Product: 'product1'
  CostCenter: 'Engineering-Product1'
  Owner: 'team-product1@company.com'
}
```

### Use Azure Cost Management:

```bash
# Get cost breakdown by product tag
az costmanagement query \
  --type Usage \
  --dataset-filter "{\"and\":[{\"dimensions\":{\"name\":\"ResourceGroup\",\"operator\":\"In\",\"values\":[\"shared-prod-eastus-rg\"]}},{\"tags\":{\"name\":\"Product\",\"operator\":\"In\",\"values\":[\"product1\"]}}]}" \
  --timeframe MonthToDate
```

Create monthly cost allocation report and distribute to product teams.

## Step 8: Document Shared Resource Governance

Create `SHARED-RESOURCES.md`:

```markdown
# Shared Resources Governance

## Shared SQL Server: shared-prod-eastus-sql-001

### Databases:
- product1-prod-db (Owner: team-product1@company.com)
- product2-prod-db (Owner: team-product2@company.com)

### Access Pattern:
- Azure AD authentication only
- Each app uses Managed Identity
- Database-level RBAC permissions

### Request New Database:
1. Create ticket in <ticketing-system>
2. Provide: Product name, expected size, owner email
3. Platform team provisions within 1 business day

### SLA:
- Uptime: 99.9%
- Backup: Daily, 35-day retention
- Maintenance window: Sunday 2-4 AM EST

## Shared Key Vault: shared-prod-eastus-kv-001

### Access Pattern:
- RBAC-based
- Request access via <ticketing-system>
- Secrets follow naming: <product>-<purpose>

## Shared App Service Plan: shared-prod-eastus-asp-001

### Capacity:
- Current: P1v3 x 2 instances
- Max apps: 10
- Auto-scale: 2-5 instances based on CPU

### Request Hosting:
1. Verify app resource requirements < 500MB RAM
2. Create ticket with deployment details
3. Receives own App Service on shared plan
```
</process>

<anti_patterns>
**Avoid:**

- **Sharing without isolation**: Don't put multiple products in same database. Always use separate databases on shared server.

- **SQL authentication**: Never use SQL username/password. Always use Azure AD + Managed Identity.

- **Access policies in Key Vault**: Use RBAC instead. Access policies are deprecated and less secure.

- **No resource limits**: Set quotas and limits to prevent one product from consuming all resources.

- **Poor naming convention**: Shared resources must be clearly named as "shared" to avoid confusion.

- **No cost allocation**: Tag all resources with product ownership for chargeback and accountability.

- **No governance documentation**: Document who owns what, how to request access, and SLAs.

- **Sharing across security boundaries**: Don't share Key Vault between dev and prod. Keep separate.

- **No blast radius consideration**: Understand that shared resource failure affects multiple products.

- **Forgetting Private Endpoints**: Shared resources like SQL and Key Vault should use Private Endpoints, not public access.

- **Over-sharing**: Don't share everything. Balance cost savings vs isolation needs.
</anti_patterns>

<success_criteria>
A well-designed shared resource architecture has:

- Clear pattern chosen (shared infra vs shared RG vs isolated)
- Shared SQL Server with isolated databases per product
- Azure AD authentication only (no SQL auth)
- Managed Identity for all app-to-resource access
- RBAC on database and Key Vault level
- Clear naming convention: `shared-<env>-<region>-<type>-<instance>`
- Comprehensive tagging for cost allocation
- Private Endpoints for secure access
- Documented governance (ownership, access request, SLAs)
- Resource limits and quotas configured
- Cost allocation report generated monthly
- Blast radius understood and documented
- No manual access (all via IaC and automation)
- Security boundaries respected (no dev/prod mixing)
- Monitoring configured for shared resources
</success_criteria>
