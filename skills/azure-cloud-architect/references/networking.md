<overview>
Comprehensive guide to Azure networking. Covers VNets, subnets, NSGs, Application Gateway, Private Endpoints, DNS, and network security patterns for 2024-2025.
</overview>

<networking_fundamentals>
## Azure Networking Core Concepts

<concept name="Virtual Network (VNet)">
**What:** Private network in Azure, isolated from other networks.

**Key features:**
- Address space: CIDR block (e.g., 10.0.0.0/16)
- Subnets: Subdivisions of VNet
- Peering: Connect VNets together
- Default route to internet (outbound)

**Example:**
```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vnet-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'  // 65,536 IP addresses
      ]
    }
    subnets: [
      {
        name: 'app-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'  // 256 IPs
        }
      }
      {
        name: 'data-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
        }
      }
      {
        name: 'gateway-subnet'
        properties: {
          addressPrefix: '10.0.255.0/24'
        }
      }
    ]
  }
}
```
</concept>

<concept name="Network Security Group (NSG)">
**What:** Firewall rules for inbound/outbound traffic.

**Applied to:** Subnets or individual NICs

**Rules:**
- Priority: 100-4096 (lower = higher priority)
- Direction: Inbound or Outbound
- Action: Allow or Deny
- Protocol: TCP, UDP, ICMP, Any

**Example:**
```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-app-subnet'
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
        name: 'allow-app-to-sql'
        properties: {
          priority: 110
          direction: 'Outbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.1.0/24'  // App subnet
          sourcePortRange: '*'
          destinationAddressPrefix: '10.0.2.0/24'  // Data subnet
          destinationPortRange: '1433'
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
</concept>

<concept name="Private Endpoints">
**What:** Private IP address for Azure PaaS services in your VNet.

**Benefits:**
- No public internet exposure
- Traffic stays on Microsoft backbone
- Access from on-premises via VPN/ExpressRoute

**Use for:**
- Azure SQL Database
- Storage Accounts
- Key Vault
- App Service (Premium tier)

**Example:**
```bicep
resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-sql-shared-prod'
  location: 'eastus'
  properties: {
    subnet: {
      id: dataSubnet.id
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

// DNS for private endpoint
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}

resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'vnet-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}
```
</concept>
</networking_fundamentals>

<application_gateway>
## Application Gateway

**What:** Layer 7 (HTTP/HTTPS) load balancer with WAF, SSL termination, URL routing.

**When to use:**
- Web applications with multiple backends
- Need WAF (Web Application Firewall)
- SSL offloading
- URL-based routing
- Multi-site hosting

**vs Load Balancer:**
- Load Balancer: Layer 4 (TCP/UDP), simpler, cheaper
- Application Gateway: Layer 7 (HTTP), advanced features, more expensive

### Pricing

**Standard_v2:**
- Fixed: $0.246/hour = $178/month
- Capacity units: $0.008 per hour
- Data processed: $0.008/GB

**WAF_v2:**
- Fixed: $0.443/hour = $320/month
- Capacity units: $0.008 per hour
- Data processed: $0.008/GB

**Typical cost:** $200-500/month depending on traffic

### Configuration

```bicep
resource appGateway 'Microsoft.Network/applicationGateways@2023-11-01' = {
  name: 'agw-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    sku: {
      name: 'WAF_v2'
      tier: 'WAF_v2'
      capacity: 2  // 2-125 instances (auto-scale)
    }
    gatewayIPConfigurations: [
      {
        name: 'gateway-ip-config'
        properties: {
          subnet: {
            id: gatewaySubnet.id
          }
        }
      }
    ]
    frontendIPConfigurations: [
      {
        name: 'frontend-ip'
        properties: {
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
    frontendPorts: [
      {
        name: 'https-port'
        properties: {
          port: 443
        }
      }
    ]
    backendAddressPools: [
      {
        name: 'backend-pool'
        properties: {
          backendAddresses: [
            {
              fqdn: 'myapp-prod-app-001.azurewebsites.net'
            }
          ]
        }
      }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'backend-http-settings'
        properties: {
          port: 443
          protocol: 'Https'
          cookieBasedAffinity: 'Disabled'
          pickHostNameFromBackendAddress: true
          requestTimeout: 30
          probe: {
            id: resourceId('Microsoft.Network/applicationGateways/probes', 'agw-myapp-prod-eastus', 'health-probe')
          }
        }
      }
    ]
    httpListeners: [
      {
        name: 'https-listener'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations', 'agw-myapp-prod-eastus', 'frontend-ip')
          }
          frontendPort: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendPorts', 'agw-myapp-prod-eastus', 'https-port')
          }
          protocol: 'Https'
          sslCertificate: {
            id: resourceId('Microsoft.Network/applicationGateways/sslCertificates', 'agw-myapp-prod-eastus', 'ssl-cert')
          }
        }
      }
    ]
    requestRoutingRules: [
      {
        name: 'routing-rule'
        properties: {
          ruleType: 'Basic'
          priority: 100
          httpListener: {
            id: resourceId('Microsoft.Network/applicationGateways/httpListeners', 'agw-myapp-prod-eastus', 'https-listener')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/applicationGateways/backendAddressPools', 'agw-myapp-prod-eastus', 'backend-pool')
          }
          backendHttpSettings: {
            id: resourceId('Microsoft.Network/applicationGateways/backendHttpSettingsCollection', 'agw-myapp-prod-eastus', 'backend-http-settings')
          }
        }
      }
    ]
    probes: [
      {
        name: 'health-probe'
        properties: {
          protocol: 'Https'
          path: '/health'
          interval: 30
          timeout: 30
          unhealthyThreshold: 3
          pickHostNameFromBackendHttpSettings: true
        }
      }
    ]
    webApplicationFirewallConfiguration: {
      enabled: true
      firewallMode: 'Prevention'
      ruleSetType: 'OWASP'
      ruleSetVersion: '3.2'
    }
    autoscaleConfiguration: {
      minCapacity: 2
      maxCapacity: 10
    }
  }
}
```

### URL-based Routing

**Route different paths to different backends:**

```bicep
backendAddressPools: [
  {
    name: 'api-pool'
    properties: {
      backendAddresses: [{ fqdn: 'api.myapp.com' }]
    }
  }
  {
    name: 'web-pool'
    properties: {
      backendAddresses: [{ fqdn: 'web.myapp.com' }]
    }
  }
]

