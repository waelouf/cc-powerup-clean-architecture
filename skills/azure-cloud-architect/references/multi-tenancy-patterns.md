<overview>
Comprehensive guide to designing shared and multi-tenant infrastructure on Azure. Covers patterns for sharing resources across multiple products, teams, and tenants with security, cost allocation, and isolation strategies.
</overview>

<fundamental_patterns>
## Three Core Multi-Tenancy Patterns

<pattern name="Shared Database, Shared Schema">
**Description:** All tenants share same database and same schema. Tenant identified by column (TenantID).

**When to use:**
- SaaS applications with thousands of small tenants
- Cost optimization critical
- Simple data model
- Low customization needs

**Azure implementation:**
- Single Azure SQL Database
- Row-Level Security (RLS) for tenant isolation
- Single App Service serving all tenants

**Example schema:**
```sql
CREATE TABLE Products (
    ProductID INT PRIMARY KEY,
    TenantID INT NOT NULL,  -- Tenant identifier
    ProductName NVARCHAR(100),
    Price DECIMAL(10,2)
)

-- Row-Level Security
CREATE FUNCTION fn_TenantAccessPredicate(@TenantID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantId') AS INT)

CREATE SECURITY POLICY TenantSecurityPolicy
ADD FILTER PREDICATE dbo.fn_TenantAccessPredicate(TenantID) ON dbo.Products
WITH (STATE = ON)
```

**Pros:**
- Lowest cost (one database for all)
- Simple to deploy and manage
- Easy to provision new tenants

**Cons:**
- Limited isolation (all data in one database)
- Noisy neighbor risk (one tenant can affect others)
- Difficult to customize per tenant
- Compliance challenges (data co-mingling)

**Cost:** 💰 (Lowest)
**Isolation:** 🔒 (Lowest)
**Complexity:** ⚙️ (Lowest)
</pattern>

<pattern name="Shared Database, Separate Schema">
**Description:** All tenants share same database but each has own schema.

**When to use:**
- Medium number of tenants (10-100)
- Need logical separation
- Different schema per tenant allowed
- Cost optimization important

**Azure implementation:**
- Single Azure SQL Database
- Each tenant gets own schema
- Connection string specifies default schema

**Example:**
```sql
-- Tenant 1 schema
CREATE SCHEMA Tenant1
CREATE TABLE Tenant1.Products (...)

-- Tenant 2 schema
CREATE SCHEMA Tenant2
CREATE TABLE Tenant2.Products (...)

-- Application connects with specific default schema
-- Connection string: "...;Initial Catalog=SharedDB;Default Schema=Tenant1"
```

**Pros:**
- Better isolation than shared schema
- Can customize schema per tenant
- Lower cost than database-per-tenant
- Easier compliance (logical separation)

**Cons:**
- More complex than shared schema
- Database size limits affect all tenants
- Still some noisy neighbor risk
- Schema migration complexity

**Cost:** 💰💰 (Low-Medium)
**Isolation:** 🔒🔒 (Medium)
**Complexity:** ⚙️⚙️ (Medium)
</pattern>

<pattern name="Database Per Tenant">
**Description:** Each tenant gets own dedicated database.

**When to use:**
- Strong isolation required
- Compliance requirements (HIPAA, financial)
- Large tenants with significant data
- Custom schema per tenant needed
- Billing per tenant

**Azure implementation:**
- Each tenant: Own Azure SQL Database
- Shared: SQL Server (with multiple databases)
- Elastic Pool for cost optimization

**Example:**
```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'shared-prod-sql-001'
  // SQL Server configuration
}

// Database for Tenant 1
resource tenant1Database 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'tenant1-prod-db'
  sku: {
    name: 'S3'
    tier: 'Standard'
  }
  tags: {
    TenantId: 'tenant1'
    CostCenter: 'Tenant1-CC'
  }
}

// Database for Tenant 2
resource tenant2Database 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'tenant2-prod-db'
  sku: {
    name: 'S1'
    tier: 'Standard'
  }
  tags: {
    TenantId: 'tenant2'
    CostCenter: 'Tenant2-CC'
  }
}

// Elastic Pool option (cost optimization)
resource elasticPool 'Microsoft.Sql/servers/elasticPools@2023-08-01-preview' = {
  parent: sqlServer
  name: 'tenant-pool'
  sku: {
    name: 'StandardPool'
    tier: 'Standard'
    capacity: 100  // eDTUs shared across databases
  }
}
```

**Pros:**
- Maximum isolation and security
- Independent scaling per tenant
- Easy to backup/restore individual tenant
- Clear cost allocation
- Compliance-friendly

**Cons:**
- Higher cost
- More databases to manage
- Complex provisioning automation
- Schema migration across many databases

