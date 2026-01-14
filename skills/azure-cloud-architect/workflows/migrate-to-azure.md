<required_reading>
Before migrating to Azure:
- `references/compute-services.md` - Choose appropriate Azure compute service
- `references/architecture-patterns.md` - Strangler Fig pattern for gradual migration
- `references/storage-data.md` - Data migration strategies
</required_reading>

<objective>
Migrate existing on-premises or cloud infrastructure to Azure using Azure Migrate, Database Migration Service, and proven migration strategies.
</objective>

<quick_start>
**5-minute migration assessment:**
```bash
# Create Azure Migrate project
az migrate project create \
  --name myapp-migration \
  --resource-group migration-rg \
  --location eastus

# Install Azure Migrate appliance (on-premises)
# Download from portal: https://portal.azure.com -> Azure Migrate -> Discover

# Start discovery (after appliance setup)
# Appliance will discover VMs, databases, web apps

# View assessment
az migrate assessment show \
  --project myapp-migration \
  --resource-group migration-rg \
  --name initial-assessment
```
</quick_start>

<process>
## Step 1: Migration Strategy

**1.1 The 6 Rs of Migration:**

```
1. Rehost (Lift-and-shift):
   - Move VMs as-is to Azure
   - Minimal changes
   - Fast, low risk
   - Example: VM → Azure VM

2. Refactor (Repackage):
   - Containerize applications
   - Minimal code changes
   - Better cloud efficiency
   - Example: App → Container Apps, AKS

3. Rearchitect:
   - Modify for cloud-native
   - Use PaaS services
   - Medium effort
   - Example: IIS → App Service

4. Rebuild:
   - Rewrite from scratch
   - Full cloud-native
   - High effort, best outcome
   - Example: Monolith → Microservices

5. Replace:
   - Use SaaS instead
   - No migration needed
   - Example: Exchange → Microsoft 365

6. Retire:
   - Decommission unused systems
   - Cost savings
```

**1.2 Choose strategy:**

```
Legacy .NET Framework app on Windows Server:
→ Rehost: Azure VM (quick win)
→ Better: Refactor to App Service (Windows)
→ Best: Rearchitect to .NET 8 + App Service (Linux)

MySQL database on-premises:
→ Rehost: MySQL on Azure VM
→ Better: Azure Database for MySQL
→ Best: Migrate to Azure SQL Database

PHP web app on Linux:
→ Rehost: Azure VM
→ Better: App Service (Linux)
→ Best: Container Apps

Static website:
→ Replace: Azure Static Web Apps
→ Or: Azure Storage static website hosting
```

## Step 2: Assessment Phase

**2.1 Azure Migrate assessment:**

```bash
# Create resource group for migration
az group create \
  --name migration-rg \
  --location eastus

# Create Azure Migrate project
az migrate project create \
  --name mycompany-migration \
  --resource-group migration-rg \
  --location eastus
```

**2.2 Discovery (using Azure Migrate appliance):**

1. Download appliance from Azure portal
2. Deploy to on-premises VMware/Hyper-V
3. Connect to Azure Migrate project
4. Appliance discovers:
   - VMs (OS, CPU, RAM, disk)
   - Databases (SQL Server, MySQL, PostgreSQL)
   - Web apps (IIS, Apache, Nginx)
   - Dependencies (which VMs talk to each other)

**2.3 Assessment report provides:**

```
- Azure VM sizing recommendations
- Estimated monthly costs
- Readiness assessment (ready, conditionally ready, not ready)
- Dependencies map
- Performance history
```

## Step 3: VM Migration (Rehost)

**3.1 Using Azure Migrate:**

```bash
# Install Azure Migrate agent on source VMs
# (done via portal or script)

# Create replication
az migrate appliance show \
  --name myapp-appliance \
  --resource-group migration-rg

# Start replication for discovered VM
# (via portal: Azure Migrate -> Servers -> Replicate)

# Initial replication:
# - Full disk copy to Azure
# - Ongoing delta replication

# Test migration (creates test VM in Azure)
# - Verify application works
# - No impact on production

# Cutover migration
# 1. Stop source VM
# 2. Final delta replication
# 3. Start Azure VM
# 4. Decommission source VM
```

**3.2 Bicep for target infrastructure:**

