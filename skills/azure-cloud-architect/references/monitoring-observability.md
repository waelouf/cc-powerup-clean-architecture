<overview>
Comprehensive guide to monitoring and observability on Azure. Covers Application Insights, Log Analytics, Azure Monitor, alerting, dashboards, and observability best practices for 2024-2025.
</overview>

<observability_fundamentals>
## Three Pillars of Observability

<pillar name="Logs">
**What:** Discrete events with timestamp and context

**Use for:**
- Debugging specific issues
- Audit trails
- Understanding event sequences

**Azure services:** Log Analytics, Application Insights

**Example:**
```
2025-01-11T14:32:15Z [INFO] User 12345 logged in from 192.168.1.1
2025-01-11T14:32:18Z [ERROR] Database connection timeout after 30s
```
</pillar>

<pillar name="Metrics">
**What:** Numeric measurements over time

**Use for:**
- Understanding trends
- Capacity planning
- Alerting on thresholds
- Performance analysis

**Azure services:** Azure Monitor Metrics

**Example:**
- CPU percentage: 45%, 52%, 48%, 51%
- Request count: 1200 req/min
- Response time (P95): 250ms
</pillar>

<pillar name="Traces">
**What:** Request flow through distributed system

**Use for:**
- Understanding request path
- Identifying bottlenecks
- Debugging distributed systems
- Dependency analysis

**Azure services:** Application Insights distributed tracing

**Example:**
```
Request → API Gateway (50ms)
  → App Service (200ms)
    → SQL Database (150ms)
  → Redis Cache (10ms)
Total: 410ms
```
</pillar>
</observability_fundamentals>

<application_insights>
## Application Insights

**Purpose:** APM (Application Performance Monitoring) for applications.

### Setup Application Insights

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'myapp-prod-insights'
  location: 'eastus'
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalyticsWorkspace.id  // Link to Log Analytics
    IngestionMode: 'LogAnalytics'  // 2024+ recommendation
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
  tags: {
    Environment: 'prod'
    Product: 'myapp'
  }
}

output instrumentationKey string = appInsights.properties.InstrumentationKey
output connectionString string = appInsights.properties.ConnectionString
```

### Integrate with Application

**App Service:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'ApplicationInsightsAgent_EXTENSION_VERSION'
          value: '~3'  // Auto-instrumentation
        }
      ]
    }
  }
}
```

**Node.js application:**
```javascript
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true, true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true)
    .setUseDiskRetryCaching(true)
    .setSendLiveMetrics(true)
    .start();

const client = appInsights.defaultClient;

// Custom event
client.trackEvent({ name: 'UserSignup', properties: { userId: 12345 } });

// Custom metric
client.trackMetric({ name: 'QueueLength', value: 42 });

// Track dependency
client.trackDependency({
    target: 'https://api.example.com',
    name: 'GET /users',
    data: '/users?limit=10',
    duration: 125,
    resultCode: 200,
    success: true
});
```

**.NET application:**
```csharp
services.AddApplicationInsightsTelemetry();

// Custom tracking
var telemetryClient = new TelemetryClient();
telemetryClient.TrackEvent("OrderPlaced", new Dictionary<string, string>
{
    { "OrderId", "12345" },
    { "Amount", "99.99" }
});
```

### Key Metrics to Monitor

**Availability:**
- Availability percentage (target: 99.9%+)
- Failed requests
- Server exceptions

**Performance:**
- Server response time (P50, P95, P99)
- Dependency duration
- Page load time

**Usage:**
- Request count
- User sessions
- Page views

**Dependencies:**
- SQL query duration
- External API response time
- Redis cache latency

### Query Application Insights (Kusto/KQL)

**Recent exceptions:**
```kql
exceptions
| where timestamp > ago(1h)
| summarize count() by type, outerMessage
| order by count_ desc
```

