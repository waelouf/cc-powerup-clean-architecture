---
name: optimize-costs
description: Analyze and reduce Azure spending using FinOps best practices
arguments: ""
---

<objective>
Quick access to optimize Azure costs. Routes directly to optimize-costs workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the optimize-costs workflow.

User intent: Optimize Azure costs
Arguments provided: $ARGUMENTS

Route to: workflows/optimize-costs.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the optimize-costs workflow.

Follow the workflow steps to:
1. Analyze current costs (Azure Cost Management)
2. Identify optimization opportunities
3. Right-size resources based on actual usage
4. Implement auto-shutdown for dev/test
5. Purchase Reserved Instances for steady-state workloads
6. Setup budget alerts
7. Tag resources for cost allocation

Focus on quick wins:
- Auto-shutdown for dev/test VMs (save 70%)
- Reserved Instances for production (save 30-50%)
- Right-size oversized resources
- Delete unused resources
- Implement auto-scaling

Provide specific Azure CLI commands to analyze costs and implement optimizations.

Do not show the skill intake menu - go directly to cost optimization workflow.
</instructions>
