<overview>
Consolidated anti-patterns across all Azure DevOps architect domains - what NOT to do and why, with corrective actions for infrastructure, deployment, security, cost, and architecture.
</overview>

<infrastructure_anti_patterns>
## Infrastructure as Code Anti-Patterns

<anti_pattern name="Manual Resource Creation">
**Problem:** Creating Azure resources via portal instead of IaC.

**Why it's bad:**
- No version control
- Can't reproduce in other environments
- Configuration drift
- No audit trail
- Team members don't know what exists

**Example:**
```
Developer creates storage account in portal for "quick test"
→ Production depends on it
→ No one knows the configuration
→ Account gets deleted accidentally
→ Production breaks
```

**Fix:**
```bicep
// Document EVERYTHING in code
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stmyappprodeastus001'
  location: 'eastus'
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}
```

**Rule:** If it's not in code, it doesn't exist.
</anti_pattern>

<anti_pattern name="Hardcoded Values">
**Problem:** Hardcoding resource names, connection strings, regions in IaC.

**Bad:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'my-app-dev-eastus'  // ❌ Hardcoded environment
  location: 'eastus'          // ❌ Hardcoded region
  properties: {
    serverFarmId: '/subscriptions/12345.../asp-dev'  // ❌ Hardcoded ID
  }
}
```

**Good:**
```bicep
param environment string
param location string = resourceGroup().location
param appServicePlanId string

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-${environment}-${location}'
  location: location
  properties: {
    serverFarmId: appServicePlanId
  }
}
```

**Fix:** Always use parameters for environment-specific values.
</anti_pattern>

<anti_pattern name="No Resource Tagging">
**Problem:** Resources created without tags.

**Why it's bad:**
- Can't allocate costs
- Can't identify owners
- Can't apply automation
- Can't query by environment

**Fix:**
```bicep
param tags object = {
  Environment: 'prod'
  Product: 'myapp'
  CostCenter: 'Engineering'
  Owner: 'team@company.com'
}

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  location: 'eastus'
  tags: tags  // ✅ Always tag
}
```

**Enforce with policy:**
```bicep
resource requireTags 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-tags'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025'
    parameters: {
      tagName: { value: 'CostCenter' }
    }
  }
}
```
</anti_pattern>

<anti_pattern name="Shared State File Without Locking">
**Problem:** Multiple developers/pipelines modifying Terraform state simultaneously.

**Why it's bad:**
- State corruption
- Lost changes
- Inconsistent infrastructure

**Bad:**
```bash
# Local state file
terraform apply  # ❌ No locking, no sharing
```

**Good:**
```hcl
# Terraform backend with locking
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod-eastus"
    storage_account_name = "sttfstateprodeastus"
    container_name       = "tfstate"
    key                  = "myapp.tfstate"
    use_azuread_auth     = true  # Managed Identity
  }
}
```

**Fix:** Always use remote backend with state locking (Azure Storage, Terraform Cloud).
</anti_pattern>
</infrastructure_anti_patterns>

<deployment_anti_patterns>
## Deployment Anti-Patterns

<anti_pattern name="No Blue-Green / Slots">
**Problem:** Deploying directly to production without staging/testing.

**Why it's bad:**
- Downtime during deployment
- No rollback option
- Users see broken state during deployment

**Bad:**
```bash
# Direct production deployment
az webapp deployment source config-zip \
  --resource-group myapp-prod-rg \
  --name app-myapp-prod \
  --src app.zip  # ❌ Downtime, no testing
```

**Good:**
```bash
# Deploy to staging slot
az webapp deployment source config-zip \
  --resource-group myapp-prod-rg \
  --name app-myapp-prod \
  --slot staging \
  --src app.zip

# Test staging slot
curl https://app-myapp-prod-staging.azurewebsites.net/health

# Swap to production (zero downtime)
az webapp deployment slot swap \
  --resource-group myapp-prod-rg \
  --name app-myapp-prod \
  --slot staging \
  --target-slot production
