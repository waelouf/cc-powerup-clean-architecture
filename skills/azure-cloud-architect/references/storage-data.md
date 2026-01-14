<overview>
Comprehensive guide to Azure storage and data services. Covers Azure SQL Database, Cosmos DB, Storage Accounts, and data persistence strategies with decision guidance for 2024-2025.
</overview>

<azure_sql_database>
## Azure SQL Database

**What:** Fully managed relational database (SQL Server in the cloud).

**When to use:**
- Relational data model
- ACID transactions
- Existing SQL Server applications
- Complex queries, joins, stored procedures

### Purchasing Models

<model name="DTU (Database Transaction Units)">
**What:** Bundled measure of compute, storage, and IO.

**Tiers:**
- **Basic:** 5 DTUs, 2 GB storage, $5/month - Dev/test only
- **Standard:** 10-3000 DTUs, up to 1 TB, $15-$1,855/month - General purpose
- **Premium:** 125-4000 DTUs, up to 4 TB, $465-$14,807/month - High performance

**When to use:** Simple, predictable workloads

**Example:**
```bicep
resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'myapp-prod-db'
  location: 'eastus'
  sku: {
    name: 'S3'  // Standard tier, 100 DTUs
    tier: 'Standard'
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 268435456000  // 250 GB
  }
}
```

**Cost (Standard):**
- S0 (10 DTUs): $15/month
- S3 (100 DTUs): $149/month
- S6 (400 DTUs): $596/month
</model>

<model name="vCore">
**What:** Choose vCores, memory, storage independently.

**Tiers:**
- **General Purpose:** Balanced, 1-80 vCores, $0.58-$9.28/hour
- **Business Critical:** High performance, in-memory, 1-80 vCores, $1.74-$27.84/hour
- **Hyperscale:** Up to 100 TB, fast backups, 1-80 vCores

**When to use:** Need flexibility, large databases, specific compute requirements

**Example:**
```bicep
resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'myapp-prod-db'
  location: 'eastus'
  sku: {
    name: 'GP_Gen5'  // General Purpose, Gen5 hardware
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 4  // 4 vCores
  }
  properties: {
    maxSizeBytes: 1099511627776  // 1 TB
  }
}
```

**Cost (General Purpose):**
- 2 vCores: $0.58/hour = $420/month
- 4 vCores: $1.16/hour = $840/month
- 8 vCores: $2.32/hour = $1,680/month
</model>

<model name="Serverless">
**What:** Auto-pause during inactivity, pay per vCore-second.

**When to use:**
- Unpredictable usage patterns
- Dev/test with idle periods
- Low to medium average compute utilization

**Example:**
```bicep
resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'myapp-dev-db'
  location: 'eastus'
  sku: {
    name: 'GP_S_Gen5'  // Serverless
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2  // Max 2 vCores
  }
  properties: {
    autoPauseDelay: 60  // Auto-pause after 60 min idle
    minCapacity: 0.5    // Min 0.5 vCore
    maxSizeBytes: 34359738368  // 32 GB
  }
}
```

**Cost:**
- $0.51/vCore-hour (when active)
- $0.14/GB/month for storage
- Auto-pauses when idle (zero compute cost)
- Example: Active 8 hours/day = ~$120/month
</model>

### SQL Server Configuration

```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-shared-prod-eastus-001'
  location: 'eastus'
  properties: {
    administratorLogin: null  // Azure AD only!
    administratorLoginPassword: null
    administrators: {
      administratorType: 'ActiveDirectory'
      login: 'dba-group@company.com'
      sid: adminGroupObjectId
      tenantId: subscription().tenantId
      azureADOnlyAuthentication: true  // No SQL authentication
    }
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'  // Private endpoint only
  }
}
```

### Elastic Pools

**What:** Share resources across multiple databases.

**When to use:**
- Multiple databases with varying usage patterns
- Cost optimization (cheaper than individual databases)

