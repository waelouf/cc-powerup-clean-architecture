<required_reading>
Before implementing disaster recovery:
- `references/disaster-recovery.md` - RTO/RPO, backup strategies, failover patterns
- `references/storage-data.md` - Geo-redundancy for SQL and Storage
- `references/kubernetes-aks.md` - AKS disaster recovery if using Kubernetes
</required_reading>

<objective>
Implement Azure disaster recovery and business continuity using Azure Backup, Site Recovery, geo-redundancy, and multi-region failover strategies.
</objective>

<quick_start>
**5-minute basic backup:**
```bash
# Create Recovery Services Vault
az backup vault create \
  --resource-group myapp-prod-eastus-rg \
  --name rsv-myapp-prod-eastus \
  --location eastus

# Enable backup for VM
az backup protection enable-for-vm \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --vm vm-myapp-prod-eastus-001 \
  --policy-name DefaultPolicy

# Enable geo-redundant SQL Database
az sql db update \
  --resource-group myapp-prod-eastus-rg \
  --server sql-myapp-prod-eastus \
  --name mydb \
  --backup-storage-redundancy Geo

# Manual backup
az backup protection backup-now \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --container-name <container-name> \
  --item-name vm-myapp-prod-eastus-001 \
  --retain-until 30-01-2026
```
</quick_start>

<process>
## Step 1: Define RTO and RPO

**1.1 Understand requirements:**

```
RTO (Recovery Time Objective):
- How long can system be down?
- Tier 1 (Critical): < 1 hour
- Tier 2 (Important): < 4 hours
- Tier 3 (Standard): < 24 hours

RPO (Recovery Point Objective):
- How much data loss is acceptable?
- Tier 1 (Critical): < 5 minutes
- Tier 2 (Important): < 1 hour
- Tier 3 (Standard): < 24 hours

Cost vs. RTO/RPO:
- Lower RTO/RPO = Higher cost
- Choose based on business impact
```

**1.2 DR strategy selection:**

```
Strategy             RTO        RPO        Cost    Use Case
Backup/Restore       Hours      Hours      $       Non-critical apps
Pilot Light          < 1 hour   Minutes    $$      Important apps
Warm Standby         Minutes    Seconds    $$$     Critical apps
Hot Standby (Active) Seconds    Near-zero  $$$$    Mission-critical
```

## Step 2: Azure Backup

**2.1 Create Recovery Services Vault:**

```bicep
resource recoveryServicesVault 'Microsoft.RecoveryServices/vaults@2024-04-01' = {
  name: 'rsv-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'RS0'
    tier: 'Standard'
  }
  properties: {}
}

// Backup policy
resource backupPolicy 'Microsoft.RecoveryServices/vaults/backupPolicies@2024-04-01' = {
  parent: recoveryServicesVault
  name: 'DailyBackupPolicy'
  properties: {
    backupManagementType: 'AzureIaasVM'
    schedulePolicy: {
      schedulePolicyType: 'SimpleSchedulePolicy'
      scheduleRunFrequency: 'Daily'
      scheduleRunTimes: ['2025-01-11T02:00:00Z']  // 2 AM UTC
    }
    retentionPolicy: {
      retentionPolicyType: 'LongTermRetentionPolicy'
      dailySchedule: {
        retentionTimes: ['2025-01-11T02:00:00Z']
        retentionDuration: {
          count: 30
          durationType: 'Days'
        }
      }
      weeklySchedule: {
        daysOfTheWeek: ['Sunday']
        retentionTimes: ['2025-01-11T02:00:00Z']
        retentionDuration: {
          count: 12
          durationType: 'Weeks'
        }
      }
      monthlySchedule: {
        retentionScheduleFormatType: 'Weekly'
        retentionScheduleWeekly: {
          daysOfTheWeek: ['Sunday']
          weeksOfTheMonth: ['First']
        }
        retentionTimes: ['2025-01-11T02:00:00Z']
        retentionDuration: {
          count: 12
          durationType: 'Months'
        }
      }
    }
  }
}
```

**2.2 Enable backup for resources:**

```bash
# VM backup
az backup protection enable-for-vm \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --vm vm-myapp-prod-eastus-001 \
  --policy-name DailyBackupPolicy

# Check backup status
az backup job list \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --output table

# List recovery points
az backup recoverypoint list \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --container-name <container> \
  --item-name vm-myapp-prod-eastus-001 \
  --output table
```

**2.3 Restore from backup:**

```bash
# Restore VM
az backup restore restore-disks \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-eastus \
  --container-name <container> \
  --item-name vm-myapp-prod-eastus-001 \
  --rp-name <recovery-point-name> \
  --storage-account stbackupprodeastus \
  --restore-to-staging-storage-account
```

## Step 3: SQL Database Geo-Redundancy

**3.1 Active geo-replication:**

