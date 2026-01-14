<required_reading>
Before implementing security, understand these foundational concepts:
- `references/security-identity.md` - Managed Identity, RBAC, Key Vault, Private Endpoints
- `references/networking.md` - Network security, NSGs, Private Endpoints
- `references/anti-patterns.md` - Security anti-patterns to avoid
</required_reading>

<objective>
Implement Azure security best practices including Managed Identity (zero credentials), RBAC, Key Vault, Private Endpoints, and network security using Azure CLI and Bicep.
</objective>

<quick_start>
**5-minute security baseline:**
```bash
# 1. Enable Managed Identity for App Service
az webapp identity assign \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg

# 2. Create Key Vault with RBAC (not access policies)
az keyvault create \
  --name kv-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --location eastus \
  --enable-rbac-authorization true

# 3. Grant app access to Key Vault
APP_PRINCIPAL_ID=$(az webapp identity show \
  --name app-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query principalId -o tsv)

az role assignment create \
  --assignee $APP_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name kv-myapp-prod-eastus --query id -o tsv)

# 4. Disable public access to storage
az storage account update \
  --name stmyappprodeastus \
  --resource-group myapp-prod-rg \
  --public-network-access Disabled

# 5. Create Private Endpoint for storage
az network private-endpoint create \
  --name pe-storage-myapp-prod \
  --resource-group myapp-prod-rg \
  --vnet-name vnet-myapp-prod-eastus \
  --subnet snet-data \
  --private-connection-resource-id $(az storage account show --name stmyappprodeastus --query id -o tsv) \
  --group-id blob \
  --connection-name storage-connection
```
</quick_start>

<process>
## Step 1: Zero Credentials - Managed Identity

**Purpose:** Eliminate secrets, passwords, and connection strings from code.

**1.1 Enable Managed Identity:**

```bicep
// System-assigned identity (recommended for single resource)
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  location: 'eastus'
  identity: {
    type: 'SystemAssigned'  // Azure creates and manages identity
  }
}

// User-assigned identity (for sharing across resources)
resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-myapp-prod-eastus'
  location: 'eastus'
}

resource appServiceWithUserIdentity 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${managedIdentity.id}': {}
    }
  }
}
```

**1.2 Grant RBAC permissions:**

```bicep
// Grant storage access
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: 'stmyappprodeastus'
}

resource storageBlobDataContributor 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'Storage Blob Data Contributor')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Grant SQL access
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' existing = {
  name: 'sql-myapp-prod-eastus'
}

resource sqlDbContributor 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: sqlServer
  name: guid(sqlServer.id, appService.id, 'SQL DB Contributor')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '9b7fa17d-e63e-47b0-bb0a-15c516ac86ec')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**1.3 Update application code:**

```csharp
// Before (BAD - credentials in code)
var blobClient = new BlobServiceClient(
  "DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=abc123..."
);

// After (GOOD - Managed Identity)
var blobClient = new BlobServiceClient(
  new Uri("https://stmyappprodeastus.blob.core.windows.net"),
  new DefaultAzureCredential()  // Automatically uses Managed Identity
);

// SQL connection with Managed Identity
var connectionString = "Server=sql-myapp-prod-eastus.database.windows.net;Database=mydb;Authentication=Active Directory Default;";
```

## Step 2: Secrets Management with Key Vault

**2.1 Create Key Vault with RBAC:**

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-myapp-prod-eastus-001'
  location: 'eastus'
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // Use RBAC, not access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true  // Cannot permanently delete for 90 days
    publicNetworkAccess: 'Disabled'  // Private endpoints only
  }
}

// Grant app access to secrets
resource kvSecretsUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: keyVault
  name: guid(keyVault.id, appService.id, 'Key Vault Secrets User')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**2.2 Store secrets:**

```bash
# Add secret to Key Vault
az keyvault secret set \
  --vault-name kv-myapp-prod-eastus-001 \
  --name "ApiKey" \
  --value "your-secret-api-key"

# Add connection string
az keyvault secret set \
  --vault-name kv-myapp-prod-eastus-001 \
  --name "SqlConnectionString" \
  --value "Server=sql-myapp-prod.database.windows.net;Database=mydb;..."
```

**2.3 Reference from App Service:**

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'ApiKey'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/ApiKey/)'
        }
        {
          name: 'SqlConnectionString'
          value: '@Microsoft.KeyVault(VaultName=kv-myapp-prod-eastus-001;SecretName=SqlConnectionString)'
        }
      ]
    }
  }
}
```

## Step 3: Network Security

**3.1 Virtual Network with subnets:**

```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vnet-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'snet-gateway'
        properties: {
          addressPrefix: '10.0.1.0/24'
          serviceEndpoints: []
        }
      }
      {
        name: 'snet-app'
        properties: {
          addressPrefix: '10.0.2.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
        }
      }
      {
        name: 'snet-data'
        properties: {
          addressPrefix: '10.0.3.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}
```

**3.2 Network Security Groups:**

```bicep
resource nsgData 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-data-prod-eastus'
  location: 'eastus'
  properties: {
    securityRules: [
      {
        name: 'AllowAppSubnet'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.2.0/24'  // App subnet only
          destinationAddressPrefix: '10.0.3.0/24'
          destinationPortRanges: ['1433', '5432']  // SQL, PostgreSQL
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
    ]
  }
}
```

**3.3 Private Endpoints:**