```

**Fix:** Always use deployment slots for App Service, blue-green for AKS.
</anti_pattern>

<anti_pattern name="Secrets in Code">
**Problem:** Committing secrets, connection strings, API keys to Git.

**Why it's bad:**
- Security breach
- Exposed in Git history forever
- Can't rotate without code change

**Bad:**
```typescript
// ❌ NEVER DO THIS
const connectionString = "Server=sql-prod.database.windows.net;Database=mydb;User Id=admin;Password=P@ssw0rd123";
```

**Good:**
```bicep
// Store in Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // Use RBAC, not access policies
  }
}

resource secret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: keyVault
  name: 'SqlConnectionString'
  properties: {
    value: 'Server=sql-prod.database.windows.net;...'
  }
}

// Reference in App Service
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  identity: { type: 'SystemAssigned' }
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'SqlConnectionString'
          value: '@Microsoft.KeyVault(SecretUri=${secret.properties.secretUri})'
        }
      ]
    }
  }
}
```

**Fix:** Always use Key Vault + Managed Identity.
</anti_pattern>

<anti_pattern name="No Health Checks">
**Problem:** Application deployed but no way to verify it's working.

**Why it's bad:**
- Can't detect failures automatically
- Load balancer sends traffic to broken instances
- No monitoring/alerting

**Fix:**
```csharp
// Add health check endpoint
app.MapHealthChecks("/health", new HealthCheckOptions {
  ResponseWriter = async (context, report) => {
    context.Response.ContentType = "application/json";
    var result = JsonSerializer.Serialize(new {
      status = report.Status.ToString(),
      checks = report.Entries.Select(e => new {
        name = e.Key,
        status = e.Value.Status.ToString()
      })
    });
    await context.Response.WriteAsync(result);
  }
});

// Configure in App Service
az webapp config set \
  --resource-group myapp-prod-rg \
  --name app-myapp-prod \
  --health-check-path "/health"
```
</anti_pattern>

<anti_pattern name="Database Deployments Alongside App">
**Problem:** Deploying database schema changes simultaneously with app code.

**Why it's bad:**
- Breaking change if not backward compatible
- Can't roll back app without rolling back DB
- Downtime

**Fix: Expand-contract pattern**
```
Week 1: Add new column (expand)
  → Deploy app that writes to both old and new column
Week 2: Migrate data
  → Backfill new column from old column
Week 3: Update app to read from new column
  → Deploy app that only uses new column
Week 4: Remove old column (contract)
  → Drop old column
```

**Always:** Database changes must be backward compatible with N-1 app version.
</anti_pattern>
</deployment_anti_patterns>

<security_anti_patterns>
## Security Anti-Patterns

<anti_pattern name="Using Access Keys/Passwords">
**Problem:** Using storage account access keys, SQL passwords, service principal secrets.

**Why it's bad:**
- Credentials in code or config
- Hard to rotate
- Easy to leak
- No audit trail

**Bad:**
```bash
# ❌ Access key in connection string
DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=abc123...;EndpointSuffix=core.windows.net
```

**Good:**
```bicep
// App Service with Managed Identity
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  identity: {
    type: 'SystemAssigned'  // ✅ Managed Identity
  }
}

// Grant access to storage
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: 'stmyappprodeastus'
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'Storage Blob Data Contributor')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Application code:**
```csharp
// ✅ Use DefaultAzureCredential (picks up Managed Identity automatically)
var blobClient = new BlobServiceClient(
  new Uri("https://stmyappprodeastus.blob.core.windows.net"),
  new DefaultAzureCredential()
);
```

**Rule:** Zero credentials. Use Managed Identity everywhere.
</anti_pattern>

<anti_pattern name="Public Endpoints for Data Services">
**Problem:** SQL Database, Storage Account, Cosmos DB accessible from internet.

**Why it's bad:**
- Attack surface
- Data exfiltration risk
- Compliance violations