**Cost:** 💰💰💰 (High)
**Isolation:** 🔒🔒🔒 (High)
**Complexity:** ⚙️⚙️⚙️ (High)
</pattern>
</fundamental_patterns>

<shared_infrastructure_patterns>
## Shared Infrastructure for Multiple Products

<scenario name="Shared SQL Server, Isolated Databases">
**Use case:** Multiple products from same organization sharing SQL Server.

**Architecture:**
```
SQL Server: shared-prod-eastus-sql-001
├── Database: product1-prod-db (Owner: Team A)
├── Database: product2-prod-db (Owner: Team B)
├── Database: product3-prod-db (Owner: Team C)
└── Database: shared-reference-db (Owner: Platform Team)
```

**Implementation:**
```bicep
@description('Azure AD admin for SQL Server')
param azureAdAdminEmail string
param azureAdAdminObjectId string

resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'shared-prod-eastus-sql-001'
  location: 'eastus'
  properties: {
    administrators: {
      administratorType: 'ActiveDirectory'
      login: azureAdAdminEmail
      sid: azureAdAdminObjectId
      tenantId: subscription().tenantId
      azureADOnlyAuthentication: true  // No SQL auth!
    }
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'  // Private endpoint only
  }
  tags: {
    Purpose: 'Shared Database Server'
    SharedBy: 'product1,product2,product3'
    ManagedBy: 'Platform Team'
  }
}

// Product 1 database
resource product1Db 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'product1-prod-db'
  sku: {
    name: 'S3'
    tier: 'Standard'
  }
  tags: {
    Product: 'product1'
    Owner: 'team-a@company.com'
    CostCenter: 'Engineering-ProductA'
  }
}
```

**Access Control:**
Each app uses Managed Identity with database-level permissions:

```sql
-- Run this for each app's managed identity
-- Connect to specific database: product1-prod-db

CREATE USER [product1-app-identity] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [product1-app-identity];
ALTER ROLE db_datawriter ADD MEMBER [product1-app-identity];
GO

-- product1 app can ONLY access product1-prod-db
-- product2 app can ONLY access product2-prod-db
```

**Benefits:**
- Cost savings: One SQL Server instead of N servers
- Centralized management and patching
- Easier monitoring and backup

**Considerations:**
- Server-level issues affect all products
- Need clear ownership and SLA documentation
- Resource limits at server level
</scenario>

<scenario name="Shared Key Vault">
**Use case:** Multiple products storing secrets in same Key Vault.

**Architecture:**
```
Key Vault: shared-prod-eastus-kv-001
├── Secret: product1-db-connection
├── Secret: product1-api-key
├── Secret: product2-db-connection
├── Secret: product2-external-api-key
└── Secret: shared-storage-key
```

**Implementation:**
```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'shared-prod-eastus-kv-001'
  location: 'eastus'
  properties: {
    sku: {
      family: 'A'
      name: 'premium'  // HSM-backed
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // RBAC, not access policies!
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'
      virtualNetworkRules: []
      ipRules: []
    }
  }
}

// Grant Product1 app access to its secrets only
resource product1SecretAccess 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, product1AppIdentity, 'secrets-user')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')  // Key Vault Secrets User
    principalId: product1AppIdentity
    principalType: 'ServicePrincipal'
  }
}
```

**Secret Naming Convention:**
```
<product>-<environment>-<purpose>

Examples:
- product1-prod-db-connection
- product2-prod-api-key
- shared-prod-storage-key
```

**Access Pattern:**
```csharp
// Product1 app code - uses Managed Identity
var credential = new DefaultAzureCredential();
var client = new SecretClient(
    new Uri("https://shared-prod-eastus-kv-001.vault.azure.net/"),
    credential
);

// Can only read secrets this identity has access to
var secret = await client.GetSecretAsync("product1-prod-db-connection");
```

**RBAC Best Practice:**
Use **secret-level permissions** (Azure RBAC 2023+) instead of vault-level:

```bash
# Grant access to specific secret only
az keyvault set-policy \
  --name shared-prod-eastus-kv-001 \
  --object-id <app-identity-id> \
  --secret-permissions get \
  --secret-name product1-prod-db-connection
```
</scenario>

<scenario name="Shared App Service Plan">
**Use case:** Multiple apps sharing compute resources.

**When to share:**
- Apps have similar traffic patterns
- All apps owned by same team
- Cost optimization priority
- Total load fits within plan limits

**When NOT to share:**
- Different scaling requirements
- One app has unpredictable spikes
- Different teams (hard to allocate costs)

