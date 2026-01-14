# Workflow: Optimize Costs

<required_reading>
**Read these reference files NOW before optimizing costs:**
1. references/cost-optimization.md
2. references/compute-services.md
3. references/storage-data.md
</required_reading>

<process>
## Step 1: Analyze Current Spending

### Get Cost Overview:

```bash
# Current month costs by resource group
az costmanagement query \
  --type Usage \
  --dataset-aggregation '{\"totalCost\":{\"name\":\"Cost\",\"function\":\"Sum\"}}' \
  --dataset-grouping name="ResourceGroup" type="Dimension" \
  --timeframe MonthToDate

# Costs by resource type
az costmanagement query \
  --type Usage \
  --dataset-aggregation '{\"totalCost\":{\"name\":\"Cost\",\"function\":\"Sum\"}}' \
  --dataset-grouping name="ResourceType" type="Dimension" \
  --timeframe MonthToDate

# Top 10 most expensive resources
az costmanagement query \
  --type Usage \
  --dataset-aggregation '{\"totalCost\":{\"name\":\"Cost\",\"function\":\"Sum\"}}' \
  --dataset-grouping name="ResourceId" type="Dimension" \
  --timeframe MonthToDate \
  --query "properties.rows" \
  -o json | jq 'sort_by(.[0]) | reverse | .[0:10]'
```

### Use Azure Cost Management in Portal:
1. Go to Cost Management + Billing
2. Cost analysis
3. Group by: Resource, Service, Location
4. Identify top cost drivers

## Step 2: Identify Quick Wins

### Check for Unattached Resources:

```bash
# Find unattached disks (paying for storage not in use)
az disk list --query "[?managedBy==null].[name, resourceGroup, diskSizeGb, sku.name]" -o table

# Delete unattached disks (after verification!)
for disk in $(az disk list --query "[?managedBy==null].id" -o tsv); do
  echo "Would delete: $disk"
  # az disk delete --ids $disk --yes --no-wait
done

# Find unused public IPs
az network public-ip list --query "[?ipConfiguration==null].[name, resourceGroup]" -o table

# Find idle VMs (running but low CPU)
az monitor metrics list \
  --resource <vm-id> \
  --metric "Percentage CPU" \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation Average

# Find old snapshots
az snapshot list --query "[?timeCreated < '2024-01-01'].[name, resourceGroup, diskSizeGb]" -o table
```

### Implement Auto-Shutdown for Dev/Test:

```bash
# Configure auto-shutdown for VM
az vm auto-shutdown \
  --name <vm-name> \
  --resource-group <rg-name> \
  --time 1900 \
  --timezone "Eastern Standard Time" \
  --email <email@company.com>

# Or use Bicep for App Services
resource autoShutdown 'Microsoft.DevTestLab/schedules@2018-09-15' = {
  name: 'shutdown-computevm-${vmName}'
  location: location
  properties: {
    status: 'Enabled'
    taskType: 'ComputeVmShutdownTask'
    dailyRecurrence: {
      time: '1900'
    }
    timeZoneId: 'Eastern Standard Time'
    targetResourceId: vm.id
    notificationSettings: {
      status: 'Enabled'
      emailRecipient: 'team@company.com'
      timeInMinutes: 30
    }
  }
}
```

## Step 3: Right-Size Resources

### Analyze App Service Plans:

```bash
# Get App Service Plan metrics
az monitor metrics list \
  --resource <asp-id> \
  --metric "CpuPercentage,MemoryPercentage" \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation Average

# If average CPU < 30% and Memory < 50%, downgrade SKU
```

**Downgrade App Service Plan** (if underutilized):

```bash
# From P1v3 ($124/mo) to B1 ($13/mo) for dev environment
az appservice plan update \
  --name myapp-dev-eastus-asp-001 \
  --resource-group myapp-dev-eastus-rg \
  --sku B1

# From Standard S3 SQL Database to S1 (if low DTU usage)
az sql db update \
  --name myapp-dev-db \
  --server shared-dev-eastus-sql-001 \
  --resource-group shared-dev-eastus-rg \
  --service-objective S1
```

