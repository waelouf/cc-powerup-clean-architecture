---
name: provision
description: Provision new Azure infrastructure from scratch using Bicep or Terraform
arguments: "[resource-type]"
---

<objective>
Quick access to provision new Azure infrastructure. Routes directly to the provision-infrastructure workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the provision-infrastructure workflow.

User intent: Provision new Azure infrastructure
Arguments provided: $ARGUMENTS

Route to: workflows/provision-infrastructure.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the provision-infrastructure workflow.

If arguments are provided (e.g., "app service", "aks cluster", "sql database"), use those to guide the infrastructure type.

Follow the workflow steps to:
1. Gather requirements
2. Choose IaC tool (Bicep vs Terraform)
3. Create infrastructure code
4. Deploy to Azure
5. Verify deployment

Do not show the skill intake menu - go directly to provisioning workflow.
</instructions>
