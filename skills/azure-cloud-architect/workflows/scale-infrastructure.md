<required_reading>
Before implementing scaling, understand:
- `references/compute-services.md` - Scaling capabilities of different Azure services
- `references/cost-optimization.md` - Cost implications of scaling
- `references/kubernetes-aks.md` - AKS auto-scaling if using Kubernetes
</required_reading>

<objective>
Configure auto-scaling for Azure infrastructure to handle traffic growth while optimizing costs using App Service auto-scale, AKS cluster autoscaler, and Azure Monitor metrics.
</objective>

<quick_start>
**5-minute auto-scale setup:**
```bash
# Enable auto-scale for App Service Plan
az monitor autoscale create \
  --resource-group myapp-prod-rg \
  --resource $(az appservice plan show --name asp-myapp-prod-eastus --resource-group myapp-prod-rg --query id -o tsv) \
  --name autoscale-asp-myapp-prod \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Scale out when CPU > 70%
az monitor autoscale rule create \
  --resource-group myapp-prod-rg \
  --autoscale-name autoscale-asp-myapp-prod \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1

# Scale in when CPU < 30%
az monitor autoscale rule create \
  --resource-group myapp-prod-rg \
  --autoscale-name autoscale-asp-myapp-prod \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1
```
</quick_start>

<process>
## Step 1: Understand Scaling Options

**Vertical scaling (scale up/down):**
- Change to larger/smaller VM size
- More CPU, RAM, disk
- Requires restart (brief downtime)
- Use for predictable growth

**Horizontal scaling (scale out/in):**
- Add/remove instances
- No downtime
- Better for variable traffic
- Requires stateless applications

**Auto-scaling:**
- Automatically adjusts instance count based on metrics
- Cost-efficient (only pay for what you need)
- Responds to traffic patterns

## Step 2: App Service Auto-Scale

**2.1 Create auto-scale profile:**

```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
    capacity: 2  // Starting capacity
  }
}

resource autoscale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: 'autoscale-asp-myapp-prod'
  location: 'eastus'
  properties: {
    enabled: true
    targetResourceUri: appServicePlan.id
    profiles: [
      {
        name: 'Default autoscale profile'
        capacity: {
          minimum: '2'  // Never go below 2 instances (HA)
          maximum: '10'  // Cost cap
          default: '2'
        }
        rules: [
          {
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: appServicePlan.id
              timeGrain: 'PT1M'  // Sample every 1 minute
              statistic: 'Average'
              timeWindow: 'PT5M'  // Evaluate over 5 minutes
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 70
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'  // Add 1 instance
              cooldown: 'PT5M'  // Wait 5 min before scaling again
            }
          }
          {
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: appServicePlan.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT10M'  // Longer window for scale-in
              timeAggregation: 'Average'
              operator: 'LessThan'
              threshold: 30
            }
            scaleAction: {
              direction: 'Decrease'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT10M'  // Longer cooldown for scale-in
            }
          }
          {
            metricTrigger: {
              metricName: 'HttpQueueLength'
              metricResourceUri: appServicePlan.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT5M'
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 100  // Queue > 100 requests
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '2'  // Add 2 instances (queue backing up)
              cooldown: 'PT5M'
            }
          }
        ]
      }
      {
        name: 'Business hours profile'  // Scale up during business hours
        capacity: {
          minimum: '4'
          maximum: '15'
          default: '4'
        }
        rules: []  // Same rules as default
        recurrence: {
          frequency: 'Week'
          schedule: {
            timeZone: 'Eastern Standard Time'
            days: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday']
            hours: [8]  // 8 AM
            minutes: [0]
          }
        }
      }
      {
        name: 'Off-hours profile'
        capacity: {
          minimum: '2'
          maximum: '6'
          default: '2'
        }
        rules: []
        recurrence: {
          frequency: 'Week'
          schedule: {
            timeZone: 'Eastern Standard Time'
            days: ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday']
            hours: [18]  // 6 PM
            minutes: [0]
          }
        }
      }
    ]
    notifications: [
      {
        operation: 'Scale'
        email: {
          sendToSubscriptionAdministrator: false
          sendToSubscriptionCoAdministrator: false
          customEmails: ['ops@company.com']
        }
      }
    ]
  }
}
```

**2.2 Metrics for auto-scale:**

```bash
# CPU-based (most common)
az monitor autoscale rule create \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 1

# Memory-based
az monitor autoscale rule create \
  --condition "MemoryPercentage > 80 avg 5m" \
  --scale out 1

# HTTP queue length (request backlog)
az monitor autoscale rule create \
  --condition "HttpQueueLength > 100 avg 5m" \
  --scale out 2

# Response time (Application Insights metric)
az monitor autoscale rule create \
  --condition "requests/duration > 1000 avg 5m" \
  --scale out 1

# Custom metric (Application Insights)
az monitor autoscale rule create \
  --condition "customMetrics/activeUsers > 1000 avg 5m" \
  --scale out 2
```

## Step 3: AKS Auto-Scaling

**3.1 Horizontal Pod Autoscaler (HPA):**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2  # Initial replicas
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:  # Control scale-up/down behavior
    scaleUp:
      stabilizationWindowSeconds: 60  # Wait 60s before scaling up
      policies:
      - type: Percent
        value: 50  # Max 50% increase at once
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1  # Remove 1 pod at a time
        periodSeconds: 60
