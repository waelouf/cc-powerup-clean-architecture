---
name: shared-resources
description: Design cross-product shared infrastructure (database servers, Key Vaults, networking)
arguments: "[resource-type]"
---

<objective>
Quick access to design shared resources architecture. Routes directly to design-shared-resources workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the design-shared-resources workflow.

User intent: Design shared resources for multiple products
Resource type: $ARGUMENTS (database server, key vault, networking, or general)

Route to: workflows/design-shared-resources.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the design-shared-resources workflow.

If arguments specify resource type, focus on that specific shared resource pattern.

Follow the workflow steps to:
1. Identify which resources to share
2. Design multi-tenancy pattern (shared database/schema vs separate)
3. Configure security and isolation
4. Implement cost allocation (tags, naming)
5. Setup governance and documentation

Cover common shared resource patterns:
- SQL Server with multiple databases (one per product)
- Shared Key Vault with RBAC isolation
- App Service Plan hosting multiple apps
- Shared networking (VNet, Application Gateway)
- Shared monitoring (Log Analytics, Application Insights)

Emphasize:
- Security isolation despite sharing
- Cost allocation strategies
- Governance and documentation
- When to share vs when to separate

Do not show the skill intake menu - go directly to shared resources workflow.
</instructions>
