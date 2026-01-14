<overview>
Comprehensive guide to Azure Kubernetes Service (AKS) best practices for production. Covers cluster configuration, node pools, networking, security, scaling, and operational patterns for 2024-2025.
</overview>

<aks_fundamentals>
## AKS Architecture

**Components:**
- **Control Plane:** Managed by Azure (free)
- **Node Pools:** VMs running containers (you pay for VMs)
- **Networking:** Azure CNI or Kubenet
- **Add-ons:** Monitoring, policy, ingress

**Cost:** Pay only for worker nodes (VMs), control plane is free

### Cluster Tiers

**Free Tier:**
- No SLA
- Use for: Dev/test
- Cost: $0 (just VM costs)

**Standard Tier:**
- 99.5% SLA (single zone)
- 99.95% SLA (availability zones)
- Use for: Production
- Cost: $73/month + VM costs

**Premium Tier (2024+):**
- 99.95% SLA (uptime)
- Longer support for Kubernetes versions
- Use for: Mission-critical workloads
- Cost: $365/month + VM costs

</aks_fundamentals>

<cluster_configuration>
## Production AKS Cluster

```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  name: 'aks-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Base'
    tier: 'Standard'  // Production SLA
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: '1.28.3'  // Use recent stable version
    dnsPrefix: 'myapp-prod'
    enableRBAC: true

    // Azure AD integration
    aadProfile: {
      managed: true
      enableAzureRBAC: true  // Use Azure RBAC for k8s auth
      adminGroupObjectIDs: [
        adminGroupObjectId  // Azure AD group for cluster admins
      ]
    }

    // Networking
    networkProfile: {
      networkPlugin: 'azure'  // Azure CNI (recommended for prod)
      networkPolicy: 'azure'
      serviceCidr: '10.0.0.0/16'
      dnsServiceIP: '10.0.0.10'
      loadBalancerSku: 'standard'
      outboundType: 'loadBalancer'
    }

    // Node pools defined separately
    agentPoolProfiles: []  // See below

    // API server access
    apiServerAccessProfile: {
      authorizedIPRanges: [
        '203.0.113.0/24'  // Office IP range
      ]
      enablePrivateCluster: false  // or true for private cluster
    }

    // Add-ons
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
      azureKeyvaultSecretsProvider: {
        enabled: true
        config: {
          enableSecretRotation: 'true'
        }
      }
    }

    // Auto-upgrade
    autoUpgradeProfile: {
      upgradeChannel: 'patch'  // Auto-upgrade to latest patch
    }

    // Security
    securityProfile: {
      defender: {
        securityMonitoring: {
          enabled: true
        }
      }
    }
  }
}
```
</cluster_configuration>

<node_pools>
## Node Pools Best Practices

### System Pool (Required)

**Purpose:** Run system pods (CoreDNS, metrics-server, etc.)

```bicep
resource systemPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-01-01' = {
  parent: aks
  name: 'system'
  properties: {
    count: 3  // Always 3+ for HA
    vmSize: 'Standard_D2s_v5'  // 2 vCPU, 8 GB RAM
    osType: 'Linux'
    osSKU: 'Ubuntu'
    mode: 'System'  // System pool
    availabilityZones: [
      '1'
      '2'
      '3'
    ]
    enableAutoScaling: true
    minCount: 3
    maxCount: 5
    maxPods: 30
    nodeLabels: {
      'workload': 'system'
    }
    nodeTaints: [
      'CriticalAddonsOnly=true:NoSchedule'  // Only system pods
    ]
  }
}
```

**Cost:** 3 × Standard_D2s_v5 = 3 × $70/month = $210/month minimum

### User Pool (Application Workloads)

```bicep
resource userPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-01-01' = {
  parent: aks
  name: 'apps'
  properties: {
    count: 3
    vmSize: 'Standard_D4s_v5'  // 4 vCPU, 16 GB RAM
    osType: 'Linux'
    mode: 'User'
    availabilityZones: [
      '1'
      '2'
      '3'
    ]
    enableAutoScaling: true
    minCount: 2
    maxCount: 20
    maxPods: 50
    nodeLabels: {
      'workload': 'applications'
    }
  }
}
```

### Specialized Pools

**GPU pool:**
```bicep
resource gpuPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-01-01' = {
  parent: aks
  name: 'gpu'
  properties: {
    count: 1
    vmSize: 'Standard_NC6s_v3'  // 6 vCPU, 112 GB RAM, 1 GPU
    mode: 'User'
    enableAutoScaling: true
    minCount: 0  // Scale to zero when not needed
    maxCount: 5
    nodeLabels: {
      'workload': 'gpu'
    }
    nodeTaints: [
      'sku=gpu:NoSchedule'  // Only GPU workloads
    ]
  }
}
```

