---
name: secure
description: Implement Azure security best practices (Managed Identity, Key Vault, RBAC, Private Endpoints)
arguments: ""
---

<objective>
Quick access to implement Azure security. Routes directly to secure-infrastructure workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the secure-infrastructure workflow.

User intent: Secure Azure infrastructure
Arguments provided: $ARGUMENTS

Route to: workflows/secure-infrastructure.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the secure-infrastructure workflow.

Follow the workflow steps to:
1. Enable Managed Identity (zero credentials)
2. Setup Key Vault with RBAC (not access policies)
3. Configure network security (VNet, NSGs, Private Endpoints)
4. Implement RBAC with least privilege
5. Enable Azure Defender
6. Apply Azure Policy for governance

Emphasize security principles:
- Zero credentials (Managed Identity for everything)
- Private Endpoints for all data services
- RBAC over legacy access policies
- Network isolation (NSGs, subnets)
- Secrets in Key Vault, never in code

Do not show the skill intake menu - go directly to security workflow.
</instructions>