```bicep
resource elasticPool 'Microsoft.Sql/servers/elasticPools@2023-08-01-preview' = {
  parent: sqlServer
  name: 'tenant-pool'
  location: 'eastus'
  sku: {
    name: 'StandardPool'
    tier: 'Standard'
    capacity: 100  // 100 DTUs shared
  }
  properties: {
    perDatabaseSettings: {
      minCapacity: 0
      maxCapacity: 100
    }
  }
}

// Databases in pool
resource database1 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'tenant1-db'
  location: 'eastus'
  properties: {
    elasticPoolId: elasticPool.id
  }
}
```

**Cost example:**
- 100 DTU pool: $149/month
- 10 databases sharing pool vs 10 individual S3 databases
- Pool: $149/month vs Individual: $1,490/month
- **Savings: 90%**

### Backup and Retention

**Automatic backups:**
- Full backup: Weekly
- Differential backup: Every 12-24 hours
- Transaction log backup: Every 5-10 minutes
- Default retention: 7 days
- Max retention: 35 days

**Long-term retention:**
```bicep
resource backupLongTermRetention 'Microsoft.Sql/servers/databases/backupLongTermRetentionPolicies@2023-08-01-preview' = {
  parent: sqlDatabase
  name: 'default'
  properties: {
    weeklyRetention: 'P5W'   // 5 weeks
    monthlyRetention: 'P12M' // 12 months
    yearlyRetention: 'P5Y'   // 5 years
    weekOfYear: 1
  }
}
```
</azure_sql_database>

<cosmos_db>
## Azure Cosmos DB

**What:** Globally distributed, multi-model NoSQL database.

**When to use:**
- Global distribution (multi-region writes)
- Low latency requirements (< 10ms)
- NoSQL data models (document, key-value, graph, column-family)
- Massive scale (millions of requests/sec)

**When NOT to use:**
- Complex joins, transactions across documents
- Budget-constrained (more expensive than SQL)
- Relational data model

### APIs

- **Core (SQL):** Document store, SQL-like queries
- **MongoDB:** Compatible with MongoDB apps
- **Cassandra:** Wide-column store
- **Gremlin:** Graph database
- **Table:** Key-value store

### Pricing Models

**Provisioned throughput:**
- RU/s (Request Units per second)
- 400 RU/s minimum = $24/month
- Each additional 100 RU/s = $5.84/month
- Reserved capacity: 20-65% discount

**Serverless:**
- Pay per request (no minimum)
- $0.28 per million RUs
- $0.25/GB/month storage
- Good for: Dev/test, low traffic, unpredictable

**Autoscale:**
- Automatically scale between min/max RU/s
- Pay for actual usage
- Max 10x min throughput

**Example:**
```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: 'cosmos-myapp-prod'
  location: 'eastus'
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    enableAutomaticFailover: true
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'  // Balance of consistency/performance
    }
    locations: [
      {
        locationName: 'East US'
        failoverPriority: 0
      }
      {
        locationName: 'West US 2'
        failoverPriority: 1
      }
    ]
  }
}

resource cosmosDatabase 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-11-15' = {
  parent: cosmosAccount
  name: 'myapp-db'
  properties: {
    resource: {
      id: 'myapp-db'
    }
  }
}

resource cosmosContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: cosmosDatabase
  name: 'users'
  properties: {
    resource: {
      id: 'users'
      partitionKey: {
        paths: ['/userId']
        kind: 'Hash'
      }
    }
    options: {
      throughput: 400  // Provisioned RU/s
    }
  }
}
```

### Cost Example

**Low traffic app (serverless):**
```
10M requests/month
1M RUs consumed
10 GB data

Cost:
- Requests: $0.28
- Storage: $2.50
Total: ~$3/month
```

**High traffic app (provisioned):**
```
1000 RU/s provisioned
100 GB data
2 regions (replication)

Cost:
- Throughput: $584/month × 2 regions = $1,168/month
- Storage: $25/month × 2 regions = $50/month
Total: ~$1,218/month
```
</cosmos_db>

