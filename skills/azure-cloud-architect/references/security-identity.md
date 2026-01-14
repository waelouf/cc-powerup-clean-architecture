<overview>
Comprehensive guide to Azure security and identity management. Covers RBAC, Managed Identity, Key Vault, Azure AD integration, security best practices, and zero-trust architecture for 2024-2025.
</overview>

<fundamental_principles>
## Security Principles for Azure

<principle name="Zero Trust">
**Concept:** Never trust, always verify. Every access request is authenticated and authorized.

**In practice:**
- No implicit trust based on network location
- Verify explicitly (identity, device, location)
- Use least privilege access
- Assume breach (segment access, verify end-to-end)
</principle>

<principle name="Defense in Depth">
**Concept:** Multiple layers of security controls.

**Layers:**
1. **Identity:** Azure AD, MFA, Conditional Access
2. **Perimeter:** Firewall, DDoS protection, WAF
3. **Network:** NSGs, Private Endpoints, VNet isolation
4. **Compute:** Patch management, endpoint protection
5. **Application:** Secure coding, secrets management
6. **Data:** Encryption at rest and in transit
</principle>

<principle name="Least Privilege">
**Concept:** Grant minimum permissions necessary to perform task.

**In practice:**
- Use RBAC with specific roles
- Time-bound access (JIT with PIM)
- Regular access reviews
- Separate dev/staging/prod access
</principle>

<principle name="Secrets Never in Code">
**Concept:** No credentials, API keys, connection strings in source code or config files.

**In practice:**
- Store secrets in Key Vault
- Use Managed Identity for authentication
- Reference Key Vault from app configuration
- Scan code for secrets (GitHub Advanced Security, GitGuardian)
</principle>
</fundamental_principles>

<managed_identity>
## Managed Identity (Zero Credentials)

**What it is:** Azure-managed identity for applications to authenticate to Azure services without credentials in code.

**Types:**

<type name="System-Assigned Managed Identity">
**Lifecycle:** Tied to Azure resource (App Service, VM, Function)
**When deleted:** Identity deleted with resource
**Use when:** Single resource needs access to Azure services

**Enable for App Service:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  location: 'eastus'
  identity: {
    type: 'SystemAssigned'  // Enable managed identity
  }
  properties: {
    serverFarmId: appServicePlan.id
  }
}

// Output the principal ID for RBAC assignment
output appIdentityPrincipalId string = appService.identity.principalId
```

**Use from application code:**
```csharp
// No credentials needed!
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var credential = new DefaultAzureCredential();  // Automatically uses managed identity
var client = new SecretClient(new Uri("https://myvault.vault.azure.net/"), credential);
var secret = await client.GetSecretAsync("database-connection");
```
</type>

<type name="User-Assigned Managed Identity">
**Lifecycle:** Independent of resources
**Reusable:** Multiple resources can use same identity
**Use when:** Multiple resources need same permissions, or need identity before resource exists

**Create:**
```bicep
resource userIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'shared-app-identity'
  location: 'eastus'
}

// Use in App Service
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  location: 'eastus'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${userIdentity.id}': {}
    }
  }
}
```
</type>

### Managed Identity vs Service Principal

**Managed Identity (Recommended):**
- ✅ No credential management
- ✅ Automatic rotation
- ✅ Tied to Azure resource
- ❌ Only works for Azure resources

**Service Principal:**
- ✅ Works outside Azure (on-premises, other clouds)
- ✅ Can be shared across subscriptions
- ❌ Manual credential management
- ❌ Credentials can be compromised
- Use only when Managed Identity not available
</managed_identity>

<rbac>
## Role-Based Access Control (RBAC)

**Concept:** Assign permissions through roles, not individual permissions.

### Common Built-in Roles

<role name="Owner">
**Permissions:** Full access including RBAC management
**Use for:** Subscription/resource group owners, admins
**Caution:** Very powerful, use sparingly
</role>

<role name="Contributor">
**Permissions:** Create/manage resources, no RBAC changes
**Use for:** Engineers deploying resources
**Common use:** CI/CD service principals
</role>

<role name="Reader">
**Permissions:** View resources only
**Use for:** Monitoring, auditing, support teams
</role>

<role name="Key Vault Secrets User">
**Permissions:** Read secrets from Key Vault
**Use for:** Applications reading secrets via Managed Identity
**Role ID:** 4633458b-17de-408a-b874-0445c86b69e6
</role>

<role name="Storage Blob Data Contributor">
**Permissions:** Read/write/delete blobs
**Use for:** Applications accessing blob storage
**Role ID:** ba92f5b4-2d11-453d-a403-e96b0029c9fe
</role>

<role name="SQL DB Contributor">
**Permissions:** Manage SQL databases (not data access)
**Use for:** Database administrators
</role>

### RBAC Assignment

**Scope hierarchy:**
```
Management Group
└── Subscription
    └── Resource Group
        └── Resource