```

**3.2 Cluster Autoscaler:**

```bicep
resource aksCluster 'Microsoft.ContainerService/managedClusters@2024-02-01' = {
  name: 'aks-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    agentPoolProfiles: [
      {
        name: 'system'
        count: 2
        vmSize: 'Standard_D2s_v3'
        mode: 'System'
        enableAutoScaling: true
        minCount: 2
        maxCount: 5
      }
      {
        name: 'user'
        count: 3
        vmSize: 'Standard_D4s_v3'
        mode: 'User'
        enableAutoScaling: true
        minCount: 3
        maxCount: 10
      }
    ]
  }
}
```

**3.3 KEDA (Event-driven autoscaling):**

```bash
# Install KEDA
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

```yaml
# ScaledObject for queue-based scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      namespace: sb-myapp-prod-eastus
      messageCount: "10"  # 1 pod per 10 messages
```

## Step 4: Azure Functions Auto-Scale

**Consumption Plan (automatic):**
```bicep
resource funcApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-processor-prod-eastus'
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: consumptionPlan.id  // Auto-scales to thousands of instances
  }
}

resource consumptionPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-consumption-prod-eastus'
  sku: {
    name: 'Y1'  // Consumption (dynamic)
    tier: 'Dynamic'
  }
}
```

**Premium Plan (pre-warmed instances):**
```bicep
resource premiumPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-func-premium-prod-eastus'
  sku: {
    name: 'EP1'  // Elastic Premium
    tier: 'ElasticPremium'
  }
  properties: {
    maximumElasticWorkerCount: 20  // Max scale-out
  }
}
```

## Step 5: Database Scaling

**5.1 Azure SQL Database auto-scale:**

```bicep
// Serverless (auto-scales compute)
resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'mydb'
  location: 'eastus'
  sku: {
    name: 'GP_S_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2  // Min vCores
  }
  properties: {
    autoPauseDelay: 60  // Auto-pause after 60 min idle
    minCapacity: 0.5  // Min 0.5 vCores
    maxSizeBytes: 34359738368  // 32 GB
  }
}
```

**5.2 Cosmos DB auto-scale:**

```bicep
resource cosmosDatabase 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2024-05-15' = {
  parent: cosmosAccount
  name: 'mydb'
  properties: {
    resource: {
      id: 'mydb'
    }
    options: {
      autoscaleSettings: {
        maxThroughput: 4000  // Auto-scale 400-4000 RU/s
      }
    }
  }
}
```

## Step 6: Monitor Auto-Scaling

```bash
# Check auto-scale history
az monitor autoscale show \
  --name autoscale-asp-myapp-prod \
  --resource-group myapp-prod-rg

# Get scale events
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --offset 24h \
  --query "[?contains(operationName.value, 'Autoscale')].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.value}" \
  --output table

# Query scale metrics in Application Insights
az monitor app-insights query \
  --app ai-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --analytics-query "performanceCounters | where name == 'Instance Count' | summarize avg(value) by bin(timestamp, 5m) | render timechart"
```

**Application Insights query for scale correlation:**
```kql
// Correlate instance count with request volume
let scaling = performanceCounters
| where name == "Instance Count"
| project timestamp, instanceCount=value;

let requests = requests
| summarize requestCount=count() by bin(timestamp, 5m);

scaling
| join kind=inner requests on timestamp
| render timechart
```
</process>

<scaling_best_practices>
## Scaling Best Practices

1. **Set appropriate minimums:**
   - Production: min 2 instances (HA)
   - Dev: min 0-1 instances (cost)

2. **Set conservative maximums:**
   - Prevent runaway costs
   - Account for quotas
   - Plan for max capacity

3. **Use different metrics:**
   - CPU/memory for compute-bound
   - Queue length for async workloads
   - Custom metrics for business logic

4. **Cooldown periods:**
   - Scale-out: 5 minutes (respond quickly)
   - Scale-in: 10-15 minutes (avoid flapping)

5. **Schedule-based scaling:**
   - Business hours: higher capacity
   - Weekends/nights: lower capacity
   - Black Friday: pre-scale

6. **Test scaling:**
   - Load test before production
   - Verify app is stateless
   - Check database connection limits
</scaling_best_practices>

<cost_estimation>
**App Service auto-scale costs:**
```
Baseline (P1v3): 2 instances × $146/month = $292/month
Peak (P1v3): 10 instances × $146/month × 10% time = $146/month
Total: ~$438/month

vs. Fixed 10 instances: $1,460/month
Savings: 70%
```

**AKS auto-scale costs:**
```
Baseline: 3 nodes × D4s_v3 ($140/month) = $420/month
Peak: 10 nodes × $140/month × 15% time = $210/month
Total: ~$630/month

vs. Fixed 10 nodes: $1,400/month
Savings: 55%
```
</cost_estimation>

<success_criteria>
Auto-scaling is properly configured when:
- Minimum instances ensure high availability (≥2 for production)
- Maximum instances prevent runaway costs
- Metrics trigger scaling at appropriate thresholds
- Cooldown periods prevent flapping
- Scale events logged and monitored
- Load testing confirms smooth scaling
- Application remains responsive during scale events
- Cost optimized (only pay for needed capacity)
</success_criteria>
