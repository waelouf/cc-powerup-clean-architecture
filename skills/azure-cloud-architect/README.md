# Azure DevOps Architect Skill

> Expert guidance for building Azure cloud infrastructure and DevOps pipelines from scratch through production. Full lifecycle coverage - provision, configure, deploy, monitor, secure, optimize.

## Overview

This domain expertise skill provides comprehensive, research-backed guidance for Azure DevOps architecture based on 2024-2025 best practices. It covers the complete lifecycle from initial provisioning through production operations, with emphasis on:

- **Infrastructure as Code**: Bicep and Terraform with real-world examples
- **Security by Default**: Managed Identity, Private Endpoints, zero credentials
- **Cost Optimization**: FinOps principles with actual pricing and savings calculations
- **Production Readiness**: Multi-environment setup, monitoring, disaster recovery
- **DevOps Automation**: CI/CD pipelines with GitHub Actions and Azure DevOps

## Quick Start

### Basic Usage

Invoke the skill to see the interactive menu:

```
/azure-cloud-architect
```

You'll be presented with 13 options:
1. Provision new infrastructure
2. Setup CI/CD pipeline
3. Deploy application
4. Setup monitoring
5. Debug deployment
6. Optimize costs
7. Secure infrastructure
8. Setup networking
9. Design shared resources
10. Scale infrastructure
11. Setup environments
12. Implement disaster recovery
13. Something else

### Fast-Track Commands

Skip the menu and go directly to a specific workflow:

```bash
# Provision infrastructure
/provision

# Deploy an application
/deploy

# Setup CI/CD with GitHub Actions
/setup-cicd github

# Debug a failed deployment
/debug

# Setup monitoring
/monitor

# Implement security best practices
/secure

# Optimize costs
/optimize-costs

# Configure networking
/setup-network

# Design shared resources (multi-product infrastructure)
/shared-resources

# Scale infrastructure
/scale

# Setup dev/staging/prod environments
/environments

# Implement disaster recovery
/dr

# Migrate to Azure
/migrate
```

## Example Scenarios

### Scenario 1: New Azure Project

**Goal**: Start a new web application in Azure with proper infrastructure.

```bash
# Step 1: Setup environments first
/environments

# Step 2: Provision infrastructure
/provision app service

# Step 3: Setup CI/CD
/setup-cicd github

# Step 4: Deploy application
/deploy

# Step 5: Setup monitoring
/monitor

# Step 6: Secure infrastructure
/secure
```

### Scenario 2: Multi-Product Shared Infrastructure

**Goal**: Design database server shared across multiple products.

```bash
# Use the shared resources workflow
/shared-resources database server
```

You'll get guidance on:
- Multi-tenancy patterns (shared DB/schema, separate schema, separate DB)
- Security architecture with Managed Identity
- Cost allocation strategies
- Governance and documentation templates

### Scenario 3: Production Issues

**Goal**: Debug deployment failure and 502 errors.

```bash
# Quick diagnostic
/debug deployment failed with 502 errors
```

You'll get:
- Immediate diagnostic commands
- Common error patterns and fixes
- Step-by-step troubleshooting methodology
- Rollback strategies

### Scenario 4: Cost Optimization

**Goal**: Reduce monthly Azure spending.

```bash
# Analyze and optimize costs
/optimize-costs
```

You'll get:
- 7 optimization strategies with scripts
- Right-sizing guidance based on metrics
- Reserved Instance calculations
- Auto-shutdown for dev/test resources
- Budget alerts configuration

## Structure

### Workflows (13)

Located in `workflows/`, these are comprehensive step-by-step guides:

