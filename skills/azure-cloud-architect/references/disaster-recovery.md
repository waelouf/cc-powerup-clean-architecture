<overview>
Comprehensive guide to Azure disaster recovery and business continuity. Covers Azure Backup, Site Recovery, geo-redundancy, RTO/RPO strategies, and DR best practices for 2024-2025.
</overview>

<dr_fundamentals>
## Disaster Recovery Concepts

<concept name="RTO (Recovery Time Objective)">
**What:** Maximum acceptable downtime after a disaster.

**Examples:**
- Tier 1 (Critical): RTO < 1 hour
- Tier 2 (Important): RTO < 4 hours
- Tier 3 (Standard): RTO < 24 hours

**Affects:**
- DR strategy choice
- Cost (lower RTO = higher cost)
- Complexity

**Example:** E-commerce site RTO = 1 hour (lose $X revenue per hour down)
</concept>

<concept name="RPO (Recovery Point Objective)">
**What:** Maximum acceptable data loss (time between last backup and disaster).

**Examples:**
- Tier 1: RPO < 15 minutes
- Tier 2: RPO < 1 hour
- Tier 3: RPO < 24 hours

**Affects:**
- Backup frequency
- Replication strategy
- Cost

**Example:** Financial system RPO = 0 (zero data loss acceptable)
</concept>

<concept name="Availability Zones">
**What:** Physically separate datacenters within same region.

**Benefits:**
- Protects against datacenter failures
- Low latency between zones
- Same region (data sovereignty)

**Use for:** High availability within region

**Example:**
```
Region: East US
├── Zone 1: Datacenter A
├── Zone 2: Datacenter B
└── Zone 3: Datacenter C
```
</concept>

<concept name="Geo-Redundancy">
**What:** Data replicated to secondary region (hundreds of miles away).

**Benefits:**
- Protects against regional disasters
- Compliance (geographic backup)

**Tradeoffs:**
- Higher cost (2x data storage)
- Latency (for reads from secondary)

**Use for:** Critical data, regulatory requirements
</concept>
</dr_fundamentals>

<azure_backup>
## Azure Backup

**What:** Managed backup service for VMs, databases, file shares.

**Backup targets:**
- Azure VMs
- SQL Server in VMs
- Azure Files
- On-premises (via agent)

### Recovery Services Vault

```bicep
resource recoveryVault 'Microsoft.RecoveryServices/vaults@2023-06-01' = {
  name: 'rsv-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'RS0'
    tier: 'Standard'
  }
  properties: {
    publicNetworkAccess: 'Disabled'  // Use private endpoint
  }
}

resource backupConfig 'Microsoft.RecoveryServices/vaults/backupconfig@2023-06-01' = {
  parent: recoveryVault
  name: 'vaultconfig'
  properties: {
    enhancedSecurityState: 'Enabled'
    softDeleteFeatureState: 'Enabled'  // Protect against accidental deletion
  }
}
```

### Backup Policies

**VM Backup Policy:**
```bicep
resource vmBackupPolicy 'Microsoft.RecoveryServices/vaults/backupPolicies@2023-06-01' = {
  parent: recoveryVault
  name: 'vm-daily-backup'
  properties: {
    backupManagementType: 'AzureIaasVM'
    schedulePolicy: {
      schedulePolicyType: 'SimpleSchedulePolicy'
      scheduleRunFrequency: 'Daily'
      scheduleRunTimes: [
        '2024-01-01T02:00:00Z'  // 2 AM UTC
      ]
    }
    retentionPolicy: {
      retentionPolicyType: 'LongTermRetentionPolicy'
      dailySchedule: {
        retentionTimes: [
          '2024-01-01T02:00:00Z'
        ]
        retentionDuration: {
          count: 30
          durationType: 'Days'
        }
      }
      weeklySchedule: {
        daysOfTheWeek: ['Sunday']
        retentionTimes: [
          '2024-01-01T02:00:00Z'
        ]
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
        retentionTimes: [
          '2024-01-01T02:00:00Z'
        ]
        retentionDuration: {
          count: 12
          durationType: 'Months'
        }
      }
      yearlySchedule: {
        retentionScheduleFormatType: 'Weekly'
        monthsOfYear: ['January']
        retentionScheduleWeekly: {
          daysOfTheWeek: ['Sunday']
          weeksOfTheMonth: ['First']
        }
        retentionTimes: [
          '2024-01-01T02:00:00Z'
        ]
        retentionDuration: {
          count: 5
          durationType: 'Years'
        }
      }
    }
    timeZone: 'UTC'
  }
}
```

