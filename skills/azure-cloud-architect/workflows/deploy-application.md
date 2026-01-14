# Workflow: Deploy Application

<required_reading>
**Read these reference files NOW before deploying:**
1. references/compute-services.md
2. references/deployment-strategies.md
3. references/monitoring-observability.md
</required_reading>

<process>
## Step 1: Determine Target Service

Ask user what type of application:
- **Web app/API** → App Service or AKS
- **Serverless/Functions** → Azure Functions
- **Container app** → Container Apps or AKS
- **Static site** → Static Web Apps

Choose appropriate service based on references/compute-services.md.

## Step 2: Prepare Application

Ensure application has:
- Health endpoint (`/health` or `/healthz`)
- Configuration via environment variables (no hardcoded values)
- Logging to stdout/stderr
- Application Insights integration

## Step 3: Deploy Based on Service Type

### App Service:

```bash
# Using Azure CLI
az webapp deployment source config-zip \
  --resource-group <rg-name> \
  --name <app-name> \
  --src app.zip

# Using deployment slot
az webapp deployment source config-zip \
  --resource-group <rg-name> \
  --name <app-name> \
  --slot staging \
  --src app.zip

# Test staging
curl https://<app-name>-staging.azurewebsites.net/health

# Swap to production
az webapp deployment slot swap \
  --resource-group <rg-name> \
  --name <app-name> \
  --slot staging \
  --target-slot production
```

### Azure Functions:

```bash
# Deploy function app
func azure functionapp publish <function-app-name>

# Or using Azure CLI
az functionapp deployment source config-zip \
  --resource-group <rg-name> \
  --name <function-app-name> \
  --src function-app.zip
```

### AKS (Kubernetes):

```bash
# Get credentials
az aks get-credentials \
  --resource-group <rg-name> \
  --name <aks-name>

# Deploy application
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Check rollout status
kubectl rollout status deployment/<app-name>

# Verify pods running
kubectl get pods -l app=<app-name>
```

## Step 4: Configure Application Settings

```bash
# Set environment variables
az webapp config appsettings set \
  --resource-group <rg-name> \
  --name <app-name> \
  --settings \
    NODE_ENV=production \
    DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://<vault>.vault.azure.net/secrets/db-connection/)" \
    APPINSIGHTS_INSTRUMENTATIONKEY="<key>"
```

## Step 5: Verify Deployment

```bash
# Check app status
az webapp show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query "state" -o tsv

# Test health endpoint
curl -f https://<app-name>.azurewebsites.net/health || exit 1

# Check logs
az webapp log tail \
  --resource-group <rg-name> \
  --name <app-name>

# Check Application Insights
az monitor app-insights metrics show \
  --app <app-insights-name> \
  --resource-group <rg-name> \
  --metric "requests/count" \
  --start-time "2025-01-01T00:00:00Z"
```

## Step 6: Monitor Post-Deployment

Monitor for 15-30 minutes:
- Error rates in Application Insights
- Response times
- CPU/Memory usage
- Failed requests

If issues detected, rollback immediately.
</process>

<anti_patterns>
**Avoid:**
- Deploying directly to production without staging slot
- No health endpoint
- Hardcoded configuration values
- Not testing after deployment
- No rollback plan
- Deploying during peak hours
</anti_patterns>

<success_criteria>
Successful deployment has:
- Application deployed to staging first
- Health checks passing
- Swapped to production with zero downtime
- Application Insights receiving telemetry
- No error spikes in monitoring
- Rollback procedure documented and tested
</success_criteria>