```bicep
// Network for migrated VMs
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vnet-migration-prod-eastus'
  location: 'eastus'
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'snet-app'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
    ]
  }
}

// Target VM
resource vm 'Microsoft.Compute/virtualMachines@2024-03-01' = {
  name: 'vm-myapp-prod-eastus-001'
  location: 'eastus'
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_D4s_v3'  // From Azure Migrate recommendation
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: '2022-Datacenter'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Premium_LRS'
        }
      }
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nic.id
        }
      ]
    }
  }
}
```

## Step 4: Database Migration

**4.1 SQL Server to Azure SQL Database:**

```bash
# Create Azure SQL Database
az sql server create \
  --name sql-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --location eastus \
  --admin-user sqladmin \
  --admin-password <password>

az sql db create \
  --resource-group myapp-prod-rg \
  --server sql-myapp-prod-eastus \
  --name mydb \
  --service-objective S3 \
  --backup-storage-redundancy Geo

# Install Data Migration Assistant (on-premises)
# Download: https://www.microsoft.com/download/details.aspx?id=53595

# Run assessment
# DMA -> New Assessment -> Select SQL Server -> Connect to source

# Migrate schema
# DMA -> New Migration -> Migrate schema only

# Migrate data using Azure Database Migration Service
az dms project create \
  --resource-group migration-rg \
  --service-name dms-myapp \
  --name mydb-migration \
  --location eastus \
  --source-platform SQL \
  --target-platform SQLDB

# Create migration task (via portal)
# - Offline migration: Downtime during migration
# - Online migration: Minimal downtime, uses log shipping
```

**4.2 MySQL migration:**

```bash
# Create Azure Database for MySQL
az mysql flexible-server create \
  --resource-group myapp-prod-rg \
  --name mysql-myapp-prod-eastus \
  --location eastus \
  --admin-user myadmin \
  --admin-password <password> \
  --sku-name Standard_D2ds_v4 \
  --tier GeneralPurpose \
  --storage-size 128

# Option 1: mysqldump (offline)
mysqldump -h source-server -u user -p mydb > mydb.sql
mysql -h mysql-myapp-prod-eastus.mysql.database.azure.com -u myadmin -p mydb < mydb.sql

# Option 2: Azure Database Migration Service
# Supports online migration with minimal downtime
```

**4.3 MongoDB to Cosmos DB:**

```bash
# Create Cosmos DB account
az cosmosdb create \
  --name cosmos-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --kind MongoDB \
  --locations regionName=eastus

# Use Azure Database Migration Service
# Or mongodump/mongorestore
mongodump --uri="mongodb://source-server:27017/mydb" --archive=mydb.archive
mongorestore --uri="mongodb://cosmos-myapp-prod-eastus.mongo.cosmos.azure.com:10255/mydb?ssl=true" --archive=mydb.archive
```

## Step 5: Web App Migration

**5.1 IIS to App Service:**

```bash
# Create App Service
az appservice plan create \
  --name asp-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --sku P1v3 \
  --is-linux false

az webapp create \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --plan asp-myapp-prod-eastus

# Deploy code
# Option 1: ZIP deploy
az webapp deployment source config-zip \
  --resource-group myapp-prod-rg \
  --name app-myapp-prod-eastus \
  --src app.zip

# Option 2: FTP
# Get FTP credentials from portal

# Option 3: GitHub Actions (recommended)
# See workflows/setup-cicd-pipeline.md
```

**5.2 Containerize and deploy:**

```bash
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# Build and push to ACR
az acr create \
  --resource-group myapp-prod-rg \
  --name acrmyappprodeastus \
  --sku Standard

az acr build \
  --registry acrmyappprodeastus \
  --image myapp:latest \
  --file Dockerfile .

# Deploy to Container Apps
az containerapp create \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --environment containerapps-env-prod \
  --image acrmyappprodeastus.azurecr.io/myapp:latest \
  --target-port 3000 \
  --ingress external \
  --min-replicas 2 \
  --max-replicas 10
```

## Step 6: Data Transfer

**6.1 Small data (<1 TB) - AzCopy:**

```bash
# Install AzCopy
# Download: https://aka.ms/downloadazcopy

# Copy files from on-premises to Azure Storage
azcopy copy \
  "/path/to/source/*" \
  "https://stmyappprodeastus.blob.core.windows.net/data?<SAS-token>" \
  --recursive

# Copy from S3 to Azure Blob
azcopy copy \
  "https://mybucket.s3.amazonaws.com/*" \
  "https://stmyappprodeastus.blob.core.windows.net/data?<SAS-token>" \
  --recursive
```

