---
name: migrate
description: Migrate on-premises or cloud infrastructure to Azure
arguments: "[source]"
---

<objective>
Quick access to Azure migration. Routes directly to migrate-to-azure workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the migrate-to-azure workflow.

User intent: Migrate to Azure
Source environment: $ARGUMENTS (on-premises, AWS, other cloud, or ask)

Route to: workflows/migrate-to-azure.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the migrate-to-azure workflow.

If arguments describe source environment, use that context to guide migration strategy.

Follow the workflow steps to:
1. Choose migration strategy (6 Rs: Rehost, Refactor, Rearchitect, Rebuild, Replace, Retire)
2. Run Azure Migrate assessment
3. Plan migration (VMs, databases, web apps, data)
4. Execute migration with minimal downtime
5. Perform cutover
6. Post-migration optimization

Cover migration paths:
- VMs: Azure Migrate
- SQL Server: Database Migration Service
- MySQL/PostgreSQL: Azure Database Migration Service
- Web apps: App Service migration assistant
- Data: AzCopy, Data Box

Emphasize:
- Assessment phase is critical
- Test migrations before production cutover
- Documented cutover plan with rollback
- Post-migration optimization (right-sizing, costs)

Do not show the skill intake menu - go directly to migration workflow.
</instructions>
