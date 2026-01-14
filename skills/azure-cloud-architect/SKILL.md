---
name: azure-cloud-architect
description: Build Azure cloud infrastructure and DevOps pipelines from scratch through production. Full lifecycle - provision, configure, deploy, monitor, secure, optimize. CLI-based with Azure CLI, Bicep/Terraform, and DevOps best practices.
---

<essential_principles>
## How Azure DevOps Architecture Works

### 1. Infrastructure as Code is the Source of Truth

All Azure resources MUST be defined in code (Bicep, Terraform, or ARM templates). Never create resources manually in the portal for production - manual changes create configuration drift and are impossible to replicate or version control.

**Use Bicep when:**
- Azure-only infrastructure
- You want the newest Azure features immediately
- Team prefers native Azure tooling
- No state file management complexity

**Use Terraform when:**
- Multi-cloud or hybrid environments
- Need mature ecosystem and community modules
- Require advanced state management and drift detection
- Team has existing Terraform expertise

### 2. Azure CLI First, Portal Never

Use Azure CLI (`az`) for all operations. The portal is for viewing and understanding, not for making changes. CLI commands are:
- Repeatable and scriptable
- Version controlled
- Auditable
- Automatable in CI/CD pipelines

### 3. Security by Default

**Never hardcode secrets.** Use:
- Azure Key Vault for secrets, keys, and certificates
- Managed Identity for authentication (no credentials in code)
- RBAC for access control (principle of least privilege)
- Azure AD integration for user authentication

RBAC has replaced legacy access policies. Always use RBAC permission model for Key Vault and other resources.

### 4. Cost Consciousness

