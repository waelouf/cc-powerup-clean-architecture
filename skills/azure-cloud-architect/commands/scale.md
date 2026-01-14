---
name: scale
description: Configure auto-scaling for App Service, AKS, or Azure Functions
arguments: "[service-type]"
---

<objective>
Quick access to configure auto-scaling. Routes directly to scale-infrastructure workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the scale-infrastructure workflow.

User intent: Configure auto-scaling
Service type: $ARGUMENTS (app service, aks, functions, or ask)

Route to: workflows/scale-infrastructure.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the scale-infrastructure workflow.

If arguments specify service type, use that to guide scaling configuration.

Follow the workflow steps to:
1. Understand scaling options (vertical vs horizontal)
2. Configure auto-scale rules based on metrics
3. Set appropriate min/max instance counts
4. Configure cooldown periods
5. Setup schedule-based scaling (business hours)
6. Test scaling behavior

Cover scaling for:
- App Service (CPU, memory, queue length)
- AKS (HPA, Cluster Autoscaler, KEDA)
- Azure Functions (Consumption vs Premium)
- SQL Database (serverless auto-scale)
- Cosmos DB (auto-scale RU/s)

Do not show the skill intake menu - go directly to scaling workflow.
</instructions>