```

Roles assigned at higher scope inherited by lower scopes.

**Assign role to Managed Identity:**
```bicep
// Grant App Service managed identity access to Key Vault
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, appService.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Assign via Azure CLI:**
```bash
# Grant Contributor role at resource group scope
az role assignment create \
  --assignee <principal-id> \
  --role Contributor \
  --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>

# Grant specific Key Vault role
az role assignment create \
  --assignee <app-identity-principal-id> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/<vault-name>
```

### Custom Roles

For specific permissions not covered by built-in roles:

```json
{
  "Name": "App Service Deployment User",
  "Description": "Can deploy code to App Service but not change configuration",
  "Actions": [
    "Microsoft.Web/sites/publish/Action",
    "Microsoft.Web/sites/config/list/Action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>/resourceGroups/myapp-prod-rg"
  ]
}
```

```bash
az role definition create --role-definition app-service-deployer-role.json
```

### Privileged Identity Management (PIM)

**Just-In-Time (JIT) access:** Grant elevated permissions temporarily.

**Setup:**
1. Azure AD → Privileged Identity Management
2. Azure resources → Add eligible assignments
3. User: "Jane Doe", Role: "Contributor", Duration: Max 8 hours

**When Jane needs access:**
1. Portal → PIM → My roles → Activate
2. Justification: "Deploy production hotfix"
3. Approval (if required)
4. Access granted for specified duration
5. Auto-revoked after time expires

**Benefits:**
- Reduces standing access
- Audit trail of elevated access
- Requires justification
- Optional approval workflow
</rbac>

<key_vault>
## Azure Key Vault

**Purpose:** Securely store and access secrets, keys, and certificates.

### Key Vault Architecture

**Best practice: One Key Vault per environment**
```
shared-dev-eastus-kv-001     (dev secrets)
shared-staging-eastus-kv-001 (staging secrets)
shared-prod-eastus-kv-001    (production secrets)
```

**Don't mix environments in same vault** (security boundary).

### RBAC vs Access Policies

**CRITICAL: Use RBAC, not access policies.**

**Why RBAC:**
- ✅ Access policies deprecated (2022)
- ✅ RBAC integrates with Azure AD
- ✅ Supports PIM (time-bound access)
- ✅ Centralized permission model
- ❌ Access policies lack PIM support

**Create Key Vault with RBAC:**
```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'shared-prod-eastus-kv-001'
  location: 'eastus'
  properties: {
    sku: {
      family: 'A'
      name: 'premium'  // HSM-backed for production
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // Use RBAC!
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true  // Can't permanently delete secrets
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'  // Block public access
      virtualNetworkRules: []
      ipRules: []
    }
  }
}
```

### Grant Access with RBAC

```bicep
// Grant app managed identity access to read secrets
resource secretsUserRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, appIdentity, 'secrets-user')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')  // Key Vault Secrets User
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Grant admin full access (secrets, keys, certificates)
resource adminRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, adminObjectId, 'administrator')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '00482a5a-887f-4fb3-b217-3c3c0d0e8e82')  // Key Vault Administrator
    principalId: adminObjectId
    principalType: 'User'
  }
}
```

### Store Secrets

```bash
# Set secret
az keyvault secret set \
  --vault-name shared-prod-eastus-kv-001 \
  --name database-connection \
  --value "Server=tcp:myserver.database.windows.net,1433;Database=mydb;"

# Set with expiration
az keyvault secret set \
  --vault-name shared-prod-eastus-kv-001 \
  --name api-key \
  --value "abc123..." \
  --expires "2025-12-31T23:59:59Z"

# Get secret
az keyvault secret show \
  --vault-name shared-prod-eastus-kv-001 \
  --name database-connection \
  --query value -o tsv
```

