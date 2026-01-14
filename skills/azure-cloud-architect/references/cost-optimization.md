<overview>
Comprehensive guide to Azure cost optimization and FinOps practices. Covers right-sizing, reserved instances, cost allocation, monitoring, and actionable strategies for reducing Azure spending while maintaining performance and reliability. Based on 2024-2025 best practices.
</overview>

<finops_fundamentals>
## What is FinOps?

**FinOps** (Financial Operations) is an evolving cloud financial management discipline and cultural practice that enables organizations to get maximum business value by helping engineering, finance, IT, and business teams collaborate on cloud spending.

<principle name="Everyone Takes Ownership">
**Core concept:** Cloud costs are not just finance's problem - every team that deploys resources is responsible for their cloud spending.

**In practice:**
- Engineers see cost impact of architectural decisions
- Product teams understand cost per customer/feature
- Finance provides frameworks and visibility
- Leadership sets budgets and expectations
</principle>

<principle name="Real-Time Decision Making">
**Core concept:** Cloud costs are variable and immediate. Decisions should be made quickly with current data.

**In practice:**
- Cost dashboards updated daily
- Alerts on budget overruns
- Teams empowered to optimize without waiting for monthly reports
- Automation to enforce policies
</principle>

<principle name="Centralized Best Practices">
**Core concept:** A central team (FinOps team / Cloud Center of Excellence) provides tools, training, and governance.

**In practice:**
- Platform team maintains cost dashboards
- Shared playbooks for optimization
- Regular cost review meetings
- Training on cost-effective architecture
</principle>
</finops_fundamentals>

<cost_drivers>
## Azure Cost Drivers (What Costs Money)

<service_category name="Compute">
**Services:** Virtual Machines, App Service Plans, AKS node pools, Azure Functions, Container Instances

**Cost factors:**
- **Size/SKU:** Larger instances = higher cost
- **Running time:** Charged per hour (or per second for some)
- **Region:** Some regions cost more (e.g., West Europe > East US)
- **OS:** Windows VMs cost more than Linux
- **Reserved instances:** Discounts for 1-3 year commitments

**Typical costs (East US, Jan 2025):**
- VM (B2s): $15/month
- VM (D4s v5): $140/month
- App Service Plan (B1): $13/month
- App Service Plan (P1v3): $124/month
- AKS node (D4s v5): $140/month per node

