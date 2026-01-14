<required_reading>
Before starting, read these references to understand monitoring architecture:
- `references/monitoring-observability.md` - Application Insights, Log Analytics, alerts
- `references/cost-optimization.md` - Data retention and monitoring costs
</required_reading>

<objective>
Configure comprehensive monitoring, logging, and alerting for Azure applications using Application Insights, Log Analytics, and Azure Monitor.
</objective>

<quick_start>
**5-minute setup:**
```bash
# Create Application Insights
az monitor app-insights component create \
  --app ai-myapp-prod-eastus \
  --location eastus \
  --resource-group myapp-prod-rg \
  --application-type web

# Get instrumentation key
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query instrumentationKey -o tsv)

# Configure App Service to use Application Insights
az webapp config appsettings set \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY=$INSTRUMENTATION_KEY \
             APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=$INSTRUMENTATION_KEY"

# Create basic alert
az monitor metrics alert create \
  --name alert-high-response-time \
  --resource-group myapp-prod-rg \
  --scopes $(az webapp show --name app-myapp-prod-eastus --resource-group myapp-prod-rg --query id -o tsv) \
  --condition "avg requests/duration > 1000" \
  --description "Alert when average response time exceeds 1 second"
```
</quick_start>

<process>
## Step 1: Create Log Analytics Workspace

**Purpose:** Central repository for all logs.

```bicep
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'la-shared-prod-eastus'
  location: 'eastus'
  properties: {
    sku: {
      name: 'PerGB2018'  // Pay per GB ingested
    }
    retentionInDays: 30  // 30-90 days for prod, 7 for dev
  }
}
```

**Cost optimization:**
- Dev: 7 days retention
- Staging: 30 days retention
- Production: 90 days retention
- Archive to Storage Account for long-term retention

## Step 2: Create Application Insights

**Purpose:** Application-level telemetry (requests, dependencies, exceptions).

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'ai-myapp-prod-eastus'
  location: 'eastus'
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id  // Link to Log Analytics
    IngestionMode: 'LogAnalytics'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

output instrumentationKey string = appInsights.properties.InstrumentationKey
output connectionString string = appInsights.properties.ConnectionString
```

## Step 3: Configure Application to Send Telemetry

**App Service:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'ApplicationInsightsAgent_EXTENSION_VERSION'
          value: '~3'  // Enable auto-instrumentation
        }
      ]
    }
  }
}
```

**Azure Functions:**
```bicep
resource functionApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-processor-prod-eastus'
  kind: 'functionapp,linux'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'node'
        }
      ]
    }
  }
}
```

**Custom application code (.NET):**
```csharp
// Install NuGet: Microsoft.ApplicationInsights.AspNetCore
builder.Services.AddApplicationInsightsTelemetry();
```

**Custom application code (Node.js):**
```javascript
// npm install applicationinsights
const appInsights = require('applicationinsights');
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING).start();
```

## Step 4: Configure Diagnostic Settings

**Purpose:** Send resource-level logs to Log Analytics.

