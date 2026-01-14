<required_reading>
Before debugging, review these patterns:
- `references/monitoring-observability.md` - Using logs and metrics for troubleshooting
- `references/deployment-strategies.md` - Common deployment issues and rollback
- `references/anti-patterns.md` - Common mistakes to look for
</required_reading>

<objective>
Systematic troubleshooting methodology for failed Azure deployments, broken infrastructure, and application errors using Azure CLI, logs, and monitoring tools.
</objective>

<quick_diagnostic>
**Immediate status check (run first):**
```bash
# Resource group deployment status
az deployment group list \
  --resource-group myapp-prod-rg \
  --query "[].{Name:name, State:properties.provisioningState, Timestamp:properties.timestamp}" \
  --output table

# Failed deployment details
DEPLOYMENT_NAME=$(az deployment group list \
  --resource-group myapp-prod-rg \
  --query "[?properties.provisioningState=='Failed'].name | [0]" -o tsv)

az deployment group show \
  --name $DEPLOYMENT_NAME \
  --resource-group myapp-prod-rg \
  --query "properties.error"

# App Service status
az webapp show \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query "{Name:name, State:state, AvailabilityState:availabilityState}"

# Recent activity log (errors)
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --offset 1h \
  --query "[?level=='Error'].{Time:eventTimestamp, Resource:resourceId, Status:status.value}" \
  --output table
```
</quick_diagnostic>

<process>
## Debugging Methodology

### Step 1: Identify Failure Type

**Infrastructure deployment failure:**
- Bicep/Terraform/ARM template failed
- Resource provisioning error
- Quota or limit exceeded
- RBAC permission denied

**Application deployment failure:**
- Code deployment succeeded but app won't start
- Health check failed
- Configuration error
- Dependency missing

**Runtime failure:**
- App deployed successfully but returns errors
- Intermittent failures
- Performance degradation
- External dependency failure

### Step 2: Infrastructure Deployment Failures

**2.1 Get detailed error:**
```bash
# Show last deployment error
az deployment group show \
  --name $DEPLOYMENT_NAME \
  --resource-group myapp-prod-rg \
  --query "properties.error.{Code:code, Message:message, Details:details}"

# Terraform errors
terraform apply 2>&1 | tee deployment-error.log

# Review operations
az deployment operation group list \
  --name $DEPLOYMENT_NAME \
  --resource-group myapp-prod-rg \
  --query "[?properties.provisioningState=='Failed'].{Resource:properties.targetResource.resourceName, Error:properties.statusMessage.error.message}"
```

**2.2 Common infrastructure errors:**

<error type="QuotaExceeded">
**Error:** `Operation results in exceeding quota limits of cores`

**Fix:**
```bash
# Check current quota
az vm list-usage --location eastus \
  --query "[?name.value=='cores'].{Name:name.localizedValue, Current:currentValue, Limit:limit}" \
  --output table

# Request quota increase
az support tickets create \
  --ticket-name "quota-increase-cores" \
  --severity "minimal" \
  --issue-type "quota" \
  --description "Need 20 additional cores in East US"
```
</error>

<error type="ResourceNotFound">
**Error:** `The Resource 'Microsoft.Web/serverfarms/asp-myapp-prod' under resource group 'myapp-prod-rg' was not found.`

**Cause:** Dependency created in wrong order or doesn't exist

**Fix:**
```bicep
// Ensure dependencies are explicit
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  dependsOn: [
    appServicePlan  // Explicit dependency
  ]
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```
</error>

<error type="AuthorizationFailed">
**Error:** `The client '...' with object id '...' does not have authorization to perform action 'Microsoft.Sql/servers/write'`

**Fix:**
```bash
# Check current permissions
az role assignment list \
  --assignee user@company.com \
  --resource-group myapp-prod-rg \
  --output table

# Grant required role
az role assignment create \
  --assignee user@company.com \
  --role "SQL Server Contributor" \
  --resource-group myapp-prod-rg
```
</error>