### Reference Secrets in App Service

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    siteConfig: {
      appSettings: [
        {
          name: 'DATABASE_CONNECTION'
          value: '@Microsoft.KeyVault(SecretUri=https://shared-prod-eastus-kv-001.vault.azure.net/secrets/database-connection/)'
        }
        {
          name: 'API_KEY'
          value: '@Microsoft.KeyVault(VaultName=shared-prod-eastus-kv-001;SecretName=api-key)'
        }
      ]
    }
  }
}
```

**App Service automatically:**
- Uses its Managed Identity
- Fetches secret from Key Vault
- Updates when secret rotates

### Secret Rotation

**Automated rotation (Azure Automation):**
```powershell
# rotation-runbook.ps1
param([string]$SecretName)

# Generate new password
$NewPassword = New-Guid | ConvertTo-SecureString -AsPlainText -Force

# Update in Azure SQL
Set-AzSqlServerAdministratorPassword -ServerName "myserver" -Password $NewPassword

# Update in Key Vault
Set-AzKeyVaultSecret -VaultName "shared-prod-kv" -Name $SecretName -SecretValue $NewPassword

Write-Output "Rotated secret: $SecretName"
```

Schedule monthly via Azure Automation.

### Private Endpoint for Key Vault

```bicep
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${keyVault.name}-pe'
  location: 'eastus'
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${keyVault.name}-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
}

// Private DNS zone for name resolution
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.vaultcore.azure.net'
  location: 'global'
}
```
</key_vault>

<azure_ad_integration>
## Azure AD Authentication

### App Service / Azure Functions

**Enable Azure AD authentication:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  properties: {
    siteConfig: {
      // App configuration
    }
  }
}

resource authSettings 'Microsoft.Web/sites/config@2023-12-01' = {
  parent: appService
  name: 'authsettingsV2'
  properties: {
    globalValidation: {
      requireAuthentication: true
      unauthenticatedClientAction: 'RedirectToLoginPage'
    }
    identityProviders: {
      azureActiveDirectory: {
        enabled: true
        registration: {
          openIdIssuer: 'https://sts.windows.net/${subscription().tenantId}/'
          clientId: azureAdAppClientId
          clientSecretSettingName: 'MICROSOFT_PROVIDER_AUTHENTICATION_SECRET'
        }
        validation: {
          allowedAudiences: [
            'api://${azureAdAppClientId}'
          ]
        }
      }
    }
    login: {
      tokenStore: {
        enabled: true
      }
    }
  }
}
```

**Users must sign in with Azure AD to access app.**

### API Authentication with Azure AD

**Protect API with Azure AD tokens:**

```csharp
// ASP.NET Core Startup.cs
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));

services.AddAuthorization(options =>
{
    options.AddPolicy("ReadAccess", policy =>
        policy.RequireRole("App.Read"));
});

// Controller
[Authorize(Policy = "ReadAccess")]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // Only users with App.Read role can access
}
```

**Client gets token:**
```csharp
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(
    new TokenRequestContext(new[] { "api://myapp/.default" })
);

var client = new HttpClient();
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token.Token);
var response = await client.GetAsync("https://myapp.com/api/products");
```

### Conditional Access

**Require MFA for production access:**

Azure AD → Security → Conditional Access → New policy:
- **Name:** Require MFA for Production
- **Users:** Production engineers
- **Cloud apps:** Azure Management (for portal)
- **Conditions:** None
- **Grant:** Require MFA
- **Session:** Sign-in frequency 8 hours

**Benefits:**
- Extra security layer
- Prevents compromised passwords
- Audit trail
</azure_ad_integration>

<network_security>
## Network Security

### Private Endpoints

**Eliminate public internet access:**

```bicep
// SQL Server with private endpoint
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'shared-prod-sql-001'
  properties: {
    publicNetworkAccess: 'Disabled'  // No public access!
  }
}

resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: '${sqlServer.name}-pe'
  location: 'eastus'
  properties: {
    subnet: {
      id: subnetId
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
```

**Accessible only from VNet, not internet.**

### Network Security Groups (NSGs)