```bicep
// Primary SQL Server (East US)
resource sqlServerPrimary 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    administratorLogin: 'sqladmin'
    minimalTlsVersion: '1.2'
  }
}

resource sqlDatabasePrimary 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServerPrimary
  name: 'mydb'
  location: 'eastus'
  sku: {
    name: 'S3'
    tier: 'Standard'
  }
  properties: {
    requestedBackupStorageRedundancy: 'Geo'  // Geo-redundant backups
  }
}

// Secondary SQL Server (West US 2)
resource sqlServerSecondary 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-myapp-prod-westus2'
  location: 'westus2'
  properties: {
    administratorLogin: 'sqladmin'
    minimalTlsVersion: '1.2'
  }
}

// Geo-replication
resource geoReplication 'Microsoft.Sql/servers/databases/replicationLinks@2023-08-01-preview' = {
  name: '${sqlServerSecondary.name}/mydb/link'
  properties: {
    partnerDatabase: sqlDatabasePrimary.id
    partnerLocation: 'eastus'
    readableSecondary: 'Enabled'  // Can read from secondary
  }
}
```

**3.2 Failover groups (recommended):**

```bicep
resource failoverGroup 'Microsoft.Sql/servers/failoverGroups@2023-08-01-preview' = {
  parent: sqlServerPrimary
  name: 'fg-myapp-prod'
  properties: {
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60  // Wait 60 min before forced failover
    }
    readOnlyEndpoint: {
      failoverPolicy: 'Disabled'
    }
    partnerServers: [
      {
        id: sqlServerSecondary.id
      }
    ]
    databases: [
      sqlDatabasePrimary.id
    ]
  }
}

output failoverGroupEndpoint string = '${failoverGroup.name}.database.windows.net'
// App always connects to: fg-myapp-prod.database.windows.net
// Azure automatically routes to primary
```

**3.3 Test failover:**

```bash
# Initiate failover
az sql failover-group set-primary \
  --name fg-myapp-prod \
  --resource-group myapp-prod-eastus-rg \
  --server sql-myapp-prod-westus2

# Failback
az sql failover-group set-primary \
  --name fg-myapp-prod \
  --resource-group myapp-prod-westus2-rg \
  --server sql-myapp-prod-eastus
```

## Step 4: Storage Geo-Redundancy

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'stmyappprodeastus'
  location: 'eastus'
  sku: {
    name: 'Standard_GRS'  // Geo-redundant storage
    // Options:
    // - Standard_LRS: Local redundant (single datacenter)
    // - Standard_ZRS: Zone redundant (3 zones, same region)
    // - Standard_GRS: Geo redundant (paired region, async)
    // - Standard_GZRS: Geo-zone redundant (best HA)
    // - Standard_RAGRS: Read-access geo redundant
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}
```

**Initiate account failover (rare, only in disaster):**

```bash
# Check if failover is available
az storage account show \
  --name stmyappprodeastus \
  --query "{Name:name, PrimaryLocation:primaryLocation, SecondaryLocation:secondaryLocation, StatusOfPrimary:statusOfPrimary}"

# Initiate failover (makes secondary the new primary)
az storage account failover \
  --name stmyappprodeastus \
  --resource-group myapp-prod-eastus-rg \
  --yes
```

## Step 5: Multi-Region Deployment

**5.1 Traffic Manager (DNS-based):**

```bicep
resource trafficManager 'Microsoft.Network/trafficManagerProfiles@2022-04-01' = {
  name: 'tm-myapp-prod'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Priority'  // Priority, Performance, Geographic, Weighted
    dnsConfig: {
      relativeName: 'myapp-prod'
      ttl: 30  // 30 seconds DNS TTL
    }
    monitorConfig: {
      protocol: 'HTTPS'
      port: 443
      path: '/health'
      intervalInSeconds: 30
      toleratedNumberOfFailures: 3
      timeoutInSeconds: 10
    }
    endpoints: [
      {
        name: 'primary-eastus'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: appServiceEastUS.id
          endpointStatus: 'Enabled'
          priority: 1  // Primary
        }
      }
      {
        name: 'secondary-westus2'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: appServiceWestUS2.id
          endpointStatus: 'Enabled'
          priority: 2  // Failover
        }
      }
    ]
  }
}

output trafficManagerUrl string = 'https://myapp-prod.trafficmanager.net'
```

**5.2 Front Door (global load balancer):**

```bicep
resource frontDoor 'Microsoft.Cdn/profiles@2024-02-01' = {
  name: 'fd-myapp-prod'
  location: 'global'
  sku: {
    name: 'Premium_AzureFrontDoor'
  }
}

resource endpoint 'Microsoft.Cdn/profiles/afdEndpoints@2024-02-01' = {
  parent: frontDoor
  name: 'myapp-prod'
  location: 'global'
  properties: {
    enabledState: 'Enabled'
  }
}

resource originGroup 'Microsoft.Cdn/profiles/originGroups@2024-02-01' = {
  parent: frontDoor
  name: 'myapp-origins'
  properties: {
    loadBalancingSettings: {
      sampleSize: 4
      successfulSamplesRequired: 3
    }
    healthProbeSettings: {
      probePath: '/health'
      probeRequestType: 'GET'
      probeProtocol: 'Https'
      probeIntervalInSeconds: 30
    }
  }
}