urlPathMaps: [
  {
    name: 'url-path-map'
    properties: {
      defaultBackendAddressPool: {
        id: webPoolId
      }
      defaultBackendHttpSettings: {
        id: httpSettingsId
      }
      pathRules: [
        {
          name: 'api-rule'
          properties: {
            paths: ['/api/*']
            backendAddressPool: {
              id: apiPoolId
            }
          }
        }
      ]
    }
  }
]
```

### WAF Rules

**Block common attacks:**
- SQL injection
- Cross-site scripting (XSS)
- Command injection
- Request smuggling

**Custom rules:**
```bicep
webApplicationFirewallConfiguration: {
  enabled: true
  firewallMode: 'Prevention'
  ruleSetType: 'OWASP'
  ruleSetVersion: '3.2'
  disabledRuleGroups: []
  requestBodyCheck: true
  maxRequestBodySizeInKb: 128
}
```
</application_gateway>

<private_endpoints_detail>
## Private Endpoints Best Practices

### When to Use

**Use Private Endpoints for:**
- ✅ Production databases (SQL, Cosmos DB)
- ✅ Production storage accounts
- ✅ Key Vault (production)
- ✅ Any service with sensitive data

**Skip for:**
- ❌ Dev/test environments (cost savings)
- ❌ Public-facing APIs (need internet access)

### Complete Setup

```bicep
// 1. Disable public access on resource
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'sql-shared-prod-eastus-001'
  location: 'eastus'
  properties: {
    publicNetworkAccess: 'Disabled'  // Critical!
  }
}

