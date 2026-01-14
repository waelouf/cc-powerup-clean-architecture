<overview>
Comprehensive guide to Azure compute services. Covers decision-making for App Service, Azure Functions, Container Apps, AKS, and VMs with pricing, use cases, and best practices for 2024-2025.
</overview>

<service_comparison>
## Azure Compute Services Decision Matrix

| Service | Use Case | Abstraction Level | Pricing | Scaling |
|---------|----------|-------------------|---------|---------|
| **App Service** | Web apps, APIs, backends | High (PaaS) | $13-$620/mo | Auto-scale |
| **Azure Functions** | Event-driven, serverless | Highest | Consumption or Premium | Auto-scale |
| **Container Apps** | Containerized apps, microservices | High (PaaS) | Consumption-based | Auto-scale to 0 |
| **AKS** | Kubernetes, complex microservices | Medium | $70+/mo (nodes) | Manual/HPA/KEDA |
| **VMs** | Full control, legacy apps | Low (IaaS) | $15-$500+/mo | Manual/VMSS |

</service_comparison>

<app_service>
## Azure App Service

**What it is:** Managed PaaS for web apps, APIs, and mobile backends.

**When to use:**
- ✅ Standard web applications or REST APIs
- ✅ Want simple deployment and management
- ✅ Don't need container orchestration
- ✅ .NET, Node.js, Python, Java, PHP apps
- ✅ Need deployment slots (blue-green)

**When NOT to use:**
- ❌ Need Kubernetes features
- ❌ Require Windows containers
- ❌ Need root access to OS
- ❌ Extremely cost-sensitive (Functions cheaper for low traffic)

### Pricing Tiers

**Basic (B1):** $13/month
- 1 core, 1.75 GB RAM
- Use for: Dev/test
- No auto-scale, no deployment slots

**Standard (S1):** $70/month
- 1 core, 1.75 GB RAM
- Auto-scale (up to 10 instances)
- Deployment slots (5)
- Custom domains, SSL
- Use for: Small production apps

**Premium v3 (P1v3):** $124/month
- 2 cores, 8 GB RAM
- Auto-scale (up to 30 instances)
- Deployment slots (20)
- Better performance (Dv3 VMs)
- Use for: Production apps with traffic

**Premium v3 (P2v3):** $248/month
- 4 cores, 16 GB RAM
- Use for: High-traffic production

### Example

```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'myapp-prod-asp'
  location: 'eastus'
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
    capacity: 2  // 2 instances
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      alwaysOn: true
      minTlsVersion: '1.2'
    }
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
    }
  }
}

// Auto-scale
resource autoScale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: '${appServicePlan.name}-autoscale'
  location: 'eastus'
  properties: {
    targetResourceUri: appServicePlan.id
    enabled: true
    profiles: [
      {
        name: 'Scale based on CPU'
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
              metricResourceUri: appServicePlan.id
              operator: 'GreaterThan'
              statistic: 'Average'
              threshold: 70
              timeAggregation: 'Average'
              timeGrain: 'PT1M'
              timeWindow: 'PT5M'
            }
          }
          {
            scaleAction: {
              direction: 'Decrease'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT10M'
            }
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: appServicePlan.id
              operator: 'LessThan'
              statistic: 'Average'
              threshold: 30
              timeAggregation: 'Average'
              timeGrain: 'PT1M'
              timeWindow: 'PT10M'
            }
          }
        ]
      }
    ]
  }
}
```

### Cost Optimization

**Reserved capacity:** Save 55-78%
```
P1v3 pay-as-you-go: $124/month
P1v3 1-year reserved: $55/month (56% savings)
P1v3 3-year reserved: $37/month (70% savings)
```

**Right-sizing:**
- Monitor CPU/memory usage
- If average < 30%, downgrade SKU
- Use B1 for dev/test ($13/mo vs $124/mo)
</app_service>

<azure_functions>
## Azure Functions

**What it is:** Serverless, event-driven compute.