| Workflow | Purpose | Command |
|----------|---------|---------|
| provision-infrastructure.md | Create new Azure infrastructure from scratch | `/provision` |
| setup-cicd-pipeline.md | Configure CI/CD with Azure DevOps or GitHub Actions | `/setup-cicd` |
| deploy-application.md | Deploy apps to Azure services | `/deploy` |
| setup-monitoring.md | Configure monitoring, logging, and alerts | `/monitor` |
| debug-deployment.md | Troubleshoot failed deployments and issues | `/debug` |
| optimize-costs.md | Analyze and reduce Azure spending | `/optimize-costs` |
| secure-infrastructure.md | Implement security best practices | `/secure` |
| setup-networking.md | Configure VNets, NSGs, Application Gateway | `/setup-network` |
| design-shared-resources.md | Architect shared infrastructure for multiple products | `/shared-resources` |
| scale-infrastructure.md | Handle growth and traffic scaling | `/scale` |
| setup-environments.md | Create dev/staging/prod environments | `/environments` |
| implement-disaster-recovery.md | Setup backup, restore, and failover | `/dr` |
| migrate-to-azure.md | Migrate existing infrastructure to Azure | `/migrate` |

### Reference Files (15)

Located in `references/`, these contain deep domain knowledge:

**Infrastructure:**
- `infrastructure-as-code.md` - Bicep vs Terraform, Azure Verified Modules, state management
- `compute-services.md` - App Service, Functions, Container Apps, AKS, VMs decision matrix
- `storage-data.md` - Azure SQL, Cosmos DB, Storage Accounts with pricing

**Architecture:**
- `architecture-patterns.md` - Microservices, API Gateway, CQRS, Circuit Breaker, BFF
- `multi-tenancy-patterns.md` - Shared database/schema, database-per-tenant, governance
- `anti-patterns.md` - What NOT to do across all domains

**Deployment:**
- `cicd-pipelines.md` - GitHub Actions with OIDC (no secrets!), Azure DevOps
- `deployment-strategies.md` - Blue-green, canary, rolling updates, zero-downtime

**Networking:**
- `networking.md` - VNets, NSGs, Private Endpoints, Application Gateway

**Kubernetes:**
- `kubernetes-aks.md` - Production AKS setup, node pools, auto-scaling, security

**Security:**
- `security-identity.md` - Managed Identity (zero credentials), RBAC, Key Vault

**Observability:**
- `monitoring-observability.md` - Application Insights, Log Analytics, KQL queries

**Operations:**
- `cost-optimization.md` - FinOps fundamentals, real pricing, savings calculations
- `disaster-recovery.md` - RTO/RPO, Azure Backup, Site Recovery, DR testing

**Organization:**
- `resource-organization.md` - Naming conventions, tagging strategies, management groups

## Key Principles

This skill follows these core principles across all workflows:

### 1. Infrastructure as Code is the Source of Truth

All Azure resources MUST be defined in code (Bicep or Terraform). Never create resources manually in the portal for production.

**Use Bicep when:**
- Azure-only infrastructure
- Want newest Azure features immediately
- Prefer native Azure tooling

**Use Terraform when:**
- Multi-cloud or hybrid environments
- Need mature ecosystem and community modules
- Require advanced state management

### 2. Azure CLI First, Portal Never

Use Azure CLI (`az`) for all operations. The portal is for viewing and understanding, not for making changes.

### 3. Security by Default

**Never hardcode secrets.** Always use:
- Azure Key Vault for secrets, keys, and certificates
- Managed Identity for authentication (no credentials in code)
- RBAC for access control (principle of least privilege)
- Private Endpoints for data services (no public access)

### 4. Cost Consciousness