### Use Azure Advisor Recommendations:

```bash
# Get cost recommendations from Azure Advisor
az advisor recommendation list \
  --category Cost \
  --query "[].{Name:name, Impact:impact, Problem:shortDescription.problem, Solution:shortDescription.solution}" \
  -o table
```

Implement each recommendation.

## Step 4: Implement Reserved Instances

For production workloads that run 24/7:

### Calculate Savings:

```bash
# Get VM costs over last 30 days
az costmanagement query \
  --type Usage \
  --dataset-filter '{\"dimensions\":{\"name\":\"ResourceType\",\"operator\":\"In\",\"values\":[\"Microsoft.Compute/virtualMachines\"]}}' \
  --timeframe Custom \
  --time-period from='2024-12-01' to='2024-12-31' \
  --dataset-aggregation '{\"totalCost\":{\"name\":\"Cost\",\"function\":\"Sum\"}}'
```

### Purchase Reserved Instance (if cost > $100/mo):

```bash
# List available reservations
az reservations catalog show \
  --subscription-id <subscription-id> \
  --reserved-resource-type VirtualMachines \
  --location eastus

# Purchase 1-year reservation (save ~40%)
az reservations reservation-order purchase \
  --reservation-order-id <order-id> \
  --sku <sku-name> \
  --location eastus \
  --reserved-resource-type VirtualMachines \
  --quantity 1 \
  --term P1Y
```

**For App Services**, use Reserved Capacity (save 50-75%):
Portal → App Service Plans → Buy reserved capacity

## Step 5: Optimize Storage

### Implement Lifecycle Policies:

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: 'mystorageaccount'
}

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
            }
          }
        }
      ]
    }
  }
}
```

### Delete Old Backups:

```bash
# List recovery points older than 90 days
az backup recoverypoint list \
  --resource-group <rg-name> \
  --vault-name <vault-name> \
  --container-name <container> \
  --item-name <item> \
  --query "[?properties.recoveryPointTime < '2024-10-01']"

# Update backup policy retention
az backup policy set \
  --resource-group <rg-name> \
  --vault-name <vault-name> \
  --policy <policy-json> \
  --name <policy-name>
```

## Step 6: Optimize Networking

### Delete Unused Resources:

```bash
# Find unused network interfaces
az network nic list --query "[?virtualMachine==null].[name, resourceGroup]" -o table

# Find unused load balancers
az network lb list --query "[?backendAddressPools[0].backendIPConfigurations==null].[name, resourceGroup]" -o table

# Find unused Application Gateways
az network application-gateway list --query "[?backendAddressPools[0].backendAddresses==null].[name, resourceGroup]" -o table
```

### Use Private Endpoints instead of Public IPs:

Public IPs cost $0.005/hour = $3.60/month each. Remove when possible.

```bash
# Delete public IP (after moving to private endpoint)
az network public-ip delete \
  --name <ip-name> \
  --resource-group <rg-name>
```

## Step 7: Implement Budgets and Alerts

### Create Budget:

```bash
az consumption budget create \
  --budget-name "monthly-budget" \
  --amount 1000 \
  --time-grain Monthly \
  --start-date 2025-01-01 \
  --end-date 2025-12-31 \
  --resource-group <rg-name> \
  --notifications \
    '{"actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["team@company.com"],
      "contactRoles": ["Owner"],
      "thresholdType": "Actual"
    }}' \
    '{"actual_GreaterThan_100_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["team@company.com", "finance@company.com"],
      "contactRoles": ["Owner"],
      "thresholdType": "Actual"
    }}'
```

### Create Cost Alerts:

```bash
az monitor metrics alert create \
  --name "high-cost-alert" \
  --resource-group <rg-name> \
  --scopes <resource-id> \
  --condition "total Cost > 100" \
  --window-size 1d \
  --evaluation-frequency 1h \
  --action <action-group-id>
```

## Step 8: Tagging for Cost Allocation

Ensure all resources are tagged:

```bash
# Tag resource group (tags inherit to resources)
az group update \
  --name myapp-prod-eastus-rg \
  --tags Environment=prod Product=myapp CostCenter=Engineering Owner=team@company.com

