<overview>
Comprehensive guide to deployment strategies on Azure. Covers blue-green, canary, rolling updates, deployment slots, traffic management, and zero-downtime deployment patterns for 2024-2025.
</overview>

<fundamental_strategies>
## Core Deployment Strategies

<strategy name="Blue-Green Deployment">
**Concept:** Two identical environments (blue=current, green=new). Switch traffic atomically.

**When to use:**
- Need instant rollback capability
- Can afford duplicate infrastructure
- Critical applications requiring zero downtime

**Azure implementation:** App Service deployment slots

**Pros:**
- ✅ Instant rollback (swap back)
- ✅ Test in production environment
- ✅ Zero downtime
- ✅ Simple to understand

**Cons:**
- ❌ 2x infrastructure cost (temporarily)
- ❌ Database migrations complex
- ❌ All traffic switches at once

**Flow:**
```
1. Blue (production) serving traffic
2. Deploy to Green (staging slot)
3. Test Green thoroughly
4. Swap Green → Production
5. Monitor for issues
6. Keep Blue as rollback (or tear down after validation)
```
</strategy>

<strategy name="Canary Deployment">
**Concept:** Gradually route small percentage of traffic to new version while monitoring.

**When to use:**
- Want gradual rollout
- Need to test with real production traffic
- Risk-averse deployments

**Azure implementation:** Traffic Manager, App Service traffic routing, AKS Ingress

**Pros:**
- ✅ Limit blast radius (only 10% affected)
- ✅ Early warning of issues
- ✅ Progressive confidence building

**Cons:**
- ❌ More complex monitoring
- ❌ Requires traffic routing capability
- ❌ Longer deployment time

**Flow:**
```
1. Deploy new version to subset of instances
2. Route 10% traffic → new version
3. Monitor metrics (errors, latency)
4. If good: 25% → 50% → 100%
5. If bad: Route back to 0%, rollback
```
</strategy>

<strategy name="Rolling Update">
**Concept:** Update instances one-by-one or in batches.

**When to use:**
- Limited resources (can't double infrastructure)
- Stateless applications
- K8s deployments

**Azure implementation:** AKS rolling updates, VMSS rolling upgrades

**Pros:**
- ✅ No duplicate infrastructure needed
- ✅ Gradual rollout
- ✅ Built into Kubernetes

**Cons:**
- ❌ Both versions running simultaneously
- ❌ Slower deployment
- ❌ Can't instant rollback (need to roll forward)

**Flow:**
```
1. Update 1 pod/instance
2. Wait for health check
3. Update next pod/instance
4. Repeat until all updated
```
</strategy>

<strategy name="Recreate">
**Concept:** Stop all old instances, deploy all new instances.

**When to use:**
- Development/testing environments
- Downtime acceptable
- Breaking changes between versions

**Pros:**
- ✅ Simple
- ✅ No mixed versions
- ✅ No additional resources

**Cons:**
- ❌ Downtime during deployment
- ❌ Not suitable for production

**Use only for non-production.**
</strategy>
</fundamental_strategies>

<app_service_slots>
## App Service Deployment Slots (Blue-Green)

**How it works:** Each App Service can have multiple slots (staging, canary, etc.). Swap slots to switch traffic.

### Create Deployment Slots

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
  }
}

// Staging slot
resource stagingSlot 'Microsoft.Web/sites/slots@2023-12-01' = {
  parent: appService
  name: 'staging'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      appSettings: [
        {
          name: 'ENVIRONMENT'
          value: 'staging'
        }
      ]
    }
  }
}