**Bad:**
```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  properties: {
    publicNetworkAccess: 'Enabled'  // ❌ Public access
  }
}
```

**Good:**
```bicep
// Disable public access
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  properties: {
    publicNetworkAccess: 'Disabled'  // ✅ No public access
  }
}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-sql-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    subnet: { id: subnet.id }
    privateLinkServiceConnections: [
      {
        name: 'sql-connection'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: ['sqlServer']
        }
      }
    ]
  }
}
```

**Fix:** Always use Private Endpoints for production data services.
</anti_pattern>

<anti_pattern name="Over-Privileged RBAC">
**Problem:** Granting Owner or Contributor to everyone.

**Why it's bad:**
- Excessive permissions
- No least privilege
- Can't audit who did what

**Bad:**
```bash
# ❌ Too broad
az role assignment create \
  --assignee user@company.com \
  --role "Contributor" \
  --scope "/subscriptions/12345678-1234-1234-1234-123456789012"
```

**Good:**
```bash
# ✅ Specific resource, specific role
az role assignment create \
  --assignee user@company.com \
  --role "App Service Contributor" \
  --scope "/subscriptions/.../resourceGroups/myapp-prod-rg"
```

**Principle of least privilege:**
- Developers: Reader on prod, Contributor on dev
- CI/CD pipeline: Only permissions needed for deployment
- Applications: Managed Identity with specific data roles
</anti_pattern>

<anti_pattern name="No Network Segmentation">
**Problem:** All resources in same subnet, no NSGs.

**Why it's bad:**
- Lateral movement in breach
- Can't isolate compromised resource
- No defense in depth

**Fix:**
```bicep
// Separate subnets
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vnet-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      {
        name: 'snet-app'      // App tier
        properties: { addressPrefix: '10.0.1.0/24' }
      }
      {
        name: 'snet-data'     // Data tier
        properties: { addressPrefix: '10.0.2.0/24' }
      }
      {
        name: 'snet-gateway'  // Gateway tier
        properties: { addressPrefix: '10.0.3.0/24' }
      }
    ]
  }
}

// NSG rules
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-data-prod-eastus'
  location: 'eastus'
  properties: {
    securityRules: [
      {
        name: 'AllowAppToData'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.1.0/24'  // Only from app subnet
          destinationAddressPrefix: '10.0.2.0/24'
          destinationPortRange: '1433'  // SQL
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
    ]
  }
}
```
</anti_pattern>
</security_anti_patterns>

<cost_anti_patterns>
## Cost Optimization Anti-Patterns

<anti_pattern name="Always-On Resources for Sporadic Workloads">
**Problem:** Running App Service 24/7 for job that runs 1 hour/day.

**Cost impact:**
```
App Service (B1): $13/month × 730 hours = $13/month
Azure Function (Consumption): $0.20/million executions
  → 1 hour/day = 30 executions/month = ~$0.01/month

Savings: 99% ($156/year)
```

**Fix:**
```bicep
// Use consumption-based services for sporadic workloads
resource funcApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-batchjob-prod-eastus'
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: consumptionPlan.id  // ✅ Pay per execution
  }
}

resource consumptionPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-consumption-prod-eastus'
  sku: {
    name: 'Y1'  // Consumption plan
    tier: 'Dynamic'
  }
}
```
</anti_pattern>

<anti_pattern name="No Auto-Shutdown for Dev/Test">
**Problem:** Dev/test VMs and environments running 24/7.

**Cost impact:**
```
D2s_v3 VM: $0.096/hour
  → 730 hours/month = $70/month
  → Only needed 40 hours/week (160 hours/month)
  → Wasted: $55/month per VM

10 dev VMs = $550/month waste ($6,600/year)
```

