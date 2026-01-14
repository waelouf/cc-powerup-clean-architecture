---
name: environments
description: Setup dev, staging, and production environments with proper isolation
arguments: ""
---

<objective>
Quick access to setup multiple environments. Routes directly to setup-environments workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the setup-environments workflow.

User intent: Setup multiple environments (dev, staging, production)
Arguments provided: $ARGUMENTS

Route to: workflows/setup-environments.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the setup-environments workflow.

Follow the workflow steps to:
1. Design environment strategy (isolation via resource groups or subscriptions)
2. Create parameterized infrastructure (Bicep/Terraform)
3. Configure environment-specific settings (SKUs, instance counts, retention)
4. Setup RBAC per environment
5. Configure CI/CD for multi-environment deployment
6. Verify environments

Cover environment-specific configurations:
- **Dev**: Smaller SKUs, auto-shutdown, single instance, 7-day logs, Contributor access
- **Staging**: Production-like SKUs, 2 instances, 30-day logs, Reader access for devs
- **Production**: Production SKUs, 3+ instances with zone redundancy, 90-day logs, restricted access

Emphasize:
- Consistent naming conventions with environment suffix
- Tags for cost allocation (Environment=dev/staging/prod)
- Parameterized IaC for multi-environment deployment
- RBAC isolation (least privilege)
- Environment-specific cost optimizations

Do not show the skill intake menu - go directly to environments setup workflow.
</instructions>