```bicep
// App Service diagnostic settings
resource appServiceDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'diag-app-myapp-prod'
  scope: appService
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'AppServiceHTTPLogs'
        enabled: true
        retentionPolicy: { enabled: false }  // Use workspace retention
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

// SQL Database diagnostic settings
resource sqlDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'diag-sql-myapp-prod'
  scope: sqlDatabase
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        category: 'SQLInsights'
        enabled: true
      }
      {
        category: 'QueryStoreRuntimeStatistics'
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

## Step 5: Create Alerts

**5.1 Response time alert:**
```bicep
resource responseTimeAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-response-time-myapp-prod'
  location: 'global'
  properties: {
    severity: 2  // 0=Critical, 1=Error, 2=Warning, 3=Info
    enabled: true
    scopes: [appInsights.id]
    evaluationFrequency: 'PT1M'  // Check every 1 minute
    windowSize: 'PT5M'  // Evaluate over 5 minutes
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighResponseTime'
          metricName: 'requests/duration'
          operator: 'GreaterThan'
          threshold: 1000  // 1 second
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

**5.2 Error rate alert:**
```bicep
resource errorRateAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'alert-error-rate-myapp-prod'
  location: 'global'
  properties: {
    severity: 1  // Error
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
          threshold: 10  // More than 10 failed requests
          timeAggregation: 'Count'
        }
      ]
    }
    actions: [{ actionGroupId: actionGroup.id }]
  }
}
```

**5.3 Availability alert:**
```bicep
resource availabilityTest 'Microsoft.Insights/webtests@2022-06-15' = {
  name: 'availtest-myapp-prod'
  location: 'eastus'
  kind: 'ping'
  properties: {
    Name: 'Home Page Availability'
    Enabled: true
    Frequency: 300  // Every 5 minutes
    Timeout: 30
    Locations: [
      { Id: 'us-va-ash-azr' }  // East US
      { Id: 'us-ca-sjc-azr' }  // West US
      { Id: 'emea-nl-ams-azr' }  // Netherlands
    ]
    Configuration: {
      WebTest: '<WebTest><Items><Request Url="https://app-myapp-prod.azurewebsites.net" /></Items></WebTest>'
    }
    SyntheticMonitorId: 'availtest-myapp-prod'
  }
  tags: {
    'hidden-link:${appInsights.id}': 'Resource'
  }
}
```

**5.4 Action Group (notification target):**
```bicep
resource actionGroup 'Microsoft.Insights/actionGroups@2023-09-01-preview' = {
  name: 'ag-oncall-prod'
  location: 'global'
  properties: {
    groupShortName: 'OnCall'
    enabled: true
    emailReceivers: [
      {
        name: 'EmailOncall'
        emailAddress: 'oncall@company.com'
        useCommonAlertSchema: true
      }
    ]
    smsReceivers: [
      {
        name: 'SMSOncall'
        countryCode: '1'
        phoneNumber: '5551234567'
      }
    ]
    webhookReceivers: [
      {
        name: 'PagerDuty'
        serviceUri: 'https://events.pagerduty.com/integration/...'
        useCommonAlertSchema: true
      }
    ]
  }
}
```

## Step 6: Create Dashboards

**KQL queries for common scenarios:**

**6.1 Response time percentiles:**
```kql
requests
| where timestamp > ago(24h)
| summarize
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
    by bin(timestamp, 5m)
| render timechart
```

**6.2 Failed requests:**
```kql
requests
| where timestamp > ago(24h) and success == false
| summarize count() by resultCode, name
| order by count_ desc
```

**6.3 Dependency failures:**
```kql
dependencies
| where timestamp > ago(24h) and success == false
| summarize count() by target, type
| order by count_ desc
```

**6.4 Exceptions:**
```kql
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc
```

**Create dashboard in portal or via CLI:**
```bash
# Export existing dashboard
az portal dashboard show \
  --name myapp-dashboard \
  --resource-group myapp-prod-rg > dashboard.json

# Import dashboard
az portal dashboard create \
  --name myapp-prod-dashboard \
  --resource-group myapp-prod-rg \
  --input-path dashboard.json
```

## Step 7: Verify Monitoring

```bash
# Check Application Insights is receiving data
az monitor app-insights metrics show \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --metric requests/count \
  --interval PT1M

# Query recent exceptions
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "exceptions | where timestamp > ago(1h) | take 10"

# Check alert rules
az monitor metrics alert list \
  --resource-group myapp-prod-rg \
  --output table

# Test alert (simulate high response time)
# Generate load on app to trigger alert
```
</process>

<monitoring_checklist>
## Production Monitoring Checklist

- [ ] Application Insights configured
- [ ] Log Analytics workspace created
- [ ] Diagnostic settings enabled for all resources
- [ ] Alerts configured:
  - [ ] Response time > 1 second
  - [ ] Error rate > 5%
  - [ ] Availability < 99%
  - [ ] CPU > 80%
  - [ ] Memory > 80%
- [ ] Action Group configured with:
  - [ ] Email notifications
  - [ ] SMS for critical alerts
  - [ ] Integration with incident management (PagerDuty, Opsgenie)
- [ ] Availability tests from multiple regions
- [ ] Dashboard created for key metrics
- [ ] Log retention configured appropriately
- [ ] Budget alert for monitoring costs
</monitoring_checklist>

<cost_estimation>
**Application Insights (production):**
- First 5 GB/month: Free
- Additional data: $2.30/GB
- Typical small app: 10 GB/month = $11.50/month
- Typical medium app: 50 GB/month = $103.50/month

**Log Analytics:**
- Pay-per-GB: $2.99/GB (first 5 GB free)
- Commitment tiers available (100 GB/day = $196/month, saves 50%)

**Data retention:**
- First 31 days: Free
- 31-730 days: $0.12/GB/month

**Cost optimization:**
- Use sampling (reduce telemetry volume by 50-90%)
- Set retention to 30 days for dev, 90 for prod
- Archive old data to Storage Account ($0.018/GB/month)
</cost_estimation>

<success_criteria>
You've successfully set up monitoring when:
- Application Insights shows real-time requests, dependencies, and exceptions
- Log Analytics contains logs from all Azure resources
- Alerts trigger appropriately (test by simulating failures)
- Dashboard displays key business and technical metrics
- On-call team receives notifications through preferred channels
- Correlation IDs allow tracing requests across services
- Query response time is acceptable (< 10 seconds for common queries)
</success_criteria>