**Fix:**
```bicep
// Auto-shutdown for dev VMs
resource vm 'Microsoft.Compute/virtualMachines@2024-03-01' = {
  name: 'vm-devbox-dev-eastus-001'
  // ...
}

resource autoShutdown 'Microsoft.DevTestLab/schedules@2018-09-15' = {
  name: 'shutdown-computevm-${vm.name}'
  location: 'eastus'
  properties: {
    status: 'Enabled'
    taskType: 'ComputeVmShutdownTask'
    dailyRecurrence: { time: '1900' }  // Shutdown at 7 PM
    timeZoneId: 'Eastern Standard Time'
    targetResourceId: vm.id
    notificationSettings: {
      status: 'Enabled'
      emailRecipient: 'dev-team@company.com'
      timeInMinutes: 30
    }
  }
}
```
</anti_pattern>

<anti_pattern name="No Reserved Instances for Steady-State">
**Problem:** Paying pay-as-you-go for resources that run continuously.

**Cost impact:**
```
App Service (P1v3): $146/month pay-as-you-go
App Service (P1v3): $101/month with 1-year RI (31% savings)
App Service (P1v3): $73/month with 3-year RI (50% savings)

10 app services × $73 savings = $730/month ($8,760/year)
```

**Fix:**
```bash
# Purchase reserved instance
az reservations reservation-order purchase \
  --reservation-order-id "..." \
  --sku "P1v3" \
  --location "eastus" \
  --reserved-resource-type "AppService" \
  --billing-scope "/subscriptions/..." \
  --term "P1Y" \
  --quantity 10
```

**Rule:** If resource runs >50% of time, use Reserved Instance.
</anti_pattern>

<anti_pattern name="Oversized Resources">
**Problem:** Running P3v3 App Service Plan when P1v3 would suffice.

**Cost impact:**
```
P1v3: 2 cores, 8 GB RAM = $146/month
P3v3: 8 cores, 32 GB RAM = $584/month

If app only uses 1 core and 4 GB RAM:
  → Waste: $438/month per app ($5,256/year)
```

**Fix:**
```bash
# Analyze actual usage
az monitor metrics list \
  --resource "/subscriptions/.../providers/Microsoft.Web/sites/app-myapp-prod" \
  --metric "CpuPercentage,MemoryPercentage" \
  --start-time "2025-01-01" \
  --end-time "2025-01-11" \
  --aggregation Average

# If avg CPU < 30% and avg Memory < 50%, downsize
az appservice plan update \
  --name asp-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --sku P1v3  # Down from P3v3
```

**Rule:** Right-size every quarter based on actual metrics.
</anti_pattern>

<anti_pattern name="No Budget Alerts">
**Problem:** No visibility into spending until bill arrives.

**Why it's bad:**
- Surprise bills
- Can't react to anomalies
- No accountability

**Fix:**
```bicep
// Budget with alerts
resource budget 'Microsoft.Consumption/budgets@2023-11-01' = {
  name: 'budget-myapp-prod-monthly'
  properties: {
    category: 'Cost'
    amount: 5000  // $5,000/month
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: '2025-01-01'
    }
    notifications: {
      actual_GreaterThan_80_Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        contactEmails: ['finance@company.com', 'team@company.com']
        contactRoles: ['Owner', 'Contributor']
      }
      forecasted_GreaterThan_100_Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Forecasted'
        contactEmails: ['finance@company.com']
      }
    }
  }
}
```
</anti_pattern>
</cost_anti_patterns>

<architecture_anti_patterns>
## Architecture Anti-Patterns

<anti_pattern name="Distributed Monolith">
**Problem:** Microservices that all must deploy together.

**Why it's bad:**
- Complexity of microservices
- Coupling of monolith
- Worst of both worlds

**Symptoms:**
- Can't deploy service A without deploying B, C, D
- Shared database across all services
- Breaking changes ripple across services

**Fix:**
```
Each service must:
- Own its data (separate database or schema)
- Have versioned API contract
- Deploy independently
- Be backward compatible
```
</anti_pattern>

<anti_pattern name="Chatty APIs">
**Problem:** Client makes 20 API calls to render one page.