**When to use:**
- ✅ Event-driven workloads (queue, blob trigger)
- ✅ Scheduled jobs (cron)
- ✅ HTTP APIs with low/variable traffic
- ✅ Pay only for execution time
- ✅ Auto-scale to zero

**When NOT to use:**
- ❌ Long-running processes (max 10 min, 60 min with Durable)
- ❌ High sustained traffic (App Service cheaper)
- ❌ Need deployment slots
- ❌ Stateful applications

### Hosting Plans

**Consumption:** Pay per execution
- First 1M executions free/month
- $0.20 per million executions
- $0.000016 per GB-second
- Auto-scale, scale to zero
- Max 10 minute execution
- Use for: Low-traffic, event-driven

**Premium:** Pay for plan
- $174/month (EP1 plan)
- Always warm instances
- Longer execution time (60 min)
- VNet integration
- Durable Functions
- Use for: Latency-sensitive, need warm start

**Dedicated (App Service Plan):** Share with App Service
- Use existing App Service Plan
- No additional cost
- Use for: Already have underutilized plan

### Example

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'funcstorageaccount'
  location: 'eastus'
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource functionApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myfunction-prod'
  location: 'eastus'
  kind: 'functionapp,linux'
  properties: {
    serverFarmId: consumptionPlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20'
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};...'
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
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

**Function code (HTTP trigger):**
```javascript
module.exports = async function (context, req) {
    context.log('HTTP trigger function processed a request.');

    const name = req.query.name || (req.body && req.body.name);

    context.res = {
        status: 200,
        body: `Hello, ${name}!`
    };
};
```

**Queue trigger:**
```javascript
module.exports = async function (context, queueItem) {
    context.log('Processing queue item:', queueItem);

    // Process message
    await processOrder(queueItem);
};
```

### Cost Example

**Low traffic API (10,000 requests/month):**
```
Consumption plan:
- 10,000 executions: $0.002
- Compute (200ms avg, 128MB): $0.003
Total: ~$0.005/month

App Service (B1):
- Fixed cost: $13/month
```

**High traffic API (10M requests/month):**
```
Consumption plan:
- 10M executions: $2
- Compute: $50
Total: ~$52/month

App Service (P1v3 x2):
- Fixed cost: $248/month
- BUT: More predictable, better perf
```

**Break-even:** ~5M requests/month
</azure_functions>

<container_apps>
## Azure Container Apps

**What it is:** Serverless containers with auto-scale to zero.

**When to use:**
- ✅ Containerized apps without Kubernetes complexity
- ✅ Microservices
- ✅ Background workers, jobs
- ✅ Event-driven containers
- ✅ Need scale to zero

**When NOT to use:**
- ❌ Need full Kubernetes features
- ❌ Complex networking requirements
- ❌ Stateful applications
- ❌ Need specific k8s controllers

### Pricing

Consumption-based:
- $0.000012 per vCPU-second
- $0.000001333 per GB-second
- 180,000 vCPU-seconds free/month
- 360,000 GB-seconds free/month

**Example cost:**
```
Container: 0.5 vCPU, 1 GB
Running 24/7:
- vCPU: 0.5 × 2,592,000 sec = 1,296,000 vCPU-sec = $15.55
- Memory: 1 × 2,592,000 sec = 2,592,000 GB-sec = $3.46
Total: ~$19/month

If scales to zero during nights (12 hrs/day):
Total: ~$9.50/month
```

### Example

```bicep
resource containerAppEnv 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: 'myapp-env'
  location: 'eastus'
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'myapp-api'
  location: 'eastus'
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 3000
        allowInsecure: false
      }
      secrets: [
        {
          name: 'registry-password'
          value: registryPassword
        }
      ]
      registries: [
        {
          server: 'myregistry.azurecr.io'
          username: 'myregistry'
          passwordSecretRef: 'registry-password'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: 'myregistry.azurecr.io/api:latest'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0  // Scale to zero!
        maxReplicas: 10
        rules: [
          {
            name: 'http-scale'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}
```
</container_apps>

<aks>
## Azure Kubernetes Service (AKS)