<storage_accounts>
## Azure Storage Accounts

**What:** Object storage for blobs, files, queues, tables.

**Types:**
- **Blob storage:** Unstructured data (images, videos, backups)
- **File storage:** SMB file shares
- **Queue storage:** Message queuing
- **Table storage:** NoSQL key-value store

### Blob Storage Tiers

**Hot:**
- Frequent access
- $0.018/GB/month
- Lowest access cost
- Use for: Active data, logs

**Cool:**
- Infrequent access (30+ days)
- $0.010/GB/month
- Higher access cost
- Use for: Backups, archives

**Archive:**
- Rare access (180+ days)
- $0.002/GB/month
- Highest access cost + retrieval time (hours)
- Use for: Long-term archives, compliance

### Configuration

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stmyappprodeastus001'
  location: 'eastus'
  sku: {
    name: 'Standard_LRS'  // Locally redundant
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
    allowBlobPublicAccess: false  // Security: no anonymous access
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'  // Private endpoint only
    }
  }
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 30  // Soft delete (recover within 30 days)
    }
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 30
    }
  }
}
```

### Redundancy Options

**LRS (Locally Redundant):** 3 copies in one datacenter
- Cost: 1x
- Durability: 99.999999999% (11 nines)
- Use for: Non-critical data

**ZRS (Zone Redundant):** 3 copies across availability zones
- Cost: 1.25x
- Durability: 99.9999999999% (12 nines)
- Use for: Production data in supported regions

**GRS (Geo-Redundant):** LRS + replicated to secondary region
- Cost: 2x
- Durability: 99.99999999999999% (16 nines)
- Use for: Critical data, disaster recovery

**GZRS (Geo-Zone Redundant):** ZRS + replicated to secondary region
- Cost: 2.5x
- Best durability
- Use for: Mission-critical data

### Lifecycle Management

```bicep
resource lifecyclePolicy 'Microsoft.Storage/storageAccounts/managementPolicies@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    policy: {
      rules: [
        {
          name: 'move-to-cool-after-30-days'
          enabled: true
          type: 'Lifecycle'
          definition: {
            filters: {
              blobTypes: ['blockBlob']
              prefixMatch: ['logs/', 'backups/']
            }
            actions: {
              baseBlob: {
                tierToCool: {
                  daysAfterModificationGreaterThan: 30
                }
                tierToArchive: {
                  daysAfterModificationGreaterThan: 90
                }
                delete: {
                  daysAfterModificationGreaterThan: 365
                }
              }
              snapshot: {
                delete: {
                  daysAfterCreationGreaterThan: 90
                }
              }
            }
          }
        }
      ]
    }
  }
}
```

### Cost Optimization

**Storage cost example (1 TB data):**
```
Hot (LRS): $18/month
Cool (LRS): $10/month
Archive (LRS): $2/month

Lifecycle policy savings:
- Month 0-1: Hot ($18)
- Month 2-3: Cool ($10)
- Month 4+: Archive ($2)
Average: $10/month vs $18/month = 44% savings
```
</storage_accounts>

<data_persistence_patterns>
## Data Persistence Decision Guide

### When to Use Each Service

**Azure SQL Database:**
- ✅ Relational data, structured schema
- ✅ Complex queries, joins, transactions
- ✅ ACID compliance required
- ✅ Existing SQL Server apps
- ❌ Massive scale (millions of req/sec)
- ❌ Global distribution

**Cosmos DB:**
- ✅ Global distribution, multi-region writes
- ✅ Low latency (< 10ms)
- ✅ NoSQL data models
- ✅ Massive scale, elastic
- ❌ Complex joins
- ❌ Budget constrained (more expensive)

**Blob Storage:**
- ✅ Unstructured data (files, images, videos)
- ✅ Large objects (GBs, TBs)
- ✅ Cheap storage
- ❌ Querying data
- ❌ Relational data

**Table Storage:**
- ✅ Simple key-value, NoSQL
- ✅ Very cheap ($0.07/GB/month)
- ✅ Large volumes of data
- ❌ Complex queries
- ❌ Relationships, joins

### Hybrid Patterns

**Pattern: SQL + Blob Storage**
```
SQL Database: Metadata, relationships
Blob Storage: Actual files (images, PDFs)

