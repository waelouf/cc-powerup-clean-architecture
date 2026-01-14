---
name: deploy
description: Deploy application to Azure (App Service, AKS, Functions, Container Apps)
arguments: "[app-type]"
---

<objective>
Quick access to deploy applications to Azure. Routes directly to deploy-application workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the deploy-application workflow.

User intent: Deploy application to Azure
Arguments provided: $ARGUMENTS

Route to: workflows/deploy-application.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the deploy-application workflow.

If arguments are provided (e.g., "app service", "aks", "functions"), use those to guide the deployment target.

Follow the workflow steps to:
1. Identify application type
2. Choose Azure compute service
3. Deploy application
4. Configure deployment slots (if applicable)
5. Verify deployment

Do not show the skill intake menu - go directly to deployment workflow.
</instructions>