```bicep
// SQL Database private endpoint
resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-sql-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    subnet: {
      id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnet.name, 'snet-data')
    }
    privateLinkServiceConnections: [
      {
        name: 'sql-connection'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: ['sqlServer']
        }
      }
    ]
  }
}

// Private DNS zone for SQL
resource sqlPrivateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}

resource sqlPrivateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: sqlPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'config'
        properties: {
          privateDnsZoneId: sqlPrivateDnsZone.id
        }
      }
    ]
  }
}

// Link DNS zone to VNet
resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: sqlPrivateDnsZone
  name: 'link-to-vnet'
  location: 'global'
  properties: {
    virtualNetwork: {
      id: vnet.id
    }
    registrationEnabled: false
  }
}
```

## Step 4: RBAC (Role-Based Access Control)

**4.1 Principle of Least Privilege:**

```bash
# Developer access (read-only in production)
az role assignment create \
  --assignee dev@company.com \
  --role "Reader" \
  --resource-group myapp-prod-rg

# CI/CD pipeline (specific permissions only)
PIPELINE_PRINCIPAL_ID=$(az ad sp show --id <service-principal-id> --query id -o tsv)

az role assignment create \
  --assignee $PIPELINE_PRINCIPAL_ID \
  --role "Website Contributor" \
  --resource-group myapp-prod-rg

# Don't grant Contributor or Owner unless absolutely necessary
```

**4.2 Use built-in roles:**

```bicep
// Common Azure built-in roles
var roles = {
  Owner: '8e3af657-a8ff-443c-a75c-2fe8c4bcb635'
  Contributor: 'b24988ac-6180-42a0-ab88-20f7382dd24c'
  Reader: 'acdd72a7-3385-48ef-bd42-f606fba81ae7'

  // Web
  'Website Contributor': 'de139f84-1756-47ae-9be6-808fbbe84772'
  'Web Plan Contributor': '2cc479cb-7b4d-49a8-b449-8c00fd0f0a4b'

  // Data
  'Storage Blob Data Contributor': 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'
  'Storage Blob Data Reader': '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1'
  'SQL DB Contributor': '9b7fa17d-e63e-47b0-bb0a-15c516ac86ec'

  // Key Vault
  'Key Vault Secrets User': '4633458b-17de-408a-b874-0445c86b69e6'
  'Key Vault Secrets Officer': 'b86a8fe4-44ce-4948-aee5-eccb2c155cd7'
}
```

## Step 5: Azure Security Center / Defender for Cloud

```bash
# Enable Azure Defender
az security pricing create \
  --name VirtualMachines \
  --tier Standard

az security pricing create \
  --name SqlServers \
  --tier Standard

az security pricing create \
  --name AppServices \
  --tier Standard

az security pricing create \
  --name StorageAccounts \
  --tier Standard

# Check security recommendations
az security assessment list \
  --resource-group myapp-prod-rg \
  --query "[?status.code=='Unhealthy'].{Name:displayName, Severity:status.severity}" \
  --output table
```

## Step 6: Azure Policy for Governance

```bicep
// Require HTTPS only
resource httpsOnlyPolicy 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-https-only'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/a4af4a39-4135-47fb-b175-47fbdf85311d'
    displayName: 'App Service apps should only be accessible over HTTPS'
    enforcementMode: 'Default'
  }
}

// Require tags
resource requireTagsPolicy 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'require-tags'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025'
    displayName: 'Require CostCenter tag on resource groups'
    parameters: {
      tagName: {
        value: 'CostCenter'
      }
    }
  }
}

// Deny public IPs in production
resource denyPublicIPs 'Microsoft.Authorization/policyAssignments@2023-04-01' = {
  name: 'deny-public-ips-prod'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/6c112d4e-5bc7-47ae-a041-ea2d9dccd749'
    displayName: 'Deny public IPs in production'
    enforcementMode: 'Default'
  }
}
```
</process>

<security_checklist>
## Production Security Checklist

### Identity & Access
- [ ] Managed Identity enabled for all apps
- [ ] Zero connection strings or passwords in code
- [ ] RBAC permissions follow least privilege
- [ ] No Owner/Contributor roles granted unnecessarily
- [ ] Service principals rotated regularly (90 days)

### Secrets Management
- [ ] Key Vault created with RBAC (not access policies)
- [ ] Soft delete and purge protection enabled
- [ ] All secrets stored in Key Vault
- [ ] App Service references Key Vault secrets
- [ ] Secret expiration dates set

### Network Security
- [ ] VNet created with proper subnets
- [ ] NSGs configured on all subnets
- [ ] Private Endpoints for SQL, Storage, Key Vault
- [ ] Public network access disabled for data services
- [ ] App Service VNet integration configured

### Data Protection
- [ ] Encryption at rest enabled (default for most services)
- [ ] TLS 1.2+ enforced (no TLS 1.0/1.1)
- [ ] HTTPS-only enforced on App Services
- [ ] SQL Database TDE enabled (default)
- [ ] Storage account secure transfer required

### Monitoring & Compliance
- [ ] Azure Defender enabled for all services
- [ ] Diagnostic logs sent to Log Analytics
- [ ] Security Center recommendations reviewed
- [ ] Azure Policy enforcing governance
- [ ] Regular security assessment scans
</security_checklist>

<success_criteria>
Security implementation is complete when:
- No credentials exist in code, config, or environment variables
- All authentication uses Managed Identity
- All data services accessible only via Private Endpoints
- RBAC follows principle of least privilege
- Azure Security Center shows no critical/high vulnerabilities
- All Azure Policy requirements met
- Security audit log shows no unauthorized access attempts
</success_criteria>