Every architectural decision has cost implications:
- Right-size resources (don't over-provision)
- Use auto-scaling to match actual demand
- Implement auto-shutdown for non-production resources
- Use Reserved Instances/Savings Plans for predictable workloads
- Tag all resources for cost allocation

### 5. Multi-Environment Architecture

Design for multiple environments from day one:
- **Dev**: Rapid iteration, minimal cost, auto-shutdown
- **Staging**: Production-like, for integration testing
- **Production**: High availability, disaster recovery, monitoring

### 6. Observability is Not Optional

Production systems without monitoring are blind systems:
- Application Insights for application telemetry
- Log Analytics for centralized logging
- Azure Monitor for infrastructure metrics
- Alerts for proactive issue detection

### 7. Deployment Automation

Manual deployments create inconsistency and errors. All deployments must flow through CI/CD pipelines with:
- YAML-based pipeline definitions (version controlled)
- Deployment strategies (blue-green, canary)
- Automated testing and validation

## Code Examples

All workflows include production-ready code examples:

### Bicep Example
```bicep
param environment string  // 'dev', 'staging', 'prod'
param location string = resourceGroup().location

var config = {
  dev: { sku: 'B1', instances: 1 }
  staging: { sku: 'P1v3', instances: 2 }
  prod: { sku: 'P2v3', instances: 3 }
}[environment]

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: 'app-myapp-${environment}-${location}'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
  }
}
```

### Terraform Example
```hcl
variable "environment" {
  type = string
}

locals {
  config = {
    dev     = { sku = "B1", instances = 1 }
    staging = { sku = "P1v3", instances = 2 }
    prod    = { sku = "P2v3", instances = 3 }
  }
}

resource "azurerm_service_plan" "main" {
  name                = "asp-myapp-${var.environment}-eastus"
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_name            = local.config[var.environment].sku
  worker_count        = local.config[var.environment].instances
}
```

### GitHub Actions CI/CD
```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

- name: Deploy infrastructure
  run: |
    az deployment group create \
      --resource-group myapp-prod-rg \
      --template-file main.bicep \
      --parameters main.parameters.prod.json
```

## What Makes This Skill Special

✅ **Research-backed** - Based on 2024-2025 Azure best practices and official documentation

✅ **Production-ready** - Real-world examples, not toy demos

✅ **Complete code** - Full Bicep/Terraform templates, not just snippets

✅ **Cost-conscious** - Actual 2024-2025 pricing with savings calculations

✅ **Security-first** - Managed Identity, Private Endpoints, zero credentials

✅ **Decision guidance** - When to use Bicep vs Terraform, App Service vs AKS, etc.

✅ **Anti-patterns included** - Shows what NOT to do (just as important as best practices)

✅ **Full lifecycle** - Build → Debug → Optimize → Ship

## Common Workflows

### Daily Development

```bash
# Deploy changes to dev
/deploy

# Check monitoring
/monitor

# Debug issues
/debug
```

### Weekly Operations

```bash
# Review costs
/optimize-costs

# Check security posture
/secure
```

### Monthly Planning

```bash
# Scale for expected growth
/scale

# Review DR readiness
/dr
```

### Project Milestones

```bash
# Setup new environment
/environments

# Configure networking
/setup-network

# Implement shared infrastructure
/shared-resources
```

## Tips for Best Results

1. **Be specific**: Instead of `/provision`, try `/provision aks cluster for production`
2. **Provide context**: Mention your current setup, constraints, or requirements
3. **Ask follow-ups**: The skill has deep knowledge - don't hesitate to ask for clarification
4. **Copy code carefully**: All examples are production-ready but adjust for your naming conventions
5. **Check references**: If you need deeper understanding, ask to see specific reference files

## Pricing Examples (2024-2025)

The skill includes real pricing throughout:

**App Service (Linux):**
- B1 (dev): $13/month
- P1v3 (staging): $146/month
- P2v3 (prod): $292/month

**Azure SQL Database:**
- Basic (dev): $5/month
- S3 (prod): $150/month
- P2 (high performance): $500/month

**Savings:**
- Reserved Instances: 31-50% off
- Auto-shutdown (dev): ~$55/month per VM
- Right-sizing: Often 30-50% savings

## Support and Feedback

This skill is part of the Azure DevOps architect expertise collection. For issues or suggestions, please provide feedback through Claude Code.

## Version

**Version**: 1.0.0
**Last Updated**: January 2025
**Azure API Versions**: 2023-2024 stable releases
**Terraform Provider**: azurerm 3.x+
**Bicep**: 0.x latest

---

**Ready to build production-grade Azure infrastructure?**

Start with: `/azure-cloud-architect` or jump directly to any workflow with the slash commands above.
