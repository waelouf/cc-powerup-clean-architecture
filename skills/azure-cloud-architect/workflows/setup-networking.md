<required_reading>
Before configuring networking, understand:
- `references/networking.md` - VNet, NSG, Application Gateway, Private Endpoints
- `references/security-identity.md` - Network security best practices
- `references/kubernetes-aks.md` - AKS networking if using Kubernetes
</required_reading>

<objective>
Configure Azure networking including VNets, subnets, NSGs, Application Gateway, Private Endpoints, and secure network architecture for production workloads.
</objective>

<quick_start>
**5-minute basic network:**
```bash
# Create VNet with subnets
az network vnet create \
  --name vnet-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --location eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-app \
  --subnet-prefix 10.0.1.0/24

# Add data subnet
az network vnet subnet create \
  --name snet-data \
  --resource-group myapp-prod-rg \
  --vnet-name vnet-myapp-prod-eastus \
  --address-prefix 10.0.2.0/24

# Create NSG
az network nsg create \
  --name nsg-app-prod-eastus \
  --resource-group myapp-prod-rg \
  --location eastus

# Add rule to allow HTTPS
az network nsg rule create \
  --name AllowHTTPS \
  --nsg-name nsg-app-prod-eastus \
  --resource-group myapp-prod-rg \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-range 443

# Associate NSG with subnet
az network vnet subnet update \
  --name snet-app \
  --resource-group myapp-prod-rg \
  --vnet-name vnet-myapp-prod-eastus \
  --network-security-group nsg-app-prod-eastus
```
</quick_start>

<process>
## Step 1: Design Network Architecture

**1.1 Hub-spoke topology (recommended for enterprises):**

```
Hub VNet (10.0.0.0/16)
├── Gateway subnet (10.0.0.0/24) - VPN/ExpressRoute
├── Firewall subnet (10.0.1.0/24) - Azure Firewall
└── Shared services (10.0.2.0/24) - DNS, monitoring

Spoke VNet 1 - App A (10.1.0.0/16)
├── App subnet (10.1.1.0/24)
├── Data subnet (10.1.2.0/24)
└── Integration subnet (10.1.3.0/24)

Spoke VNet 2 - App B (10.2.0.0/16)
├── App subnet (10.2.1.0/24)
└── Data subnet (10.2.2.0/24)
```

**1.2 Simple topology (for small/medium apps):**

```
VNet (10.0.0.0/16)
├── Gateway subnet (10.0.0.0/24) - App Gateway
├── App subnet (10.0.1.0/24) - App Services
├── AKS subnet (10.0.2.0/23) - Kubernetes (needs /23 or larger)
├── Data subnet (10.0.4.0/24) - Private Endpoints
└── Management (10.0.5.0/24) - Bastion, VMs
```

## Step 2: Create Virtual Network

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
          addressPrefix: '10.0.0.0/24'
        }
      }
      {
        name: 'snet-app'
        properties: {
          addressPrefix: '10.0.1.0/24'
          serviceEndpoints: [
            { service: 'Microsoft.Web' }
          ]
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'  // For App Service VNet integration
              }
            }
          ]
        }
      }
      {
        name: 'snet-aks'
        properties: {
          addressPrefix: '10.0.2.0/23'  // AKS needs larger subnet
          serviceEndpoints: []
        }
      }
      {
        name: 'snet-data'
        properties: {
          addressPrefix: '10.0.4.0/24'
          privateEndpointNetworkPolicies: 'Disabled'  // Required for Private Endpoints
          privateLinkServiceNetworkPolicies: 'Disabled'
        }
      }
      {
        name: 'AzureBastionSubnet'  // Exact name required
        properties: {
          addressPrefix: '10.0.5.0/27'  // Minimum /27 for Bastion
        }
      }
    ]
  }
}
```

## Step 3: Network Security Groups (NSGs)

**3.1 App tier NSG:**

```bicep
resource nsgApp 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-app-prod-eastus'
  location: 'eastus'
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          destinationAddressPrefix: '10.0.1.0/24'
          destinationPortRange: '443'
        }
      }
      {
        name: 'AllowHTTP'  // Optional, redirect to HTTPS in app
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          destinationAddressPrefix: '10.0.1.0/24'
          destinationPortRange: '80'
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

**3.2 Data tier NSG:**

```bicep
resource nsgData 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-data-prod-eastus'
  location: 'eastus'
  properties: {
    securityRules: [
      {
        name: 'AllowAppToSQL'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.1.0/24'  // App subnet only
          destinationAddressPrefix: '10.0.4.0/24'
          destinationPortRange: '1433'
        }
      }
      {
        name: 'AllowAKSToData'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.2.0/23'  // AKS subnet
          destinationAddressPrefix: '10.0.4.0/24'
          destinationPortRanges: ['1433', '5432', '3306']  // SQL, PostgreSQL, MySQL
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

**3.3 Associate NSGs with subnets:**

```bicep
resource subnetApp 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = {
  parent: vnet
  name: 'snet-app'
  properties: {
    addressPrefix: '10.0.1.0/24'
    networkSecurityGroup: {
      id: nsgApp.id
    }
  }
}
```

## Step 4: Application Gateway (Layer 7 Load Balancer + WAF)

```bicep
// Public IP for Application Gateway
resource publicIP 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: 'pip-appgw-myapp-prod-eastus'
  location: 'eastus'
  sku: {
    name: 'Standard'
    tier: 'Regional'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
    dnsSettings: {
      domainNameLabel: 'myapp-prod'
    }
  }
}