**Retention:**
- Daily: 30 days
- Weekly: 12 weeks
- Monthly: 12 months
- Yearly: 5 years

### SQL Database Backup

**Automatic (built-in):**
- Full backup: Weekly
- Differential: Every 12-24 hours
- Transaction log: Every 5-10 minutes
- Retention: 7-35 days (default: 7)

**Point-in-time restore:**
```bash
# Restore to specific point in time
az sql db restore \
  --resource-group myapp-prod-rg \
  --server sql-shared-prod-eastus-001 \
  --name myapp-prod-db \
  --dest-name myapp-prod-db-restored \
  --time "2025-01-11T14:00:00Z"
```

**Long-term retention:**
```bicep
resource backupLongTermRetention 'Microsoft.Sql/servers/databases/backupLongTermRetentionPolicies@2023-08-01-preview' = {
  parent: sqlDatabase
  name: 'default'
  properties: {
    weeklyRetention: 'P5W'   // 5 weeks
    monthlyRetention: 'P12M' // 12 months
    yearlyRetention: 'P10Y'  // 10 years
    weekOfYear: 1
  }
}
```

### Backup Costs

**VM backup:**
- Protected instance: $10/month (< 50 GB)
- Storage: $0.10/GB/month
- Example: 100 GB VM = $10 + $10 = $20/month

**SQL Database backup:**
- Included in database cost (first 7 days)
- Long-term retention: $0.10/GB/month

**Optimize:**
- Adjust retention (30 days vs 365 days)
- Delete old recovery points
- Use cheaper storage tiers for old backups
</azure_backup>

<site_recovery>
## Azure Site Recovery

**What:** Disaster recovery orchestration and replication service.

**Use cases:**
- Azure VM to Azure (region-to-region DR)
- On-premises to Azure
- Automated failover and failback

### VM Replication

**Setup:**
```bicep
// Secondary region Recovery Services Vault
resource recoveryVaultSecondary 'Microsoft.RecoveryServices/vaults@2023-06-01' = {
  name: 'rsv-myapp-prod-westus2'
  location: 'westus2'
  sku: {
    name: 'RS0'
    tier: 'Standard'
  }
}

// Replication policy
resource replicationPolicy 'Microsoft.RecoveryServices/vaults/replicationPolicies@2023-06-01' = {
  parent: recoveryVaultSecondary
  name: 'vm-replication-policy'
  properties: {
    providerSpecificInput: {
      instanceType: 'A2A'
      recoveryPointHistory: 24  // 24-hour recovery window
      crashConsistentFrequencyInMinutes: 5
      appConsistentFrequencyInMinutes: 60
      multiVmSyncStatus: 'Enable'
    }
  }
}
```

**Enable replication:**
```bash
# Enable Site Recovery for VM
az backup protection enable-for-vm \
  --resource-group myapp-prod-eastus-rg \
  --vault-name rsv-myapp-prod-westus2 \
  --vm myapp-prod-vm-001 \
  --policy-name vm-replication-policy
```

**Replication process:**
1. Initial replication (full VM copy to secondary region)
2. Continuous replication (delta changes)
3. Crash-consistent snapshots every 5 minutes
4. App-consistent snapshots every hour

**RPO:** ~5 minutes (based on replication frequency)

### Failover Process

**Test failover (no impact):**
```bash
# Create test failover
az backup protection test-failover \
  --resource-group myapp-prod-westus2-rg \
  --vault-name rsv-myapp-prod-westus2 \
  --vm myapp-prod-vm-001
```

**Planned failover (maintenance):**
1. Shut down primary VM
2. Final replication sync
3. Failover to secondary
4. Start VM in secondary region