// Canary slot (optional)
resource canarySlot 'Microsoft.Web/sites/slots@2023-12-01' = {
  parent: appService
  name: 'canary'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```

### Blue-Green Deployment Process

**Step 1: Deploy to staging**
```bash
# Deploy new version to staging slot
az webapp deployment source config-zip \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --slot staging \
  --src app.zip
```

**Step 2: Test staging slot**
```bash
# Staging URL: https://myapp-prod-app-staging.azurewebsites.net

# Health check
curl https://myapp-prod-app-staging.azurewebsites.net/health

# Run smoke tests
npm run test:e2e:staging

# Manual testing in browser
```

**Step 3: Swap to production**
```bash
# Atomic swap (zero downtime)
az webapp deployment slot swap \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --slot staging \
  --target-slot production

# Now staging has old version (for rollback)
# Production has new version
```

**Step 4: Verify production**
```bash
# Test production
curl https://myapp-prod-app.azurewebsites.net/health

# Monitor Application Insights for errors
az monitor app-insights metrics show \
  --app myapp-prod-insights \
  --resource-group myapp-prod-rg \
  --metric "requests/failed" \
  --start-time "5 minutes ago"
```

**Step 5: Rollback if needed**
```bash
# If issues detected, swap back immediately
az webapp deployment slot swap \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --slot production \
  --target-slot staging

# Back to previous version in seconds
```

### Slot Settings

**Slot-specific settings** (don't swap):
```bash
# Mark setting as slot-specific (doesn't swap)
az webapp config appsettings set \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --slot staging \
  --settings ENVIRONMENT=staging \
  --slot-settings ENVIRONMENT

# ENVIRONMENT stays "staging" even after swap
```

**Swap-with settings** (do swap):
- Connection strings
- App settings (unless marked slot-specific)
- Configuration
- Certificates

### Auto-Swap

```bicep
resource stagingSlot 'Microsoft.Web/sites/slots@2023-12-01' = {
  parent: appService
  name: 'staging'
  properties: {
    siteConfig: {
      autoSwapSlotName: 'production'  // Auto-swap after deployment
    }
  }
}
```

**Use with caution:** Only for trusted CI/CD pipelines with good tests.
</app_service_slots>

<canary_deployments>
## Canary Deployments

### Method 1: App Service Traffic Routing

**Route percentage of traffic to canary slot:**

```bash
# Route 10% to canary, 90% to production
az webapp traffic-routing set \
  --resource-group myapp-prod-rg \
  --name myapp-prod-app \
  --distribution canary=10

# Monitor canary metrics
# If good, increase gradually
az webapp traffic-routing set \
  --distribution canary=25

az webapp traffic-routing set \
  --distribution canary=50

# Full canary (100%)
az webapp traffic-routing set \
  --distribution canary=100

# Then swap slots (canary becomes production)
az webapp deployment slot swap \
  --slot canary \
  --target-slot production
```

**Monitor canary:**
```kql
// Application Insights query
requests
| where cloud_RoleName == "canary"
| where timestamp > ago(15m)
| summarize
    requests = count(),
    failures = countif(success == false),
    avgDuration = avg(duration)
| extend errorRate = (failures * 100.0) / requests
```

**Automated canary with thresholds:**
```bash
#!/bin/bash
# canary-deploy.sh

deploy_canary() {
  az webapp deployment source config-zip --slot canary --src app.zip
}

test_canary() {
  local percentage=$1

  # Route traffic
  az webapp traffic-routing set --distribution canary=$percentage

  # Wait for metrics
  sleep 300  # 5 minutes

  # Check error rate
  ERROR_RATE=$(az monitor app-insights query \
    --app myapp-prod-insights \
    --analytics-query "requests | where cloud_RoleName == 'canary' | where timestamp > ago(5m) | summarize errorRate = (countif(success==false) * 100.0) / count()" \
    --query "tables[0].rows[0][0]" -o tsv)

  if (( $(echo "$ERROR_RATE > 1" | bc -l) )); then
    echo "Error rate too high: $ERROR_RATE%"
    return 1
  fi

  return 0
}

# Deploy to canary
deploy_canary

# Progressive rollout
for percentage in 10 25 50 100; do
  echo "Testing $percentage% traffic..."
  if test_canary $percentage; then
    echo "✓ $percentage% successful"
  else
    echo "✗ Canary failed at $percentage%, rolling back"
    az webapp traffic-routing clear
    exit 1
  fi
done

# Success - swap to production
az webapp deployment slot swap --slot canary --target-slot production
echo "✓ Deployment complete"
```

### Method 2: AKS Canary with Ingress

**Create canary deployment:**
```yaml
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 10% of total (if 10 total pods)
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: app
        image: myregistry.azurecr.io/app:v2.0.0
```

**Ingress with weighted routing:**
```yaml
# nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% to canary
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

**Increase canary weight:**
```bash
kubectl patch ingress myapp-ingress -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"50"}}}'
```
</canary_deployments>

<rolling_updates>
## Rolling Updates (Kubernetes)

**Deployment strategy:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max 2 extra pods during update
      maxUnavailable: 1  # Max 1 pod unavailable during update
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myregistry.azurecr.io/app:v2.0.0
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
```

**How it works:**
1. Create 2 new pods (maxSurge=2)
2. Wait for readiness probe
3. Terminate 1 old pod
4. Create 1 new pod
5. Repeat until all updated

**Monitor rollout:**
```bash
# Watch rollout
kubectl rollout status deployment/myapp

# Pause rollout
kubectl rollout pause deployment/myapp

# Resume rollout
kubectl rollout resume deployment/myapp

# Rollback
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2
```
</rolling_updates>

<database_migrations>
## Database Migrations During Deployment

**Challenge:** Database schema changes during blue-green deployments.

### Forward-Compatible Migrations

**Bad approach:**
```sql
-- NEVER do this during deployment
ALTER TABLE Users DROP COLUMN OldColumn;  -- Breaks old version!
```

**Good approach (backward compatible):**
```sql
-- Deployment 1: Add new column
ALTER TABLE Users ADD COLUMN NewColumn VARCHAR(100);

-- Code supports both columns during transition
-- App version N reads OldColumn, writes both
-- App version N+1 reads NewColumn

-- Deployment 2 (after old version gone): Remove old column
ALTER TABLE Users DROP COLUMN OldColumn;
```

### Expand-Contract Pattern

**Phase 1: Expand (add new)**
- Add new table/column
- Old code ignores it
- New code uses it

**Phase 2: Migrate data**
- Background job copies data
- Dual-write if needed

**Phase 3: Contract (remove old)**
- Remove old table/column
- Only after all code updated

### Database Deployment Strategies

**Separate deployment:**
```yaml
# deploy-pipeline.yml
stages:
  - stage: Database
    jobs:
      - job: RunMigrations
        steps:
          - script: |
              npx db-migrate up
            displayName: 'Run database migrations'

  - stage: Application
    dependsOn: Database  # Run after DB migrations
    jobs:
      - job: DeployApp
        steps:
          - script: |
              az webapp deployment slot swap
```

**Feature flags for breaking changes:**
```javascript
if (featureFlags.useNewColumn) {
  // Use NewColumn
} else {
  // Use OldColumn (backward compatibility)
}
```
</database_migrations>

<zero_downtime_checklist>
## Zero-Downtime Deployment Checklist

**Before deployment:**
- [ ] Health endpoint implemented (`/health`)
- [ ] Readiness probe configured
- [ ] Liveness probe configured
- [ ] Database migrations backward compatible
- [ ] Graceful shutdown implemented
- [ ] Connection draining configured

**During deployment:**
- [ ] Deploy to staging/canary first
- [ ] Run smoke tests
- [ ] Monitor error rates
- [ ] Monitor response times
- [ ] Check logs for exceptions

**After deployment:**
- [ ] Verify health endpoint
- [ ] Monitor Application Insights (15 min)
- [ ] Check error rate vs baseline
- [ ] Keep old version for rollback
- [ ] Document deployment (version, time, who)

**Rollback plan:**
- [ ] Documented rollback procedure
- [ ] Tested rollback process
- [ ] Can rollback in < 5 minutes
- [ ] Rollback doesn't require DB changes
</zero_downtime_checklist>

<anti_patterns>
**Avoid:**

<anti_pattern name="No Staging Environment">
**Problem:** Deploy directly to production

**Why bad:** No testing in prod-like environment, higher risk

**Instead:** Always deploy to staging first, test, then production
</anti_pattern>

<anti_pattern name="All Traffic at Once">
**Problem:** Switch 100% traffic immediately to new version

**Why bad:** If bug exists, 100% of users affected

**Instead:** Use canary deployment, gradual rollout
</anti_pattern>

<anti_pattern name="No Rollback Plan">
**Problem:** Can't quickly revert if issues found

**Why bad:** Extended downtime, manual recovery

**Instead:** Keep previous version in staging slot, test rollback procedure
</anti_pattern>

<anti_pattern name="Breaking Database Changes">
**Problem:** Incompatible DB schema changes during deployment

**Why bad:** Old version crashes, requires coordination

**Instead:** Use expand-contract pattern, forward-compatible migrations
</anti_pattern>

<anti_pattern name="No Health Checks">
**Problem:** Route traffic before app ready

**Why bad:** 500 errors, failed requests

**Instead:** Implement /health endpoint, configure readiness probes
</anti_pattern>
</anti_patterns>