**Implementation:**
```bicep
resource sharedPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'shared-prod-eastus-asp-001'
  location: 'eastus'
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
    capacity: 2  // Start with 2 instances
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
  tags: {
    Purpose: 'Shared App Service Plan'
    SharedBy: 'product1-api,product2-api,product3-web'
  }
}

// Product 1 API on shared plan
resource product1Api 'Microsoft.Web/sites@2023-12-01' = {
  name: 'product1-prod-api-001'
  location: 'eastus'
  properties: {
    serverFarmId: sharedPlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      alwaysOn: true
    }
  }
  tags: {
    Product: 'product1'
    CostCenter: 'Product1-CC'
  }
}

// Product 2 API on same shared plan
resource product2Api 'Microsoft.Web/sites@2023-12-01' = {
  name: 'product2-prod-api-001'
  location: 'eastus'
  properties: {
    serverFarmId: sharedPlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'PYTHON|3.11'
      alwaysOn: true
    }
  }
  tags: {
    Product: 'product2'
    CostCenter: 'Product2-CC'
  }
}

// Auto-scaling based on combined load
resource autoScale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: '${sharedPlan.name}-autoscale'
  location: 'eastus'
  properties: {
    targetResourceUri: sharedPlan.id
    enabled: true
    profiles: [
      {
        name: 'Auto scale condition'
        capacity: {
          minimum: '2'
          maximum: '10'
          default: '2'
        }
        rules: [
          {
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT5M'
            }
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: sharedPlan.id
              operator: 'GreaterThan'
              statistic: 'Average'
              threshold: 70
              timeAggregation: 'Average'
              timeGrain: 'PT1M'
              timeWindow: 'PT5M'
            }
          }
        ]
      }
    ]
  }
}
```

**Cost Allocation:**
Can't split App Service Plan cost automatically. Use tagging and manual allocation based on:
- Number of apps per team
- CPU/memory usage metrics per app
- Business agreement

**Monitor resource usage per app:**
```bash
az monitor metrics list \
  --resource <app-id> \
  --metric "CpuTime,MemoryWorkingSet,HttpResponseTime" \
  --start-time "2025-01-01T00:00:00Z" \
  --end-time "2025-01-07T23:59:59Z" \
  --aggregation Average
```
</scenario>
</shared_infrastructure_patterns>

<resource_organization>
## Resource Organization Strategies

<strategy name="Single Shared Resource Group">
**Structure:**
```
shared-prod-eastus-rg/
├── SQL Server (shared)
├── Databases (one per product, tagged)
├── Key Vault (shared)
├── App Service Plan (shared)
└── Application Insights (shared or per-product)
```

**Pros:** Simple, centralized management
**Cons:** No automatic cost breakout, RBAC at resource level needed

**Use when:** Small number of products, same team
</strategy>

<strategy name="Shared + Product Resource Groups">
**Structure:**
```
shared-prod-eastus-rg/
├── SQL Server (shared infrastructure)
└── Key Vault (shared secrets)

product1-prod-eastus-rg/
├── App Service (references shared ASP)
├── Database (on shared SQL Server)
└── Storage Account

product2-prod-eastus-rg/
├── App Service
├── Database (on shared SQL Server)
└── Storage Account
```

**Pros:** Clear cost allocation, better RBAC boundaries
**Cons:** More resource groups to manage

**Use when:** Multiple teams, need cost visibility
**Recommended:** This approach for enterprise scenarios
</strategy>

<strategy name="Management Group Hierarchy">
For large organizations:

```
Root Management Group
├── Platform Services (shared infra team)
│   └── Shared-Prod subscription
│       └── shared-prod-eastus-rg
├── Product A (team A)
│   └── ProductA-Prod subscription
│       └── product-a-prod-eastus-rg
└── Product B (team B)
    └── ProductB-Prod subscription
        └── product-b-prod-eastus-rg
```

**Benefits:**
- Clear ownership and billing
- Policies applied hierarchically
- Subscription-level isolation
</strategy>
</resource_organization>

<cost_allocation>
## Cost Allocation for Shared Resources

### Tagging Strategy:

**Required tags for all resources:**
```bicep
tags: {
  Environment: 'prod'              // dev/staging/prod
  Product: 'product1'              // Which product owns this
  CostCenter: 'Engineering-P1'     // For chargeback
  Owner: 'team-a@company.com'      // Contact
  SharedResource: 'yes/no'         // Is this shared?
  SharedBy: 'p1,p2,p3'            // Which products share (if applicable)
}
```

### Cost Allocation Methods:

<method name="Usage-Based">
**For:** Shared databases, storage, compute

**Approach:** Measure actual usage per product:
- Database size/DTU usage
- Storage GB per container/blob prefix
- Compute hours per app

**Implementation:**
```sql
-- Get database size per product
SELECT
    d.name AS DatabaseName,
    t.value AS Product,
    SUM(a.total_pages) * 8/1024/1024.0 AS SizeGB
FROM sys.databases d
JOIN sys.dm_db_partition_stats a
    ON d.database_id = a.database_id
LEFT JOIN sys.tags t
    ON d.database_id = t.resource_id AND t.key = 'Product'
GROUP BY d.name, t.value
```