**Unplanned failover (disaster):**
1. Primary region unavailable
2. Failover to latest recovery point
3. Possible data loss (up to 5 minutes)

**Failback:**
1. Primary region restored
2. Reverse replication (secondary → primary)
3. Failover back to primary
4. Resume normal replication

### Cost

**Site Recovery:**
- Protected instance: $25/month per VM
- Storage: Standard storage costs for replica
- Network: Outbound data transfer during replication

**Example:**
- 10 VMs protected: 10 × $25 = $250/month
- Replica storage: 10 × 100 GB × $0.05 = $50/month
- **Total: $300/month for DR**
</site_recovery>

<geo_redundancy>
## Geo-Redundant Services

### Storage Accounts

**GRS (Geo-Redundant Storage):**
- Replicates to secondary region
- Read-only access to secondary (RA-GRS)
- Automatic failover (Microsoft-initiated)

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stmyappprodeastus001'
  location: 'eastus'
  sku: {
    name: 'Standard_GRS'  // or Standard_RAGRS for read access
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}
```

**RPO:** ~15 minutes (replication delay)
**RTO:** Hours (failover process)

**Cost:** 2x storage cost

### SQL Database Geo-Replication

**Active geo-replication:**
- Up to 4 readable secondaries
- Any Azure region
- Automatic failover groups

```bicep
// Primary database (East US)
resource primaryDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: primaryServer
  name: 'myapp-prod-db'
  location: 'eastus'
  sku: {
    name: 'S3'
    tier: 'Standard'
  }
}

// Failover group
resource failoverGroup 'Microsoft.Sql/servers/failoverGroups@2023-08-01-preview' = {
  parent: primaryServer
  name: 'myapp-failover-group'
  properties: {
    partnerServers: [
      {
        id: secondaryServer.id
      }
    ]
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60
    }
    readOnlyEndpoint: {
      failoverPolicy: 'Disabled'
    }
    databases: [
      primaryDatabase.id
    ]
  }
}
```

**Connection string:**
```
# Always connect to failover group endpoint (not server)
Server=tcp:myapp-failover-group.database.windows.net,1433;Database=myapp-prod-db;
```

**Failover automatically routes to healthy server.**

**RPO:** 0-5 seconds
**RTO:** ~30 seconds (automatic failover)

**Cost:** Cost of secondary database (same as primary)

### Cosmos DB Multi-Region

**Automatic global distribution:**

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: 'cosmos-myapp-prod'
  location: 'eastus'
  properties: {
    databaseAccountOfferType: 'Standard'
    enableAutomaticFailover: true
    enableMultipleWriteLocations: true  // Multi-region writes
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    locations: [
      {
        locationName: 'East US'
        failoverPriority: 0
        isZoneRedundant: true
      }
      {
        locationName: 'West US 2'
        failoverPriority: 1
        isZoneRedundant: true
      }
      {
        locationName: 'North Europe'
        failoverPriority: 2
        isZoneRedundant: true
      }
    ]
  }
}
```

**RPO:** Zero (multi-master replication)
**RTO:** Automatic failover (seconds)

**Cost:** Cost per region (3 regions = 3x base cost)
</geo_redundancy>

<dr_strategies>
## DR Strategy Selection

### Backup and Restore (Cheapest)

**When to use:**
- RTO: 24+ hours
- RPO: 24 hours
- Non-critical workloads

**Implementation:**
- Daily backups to storage
- Restore when needed

**Cost:** $$ (storage only)
**RTO:** Hours to days
**RPO:** Hours

### Pilot Light

**When to use:**
- RTO: 4-12 hours
- RPO: 1 hour
- Important workloads

**Implementation:**
- Core infrastructure in secondary (minimal)
- Data replicated continuously
- Scale up during disaster

**Cost:** $$$ (minimal secondary infrastructure)
**RTO:** Hours
**RPO:** Minutes

### Warm Standby

**When to use:**
- RTO: 1-4 hours
- RPO: Minutes
- Business-critical workloads

**Implementation:**
- Secondary region with scaled-down version
- Data replicated continuously
- Scale up to full capacity during disaster

**Cost:** $$$$ (partial secondary infrastructure)
**RTO:** Minutes to hours
**RPO:** Minutes