<error type="NameAlreadyExists">
**Error:** `Storage account 'stmyappprodeastus' is already taken`

**Cause:** Storage account names are globally unique

**Fix:**
```bicep
// Add unique suffix
param uniqueSuffix string = uniqueString(resourceGroup().id)

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stmyappprod${uniqueSuffix}'  // Globally unique
  location: 'eastus'
  // ...
}
```
</error>

### Step 3: Application Deployment Failures

**3.1 Check deployment logs:**
```bash
# App Service deployment logs
az webapp log tail \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg

# Download full logs
az webapp log download \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --log-file app-logs.zip

# Container logs (if containerized)
az webapp log config \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --docker-container-logging filesystem

az webapp log tail --name app-myapp-prod-eastus --resource-group myapp-prod-rg
```

**3.2 Common application errors:**

<error type="ApplicationStartupFailed">
**Error:** App deployed but returns 503 Service Unavailable

**Diagnostics:**
```bash
# Check application logs
az webapp log show \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg

# SSH into container (Linux App Service)
az webapp ssh \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg

# Check startup command
az webapp config show \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query "{StartupCommand:linuxFxVersion, AppCommand:appCommandLine}"
```

**Common causes:**
- Missing environment variables
- Incorrect startup command
- Port not matching WEBSITES_PORT
- Package dependencies not installed

**Fix:**
```bash
# Set required environment variables
az webapp config appsettings set \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --settings DATABASE_URL="..." API_KEY="..." PORT=8000

# Fix startup command (Node.js example)
az webapp config set \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --startup-file "npm start"
```
</error>

<error type="DependencyFailed">
**Error:** App starts but crashes immediately

**Diagnostics:**
```bash
# Check recent exceptions in Application Insights
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "exceptions | where timestamp > ago(1h) | project timestamp, type, outerMessage, innermostMessage | take 20"

# Check dependency calls
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "dependencies | where timestamp > ago(1h) and success == false | project timestamp, name, target, resultCode | take 20"
```

**Common causes:**
- Database connection string incorrect
- SQL Database firewall blocking App Service
- Key Vault access denied
- External API unreachable

**Fix for SQL firewall:**
```bash
# Allow Azure services to access SQL
az sql server firewall-rule create \
  --resource-group myapp-prod-rg \
  --server sql-myapp-prod-eastus \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Better: Use Private Endpoint (no public access)
# See references/networking.md for Private Endpoint setup
```

**Fix for Key Vault access:**
```bash
# Grant Managed Identity access to Key Vault
APP_PRINCIPAL_ID=$(az webapp identity show \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query principalId -o tsv)

az role assignment create \
  --assignee $APP_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name kv-myapp-prod-eastus --query id -o tsv)
```
</error>

### Step 4: Runtime Failures

**4.1 Intermittent 5xx errors:**
```bash
# Check error rate in Application Insights
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "requests | where timestamp > ago(1h) | summarize TotalRequests=count(), FailedRequests=countif(success==false), ErrorRate=100.0*countif(success==false)/count() by bin(timestamp, 5m) | render timechart"

# Check specific error codes
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "requests | where timestamp > ago(1h) and success == false | summarize count() by resultCode, name | order by count_ desc"
```

**Common causes:**
- **502 Bad Gateway:** App Service instance unhealthy, crashed, or restarting
- **503 Service Unavailable:** App Service overloaded, auto-scaling not fast enough
- **504 Gateway Timeout:** Request taking >230 seconds (App Service timeout)

**Fix for 502:**
```bash
# Enable health check
az webapp config set \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --health-check-path "/health"

# Increase instance count
az appservice plan update \
  --name asp-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --number-of-workers 3

# Review app restart history
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --offset 24h \
  --query "[?contains(resourceId, 'app-myapp-prod')].{Time:eventTimestamp, Operation:operationName.value, Status:status.value}" \
  --output table
```