Azure costs accumulate quickly. Every architectural decision has cost implications:
- Right-size resources (don't over-provision)
- Use auto-scaling to match actual demand
- Implement auto-shutdown for non-production resources
- Use Reserved Instances/Savings Plans for predictable workloads
- Tag all resources for cost allocation and tracking

### 5. Multi-Environment Architecture

Design for multiple environments from day one:
- **Dev**: Rapid iteration, minimal cost, auto-shutdown
- **Staging**: Production-like, for integration testing
- **Production**: High availability, disaster recovery, monitoring

Use consistent naming conventions and resource organization across environments.

### 6. Observability is Not Optional

Production systems without monitoring are blind systems:
- Application Insights for application telemetry
- Log Analytics for centralized logging
- Azure Monitor for infrastructure metrics
- Alerts for proactive issue detection

### 7. Deployment Automation

Manual deployments create inconsistency and errors. All deployments must flow through CI/CD pipelines:
- Azure DevOps Pipelines (Azure-native)
- GitHub Actions (code-to-cloud integration)
- YAML-based pipeline definitions (version controlled)

Use deployment strategies like blue-green or canary to minimize risk.
</essential_principles>

<intake>
What would you like to do?

1. **Provision new infrastructure** - Create Azure resources from scratch
2. **Setup CI/CD pipeline** - Configure automated deployment pipeline
3. **Deploy application** - Deploy app to Azure (App Service, AKS, Functions, etc.)
4. **Setup monitoring** - Configure Application Insights, Log Analytics, alerts
5. **Debug deployment** - Troubleshoot failed deployments or infrastructure issues
6. **Optimize costs** - Analyze and reduce Azure spending
7. **Secure infrastructure** - Implement security best practices (Key Vault, RBAC, MI)
8. **Setup networking** - Configure VNets, NSGs, Application Gateway
9. **Design shared resources** - Architect multi-product shared infrastructure
10. **Scale infrastructure** - Handle growth and traffic increases
11. **Setup environments** - Create dev/staging/prod environments
12. **Implement disaster recovery** - Setup backup, restore, failover
13. **Something else** - Describe what you need

**Wait for response before proceeding.**
</intake>

<routing>
| Response | Workflow |
|----------|----------|
| 1, "provision", "create", "infrastructure", "new", "bicep", "terraform" | `workflows/provision-infrastructure.md` |
| 2, "cicd", "ci/cd", "pipeline", "devops", "github actions", "deploy automation" | `workflows/setup-cicd-pipeline.md` |
| 3, "deploy", "app", "application", "release", "publish" | `workflows/deploy-application.md` |
| 4, "monitor", "monitoring", "logs", "alerts", "insights", "observability" | `workflows/setup-monitoring.md` |
| 5, "debug", "troubleshoot", "fix", "broken", "error", "failed", "issue" | `workflows/debug-deployment.md` |
| 6, "cost", "optimize", "expensive", "finops", "spending", "budget" | `workflows/optimize-costs.md` |
| 7, "secure", "security", "rbac", "key vault", "identity", "managed identity" | `workflows/secure-infrastructure.md` |
| 8, "network", "vnet", "nsg", "gateway", "subnet", "firewall" | `workflows/setup-networking.md` |
| 9, "shared", "multi-product", "multi-tenant", "share", "database server" | `workflows/design-shared-resources.md` |
| 10, "scale", "scaling", "autoscale", "grow", "traffic", "performance" | `workflows/scale-infrastructure.md` |
| 11, "environment", "dev", "staging", "prod", "production", "test" | `workflows/setup-environments.md` |
| 12, "disaster", "recovery", "backup", "restore", "failover", "dr", "bcdr" | `workflows/implement-disaster-recovery.md` |
| 13, "migrate", "migration", "move to azure" | `workflows/migrate-to-azure.md` |

**After reading the workflow, follow it exactly.**
</routing>

<verification_loop>
## After Every Change

Run these verification commands to ensure your changes work:

```bash
# 1. Validate IaC templates
az bicep build --file main.bicep
# OR for Terraform
terraform validate && terraform plan

# 2. Check resource deployment status
az deployment group show --name <deployment-name> --resource-group <rg-name>

# 3. Verify resources are running
az resource list --resource-group <rg-name> --output table

# 4. Test application endpoint (if applicable)
curl https://<your-app>.azurewebsites.net/health
# OR
az webapp show --name <app-name> --resource-group <rg-name> --query "state"

# 5. Check recent deployments and errors
az monitor activity-log list --resource-group <rg-name> --max-events 10
```

Report to user:
- **Infrastructure**: ✓ Deployed / ✗ Failed (with error details)
- **Application**: ✓ Running / ✗ Stopped
- **Monitoring**: ✓ Collecting data / ⚠ Needs configuration
- **Cost estimate**: $X/month for current configuration

Always provide next steps or recommendations for improvement.
</verification_loop>

<reference_index>
## Domain Knowledge

All in `references/`:

**Infrastructure:** infrastructure-as-code.md, compute-services.md, storage-data.md
**Architecture:** architecture-patterns.md, multi-tenancy-patterns.md
**Deployment:** cicd-pipelines.md, deployment-strategies.md
**Networking:** networking.md
**Kubernetes:** kubernetes-aks.md
**Security:** security-identity.md
**Observability:** monitoring-observability.md
**Operations:** cost-optimization.md, disaster-recovery.md
**Organization:** resource-organization.md, anti-patterns.md
</reference_index>

<workflows_index>
## Workflows

All in `workflows/`:

| Workflow | Purpose |
|----------|---------|
| provision-infrastructure.md | Create new Azure infrastructure from scratch |
| setup-cicd-pipeline.md | Configure CI/CD with Azure DevOps or GitHub Actions |
| deploy-application.md | Deploy apps to Azure services |
| setup-monitoring.md | Configure monitoring, logging, and alerts |
| debug-deployment.md | Troubleshoot failed deployments and issues |
| optimize-costs.md | Analyze and reduce Azure spending |
| secure-infrastructure.md | Implement security best practices |
| setup-networking.md | Configure VNets, NSGs, Application Gateway |
| design-shared-resources.md | Architect shared infrastructure for multiple products |
| scale-infrastructure.md | Handle growth and traffic scaling |
| setup-environments.md | Create dev/staging/prod environments |
| implement-disaster-recovery.md | Setup backup, restore, and failover |
| migrate-to-azure.md | Migrate existing infrastructure to Azure |
</workflows_index>
