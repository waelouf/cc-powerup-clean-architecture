---
name: debug
description: Debug failed Azure deployments or troubleshoot infrastructure issues
arguments: "[issue-description]"
---

<objective>
Quick access to troubleshoot Azure deployment failures. Routes directly to debug-deployment workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the debug-deployment workflow.

User intent: Debug Azure deployment or infrastructure issue
Issue description: $ARGUMENTS

Route to: workflows/debug-deployment.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the debug-deployment workflow.

If arguments describe the issue, use that context to guide troubleshooting.

Follow the systematic debugging methodology:
1. Run quick diagnostic commands
2. Identify failure type (infrastructure, application, runtime)
3. Get detailed error messages
4. Apply fixes based on common error patterns
5. Verify resolution

Provide specific Azure CLI commands to:
- Check deployment status
- View logs
- Check resource health
- Query Application Insights
- Review activity logs

Do not show the skill intake menu - go directly to debugging workflow.
</instructions>