**Slow requests (P95):**
```kql
requests
| where timestamp > ago(24h)
| summarize percentile(duration, 95) by bin(timestamp, 1h), name
| render timechart
```

**Failed dependencies:**
```kql
dependencies
| where success == false
| where timestamp > ago(1h)
| summarize count() by target, type, resultCode
```

**Custom events funnel:**
```kql
customEvents
| where timestamp > ago(7d)
| where name in ("ViewProduct", "AddToCart", "Checkout", "Purchase")
| summarize count() by name
| order by count_ desc
```

### Live Metrics

**Real-time monitoring:** Portal → Application Insights → Live Metrics

Shows:
- Incoming requests per second
- Outgoing requests per second
- Overall health
- Server CPU/memory
- Exception rate

Use during:
- Deployments (verify health)
- Load testing
- Incident response

### Availability Tests

**Monitor endpoint availability:**

```bicep
resource availabilityTest 'Microsoft.Insights/webtests@2022-06-15' = {
  name: 'myapp-availability-test'
  location: 'eastus'
  kind: 'ping'
  properties: {
    Name: 'Homepage availability'
    Description: 'Ping homepage every 5 minutes'
    Enabled: true
    Frequency: 300  // 5 minutes
    Timeout: 30
    Kind: 'ping'
    RetryEnabled: true
    Locations: [
      { Id: 'us-va-ash-azr' }  // East US
      { Id: 'us-ca-sjc-azr' }  // West US
      { Id: 'emea-nl-ams-azr' }  // Europe
    ]
    Configuration: {
      WebTest: '<WebTest><Items><Request Url="https://myapp.azurewebsites.net/health" ThinkTime="0" Timeout="30" /></Items></WebTest>'
    }
  }
}
```

**Alert on availability < 100%:**
```bicep
resource availabilityAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'availability-alert'
  location: 'global'
  properties: {
    severity: 1
    enabled: true
    scopes: [appInsights.id]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'Availability < 100%'
          metricName: 'availabilityResults/availabilityPercentage'
          operator: 'LessThan'
          threshold: 100
          timeAggregation: 'Average'
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
</application_insights>

<log_analytics>
## Log Analytics Workspace

**Purpose:** Centralized log storage and analysis for all Azure resources.

### Create Log Analytics Workspace

```bicep
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'shared-prod-logs'
  location: 'eastus'
  properties: {
    sku: {
      name: 'PerGB2018'  // Pay per GB ingested
    }
    retentionInDays: 90  // 90-730 days
    features: {
      enableLogAccessUsingOnlyResourcePermissions: true
    }
    workspaceCapping: {
      dailyQuotaGb: 10  // Optional: limit ingestion cost
    }
  }
}
```

### Send Logs to Log Analytics

**App Service diagnostic settings:**
```bicep
resource appServiceDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'send-to-log-analytics'
  scope: appService
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'AppServiceHTTPLogs'
        enabled: true
      }
      {
        category: 'AppServiceConsoleLogs'
        enabled: true
      }
      {
        category: 'AppServiceAppLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}
```

**SQL Database diagnostic settings:**
```bicep
resource sqlDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'send-to-log-analytics'
  scope: sqlDatabase
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'QueryStoreRuntimeStatistics'
        enabled: true
      }
      {
        category: 'QueryStoreWaitStatistics'
        enabled: true
      }
      {
        category: 'Errors'
        enabled: true
      }
      {
        category: 'DatabaseWaitStatistics'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'Basic'
        enabled: true
      }
    ]
  }
}
```

### KQL Queries

**HTTP 500 errors in last hour:**
```kql
AppServiceHTTPLogs
| where TimeGenerated > ago(1h)
| where ScStatus >= 500
| summarize count() by ScStatus, CsUriStem
| order by count_ desc
```

**Slow SQL queries (> 1 second):**
```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL"
| where Category == "QueryStoreRuntimeStatistics"
| where avg_duration_s > 1
| project TimeGenerated, query_hash_s, avg_duration_s, execution_count_d
| order by avg_duration_s desc
```

**Resource health:**
```kql
AzureActivity
| where TimeGenerated > ago(24h)
| where CategoryValue == "ResourceHealth"
| summarize count() by ResourceId, Properties
```

### Cost Optimization

**Monitor ingestion:**
```kql
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize IngestedGB = sum(Quantity) / 1000 by bin(TimeGenerated, 1d), DataType
| render timechart
```

**Reduce costs:**
- Set daily cap (workspaceCapping)
- Adjust retention (30 days for non-prod)
- Filter unnecessary logs
- Use sampling for high-volume logs
</log_analytics>

<azure_monitor>
## Azure Monitor

**Purpose:** Unified monitoring platform for metrics, alerts, and dashboards.

### Metrics

**View metrics:**
```bash
# Get App Service CPU percentage
az monitor metrics list \
  --resource <app-service-id> \
  --metric "CpuPercentage" \
  --start-time "2025-01-10T00:00:00Z" \
  --end-time "2025-01-11T00:00:00Z" \
  --interval PT1H \
  --aggregation Average