### Hot Standby (Active-Active)

**When to use:**
- RTO: < 1 hour
- RPO: Near-zero
- Mission-critical workloads

**Implementation:**
- Full infrastructure in both regions
- Active-active (both serving traffic)
- Traffic Manager or Front Door for routing

**Cost:** $$$$$ (full duplication)
**RTO:** Seconds to minutes
**RPO:** Seconds

**Example:**
```bicep
resource trafficManager 'Microsoft.Network/trafficManagerProfiles@2022-04-01' = {
  name: 'tm-myapp-prod'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Priority'  // Primary/failover
    dnsConfig: {
      relativeName: 'myapp-prod'
      ttl: 30  // Low TTL for fast failover
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
```
</dr_strategies>

<dr_testing>
## DR Testing

**Test DR plan regularly:**

### Test Frequency

- **Quarterly:** Full DR test with failover
- **Monthly:** Backup restore test
- **Weekly:** Monitoring and alert validation

### DR Test Checklist

```markdown
# DR Test Procedure

## Pre-Test
- [ ] Notify team of DR test
- [ ] Document current production state
- [ ] Prepare test environment in secondary region

## Test Execution
- [ ] Initiate failover to secondary region
- [ ] Verify all services started
- [ ] Test application functionality
- [ ] Verify data integrity
- [ ] Check monitoring and alerts
- [ ] Test failback to primary region

## Post-Test
- [ ] Document test results
- [ ] Record RTO achieved
- [ ] Record RPO (data loss if any)
- [ ] Update runbooks based on learnings
- [ ] Schedule fixes for identified issues

## Metrics
- Planned RTO: X hours
- Actual RTO: Y hours
- Planned RPO: A minutes
- Actual RPO: B minutes
- Issues found: N
```

### Automated DR Testing

```bash
#!/bin/bash
# dr-test.sh

echo "Starting DR test..."

# 1. Initiate failover
az network traffic-manager endpoint update \
  --name primary-eastus \
  --profile-name tm-myapp-prod \
  --resource-group myapp-prod-rg \
  --type azureEndpoints \
  --endpoint-status Disabled

# 2. Wait for traffic shift
sleep 120

# 3. Test secondary endpoint
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" https://myapp-prod-westus2.azurewebsites.net/health)

if [ $RESPONSE -eq 200 ]; then
  echo "✓ Secondary region healthy"
else
  echo "✗ Secondary region unhealthy: HTTP $RESPONSE"
  exit 1
fi

# 4. Test application functionality
npm run test:e2e -- --baseUrl=https://myapp-prod-westus2.azurewebsites.net

# 5. Failback to primary
az network traffic-manager endpoint update \
  --name primary-eastus \
  --profile-name tm-myapp-prod \
  --resource-group myapp-prod-rg \
  --type azureEndpoints \
  --endpoint-status Enabled

echo "✓ DR test complete"
```
</dr_testing>

<anti_patterns>
**Avoid:**

<anti_pattern name="No DR Plan">
**Problem:** No disaster recovery strategy

**Why bad:** Unprepared for disasters, extended downtime, data loss

**Instead:** Choose DR strategy based on RTO/RPO, implement and test
</anti_pattern>

<anti_pattern name="Untested DR">
**Problem:** DR plan exists but never tested

**Why bad:** Plan may not work when needed, unknown RTO/RPO

**Instead:** Test quarterly, document results, update runbooks
</anti_pattern>

<anti_pattern name="Single Region Only">
**Problem:** All resources in one Azure region

**Why bad:** Regional outage = complete downtime

**Instead:** Critical apps in 2+ regions with failover
</anti_pattern>

<anti_pattern name="No Backup Verification">
**Problem:** Backups running but never tested restore

**Why bad:** Backups may be corrupt or incomplete

**Instead:** Test restore monthly, verify data integrity
</anti_pattern>

<anti_pattern name="Same DR Strategy for All Apps">
**Problem:** Using hot standby for everything (or backup-only for everything)

**Why bad:** Over-spending or under-protecting

**Instead:** Match strategy to RTO/RPO requirements (tier-based approach)
</anti_pattern>
</anti_patterns>