Allocate cost proportionally to usage.
</method>

<method name="Equal Split">
**For:** Shared infrastructure with similar usage

**Approach:** Divide cost equally among products

**Example:** SQL Server costs $500/mo shared by 4 products = $125 each
</method>

<method name="Fixed Allocation">
**For:** Known capacity reservations

**Approach:** Pre-agreed percentages

**Example:**
- Product A: 50% of shared plan capacity = 50% of cost
- Product B: 30% = 30%
- Product C: 20% = 20%
</method>

### Automated Cost Reports:

```bash
#!/bin/bash
# monthly-cost-allocation.sh

# Get costs by product tag
az costmanagement query \
  --type Usage \
  --timeframe Custom \
  --time-period from="2025-01-01" to="2025-01-31" \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="TagKey" type="Dimension" \
  --query "properties.rows" \
  | jq '.[] | select(.[1] == "Product") | {product: .[2], cost: .[0]}'

# Export to CSV for finance team
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-grouping name="TagKey" type="Dimension" \
  --output tsv > cost-allocation.csv
```
</cost_allocation>

<governance>
## Governance and Documentation

### Shared Resource Registry:

Create `SHARED-RESOURCES.md`:

```markdown
# Shared Infrastructure Registry

## Shared SQL Server: shared-prod-eastus-sql-001

**Purpose:** Multi-product database hosting
**Managed by:** Platform Team (platform@company.com)
**SLA:** 99.99% uptime
**Backup:** Daily, 35-day retention
**Maintenance Window:** Sunday 2-4 AM EST

### Databases:
| Database | Product | Owner | Size Limit | Cost Allocation |
|----------|---------|-------|------------|-----------------|
| product1-prod-db | Product 1 | team-a@company.com | 100 GB | Usage-based |
| product2-prod-db | Product 2 | team-b@company.com | 50 GB | Usage-based |
| product3-prod-db | Product 3 | team-c@company.com | 25 GB | Usage-based |

### Access:
- Azure AD authentication only
- Each app uses Managed Identity
- Database-level RBAC permissions
- No direct access (apps only)

### Request New Database:
1. Submit ticket: <ticketing-system-url>
2. Provide: Product name, expected size, owner email
3. Provisioning time: 1 business day

## Shared Key Vault: shared-prod-eastus-kv-001

**Purpose:** Centralized secret storage
**Managed by:** Security Team (security@company.com)

### Secret Naming Convention:
`<product>-<env>-<purpose>`

### Access Request:
Submit Security Request Form
```

### Access Control Matrix:

| Resource | Platform Team | Product A Team | Product B Team |
|----------|--------------|----------------|----------------|
| SQL Server | Owner | - | - |
| product-a-db | Contributor | Owner (via MI) | - |
| product-b-db | Contributor | - | Owner (via MI) |
| Shared Key Vault | Owner | Reader (specific secrets) | Reader (specific secrets) |
| Shared App Service Plan | Owner | Reader | Reader |

### Monitoring and Alerts:

```bicep
// Alert if shared SQL Server DTU > 80%
resource sqlAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'sql-server-high-dtu'
  location: 'global'
  properties: {
    severity: 2
    enabled: true
    scopes: [sqlServer.id]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'High DTU'
          metricName: 'dtu_consumption_percent'
          operator: 'GreaterThan'
          threshold: 80
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: platformTeamActionGroup.id
      }
    ]
  }
}
```
</governance>

<anti_patterns>
**Avoid:**

<anti_pattern name="Sharing Across Environments">
**Problem:** Using same SQL Server for dev and prod

**Why bad:** Security boundary violation, dev issues affect prod

**Instead:** Separate servers for dev/staging/prod
</anti_pattern>

<anti_pattern name="No Resource Limits">
**Problem:** No quotas on shared resources

**Why bad:** One product can consume all capacity

**Instead:** Set database size limits, DTU limits, connection limits
</anti_pattern>

<anti_pattern name="Poor Naming Convention">
**Problem:** Resources named generically: db1, db2, kv1

**Why bad:** Can't identify ownership or purpose

**Instead:** Use pattern: `<product>-<env>-<region>-<type>-<instance>`
</anti_pattern>

<anti_pattern name="Manual Cost Allocation">
**Problem:** Calculating chargeback manually each month

**Why bad:** Time consuming, error prone

**Instead:** Automate with tags and Azure Cost Management queries
</anti_pattern>

<anti_pattern name="No Blast Radius Documentation">
**Problem:** Don't know what fails if shared resource fails

**Why bad:** Can't assess risk or plan DR

**Instead:** Document dependencies and impact of failures
</anti_pattern>
</anti_patterns>