# Get SQL Database DTU usage
az monitor metrics list \
  --resource <sql-database-id> \
  --metric "dtu_consumption_percent" \
  --aggregation Average,Maximum
```

### Alerts

**Create metric alert:**
```bicep
resource cpuAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'high-cpu-alert'
  location: 'global'
  properties: {
    severity: 2  // 0=Critical, 1=Error, 2=Warning, 3=Info, 4=Verbose
    enabled: true
    scopes: [appServicePlan.id]
    evaluationFrequency: 'PT5M'  // Check every 5 minutes
    windowSize: 'PT15M'  // Over 15-minute window
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'CPU > 80%'
          metricName: 'CpuPercentage'
          operator: 'GreaterThan'
          threshold: 80
          timeAggregation: 'Average'
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

**Log query alert:**
```bicep
resource errorAlert 'Microsoft.Insights/scheduledQueryRules@2023-03-15-preview' = {
  name: 'high-error-rate-alert'
  location: 'eastus'
  properties: {
    displayName: 'High error rate'
    severity: 1
    enabled: true
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    scopes: [logAnalytics.id]
    criteria: {
      allOf: [
        {
          query: '''
            requests
            | where success == false
            | summarize errorCount = count() by bin(timestamp, 5m)
            | where errorCount > 10
          '''
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
        }
      ]
    }
    actions: {
      actionGroups: [actionGroup.id]
    }
  }
}
```

### Action Groups

**Send alerts to multiple channels:**
```bicep
resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'production-alerts'
  location: 'global'
  properties: {
    groupShortName: 'ProdAlerts'
    enabled: true
    emailReceivers: [
      {
        name: 'OnCall Engineer'
        emailAddress: 'oncall@company.com'
        useCommonAlertSchema: true
      }
    ]
    smsReceivers: [
      {
        name: 'OnCall SMS'
        countryCode: '1'
        phoneNumber: '5551234567'
      }
    ]
    webhookReceivers: [
      {
        name: 'Slack'
        serviceUri: 'https://hooks.slack.com/services/...'
        useCommonAlertSchema: true
      }
    ]
    azureFunctionReceivers: [
      {
        name: 'Auto-remediation'
        functionAppResourceId: functionApp.id
        functionName: 'AutoRemediate'
        httpTriggerUrl: 'https://...'
        useCommonAlertSchema: true
      }
    ]
  }
}
```

### Dashboards

**Create shared dashboard:**
```bicep
resource dashboard 'Microsoft.Portal/dashboards@2020-09-01-preview' = {
  name: 'myapp-production-dashboard'
  location: 'eastus'
  properties: {
    lenses: [
      {
        order: 0
        parts: [
          {
            position: { x: 0, y: 0, rowSpan: 4, colSpan: 6 }
            metadata: {
              type: 'Extension/HubsExtension/PartType/MonitorChartPart'
              inputs: [
                {
                  name: 'resourceId'
                  value: appService.id
                }
                {
                  name: 'metricName'
                  value: 'Http2xx'
                }
              ]
            }
          }
        ]
      }
    ]
  }
}
```
</azure_monitor>

<best_practices>
## Monitoring Best Practices

### 1. Instrument Everything

**Application:**
- HTTP requests/responses
- Database queries
- External API calls
- Background jobs
- Custom business events

**Infrastructure:**
- CPU, memory, disk
- Network throughput
- Request count, latency
- Error rates

### 2. Use Structured Logging

**Good:**
```javascript
logger.info('User login', {
  userId: 12345,
  ipAddress: '192.168.1.1',
  userAgent: 'Mozilla/5.0...',
  duration: 125
});
```

**Bad:**
```javascript
logger.info(`User 12345 logged in from 192.168.1.1`);
```

Structured logs are queryable in KQL.

### 3. Set Meaningful Alerts

**Alert on:**
- Availability < 99.9%
- Error rate > 1%
- P95 latency > SLA threshold
- Resource exhaustion (CPU > 80%, Memory > 85%)
- Failed deployments

**Don't alert on:**
- Informational events
- Expected transient failures
- Metrics that don't require action

### 4. Alert Severity Levels

**Sev 0 (Critical):** Production down, immediate page
**Sev 1 (Error):** Significant degradation, page during business hours
**Sev 2 (Warning):** Potential issue, notify via email/Slack
**Sev 3 (Info):** FYI, log only

### 5. Sampling for High Volume

For high-traffic apps, use sampling:

```javascript
appInsights.setup()
    .setSamplingPercentage(10)  // Sample 10% of telemetry
    .start();
```

Still captures all errors and exceptions.

### 6. Custom Dimensions

Add context to telemetry:

```csharp
var telemetry = new RequestTelemetry();
telemetry.Properties.Add("TenantId", "tenant-123");
telemetry.Properties.Add("Region", "us-east");
telemetry.Properties.Add("Version", "2.1.0");
```

Query by dimension:
```kql
requests
| where customDimensions.TenantId == "tenant-123"
| summarize count() by bin(timestamp, 1h)
```

### 7. Distributed Tracing

Enable correlation:

```javascript
// Node.js
appInsights.setup()
    .setAutoDependencyCorrelation(true)
    .start();
```

Traces entire request flow across microservices.

### 8. Regular Review

**Weekly:** Review dashboard, check for anomalies
**Monthly:** Review alert effectiveness, tune thresholds
**Quarterly:** Review retention, cost, unused resources
</best_practices>

<anti_patterns>
**Avoid:**

<anti_pattern name="No Monitoring">
**Problem:** Deploying without Application Insights or logging

**Why bad:** Blind to issues, slow incident response, no performance data

**Instead:** Set up Application Insights from day one
</anti_pattern>

<anti_pattern name="Alert Fatigue">
**Problem:** Too many alerts, mostly false positives

**Why bad:** Important alerts ignored, on-call burnout

**Instead:** Tune thresholds, aggregate similar alerts, actionable only
</anti_pattern>

<anti_pattern name="Logging Sensitive Data">
**Problem:** Logging passwords, tokens, PII

**Why bad:** Security breach, compliance violation

**Instead:** Sanitize logs, use correlation IDs instead of PII
</anti_pattern>

<anti_pattern name="No Runbooks">
**Problem:** Alerts without guidance on how to respond

**Why bad:** Slow resolution, inconsistent response

**Instead:** Link alerts to runbooks with investigation steps
</anti_pattern>

<anti_pattern name="Ignoring Costs">
**Problem:** Unlimited log ingestion, no retention policy

**Why bad:** High costs, wasted data

**Instead:** Set daily cap, appropriate retention, sampling
</anti_pattern>
</anti_patterns>