**Why it's bad:**
- High latency
- Network overhead
- Poor mobile experience

**Bad:**
```typescript
// ❌ N+1 problem
const orders = await fetch('/api/orders');  // 1 call
for (const order of orders) {
  const customer = await fetch(`/api/customers/${order.customerId}`);  // N calls
  const items = await fetch(`/api/orders/${order.id}/items`);  // N calls
}
// Total: 1 + 2N calls
```

**Good:**
```typescript
// ✅ Single call with aggregated data
const ordersWithDetails = await fetch('/api/orders?include=customer,items');
// Total: 1 call
```

**Fix:** Backend for Frontend (BFF) or GraphQL to aggregate data.
</anti_pattern>

<anti_pattern name="No Retry Logic">
**Problem:** Application fails on first transient error.

**Why it's bad:**
- Poor reliability
- Temporary network blips cause failures
- Bad user experience

**Fix:**
```csharp
// Use Polly for retry
var retryPolicy = Policy
  .Handle<HttpRequestException>()
  .WaitAndRetryAsync(
    retryCount: 3,
    sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
    onRetry: (exception, timeSpan, retryCount, context) => {
      logger.LogWarning($"Retry {retryCount} after {timeSpan}");
    }
  );

await retryPolicy.ExecuteAsync(async () => {
  return await httpClient.GetAsync("https://api.example.com");
});
```

**Rule:** Always retry on transient failures (network, 429, 503).
</anti_pattern>

<anti_pattern name="Sync Communication for Long Operations">
**Problem:** HTTP request waits for long-running operation to complete.

**Why it's bad:**
- Timeouts
- Resource exhaustion
- Poor scalability

**Bad:**
```typescript
// ❌ Sync - client waits 10 minutes
POST /api/reports/generate
Response: (waits 10 minutes) { reportUrl: "..." }
```

**Good:**
```typescript
// ✅ Async - immediate response, poll for status
POST /api/reports/generate
Response: { jobId: "abc123", status: "Processing" }

GET /api/reports/status/abc123
Response: { status: "Processing", progress: 50 }

GET /api/reports/status/abc123
Response: { status: "Completed", reportUrl: "https://..." }
```

**Fix:** Use async patterns for operations >5 seconds.
</anti_pattern>

<anti_pattern name="Ignoring Idempotency">
**Problem:** Retrying non-idempotent operation causes duplicates.

**Example:**
```
Client sends: "Charge credit card $100"
Network timeout (but server processed it)
Client retries: "Charge credit card $100"
Result: Customer charged $200
```

**Fix:**
```csharp
// Idempotency key
[HttpPost("/api/payments")]
public async Task<IActionResult> ProcessPayment(
  [FromBody] PaymentRequest request,
  [FromHeader(Name = "Idempotency-Key")] string idempotencyKey)
{
  // Check if already processed
  var existing = await db.Payments
    .FirstOrDefaultAsync(p => p.IdempotencyKey == idempotencyKey);

  if (existing != null) {
    return Ok(existing);  // Return existing result
  }

  // Process payment
  var payment = await paymentService.Charge(request);
  payment.IdempotencyKey = idempotencyKey;
  await db.Payments.AddAsync(payment);
  await db.SaveChangesAsync();

  return Ok(payment);
}
```

**Rule:** All write operations should be idempotent.
</anti_pattern>

<anti_pattern name="No Correlation IDs">
**Problem:** Can't trace request across multiple services.

**Why it's bad:**
- Impossible to debug distributed issues
- Can't correlate logs
- No end-to-end visibility

**Fix:**
```csharp
// Generate correlation ID at gateway
app.Use(async (context, next) => {
  var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
    ?? Guid.NewGuid().ToString();

  context.Items["CorrelationId"] = correlationId;
  context.Response.Headers["X-Correlation-ID"] = correlationId;

  using (logger.BeginScope(new Dictionary<string, object> {
    ["CorrelationId"] = correlationId
  })) {
    await next();
  }
});

// Pass to downstream services
var request = new HttpRequestMessage(HttpMethod.Get, "https://downstream/api");
request.Headers.Add("X-Correlation-ID", correlationId);
await httpClient.SendAsync(request);
```