resource originEastUS 'Microsoft.Cdn/profiles/originGroups/origins@2024-02-01' = {
  parent: originGroup
  name: 'eastus'
  properties: {
    hostName: 'app-myapp-prod-eastus.azurewebsites.net'
    httpPort: 80
    httpsPort: 443
    priority: 1
    weight: 1000
  }
}

resource originWestUS2 'Microsoft.Cdn/profiles/originGroups/origins@2024-02-01' = {
  parent: originGroup
  name: 'westus2'
  properties: {
    hostName: 'app-myapp-prod-westus2.azurewebsites.net'
    httpPort: 80
    httpsPort: 443
    priority: 2  // Failover
    weight: 1000
  }
}
```

## Step 6: DR Testing

**6.1 Create DR runbook:**

```markdown
# Disaster Recovery Runbook

## Scenario 1: Primary Region Failure

1. **Detect:**
   - Azure health alert triggers
   - Traffic Manager marks primary unhealthy
   - Monitoring shows 100% failure rate

2. **Validate:**
   - Check Azure status page
   - Verify secondary region is healthy
   - Confirm data replication is current

3. **Failover SQL Database:**
   ```bash
   az sql failover-group set-primary \
     --name fg-myapp-prod \
     --server sql-myapp-prod-westus2
   ```

4. **Update DNS (if not using Traffic Manager):**
   ```bash
   # Update A record to point to West US 2
   ```

5. **Verify:**
   - Test application in secondary region
   - Check data integrity
   - Monitor error rates

6. **Communicate:**
   - Notify stakeholders
   - Update status page
   - Post incident report

## Scenario 2: Data Corruption

1. **Identify:**
   - Timestamp of corruption
   - Affected data scope

2. **Restore:**
   ```bash
   # Restore from point-in-time before corruption
   az sql db restore \
     --dest-name mydb-restored \
     --server sql-myapp-prod-eastus \
     --time "2025-01-11T10:00:00Z"
   ```

3. **Validate:**
   - Verify restored data
   - Compare with production

4. **Cutover:**
   - Rename databases
   - Update connection strings
```

**6.2 Test schedule:**

```
Monthly: Test backups (restore to test environment)
Quarterly: Failover test (non-production hours)
Annually: Full DR drill (with communication plan)
```

**6.3 Execute test failover:**

```bash
# 1. Document current state
az resource list --resource-group myapp-prod-eastus-rg > pre-test-state.json

# 2. Initiate SQL failover
az sql failover-group set-primary \
  --name fg-myapp-prod \
  --server sql-myapp-prod-westus2

# 3. Verify traffic routing
curl https://myapp-prod.trafficmanager.net/health

# 4. Monitor metrics
# Check Application Insights for errors

# 5. Failback
az sql failover-group set-primary \
  --name fg-myapp-prod \
  --server sql-myapp-prod-eastus

# 6. Document results
# - RTO achieved: X minutes
# - RPO achieved: Y minutes
# - Issues encountered: ...
```
</process>

<dr_checklist>
## Disaster Recovery Checklist

- [ ] RTO and RPO defined for each service tier
- [ ] Recovery Services Vault created
- [ ] Backup policies configured
- [ ] Automated daily backups running
- [ ] SQL Database geo-replication enabled
- [ ] Failover groups configured (not just replication)
- [ ] Storage accounts using GRS or GZRS
- [ ] Multi-region deployment completed
- [ ] Traffic Manager or Front Door configured
- [ ] Health checks configured for all endpoints
- [ ] DR runbook documented
- [ ] DR testing performed quarterly
- [ ] Backup restoration tested monthly
- [ ] Monitoring alerts for backup failures
- [ ] Communication plan for incidents
</dr_checklist>

<cost_estimation>
**Disaster Recovery costs:**

```
Azure Backup:
- Protected instances: $10/month per VM
- Storage: $0.10/GB/month

SQL Geo-Replication:
- Secondary database: 100% of primary cost
- S3 primary + S3 secondary = $200/month

Storage GRS:
- Premium: $0.04/GB/month (vs $0.02 for LRS)
- 1 TB = $20/month additional

Traffic Manager:
- $0.54/million DNS queries
- Health checks: Free
- Typical cost: $5-20/month

Multi-region App Service:
- 2× regions = 2× cost
- P2v3 × 2 = $584/month
```
</cost_estimation>

<success_criteria>
Disaster recovery is properly implemented when:
- Automated backups running daily for all critical data
- Backup restoration tested and verified monthly
- SQL failover groups configured with < 60 min RTO
- Storage accounts using geo-redundant options
- Multi-region deployment with automatic failover
- DR runbook documented and accessible
- DR test performed with results documented
- RTO/RPO targets met in testing
- Monitoring alerts configured for DR components
</success_criteria>