// 2. Create private endpoint
resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-sql-shared-prod'
  location: 'eastus'
  properties: {
    subnet: {
      id: dataSubnet.id
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

// 3. Create Private DNS Zone
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}

// 4. Link DNS Zone to VNet
resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'vnet-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// 5. Create DNS record for private endpoint
resource privateDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: sqlPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'config'
        properties: {
          privateDnsZoneId: privateDnsZone.id
        }
      }
    ]
  }
}
```

### Verify Private Endpoint

```bash
# Check private endpoint IP
az network private-endpoint show \
  --name pe-sql-shared-prod \
  --resource-group myapp-prod-rg \
  --query 'customDnsConfigs[0].ipAddresses[0]' -o tsv

# DNS resolution should return private IP
nslookup sql-shared-prod-eastus-001.database.windows.net

# Should return 10.0.x.x (private IP), not public IP
```

### Cost Comparison

**Service Endpoint (free):**
- Traffic on Microsoft backbone
- Still uses public IP of service
- Can restrict by VNet

**Private Endpoint ($7.30/month):**
- $0.01/hour = $7.30/month per endpoint
- Private IP in your VNet
- No public access
- Full isolation

**Recommendation:** Use Private Endpoints for production, Service Endpoints for dev/test.
</private_endpoints_detail>

<vnet_integration>
## VNet Integration for App Service

**What:** Connect App Service to VNet for outbound traffic.

**Use cases:**
- Access resources in VNet (VMs, databases)
- Access on-premises via VPN
- Route through Azure Firewall

**Requirements:**
- App Service: Standard tier or higher
- Regional VNet Integration (recommended)

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'myapp-prod-app-001'
  location: 'eastus'
  properties: {
    serverFarmId: appServicePlan.id
    virtualNetworkSubnetId: appIntegrationSubnet.id  // Enable VNet integration
    vnetRouteAllEnabled: true  // Route all traffic through VNet
  }
}

// Dedicated subnet for VNet integration
resource appIntegrationSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = {
  parent: vnet
  name: 'app-integration-subnet'
  properties: {
    addressPrefix: '10.0.10.0/24'
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
```

**Now app can:**
- Access SQL via private endpoint (10.0.2.5)
- Access VMs in VNet
- Access on-premises resources (via VPN)
</vnet_integration>

<dns>
## DNS Configuration

### Azure DNS

**Public DNS zones:**
```bicep
resource dnsZone 'Microsoft.Network/dnsZones@2018-05-01' = {
  name: 'myapp.com'
  location: 'global'
}

resource aRecord 'Microsoft.Network/dnsZones/A@2018-05-01' = {
  parent: dnsZone
  name: 'www'
  properties: {
    TTL: 3600
    ARecords: [
      {
        ipv4Address: publicIP.properties.ipAddress
      }
    ]
  }
}

resource cnameRecord 'Microsoft.Network/dnsZones/CNAME@2018-05-01' = {
  parent: dnsZone
  name: 'api'
  properties: {
    TTL: 3600
    CNAMERecord: {
      cname: 'myapp-prod-app-001.azurewebsites.net'
    }
  }
}
```

**Configure nameservers at domain registrar:**
```bash
az network dns zone show \
  --name myapp.com \
  --resource-group myapp-prod-rg \
  --query nameServers

# Returns:
# ns1-01.azure-dns.com
# ns2-01.azure-dns.net
# ns3-01.azure-dns.org
# ns4-01.azure-dns.info
```

### Private DNS Zones

**For private endpoint name resolution:**

```bicep
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}

resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: '${vnet.name}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}
```

**Common private DNS zones:**
- `privatelink.database.windows.net` - SQL Database
- `privatelink.blob.core.windows.net` - Storage Blobs
- `privatelink.vaultcore.azure.net` - Key Vault
- `privatelink.azurewebsites.net` - App Service
</dns>