**4.2 Performance degradation:**
```bash
# Check CPU and memory metrics
az monitor metrics list \
  --resource $(az webapp show --name app-myapp-prod-eastus --resource-group myapp-prod-rg --query id -o tsv) \
  --metric "CpuPercentage,MemoryPercentage" \
  --start-time 2025-01-11T00:00:00Z \
  --end-time 2025-01-11T23:59:59Z \
  --interval PT1M \
  --aggregation Average

# Check slow requests
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "requests | where timestamp > ago(1h) and duration > 5000 | project timestamp, name, url, duration, resultCode | order by duration desc | take 20"

# Check slow database queries
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "dependencies | where timestamp > ago(1h) and type == 'SQL' and duration > 1000 | project timestamp, name, data, duration | order by duration desc | take 20"
```

**Fix:**
```bash
# Scale up (more powerful instances)
az appservice plan update \
  --name asp-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --sku P2v3

# Scale out (more instances)
az appservice plan update \
  --name asp-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --number-of-workers 5

# Enable auto-scale
az monitor autoscale create \
  --resource-group myapp-prod-rg \
  --resource $(az appservice plan show --name asp-myapp-prod-eastus --resource-group myapp-prod-rg --query id -o tsv) \
  --min-count 2 \
  --max-count 10 \
  --count 2

az monitor autoscale rule create \
  --resource-group myapp-prod-rg \
  --autoscale-name autoscale-asp-myapp-prod \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1
```

### Step 5: Rollback Strategies

**5.1 App Service deployment rollback:**
```bash
# List deployment history
az webapp deployment list \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --output table

# Rollback to previous deployment
PREVIOUS_DEPLOYMENT_ID=$(az webapp deployment list \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query "[1].id" -o tsv)

az webapp deployment source delete \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --ids $PREVIOUS_DEPLOYMENT_ID

# Or use deployment slots
az webapp deployment slot swap \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --slot staging \
  --target-slot production \
  --action swap  # Swap back to previous version
```

**5.2 Infrastructure rollback:**
```bash
# Bicep: Redeploy previous version
git checkout <previous-commit>
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --parameters main.parameters.prod.json

# Terraform: Rollback state
terraform state pull > backup.tfstate
terraform apply -auto-approve  # Apply previous config
```
</process>

<debugging_checklist>
## Systematic Debugging Steps

1. [ ] Get deployment status (success/failed)
2. [ ] Check recent activity log for errors
3. [ ] Review deployment operation details
4. [ ] Check application logs (last 100 lines)
5. [ ] Verify environment variables and configuration
6. [ ] Test external dependencies (database, API, Key Vault)
7. [ ] Check resource metrics (CPU, memory, network)
8. [ ] Review Application Insights for exceptions
9. [ ] Verify RBAC permissions
10. [ ] Check quotas and limits
11. [ ] Review recent changes (code, config, infrastructure)
12. [ ] Test in isolation (local, staging slot)
</debugging_checklist>

<tools_reference>
## Essential Debugging Commands

```bash
# Resource status
az resource list --resource-group <rg> --output table
az webapp show --name <app> --resource-group <rg> --query state

# Logs
az webapp log tail --name <app> --resource-group <rg>
az monitor activity-log list --resource-group <rg> --offset 1h

# Metrics
az monitor metrics list --resource <resource-id> --metric "CpuPercentage,MemoryPercentage"

# Application Insights queries
az monitor app-insights query --app <ai-name> --resource-group <rg> --analytics-query "<KQL>"

# Deployments
az deployment group list --resource-group <rg>
az deployment group show --name <deployment> --resource-group <rg>

# Configuration
az webapp config show --name <app> --resource-group <rg>
az webapp config appsettings list --name <app> --resource-group <rg>

# SSH/Console
az webapp ssh --name <app> --resource-group <rg>
```
</tools_reference>

<success_criteria>
Debugging is complete when:
- Root cause identified and documented
- Issue resolved or workaround implemented
- Application health check returns 200 OK
- Error rate returns to baseline (< 1%)
- Monitoring confirms stability over 30 minutes
- Rollback plan documented for future
- Post-mortem written (for production incidents)
</success_criteria>