**Spot VM pool (cost savings):**
```bicep
resource spotPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-01-01' = {
  parent: aks
  name: 'spot'
  properties: {
    count: 3
    vmSize: 'Standard_D4s_v5'
    mode: 'User'
    scaleSetPriority: 'Spot'  // Spot VMs (up to 90% discount)
    scaleSetEvictionPolicy: 'Delete'
    spotMaxPrice: -1  // Pay up to on-demand price
    enableAutoScaling: true
    minCount: 0
    maxCount: 10
    nodeLabels: {
      'workload': 'batch'
      'kubernetes.azure.com/scalesetpriority': 'spot'
    }
    nodeTaints: [
      'kubernetes.azure.com/scalesetpriority=spot:NoSchedule'
    ]
  }
}
```

**Use Spot VMs for:**
- Batch processing
- CI/CD build agents
- Fault-tolerant workloads
- Dev/test environments

**Don't use for:**
- Stateful applications
- Critical workloads
- Databases
</node_pools>

<networking>
## AKS Networking

### CNI Plugins

**Azure CNI (Recommended for production):**
- Each pod gets IP from VNet
- Direct pod-to-pod communication
- Can apply NSGs to pods
- Requires large VNet (each pod uses an IP)

**Kubenet:**
- Pods use private IP range
- NAT for external communication
- Cheaper (fewer IPs needed)
- Use for: Dev/test, cost-sensitive

### Network Policies

**Enable network policies for pod-to-pod security:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api  # Only API pods can access DB
    ports:
    - protocol: TCP
      port: 5432
```

### Ingress Controllers

**NGINX Ingress Controller (most common):**

```bash
# Install via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

**Ingress resource:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Application Gateway Ingress Controller (AGIC)

**When to use:** Need WAF, Azure integration, SSL offloading

```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  properties: {
    addonProfiles: {
      ingressApplicationGateway: {
        enabled: true
        config: {
          applicationGatewayId: applicationGateway.id
        }
      }
    }
  }
}
```
</networking>

<auto_scaling>
## Auto-Scaling

### Horizontal Pod Autoscaler (HPA)

**Scale pods based on CPU/memory:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50  # Max 50% increase at once
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Pods
        value: 1  # Remove 1 pod at a time
        periodSeconds: 60
```

### Cluster Autoscaler

**Automatically add/remove nodes:**

```bicep
resource userPool 'Microsoft.ContainerService/managedClusters/agentPools@2024-01-01' = {
  properties: {
    enableAutoScaling: true
    minCount: 2
    maxCount: 20
  }
}
```

**How it works:**
1. Pods can't schedule (not enough resources)
2. Cluster autoscaler adds node
3. Pods schedule on new node
4. When nodes underutilized, scale down

### KEDA (Event-Driven Autoscaling)

**Scale based on events (queue length, HTTP requests, etc.):**

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

**Scale based on Azure Storage Queue:**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-scaler
spec:
  scaleTargetRef:
    name: queue-processor
  minReplicaCount: 0  # Scale to zero!
  maxReplicaCount: 30
  triggers:
  - type: azure-queue
    metadata:
      queueName: orders
      queueLength: '5'  # 1 pod per 5 messages
      connectionFromEnv: STORAGE_CONNECTION
```
</auto_scaling>

<security>
## AKS Security Best Practices

### 1. Azure AD Integration

**Use Azure AD for authentication:**
```bash
# Get credentials
az aks get-credentials --resource-group myapp-prod-rg --name aks-myapp-prod

# Users authenticate with Azure AD
kubectl get pods  # Prompts for Azure AD login
```

**RBAC roles:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-edit
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit  # Built-in role
subjects:
- kind: Group
  name: "dev-team-group-id"  # Azure AD group
  apiGroup: rbac.authorization.k8s.io
```

### 2. Pod Identity

**Access Azure resources from pods without secrets:**

```bash
# Enable Pod Identity
az aks update \
  --resource-group myapp-prod-rg \
  --name aks-myapp-prod \
  --enable-pod-identity
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    aadpodidbinding: myapp-identity
spec:
  containers:
  - name: api
    image: myregistry.azurecr.io/api:v1.0.0
    env:
    - name: AZURE_CLIENT_ID
      value: "managed-identity-client-id"
```

**Pod can now access Key Vault, SQL, Storage without credentials.**

### 3. Network Policies

**Deny all traffic by default:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Then allow specific traffic explicitly.

### 4. Pod Security Standards

**Enforce restricted pod security:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Restricted prevents:**
- Running as root
- Privileged containers
- Host network access
- Host path mounts

### 5. Azure Policy

**Enforce policies cluster-wide:**

```bash
# Enable Azure Policy add-on
az aks update \
  --resource-group myapp-prod-rg \
  --name aks-myapp-prod \
  --enable-azure-policy
