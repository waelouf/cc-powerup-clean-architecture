---
name: monitor
description: Setup monitoring, logging, and alerts with Application Insights and Log Analytics
arguments: ""
---

<objective>
Quick access to configure Azure monitoring. Routes directly to setup-monitoring workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the setup-monitoring workflow.

User intent: Setup monitoring and observability
Arguments provided: $ARGUMENTS

Route to: workflows/setup-monitoring.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the setup-monitoring workflow.

Follow the workflow steps to:
1. Create Log Analytics workspace
2. Create Application Insights
3. Configure application telemetry
4. Setup diagnostic settings
5. Create alerts (response time, error rate, availability)
6. Configure dashboards

Focus on production monitoring checklist:
- Application Insights for app telemetry
- Alerts for response time, errors, availability, CPU, memory
- Action groups for notifications
- Availability tests from multiple regions
- KQL queries for common scenarios

Do not show the skill intake menu - go directly to monitoring setup workflow.
</instructions>