**6.2 Large data (>10 TB) - Azure Data Box:**

```bash
# Order Data Box from portal
# 1. Create Data Box order
# 2. Microsoft ships physical device
# 3. Copy data to device (up to 80 TB)
# 4. Ship back to Microsoft
# 5. Microsoft uploads to Azure Storage

# Track order
az databox job show \
  --resource-group migration-rg \
  --name databox-migration-001
```

## Step 7: Cutover Plan

**7.1 Cutover checklist:**

```markdown
# Cutover Plan Template

## Pre-cutover (1 week before)
- [ ] Final data sync completed
- [ ] Azure resources provisioned and tested
- [ ] DNS records prepared (don't activate yet)
- [ ] Monitoring configured in Azure
- [ ] Rollback plan documented
- [ ] Stakeholders notified of cutover window

## Cutover window (example: Saturday 2 AM - 6 AM)
T-0:00 (2:00 AM):
- [ ] Stop source applications
- [ ] Final data migration
- [ ] Verify data integrity in Azure

T+0:30 (2:30 AM):
- [ ] Update DNS records to point to Azure
- [ ] Start Azure applications

T+0:45 (2:45 AM):
- [ ] Smoke tests (login, critical paths)
- [ ] Monitor error rates

T+1:00 (3:00 AM):
- [ ] Go/No-Go decision
- [ ] If GO: Communicate success
- [ ] If NO-GO: Initiate rollback

## Post-cutover
- [ ] Monitor for 24 hours
- [ ] Decommission source after 30 days
- [ ] Post-migration review meeting
```

**7.2 DNS update:**

```bash
# Before (pointing to on-premises)
# A record: myapp.com -> 203.0.113.10

# After (pointing to Azure)
# Update A record or CNAME
# A record: myapp.com -> 20.51.7.123 (Azure Public IP)
# Or CNAME: myapp.com -> app-myapp-prod-eastus.azurewebsites.net
```

## Step 8: Post-Migration Optimization

```bash
# Right-size VMs based on actual usage
az vm resize \
  --resource-group myapp-prod-rg \
  --name vm-myapp-prod-eastus-001 \
  --size Standard_D2s_v3

# Enable auto-shutdown for dev VMs
# See workflows/optimize-costs.md

# Configure backup
# See workflows/implement-disaster-recovery.md

# Enable monitoring
# See workflows/setup-monitoring.md

# Implement security best practices
# See workflows/secure-infrastructure.md
```
</process>

<migration_checklist>
## Migration Checklist

### Assessment Phase
- [ ] Azure Migrate project created
- [ ] Discovery appliance deployed
- [ ] All workloads discovered
- [ ] Dependencies mapped
- [ ] Azure sizing recommendations reviewed
- [ ] Cost estimation approved
- [ ] Migration strategy selected per workload

### Preparation Phase
- [ ] Azure landing zone prepared (VNet, NSGs, etc.)
- [ ] Target resources provisioned
- [ ] Network connectivity established (VPN/ExpressRoute if needed)
- [ ] RBAC configured
- [ ] Monitoring configured

### Migration Phase
- [ ] Test migrations performed
- [ ] Applications verified in Azure
- [ ] Data migration tested
- [ ] Cutover plan documented
- [ ] Rollback plan documented
- [ ] Stakeholders notified

### Cutover Phase
- [ ] Final data sync completed
- [ ] Source systems stopped
- [ ] DNS updated
- [ ] Azure applications started
- [ ] Smoke tests passed
- [ ] Monitoring shows healthy state

### Post-Migration Phase
- [ ] 24-hour monitoring completed
- [ ] Performance optimized
- [ ] Costs reviewed and optimized
- [ ] Source systems decommissioned
- [ ] Documentation updated
- [ ] Lessons learned documented
</migration_checklist>

<success_criteria>
Migration is successful when:
- All workloads running in Azure
- Application performance meets or exceeds baseline
- No data loss during migration
- RTO/RPO targets met during cutover
- Users can access applications via new Azure endpoints
- Monitoring shows healthy state for 24+ hours
- Costs within 10% of estimates
- Source systems safely decommissioned
</success_criteria>
