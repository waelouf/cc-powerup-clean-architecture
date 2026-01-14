---
name: dr
description: Implement disaster recovery with Azure Backup, Site Recovery, and geo-redundancy
arguments: ""
---

<objective>
Quick access to implement disaster recovery. Routes directly to implement-disaster-recovery workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the implement-disaster-recovery workflow.

User intent: Implement disaster recovery and business continuity
Arguments provided: $ARGUMENTS

Route to: workflows/implement-disaster-recovery.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the implement-disaster-recovery workflow.

Follow the workflow steps to:
1. Define RTO and RPO requirements
2. Choose DR strategy (Backup/Restore, Pilot Light, Warm Standby, Hot Standby)
3. Configure Azure Backup
4. Enable SQL Database geo-replication and failover groups
5. Configure storage geo-redundancy
6. Setup multi-region deployment with Traffic Manager/Front Door
7. Create DR runbook and test plan

Cover key DR components:
- Recovery Services Vault with backup policies
- SQL failover groups (automatic failover)
- Storage GRS/GZRS
- Multi-region deployment
- Traffic Manager or Front Door for failover routing
- DR testing schedule (monthly backups, quarterly failover tests)

Emphasize:
- RTO/RPO targets drive DR strategy
- Cost vs availability tradeoffs
- Regular testing is critical
- Documented runbook for incident response

Do not show the skill intake menu - go directly to disaster recovery workflow.
</instructions>