```

**Example policies:**
- Require resource limits
- Block privileged containers
- Enforce image sources (only from approved registries)
- Require labels
</security>

<operational_best_practices>
## Production Operations

### 1. Resource Limits

**Always set resource requests and limits:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/api:v1.0.0
        resources:
          requests:
            cpu: 100m      # Minimum needed
            memory: 128Mi
          limits:
            cpu: 500m      # Maximum allowed
            memory: 512Mi
```

**Why:**
- Requests: Used for scheduling (guarantee minimum)
- Limits: Prevent resource exhaustion

### 2. Health Probes

```yaml
spec:
  containers:
  - name: api
    image: api:v1.0.0
    ports:
    - containerPort: 3000
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health/live
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
```

**Readiness:** Is pod ready to receive traffic?
**Liveness:** Is pod healthy (restart if failing)?

### 3. Monitoring

**Container Insights (enabled via add-on):**
- CPU/memory per pod
- Node metrics
- Logs aggregation

**Query logs:**
```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where Name contains "api"
| where LogEntry contains "ERROR"
| project TimeGenerated, LogEntry
```

**Prometheus + Grafana (advanced):**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### 4. GitOps with Flux/ArgoCD

**Flux for GitOps:**
```bash
# Install Flux
flux install

# Add Git repository
flux create source git myapp \
  --url=https://github.com/myorg/myapp \
  --branch=main

# Auto-deploy from Git
flux create kustomization myapp \
  --source=myapp \
  --path="./k8s/production" \
  --prune=true
```

**Benefits:**
- Git as source of truth
- Automated deployments
- Easy rollback (revert Git commit)
- Audit trail

### 5. Cluster Upgrades

**Test in dev first:**
```bash
# Upgrade dev cluster
az aks upgrade \
  --resource-group myapp-dev-rg \
  --name aks-myapp-dev \
  --kubernetes-version 1.28.5

# Test thoroughly

# Upgrade production
az aks upgrade \
  --resource-group myapp-prod-rg \
  --name aks-myapp-prod \
  --kubernetes-version 1.28.5 \
  --yes
```

**Or use auto-upgrade:**
```bicep
autoUpgradeProfile: {
  upgradeChannel: 'patch'  // Auto-upgrade to patch releases only
}
```
</operational_best_practices>

<cost_optimization>
## AKS Cost Optimization

### 1. Right-Size Node Pools

**Monitor actual usage:**
```kql
Perf
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsagePercentage"
| summarize AvgCPU = avg(CounterValue) by Computer
```

If average CPU < 30%, use smaller VMs.

### 2. Use Spot VMs

**Save 60-90% on fault-tolerant workloads.**

### 3. Scale to Zero

**User pools can scale to 0 when not needed:**
```bicep
enableAutoScaling: true
minCount: 0
maxCount: 10
```

### 4. Cluster Stop/Start

**Stop dev/test clusters nights/weekends:**
```bash
# Stop cluster (no VM costs while stopped)
az aks stop --resource-group myapp-dev-rg --name aks-myapp-dev

# Start cluster
az aks start --resource-group myapp-dev-rg --name aks-myapp-dev
```

**Savings:** 70% for dev clusters (stop 12 hours/day + weekends)

### 5. Reserved Instances

**For production nodes running 24/7:**
- 1-year RI: 40% savings
- 3-year RI: 60% savings

**Example:**
```
3 × Standard_D4s_v5 nodes × $140/month = $420/month
3-year RI: $165/month
Savings: $255/month = $3,060/year
```
</cost_optimization>

<anti_patterns>
**Avoid:**

<anti_pattern name="No Resource Limits">
**Problem:** Pods without resource limits

**Why bad:** One pod can consume all node resources

**Instead:** Always set requests and limits
</anti_pattern>

<anti_pattern name="Single System Pool">
**Problem:** Running apps on system pool

**Why bad:** App issues affect system pods, cluster instability

**Instead:** Separate system and user pools
</anti_pattern>

<anti_pattern name="No Health Probes">
**Problem:** No readiness/liveness probes

**Why bad:** Traffic routed to unhealthy pods, failed pods not restarted

**Instead:** Implement both readiness and liveness probes
</anti_pattern>

<anti_pattern name="Running as Root">
**Problem:** Containers running as root user

**Why bad:** Security risk, privilege escalation

**Instead:** Run as non-root, enforce with Pod Security Standards
</anti_pattern>

<anti_pattern name="Ignoring Security">
**Problem:** No network policies, no RBAC, no pod security

**Why bad:** Lateral movement in breach, privilege escalation

**Instead:** Enable all security features (Azure AD, Network Policies, Pod Security)
</anti_pattern>
</anti_patterns>
