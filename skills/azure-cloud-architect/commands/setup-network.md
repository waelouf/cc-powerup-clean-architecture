---
name: setup-network
description: Configure Azure networking (VNets, NSGs, Application Gateway, Private Endpoints)
arguments: ""
---

<objective>
Quick access to setup Azure networking. Routes directly to setup-networking workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the setup-networking workflow.

User intent: Setup Azure networking
Arguments provided: $ARGUMENTS

Route to: workflows/setup-networking.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the setup-networking workflow.

Follow the workflow steps to:
1. Design network architecture (hub-spoke vs simple)
2. Create Virtual Network with subnets
3. Configure Network Security Groups
4. Setup Application Gateway with WAF
5. Create Private Endpoints for data services
6. Configure App Service VNet integration
7. Verify network configuration

Cover key networking components:
- VNet with proper subnetting (gateway, app, data, management)
- NSGs with least-privilege rules
- Application Gateway for external traffic (Layer 7 + WAF)
- Private Endpoints for SQL, Storage, Key Vault
- App Service VNet integration
- Private DNS zones

Emphasize security:
- Network isolation (subnets, NSGs)
- Private Endpoints (no public access to data)
- Application Gateway as single entry point
- Defense in depth

Do not show the skill intake menu - go directly to networking workflow.
</instructions>