<network_security_best_practices>
## Network Security Best Practices

### 1. Defense in Depth

**Multiple security layers:**
```
Internet
  ↓
Azure Firewall / Application Gateway WAF
  ↓
NSG on subnet
  ↓
Private Endpoints (no public access)
  ↓
Resource (SQL, Storage)
```

### 2. Zero Trust Network

**Principles:**
- Verify explicitly
- Least privilege access
- Assume breach

**Implementation:**
- Private Endpoints for all data services
- NSG rules: Allow only required traffic
- Service Endpoints where Private Endpoints too expensive
- Network segmentation (separate subnets for app/data)

### 3. NSG Best Practices

**Rule organization:**
```
Priority 100-199: Allow rules (specific)
Priority 200-299: Allow rules (broader)
Priority 4000+: Deny rules (catch-all)
```

**Example:**
```bicep
securityRules: [
  // Allow HTTPS from internet
  { priority: 100, name: 'allow-https', ... }

  // Allow app → SQL on private subnet
  { priority: 110, name: 'allow-app-to-sql', ... }

  // Deny all other inbound (catch-all)
  { priority: 4096, name: 'deny-all-inbound', ... }
]
```

### 4. Subnet Design

**Recommended subnets:**
- `GatewaySubnet` - VPN/ExpressRoute Gateway
- `AzureFirewallSubnet` - Azure Firewall
- `app-subnet` - App Services (via VNet integration)
- `data-subnet` - Private endpoints for databases
- `vm-subnet` - Virtual machines
- `aks-subnet` - AKS nodes

**Sizing:**
```
/24 subnet = 256 IPs - 5 (Azure reserved) = 251 usable
/25 subnet = 128 IPs - 5 = 123 usable
/26 subnet = 64 IPs - 5 = 59 usable
```

Plan for growth: Use /24 or larger for production subnets.

### 5. Monitoring

**Enable NSG flow logs:**
```bicep
resource nsgFlowLog 'Microsoft.Network/networkWatchers/flowLogs@2023-11-01' = {
  name: 'nsg-flow-log'
  location: 'eastus'
  properties: {
    targetResourceId: nsg.id
    storageId: storageAccount.id
    enabled: true
    retentionPolicy: {
      days: 30
      enabled: true
    }
    format: {
      type: 'JSON'
      version: 2
    }
  }
}
```

**Query blocked traffic:**
```kql
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where FlowDirection_s == "I"  // Inbound
| where FlowStatus_s == "D"     // Denied
| summarize count() by SrcIP_s, DestPort_d
| order by count_ desc
```
</network_security_best_practices>

<anti_patterns>
**Avoid:**

<anti_pattern name="Public Endpoints for Production">
**Problem:** SQL Database, Storage with public internet access

**Why bad:** Attack surface, data exposure risk, compliance issues

**Instead:** Use Private Endpoints for production resources
</anti_pattern>

<anti_pattern name="No NSG Rules">
**Problem:** Default allow all traffic

**Why bad:** No network security, lateral movement in breach

**Instead:** Explicit allow rules, deny all by default
</anti_pattern>

<anti_pattern name="Overly Permissive NSG">
**Problem:** Allow 0.0.0.0/0 on all ports

**Why bad:** Defeats purpose of firewall

**Instead:** Specific source IPs, specific ports, specific protocols
</anti_pattern>

<anti_pattern name="No Network Segmentation">
**Problem:** All resources in one subnet

**Why bad:** Lateral movement, hard to apply different security policies

**Instead:** Separate subnets for app, data, management
</anti_pattern>

<anti_pattern name="Mixing Environments in VNet">
**Problem:** Dev and prod in same VNet

**Why bad:** Security boundary violation, mistakes can affect production

**Instead:** Separate VNets per environment, peering if needed
</anti_pattern>
</anti_patterns>