**Result:** All logs across all services have same correlation ID.
```
Service A: [CorrelationId: abc-123] Processing order
Service B: [CorrelationId: abc-123] Checking inventory
Service C: [CorrelationId: abc-123] Processing payment
```
</anti_pattern>
</architecture_anti_patterns>

<monitoring_anti_patterns>
## Monitoring & Observability Anti-Patterns

<anti_pattern name="No Alerts">
**Problem:** Monitoring dashboards exist but no one is alerted to problems.

**Why it's bad:**
- Issues discovered by users, not operations
- Slow response time
- Revenue loss

**Fix:**
```bicep
// Alert on high error rate
resource errorRateAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-error-rate-myapp-prod'
  location: 'global'
  properties: {
    severity: 1
    enabled: true
    scopes: [appInsights.id]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighErrorRate'
          metricName: 'requests/failed'
          operator: 'GreaterThan'
          threshold: 10
          timeAggregation: 'Count'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroup.id
      }
    ]
  }
}
```
</anti_pattern>

<anti_pattern name="Logging Everything at Info Level">
**Problem:** Gigabytes of logs, can't find important information.

**Why it's bad:**
- High cost (Log Analytics charges per GB)
- Slow queries
- Signal-to-noise ratio

**Bad:**
```csharp
// ❌ Too verbose
logger.LogInformation("Starting method Foo");
logger.LogInformation("Variable x = {x}", x);
logger.LogInformation("Calling service Bar");
logger.LogInformation("Service Bar returned");
logger.LogInformation("Method Foo completed");
```

**Good:**
```csharp
// ✅ Structured, appropriate levels
logger.LogDebug("Processing order {OrderId}", orderId);  // Only in dev
logger.LogWarning("Retry attempt {RetryCount} for {Operation}", retryCount, operation);
logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);
```

**Fix:**
- Debug: Detailed info, only in dev
- Information: Important business events (order created, payment processed)
- Warning: Recoverable issues (retry, fallback)
- Error: Failures requiring attention
- Critical: System-wide failures
</anti_pattern>

<anti_pattern name="No Structured Logging">
**Problem:** Logs are plain text, can't query efficiently.

**Bad:**
```csharp
// ❌ Unstructured
logger.LogInformation($"Order {orderId} created by user {userId} with total {total}");
```

**Good:**
```csharp
// ✅ Structured
logger.LogInformation("Order created", new {
  OrderId = orderId,
  UserId = userId,
  Total = total,
  ItemCount = items.Count
});
```

**Query in Log Analytics:**
```kql
// Can't do this with unstructured logs
traces
| where customDimensions.Total > 1000
| where customDimensions.ItemCount > 10
| summarize count() by tostring(customDimensions.UserId)
```
</anti_pattern>
</monitoring_anti_patterns>

<summary>
## Key Takeaways

**Infrastructure:**
- Everything in code, nothing manual
- Parameterize everything
- Tag all resources
- Use remote state with locking

**Deployment:**
- Blue-green/slots for zero downtime
- Secrets in Key Vault, not code
- Health checks required
- Database changes must be backward compatible

**Security:**
- Managed Identity, zero credentials
- Private Endpoints for data
- Least privilege RBAC
- Network segmentation

**Cost:**
- Consumption plans for sporadic workloads
- Auto-shutdown for dev/test
- Reserved Instances for steady-state
- Right-size based on metrics
- Budget alerts

**Architecture:**
- Independent deployment
- Aggregate data to reduce chattiness
- Retry transient failures
- Async for long operations
- Idempotent operations
- Correlation IDs

**Monitoring:**
- Alerts, not just dashboards
- Appropriate log levels
- Structured logging
</summary>