**What it is:** Managed Kubernetes for container orchestration.

**When to use:**
- ✅ Complex microservices architectures
- ✅ Need Kubernetes-specific features (CRDs, operators)
- ✅ Multi-container deployments
- ✅ Existing Kubernetes expertise
- ✅ Advanced networking requirements

**When NOT to use:**
- ❌ Simple web app (use App Service)
- ❌ Want serverless (use Container Apps)
- ❌ No Kubernetes knowledge
- ❌ Small team/simple app

### Pricing

**Node pool:**
- Standard_D4s_v5 (4 vCPU, 16 GB): $140/month per node
- Minimum 2 nodes: $280/month
- AKS control plane: Free

**Cost optimization:**
- Use Standard tier for production ($73/mo for SLA)
- User node pools: Standard_D2s_v5 for small workloads
- Spot VMs for fault-tolerant workloads (up to 90% discount)

### Example

```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  name: 'myapp-prod-aks'
  location: 'eastus'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    dnsPrefix: 'myapp-prod'
    kubernetesVersion: '1.28'
    enableRBAC: true
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'azure'
      serviceCidr: '10.0.0.0/16'
      dnsServiceIP: '10.0.0.10'
    }
    agentPoolProfiles: [
      {
        name: 'system'
        count: 2
        vmSize: 'Standard_D2s_v5'
        osType: 'Linux'
        mode: 'System'
        enableAutoScaling: true
        minCount: 2
        maxCount: 5
      }
      {
        name: 'user'
        count: 3
        vmSize: 'Standard_D4s_v5'
        osType: 'Linux'
        mode: 'User'
        enableAutoScaling: true
        minCount: 1
        maxCount: 10
      }
    ]
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalytics.id
        }
      }
      azurepolicy: {
        enabled: true
      }
    }
  }
}
```

**Deploy app:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/api:v1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
```

### AKS Best Practices

1. **Use system + user node pools** (separate concerns)
2. **Enable auto-scaling** (HPA + cluster autoscaler)
3. **Set resource requests/limits** (proper scheduling)
4. **Use Azure CNI** for production (better networking)
5. **Enable monitoring** (Container Insights)
6. **Use Azure Policy** for governance
7. **Implement network policies** (pod-to-pod security)
</aks>

<decision_guide>
## Decision Flow Chart

**Do you need containers?**
- No → App Service or Functions
- Yes → Continue

**Do you need Kubernetes features?**
- No → Container Apps
- Yes → AKS

**Is it event-driven or low traffic?**
- Yes → Functions (Consumption)
- No → Continue

**Do you need always-on performance?**
- Yes → App Service
- No → Functions (if < 5M req/mo)

**Do you need deployment slots / blue-green?**
- Yes → App Service
- No → Container Apps or Functions

**Do you need to scale to zero?**
- Yes → Functions or Container Apps
- No → App Service or AKS

**Do you have complex microservices?**
- Yes → AKS
- No → App Service or Container Apps

</decision_guide>

<anti_patterns>
**Avoid:**

<anti_pattern name="AKS for Simple App">
**Problem:** Using Kubernetes for single web app

**Why bad:** Overkill complexity, higher cost, steep learning curve

**Instead:** Use App Service for simple apps
</anti_pattern>

<anti_pattern name="Always-On Functions">
**Problem:** Using Consumption plan for 24/7 high-traffic API

**Why bad:** More expensive than App Service, cold starts

**Instead:** Use Premium Functions or App Service for sustained traffic
</anti_pattern>

<anti_pattern name="No Auto-Scale">
**Problem:** Fixed instance count regardless of load

**Why bad:** Waste money during low traffic, insufficient capacity during spikes

**Instead:** Configure auto-scaling based on CPU/memory/requests
</anti_pattern>

<anti_pattern name="Wrong SKU">
**Problem:** Using Premium tier for dev/test

**Why bad:** Wasting $100+/month per environment

**Instead:** Use Basic tier for dev/test, Premium only for production
</anti_pattern>
</anti_patterns>