# Tag individual resource
az resource tag \
  --resource-group <rg-name> \
  --name <resource-name> \
  --resource-type <resource-type> \
  --tags Project=ProjectA Department=Engineering
```

Use Azure Policy to enforce tagging:

```bicep
resource tagPolicy 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-tags'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/require-tag'
    parameters: {
      tagName: {
        value: 'CostCenter'
      }
    }
  }
}
```

## Step 9: Generate Cost Reports

Create monthly cost report:

```bash
#!/bin/bash
# cost-report.sh

REPORT_DATE=$(date +%Y-%m)
OUTPUT_FILE="cost-report-${REPORT_DATE}.json"

az costmanagement query \
  --type Usage \
  --timeframe Custom \
  --time-period from="$(date +%Y-%m-01)" to="$(date +%Y-%m-%d)" \
  --dataset-aggregation '{\"totalCost\":{\"name\":\"Cost\",\"function\":\"Sum\"}}' \
  --dataset-grouping name="ResourceGroup" type="Dimension" \
  > "$OUTPUT_FILE"

echo "Cost report saved to $OUTPUT_FILE"

# Parse and send email (requires mail setup)
TOTAL=$(jq '.properties.rows | map(.[0]) | add' "$OUTPUT_FILE")
echo "Total cost for $REPORT_DATE: \$$TOTAL" | mail -s "Azure Cost Report" team@company.com
```

Schedule this to run monthly via cron or Azure Automation.

## Step 10: Document Savings

Create cost optimization tracking sheet:

```markdown
# Cost Optimization Results

## January 2025

### Actions Taken:
1. Deleted 15 unattached disks → **$75/month saved**
2. Downgraded dev App Service Plans B1 → **$222/month saved**
3. Implemented storage lifecycle policies → **$45/month saved**
4. Purchased App Service Reserved Capacity → **$620/month saved**
5. Deleted 8 unused public IPs → **$29/month saved**

**Total Monthly Savings: $991**
**Annual Savings: $11,892**

### Current Monthly Cost: $2,450
### Previous Monthly Cost: $3,441
### Reduction: 28.8%

### Next Actions:
- [ ] Review SQL Database DTU usage (potential $200/mo savings)
- [ ] Implement auto-shutdown for staging VMs
- [ ] Migrate blob storage to cool tier
```

Share results with stakeholders monthly.
</process>

<anti_patterns>
**Avoid:**

- **Deleting resources without verification**: Always confirm resources are truly unused before deleting.

- **Over-optimizing production**: Don't sacrifice reliability for cost savings in production.

- **Ignoring Reserved Instances**: For 24/7 workloads, RIs save 40-75%. Not using them wastes money.

- **No cost monitoring**: Set up budgets and alerts. Don't wait for bill shock.

- **Using Free/Basic tiers in production**: These lack SLA and features. False economy.

- **No tagging strategy**: Can't allocate costs without tags.

- **Optimizing once and forgetting**: Cost optimization is continuous, not one-time.

- **Not tracking savings**: Document what you've saved to show value.

- **Keeping old snapshots forever**: Implement retention policies.

- **Running dev/test 24/7**: Auto-shutdown saves 70% on non-production resources.

- **Ignoring Azure Advisor**: Free recommendations from Azure. Use them.
</anti_patterns>

<success_criteria>
Successful cost optimization includes:

- Current spending analyzed and top cost drivers identified
- All unattached resources reviewed and cleaned up
- Auto-shutdown configured for dev/test resources
- Resources right-sized based on actual usage
- Azure Advisor recommendations reviewed and implemented
- Reserved Instances purchased for 24/7 production workloads
- Storage lifecycle policies implemented
- Unused networking resources deleted
- Budgets and alerts configured
- All resources properly tagged for cost allocation
- Monthly cost reports automated
- Savings documented and tracked
- At least 15-30% cost reduction achieved
- No impact to production reliability
- Optimization process documented for ongoing use
</success_criteria>