resource applicationGateway 'Microsoft.Network/applicationGateways@2023-11-01' = {
  name: 'ag-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    sku: {
      name: 'WAF_v2'
      tier: 'WAF_v2'
      capacity: 2  // Auto-scale between 2-10
    }
    autoscaleConfiguration: {
      minCapacity: 2
      maxCapacity: 10
    }
    gatewayIPConfigurations: [
      {
        name: 'appGatewayIpConfig'
        properties: {
          subnet: {
            id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnet.name, 'snet-gateway')
          }
        }
      }
    ]
    frontendIPConfigurations: [
      {
        name: 'appGatewayFrontendIP'
        properties: {
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
    frontendPorts: [
      {
        name: 'port_443'
        properties: {
          port: 443
        }
      }
      {
        name: 'port_80'
        properties: {
          port: 80
        }
      }
    ]
    backendAddressPools: [
      {
        name: 'appServiceBackend'
        properties: {
          backendAddresses: [
            {
              fqdn: 'app-myapp-prod-eastus.azurewebsites.net'
            }
          ]
        }
      }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'appServiceSettings'
        properties: {
          port: 443
          protocol: 'Https'
          cookieBasedAffinity: 'Disabled'
          pickHostNameFromBackendAddress: true
          requestTimeout: 30
          probe: {
            id: resourceId('Microsoft.Network/applicationGateways/probes', 'ag-myapp-prod-eastus', 'healthProbe')
          }
        }
      }
    ]
    httpListeners: [
      {
        name: 'httpsListener'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations', 'ag-myapp-prod-eastus', 'appGatewayFrontendIP')
          }
          frontendPort: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendPorts', 'ag-myapp-prod-eastus', 'port_443')
          }
          protocol: 'Https'
          sslCertificate: {
            id: resourceId('Microsoft.Network/applicationGateways/sslCertificates', 'ag-myapp-prod-eastus', 'appGatewaySslCert')
          }
        }
      }
    ]
    requestRoutingRules: [
      {
        name: 'rule1'
        properties: {
          ruleType: 'Basic'
          priority: 100
          httpListener: {
            id: resourceId('Microsoft.Network/applicationGateways/httpListeners', 'ag-myapp-prod-eastus', 'httpsListener')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/applicationGateways/backendAddressPools', 'ag-myapp-prod-eastus', 'appServiceBackend')
          }
          backendHttpSettings: {
            id: resourceId('Microsoft.Network/applicationGateways/backendHttpSettingsCollection', 'ag-myapp-prod-eastus', 'appServiceSettings')
          }
        }
      }
    ]
    probes: [
      {
        name: 'healthProbe'
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
      firewallMode: 'Prevention'  // Detection or Prevention
      ruleSetType: 'OWASP'
      ruleSetVersion: '3.2'
    }
  }
}
```

## Step 5: Private Endpoints

**5.1 SQL Database Private Endpoint:**

```bicep
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

// Private DNS Zone
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

## Step 6: App Service VNet Integration

```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-prod-eastus'
  location: 'eastus'
  properties: {
    virtualNetworkSubnetId: resourceId('Microsoft.Network/virtualNetworks/subnets', vnet.name, 'snet-app')
    httpsOnly: true
  }
}

// Allow only traffic from Application Gateway
resource appAccessRestriction 'Microsoft.Web/sites/config@2023-12-01' = {
  parent: appService
  name: 'web'
  properties: {
    ipSecurityRestrictions: [
      {
        vnetSubnetResourceId: resourceId('Microsoft.Network/virtualNetworks/subnets', vnet.name, 'snet-gateway')
        action: 'Allow'
        priority: 100
        name: 'AllowAppGateway'
      }
      {
        ipAddress: 'Any'
        action: 'Deny'
        priority: 2147483647
        name: 'DenyAll'
      }
    ]
  }
}
```

## Step 7: Verify Network Configuration

```bash
# Check VNet
az network vnet show \
  --name vnet-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Subnets:subnets[].name}"

# Check NSG rules
az network nsg rule list \
  --nsg-name nsg-app-prod-eastus \
  --resource-group myapp-prod-rg \
  --output table

# Check Private Endpoint
az network private-endpoint show \
  --name pe-sql-myapp-prod-eastus \
  --resource-group myapp-prod-rg \
  --query "{Name:name, PrivateIP:networkInterfaces[0].ipConfigurations[0].privateIPAddress}"

# Test connectivity from App Service
az webapp ssh --name app-myapp-prod-eastus --resource-group myapp-prod-rg
# Inside container:
nslookup sql-myapp-prod-eastus.database.windows.net  # Should resolve to private IP
```
</process>

<networking_checklist>
## Production Networking Checklist

- [ ] VNet created with appropriate address space
- [ ] Subnets designed for each tier (app, data, gateway)
- [ ] NSGs configured on all subnets
- [ ] NSG rules follow least privilege (deny by default)
- [ ] Application Gateway configured with WAF
- [ ] SSL certificate configured on Application Gateway
- [ ] Health probes configured for backend pools
- [ ] Private Endpoints created for all data services
- [ ] Private DNS zones configured and linked to VNet
- [ ] App Service VNet integration enabled
- [ ] Access restrictions configured on App Service
- [ ] Public network access disabled for data services

### Optional (for complex scenarios)
- [ ] Hub-spoke topology with VNet peering
- [ ] Azure Firewall for centralized traffic filtering
- [ ] VPN Gateway or ExpressRoute for on-premises connectivity
- [ ] Azure Bastion for secure VM access
- [ ] DDoS Protection Standard enabled
</networking_checklist>

<success_criteria>
Networking is properly configured when:
- All resources communicate via private IPs only
- Internet traffic flows through Application Gateway + WAF
- NSG flow logs show no unauthorized access attempts
- Private Endpoints resolve to private IPs from VNet
- App Service cannot be accessed directly (only via Application Gateway)
- Health checks pass for all backend pools
- SSL/TLS termination works correctly at Application Gateway
</success_criteria>