Example:
Users table → profile picture URL → Blob storage
```

**Pattern: SQL + Cosmos DB**
```
SQL Database: Transactional data (orders, payments)
Cosmos DB: Catalog, product search (global, low latency)
```

**Pattern: Cosmos DB + Blob**
```
Cosmos DB: Document metadata
Blob Storage: Large documents
```
</data_persistence_patterns>

<database_connections>
## Database Connection Best Practices

### Connection Strings in Key Vault

**Never hardcode connection strings:**

```bicep
// 1. Store in Key Vault
resource secret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: keyVault
  name: 'database-connection'
  properties: {
    value: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDatabase.name};'
  }
}

// 2. Reference from App Service
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  properties: {
    siteConfig: {
      connectionStrings: [
        {
          name: 'DefaultConnection'
          connectionString: '@Microsoft.KeyVault(SecretUri=${secret.properties.secretUri})'
          type: 'SQLAzure'
        }
      ]
    }
  }
}
```

### Connection Pooling

**Use connection pooling to avoid exhausting database connections:**

```javascript
// Node.js (mssql)
const pool = new sql.ConnectionPool({
  server: process.env.DB_SERVER,
  database: process.env.DB_NAME,
  authentication: {
    type: 'azure-active-directory-default'  // Managed Identity
  },
  pool: {
    max: 10,  // Max connections
    min: 0,
    idleTimeoutMillis: 30000
  }
});

await pool.connect();
```

```csharp
// .NET
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        configuration.GetConnectionString("DefaultConnection"),
        sqlServerOptions => {
            sqlServerOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null
            );
        }
    )
);
```

### Managed Identity Authentication

**No passwords in connection string:**

```csharp
// Connection string with Managed Identity
Server=tcp:sql-server.database.windows.net,1433;Database=mydb;Authentication=Active Directory Default;

// Code
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(new TokenRequestContext(new[] {
    "https://database.windows.net/.default"
}));

connection.AccessToken = token.Token;
```

**Grant access:**
```sql
-- Run as SQL admin
CREATE USER [myapp-prod-app-001] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [myapp-prod-app-001];
ALTER ROLE db_datawriter ADD MEMBER [myapp-prod-app-001];
```
</database_connections>

<anti_patterns>
**Avoid:**

<anti_pattern name="Connection Strings in Code">
**Problem:** Hardcoded database passwords

**Why bad:** Security breach, credentials in Git

**Instead:** Store in Key Vault, use Managed Identity
</anti_pattern>

<anti_pattern name="No Connection Pooling">
**Problem:** Opening new connection for each query

**Why bad:** Exhausts database connections, slow performance

**Instead:** Use connection pooling, reuse connections
</anti_pattern>

<anti_pattern name="Wrong Database Choice">
**Problem:** Using SQL for unstructured data, Cosmos DB for relational

**Why bad:** Poor performance, high cost

**Instead:** Match database to data model (SQL for relational, Blob for files)
</anti_pattern>

<anti_pattern name="No Lifecycle Policy">
**Problem:** Storing all data in Hot tier forever

**Why bad:** Wasting money on infrequently accessed data

**Instead:** Implement lifecycle policies to move to Cool/Archive
</anti_pattern>

<anti_pattern name="Wrong Redundancy">
**Problem:** GRS for dev/test data

**Why bad:** Paying 2x for data that doesn't need geo-redundancy

**Instead:** LRS for non-critical, GRS for production only
</anti_pattern>
</anti_patterns>