**Optimization levers:**
- Right-size (don't over-provision)
- Auto-scale (scale down when not needed)
- Auto-shutdown (dev/test environments)
- Reserved instances (40-72% savings)
- Spot VMs for fault-tolerant workloads (up to 90% savings)
</service_category>

<service_category name="Storage">
**Services:** Storage Accounts (blobs, files), Managed Disks, SQL Database storage

**Cost factors:**
- **Capacity:** GB stored
- **Access tier:** Hot (expensive) > Cool > Archive (cheapest)
- **Redundancy:** GRS > LRS
- **Transactions:** Read/write operations
- **Data egress:** Outbound data transfer

**Typical costs (East US):**
- Blob storage (Hot, LRS): $0.018/GB/month
- Blob storage (Cool, LRS): $0.01/GB/month
- Blob storage (Archive, LRS): $0.002/GB/month
- Managed disk (P30, 1TB): $122/month

**Optimization levers:**
- Lifecycle policies (move to cool/archive)
- Delete old snapshots and backups
- Use appropriate redundancy (LRS vs GRS)
- Unattach unused disks
- Compress data
</service_category>

<service_category name="Database">
**Services:** Azure SQL Database, Cosmos DB, PostgreSQL, MySQL

**Cost factors:**
- **Provisioned capacity:** DTUs or vCores
- **Storage:** GB used
- **Backup retention:** Longer retention = higher cost
- **Replication:** Geo-replication costs extra

**Typical costs (SQL Database, East US):**
- Basic (5 DTUs): $5/month
- Standard S3 (100 DTUs): $149/month
- Premium P2 (250 DTUs): $928/month
- Serverless vCore: $0.51/vCore-hour

**Optimization levers:**
- Right-size DTUs/vCores (monitor usage)
- Use serverless for unpredictable workloads
- Elastic pools for multiple databases
- Shorter backup retention
- Use read replicas selectively
</service_category>

<service_category name="Networking">
**Services:** Load Balancer, Application Gateway, VPN Gateway, Public IPs, Bandwidth

**Cost factors:**
- **Data processed:** GB through load balancer/gateway
- **Data egress:** Outbound internet traffic
- **Public IPs:** $3.60/month each
- **Gateway running time:** Per hour

**Typical costs:**
- Public IP: $3.60/month
- Load Balancer (Standard): $25/month + $0.005/GB processed
- Application Gateway (v2): $190/month + $0.008/GB processed
- VPN Gateway (Basic): $27/month

**Optimization levers:**
- Delete unused public IPs
- Use Private Endpoints instead of public access
- Reduce data egress (cache, CDN)
- Right-size gateways
</service_category>
</cost_drivers>

<optimization_strategies>
## Cost Optimization Strategies

<strategy name="Right-Sizing">
**Concept:** Use resources sized appropriately for actual workload, not "just in case" oversized resources.

**How to identify:**
```bash
# Check App Service Plan CPU/Memory usage (last 30 days)
az monitor metrics list \
  --resource <app-service-plan-id> \
  --metric "CpuPercentage,MemoryPercentage" \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation Average

# Check VM CPU usage
az monitor metrics list \
  --resource <vm-id> \
  --metric "Percentage CPU" \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation Average
```

**Decision criteria:**
- If average CPU < 30% and Memory < 50% for 30 days → **Downsize**
- If CPU > 80% frequently → **Upsize**
- If mixed pattern → **Consider auto-scaling**

**Example downsizing:**
```bash
# From P1v3 ($124/mo) to B1 ($13/mo) for dev environment
az appservice plan update \
  --name myapp-dev-asp \
  --resource-group myapp-dev-rg \
  --sku B1

# Savings: $111/month per plan
```

**Annual savings from right-sizing:**
- One oversized App Service Plan: ~$1,300/year
- Ten oversized VMs: ~$15,000/year
</strategy>

<strategy name="Reserved Instances & Savings Plans">
**Concept:** Commit to 1 or 3 years of usage for 40-72% discounts.

**When to use:**
- Workload runs 24/7 (production)
- Predictable usage for next 1-3 years
- Monthly cost > $100 (makes math work)

**Don't use for:**
- Dev/test environments (use auto-shutdown instead)
- Unpredictable workloads
- Short-lived projects

**Types:**

<type name="Reserved VM Instances">
**Savings:** 40-72% depending on term and upfront payment
**Flexibility:** Can change VM size within same family

**Example:**
```
Standard D4s v5 VM (4 vCores, 16 GB)
Pay-as-you-go: $140/month = $1,680/year
1-year reserved: $87/month = $1,044/year (38% savings)
3-year reserved: $58/month = $696/year (59% savings)
```

**Purchase:**
```bash
# Portal: Reservations → Purchase reservations → Virtual machines
# Select: Region, VM size, Term (1 or 3 years), Payment (all upfront, monthly)
```
</type>

<type name="App Service Reserved Capacity">
**Savings:** 55-78% for Premium v3
**Applies to:** Premium App Service Plans

**Example:**
```
P1v3 App Service Plan
Pay-as-you-go: $124/month
1-year reserved: $55/month (56% savings)
3-year reserved: $37/month (70% savings)
```

**Purchase:** Portal → App Service Plans → Buy reserved capacity
</type>

<type name="Azure SQL Reserved Capacity">
**Savings:** 33-80% depending on SKU and term

**Example:**
```
Standard S3 (100 DTUs)
Pay-as-you-go: $149/month
1-year reserved: $101/month (32% savings)
3-year reserved: $65/month (56% savings)
```
</type>

<type name="Azure Savings Plans">
**New option (2023+):** Commit to $/hour across multiple services
**Flexibility:** Applies to compute across regions, instance sizes, OS
**Savings:** Up to 65%

**Best for:** Mixed workloads across VMs, App Service, Container Instances
</type>

**ROI calculation:**
```
Annual spend on production VMs: $20,000
3-year reserved instance savings: 60%
Annual savings: $12,000
3-year savings: $36,000

Upfront cost (all upfront): $8,000/year = $24,000 total
Net 3-year savings: $36,000 - $24,000 = $12,000
```
</strategy>

<strategy name="Azure Hybrid Benefit">
**Concept:** Use existing on-premises Windows Server or SQL Server licenses in Azure.

**Eligible licenses:**
- Windows Server with Software Assurance
- SQL Server with Software Assurance

**Savings:**
- **Windows VMs:** Up to 40% savings
- **SQL Database:** Up to 55% savings
- **Combines with reserved instances** for maximum savings

**Example (SQL Database):**
```
Premium P2 SQL Database (250 DTUs)
Pay-as-you-go: $928/month
With Hybrid Benefit: $417/month (55% savings)
With Hybrid + 3-year RI: $194/month (79% total savings!)
```

**How to enable:**
```bash
# Enable for SQL Database
az sql db update \
  --resource-group myapp-prod-rg \
  --server shared-prod-sql-001 \
  --name myapp-prod-db \
  --license-type BasePrice  # BasePrice = Hybrid Benefit

# Enable for VM
az vm update \
  --resource-group myapp-prod-rg \
  --name myapp-vm \
  --license-type Windows_Server
```

**Important:** Must have eligible licenses with Software Assurance. Consult licensing team.
</strategy>

<strategy name="Auto-Shutdown for Dev/Test">
**Concept:** Automatically shut down non-production resources outside business hours.

**Savings:**
- Shut down 8 PM - 8 AM (12 hours) = 50% savings
- Shut down nights + weekends = 70% savings

**Example:**
```
Dev VM running 24/7: $140/month
With auto-shutdown (nights/weekends): $42/month
Savings: $98/month = $1,176/year per VM
```

**Implementation:**
```bash
# VM auto-shutdown (built-in)
az vm auto-shutdown \
  --resource-group myapp-dev-rg \
  --name myapp-dev-vm \
  --time 1900 \
  --timezone "Eastern Standard Time" \
  --email engineer@company.com

# App Service auto-shutdown (using Azure Automation)
# Or manually via Azure DevOps scheduled pipeline
az webapp stop --name myapp-dev-app --resource-group myapp-dev-rg
```

**Best practice:** Tag dev/test resources with `AutoShutdown=yes` and automate via policy.
</strategy>

<strategy name="Storage Lifecycle Policies">
**Concept:** Automatically move data to cheaper tiers as it ages.

**Tiers and costs:**
- Hot: $0.018/GB/month (frequent access)
- Cool: $0.010/GB/month (infrequent, 30+ days)
- Archive: $0.002/GB/month (rare access, 180+ days)

**Example policy:**
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

**Savings example:**
```
1 TB of logs stored in Hot tier: $18/month
Move to Cool after 30 days (average): $10/month
Savings: $8/month = $96/year per TB
```
</strategy>

<strategy name="Delete Unused Resources">
**Concept:** Identify and delete resources not attached or in use.

**Common waste:**

**Unattached disks:**
```bash
# Find unattached disks
az disk list --query "[?managedBy==null].[name, resourceGroup, diskSizeGb, sku.name]" -o table

# Cost: Premium SSD 128GB = $19/month each
# 10 unattached disks = $190/month waste = $2,280/year
```

**Unused public IPs:**
```bash
# Find unused IPs
az network public-ip list --query "[?ipConfiguration==null].[name, resourceGroup]" -o table

# Cost: $3.60/month each
# 20 unused IPs = $72/month = $864/year
```

**Old snapshots:**
```bash
# Find snapshots older than 90 days
az snapshot list --query "[?timeCreated < '$(date -u -d '90 days ago' +%Y-%m-%d)'].[name, resourceGroup, diskSizeGb]" -o table

# Cost: Premium snapshot 500GB = $9/month
# 30 old snapshots = $270/month = $3,240/year
```

**Old VM images:**
```bash
az image list --query "[?timeCreated < '2024-01-01'].[name, resourceGroup]" -o table
```

**Cleanup script:**
```bash
#!/bin/bash
# cleanup-unused-resources.sh

echo "=== Finding Unused Resources ==="

# Unattached disks
echo "Unattached disks:"
az disk list --query "[?managedBy==null].[name, resourceGroup]" -o tsv | while read name rg; do
  echo "Would delete disk: $name in $rg"
  # Uncomment to actually delete:
  # az disk delete --name "$name" --resource-group "$rg" --yes --no-wait
done

# Unused public IPs
echo "Unused public IPs:"
az network public-ip list --query "[?ipConfiguration==null].[name, resourceGroup]" -o tsv | while read name rg; do
  echo "Would delete IP: $name in $rg"
  # az network public-ip delete --name "$name" --resource-group "$rg" --yes
done

# Old snapshots (>90 days)
echo "Old snapshots:"
CUTOFF_DATE=$(date -u -d '90 days ago' +%Y-%m-%d)
az snapshot list --query "[?timeCreated < '$CUTOFF_DATE'].[name, resourceGroup]" -o tsv | while read name rg; do
  echo "Would delete snapshot: $name in $rg"
  # az snapshot delete --name "$name" --resource-group "$rg" --yes
done
```

**Run monthly via scheduled pipeline or Azure Automation.**
</strategy>

<strategy name="Tagging for Cost Allocation">
**Concept:** Tag every resource with owner, product, cost center for chargeback.

**Required tags:**
```bicep
tags: {
  Environment: 'prod'              // dev/staging/prod
  Product: 'myapp'                 // Product name
  CostCenter: 'Engineering-MyApp'  // For finance chargeback
  Owner: 'team@company.com'        // Who to contact
  Project: 'ProjectAlpha'          // Optional: project code
}
```

**Enforce with Azure Policy:**
```bicep
resource requireTagsPolicy 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-cost-tags'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/require-tag'
    displayName: 'Require CostCenter tag'
    parameters: {
      tagName: {
        value: 'CostCenter'
      }
    }
  }
}
```

**Cost queries by tag:**
```bash
# Get costs by Product tag
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="TagKey" type="Dimension" \
  --query "properties.rows[?contains(@[2], 'Product')]"

# Get costs for specific product
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-filter '{"tags":{"name":"Product","operator":"In","values":["myapp"]}}' \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}'
```
</strategy>
</optimization_strategies>

<budgets_and_alerts>
## Budgets and Cost Alerts

### Create Budget

```bash
az consumption budget create \
  --budget-name "myapp-monthly-budget" \
  --amount 1000 \
  --time-grain Monthly \
  --start-date 2025-01-01 \
  --end-date 2025-12-31 \
  --resource-group myapp-prod-rg \
  --notifications '{
    "actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["team@company.com"],
      "thresholdType": "Actual"
    },
    "actual_GreaterThan_100_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["team@company.com", "finance@company.com"],
      "thresholdType": "Actual"
    },
    "forecasted_GreaterThan_110_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 110,
      "contactEmails": ["team@company.com"],
      "thresholdType": "Forecasted"
    }
  }'
```

**Budget hierarchy:**
- **Management group level:** Entire organization
- **Subscription level:** Per subscription
- **Resource group level:** Per product/team

**Notification types:**
- **Actual:** Alert when actual spending reaches threshold
- **Forecasted:** Alert when Azure predicts spending will exceed threshold

### Automated Actions on Budget

Use Azure Automation to take action when budget exceeded:

```bicep
resource budgetAlert 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'budget-exceeded-action'
  location: 'global'
  properties: {
    groupShortName: 'BudgetAlert'
    enabled: true
    emailReceivers: [
      {
        name: 'Team Email'
        emailAddress: 'team@company.com'
      }
    ]
    automationRunbookReceivers: [
      {
        automationAccountId: automationAccount.id
        runbookName: 'shutdown-dev-resources'
        webhookResourceId: webhook.id
        isGlobalRunbook: false
      }
    ]
  }
}
```

**Runbook example (auto-shutdown dev VMs on budget exceeded):**
```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName
)

# Stop all VMs in dev resource group
$VMs = Get-AzVM -ResourceGroupName $ResourceGroupName
foreach ($VM in $VMs) {
    Write-Output "Stopping VM: $($VM.Name)"
    Stop-AzVM -ResourceGroupName $ResourceGroupName -Name $VM.Name -Force
}
```
</budgets_and_alerts>

<cost_monitoring>
## Cost Monitoring and Reporting

### Azure Cost Management Tools

**Azure Cost Management + Billing:**
- Portal → Cost Management + Billing
- Cost analysis by service, resource, tag
- Cost forecasting
- Budget alerts
- Recommendations

**Azure Advisor (Cost tab):**
```bash
# Get cost recommendations
az advisor recommendation list \
  --category Cost \
  --output table

# Common recommendations:
# - Right-size or shutdown underutilized VMs
# - Buy reserved instances
# - Delete unattached disks
# - Use lifecycle management for storage
```

### Custom Cost Dashboard

Create Power BI dashboard with Cost Management data:

1. Export cost data to Storage Account
2. Connect Power BI to storage
3. Create visualizations:
   - Cost by product (pie chart)
   - Trend over time (line chart)
   - Top 10 resources (bar chart)
   - Budget vs actual (gauge)

### Automated Monthly Cost Report

```bash
#!/bin/bash
# monthly-cost-report.sh

REPORT_MONTH=$(date +%Y-%m)
OUTPUT_FILE="cost-report-${REPORT_MONTH}.json"

# Get costs for current month
az costmanagement query \
  --type Usage \
  --timeframe MonthToDate \
  --dataset-aggregation '{"totalCost":{"name":"Cost","function":"Sum"}}' \
  --dataset-grouping name="ResourceGroup" type="Dimension" \
  > "$OUTPUT_FILE"

# Parse and calculate
TOTAL=$(jq '.properties.rows | map(.[0]) | add' "$OUTPUT_FILE")
PREV_MONTH_TOTAL=5000  # Get from last month's report

# Calculate change
CHANGE=$(echo "scale=2; (($TOTAL - $PREV_MONTH_TOTAL) / $PREV_MONTH_TOTAL) * 100" | bc)

# Email report
cat <<EOF | mail -s "Azure Cost Report - $REPORT_MONTH" team@company.com
Azure Cost Report for $REPORT_MONTH

Total: \$$TOTAL
Previous month: \$$PREV_MONTH_TOTAL
Change: $CHANGE%

Top 5 resource groups by cost:
$(jq -r '.properties.rows | sort_by(.[0]) | reverse | .[0:5] | .[] | "\(.[1]): $\(.[0])"' "$OUTPUT_FILE")

Full report attached.
EOF
```

Schedule via Azure DevOps pipeline (run on 1st of month) or Azure Automation.
</cost_monitoring>

<anti_patterns>
**Avoid:**

<anti_pattern name="Set and Forget">
**Problem:** Create resources and never review costs again

**Why bad:** Costs accumulate, waste grows, bill shock

**Instead:** Monthly cost reviews, automated optimization, continuous monitoring
</anti_pattern>

<anti_pattern name="Over-Optimizing Production">
**Problem:** Cutting costs so aggressively that reliability suffers

**Why bad:** Downtime costs more than savings, customer trust lost

**Instead:** Focus on dev/test optimization, use appropriate production SKUs
</anti_pattern>

<anti_pattern name="No Tagging Strategy">
**Problem:** Can't tell which team/product owns resources

**Why bad:** Can't allocate costs, no accountability

**Instead:** Enforce tagging with Azure Policy, tag all resources on creation
</anti_pattern>

<anti_pattern name="Ignoring Azure Advisor">
**Problem:** Not implementing free recommendations

**Why bad:** Leaving money on table, Azure tells you how to save

**Instead:** Review Advisor monthly, implement recommendations
</anti_pattern>

<anti_pattern name="Running Dev 24/7">
**Problem:** Dev and test environments run all night and weekends

**Why bad:** Waste 70% of compute costs on unused resources

**Instead:** Auto-shutdown, use lower SKUs, delete when not needed
</anti_pattern>
</anti_patterns>

<quick_wins>
## Quick Cost Optimization Wins

**Week 1: Clean up waste (2-4 hours work, $200-500/month savings)**
1. Delete unattached disks
2. Delete unused public IPs
3. Delete old snapshots
4. Delete old VM images

**Week 2: Right-size dev environments (4-6 hours, $300-800/month)**
1. Downgrade dev App Service Plans to B1
2. Downgrade dev SQL Databases to Basic or S0
3. Reduce dev VM sizes

**Week 3: Implement auto-shutdown (2-3 hours, $500-1000/month)**
1. Enable VM auto-shutdown for all dev/test
2. Create pipeline to stop App Services nights/weekends

**Week 4: Storage optimization (2-3 hours, $100-300/month)**
1. Implement lifecycle policies
2. Move old backups to Archive tier
3. Delete backups older than retention policy

**Month 2: Reserved instances (1-2 hours research, $1000+/month)**
1. Identify production VMs/App Services running 24/7
2. Calculate ROI for 1-year reserved instances
3. Purchase reservations for high-cost resources

**Typical results:**
- Month 1: 15-25% cost reduction
- Month 3: 30-40% cost reduction
- Month 6: 40-50% cost reduction (with reserved instances)
</quick_wins>