**Control traffic between subnets:**

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'app-subnet-nsg'
  location: 'eastus'
  properties: {
    securityRules: [
      {
        name: 'allow-https-inbound'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: 'Internet'
          destinationAddressPrefix: '*'
        }
      }
      {
        name: 'deny-all-inbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}
```

### Service Endpoints vs Private Endpoints

**Service Endpoints:**
- Traffic stays on Microsoft backbone
- Still uses public IP of service
- Cheaper (free)
- Use for: Non-sensitive data, cost optimization

**Private Endpoints:**
- Private IP in your VNet
- No public IP access
- Fully isolated
- Cost: $0.01/hour per endpoint
- Use for: Production, sensitive data, compliance

**Recommendation:** Use Private Endpoints for production.
</network_security>

<encryption>
## Encryption

### Encryption at Rest

**Default:** All Azure services encrypt data at rest (Microsoft-managed keys).

**Customer-Managed Keys (CMK):**

For compliance, use your own encryption keys:

```bicep
// Key Vault for encryption keys
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'encryption-keys-kv'
  properties: {
    enablePurgeProtection: true  // Required for CMK
    sku: { name: 'premium' }
  }
}

// Encryption key
resource key 'Microsoft.KeyVault/vaults/keys@2023-07-01' = {
  parent: keyVault
  name: 'data-encryption-key'
  properties: {
    kty: 'RSA'
    keySize: 4096
  }
}

// Storage account with CMK
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mystorageaccount'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    encryption: {
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
        keyname: key.name
        keyvaulturi: keyVault.properties.vaultUri
      }
    }
  }
}
```

### Encryption in Transit

**Always use TLS 1.2+:**

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  properties: {
    httpsOnly: true  // Redirect HTTP to HTTPS
    siteConfig: {
      minTlsVersion: '1.2'  // Minimum TLS version
      ftpsState: 'Disabled'  // No insecure FTP
    }
  }
}

resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'shared-prod-sql-001'
  properties: {
    minimalTlsVersion: '1.2'
  }
}
```
</encryption>

<security_best_practices>
## Security Best Practices Checklist

**Identity & Access:**
- [ ] Use Managed Identity for all Azure resource access
- [ ] Enable RBAC on Key Vault (not access policies)
- [ ] Grant least privilege access
- [ ] Use PIM for administrative access
- [ ] Enable MFA for all users
- [ ] Implement Conditional Access policies

**Secrets Management:**
- [ ] Store all secrets in Key Vault
- [ ] No secrets in code or configuration files
- [ ] Reference Key Vault from app settings
- [ ] Enable soft delete and purge protection
- [ ] Rotate secrets regularly (automated)

**Network Security:**
- [ ] Use Private Endpoints for production resources
- [ ] Disable public network access on databases
- [ ] Implement NSGs on all subnets
- [ ] Use Azure Firewall or Application Gateway WAF

**Encryption:**
- [ ] Enforce HTTPS only (httpsOnly: true)
- [ ] Set minimum TLS version to 1.2
- [ ] Disable FTP/FTPS
- [ ] Use customer-managed keys for sensitive data

**Monitoring & Auditing:**
- [ ] Enable Azure AD audit logs
- [ ] Enable diagnostic logs on all resources
- [ ] Set up alerts for suspicious activity
- [ ] Regular access reviews

**Compliance:**
- [ ] Tag resources with data classification
- [ ] Implement Azure Policy for governance
- [ ] Use Azure Blueprints for compliant deployments
- [ ] Regular security assessments (Microsoft Defender)
</security_best_practices>

<anti_patterns>
**Avoid:**

<anti_pattern name="Hardcoded Secrets">
**Problem:** Connection strings, API keys in code or app settings

**Why bad:** Exposed in Git history, accessible to anyone with repo access

**Instead:** Store in Key Vault, reference via Managed Identity
</anti_pattern>

<anti_pattern name="Using SQL Authentication">
**Problem:** SQL username/password instead of Azure AD

**Why bad:** Credentials can be compromised, no MFA, difficult to rotate

**Instead:** Use Azure AD authentication + Managed Identity
</anti_pattern>

<anti_pattern name="Key Vault Access Policies">
**Problem:** Using deprecated access policies instead of RBAC

**Why bad:** No PIM support, less secure, harder to audit

**Instead:** Enable RBAC authorization on Key Vault
</anti_pattern>

<anti_pattern name="Public Network Access">
**Problem:** Leaving public access enabled on databases, storage

**Why bad:** Exposed to internet attacks, no network isolation

**Instead:** Use Private Endpoints, disable public access
</anti_pattern>

<anti_pattern name="Overly Broad Permissions">
**Problem:** Granting Owner or Contributor when Reader sufficient

**Why bad:** Violates least privilege, increases blast radius

**Instead:** Grant minimum required role, use custom roles if needed
</anti_pattern>

<anti_pattern name="No MFA">
**Problem:** Allowing password-only authentication

**Why bad:** Vulnerable to credential theft, phishing

**Instead:** Require MFA for all users, especially admins
</anti_pattern>
</anti_patterns>
