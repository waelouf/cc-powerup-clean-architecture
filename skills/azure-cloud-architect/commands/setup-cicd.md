---
name: setup-cicd
description: Configure CI/CD pipeline with GitHub Actions or Azure DevOps Pipelines
arguments: "[platform]"
---

<objective>
Quick access to setup CI/CD automation. Routes directly to setup-cicd-pipeline workflow.
</objective>

<invocation>
Execute the azure-cloud-architect skill with context to immediately start the setup-cicd-pipeline workflow.

User intent: Setup CI/CD pipeline
Arguments provided: $ARGUMENTS
Platform preference: $ARGUMENTS (github actions, azure devops, or ask)

Route to: workflows/setup-cicd-pipeline.md
</invocation>

<instructions>
Load the azure-cloud-architect skill and immediately execute the setup-cicd-pipeline workflow.

If arguments specify platform ("github" or "azure devops"), use that preference.

Follow the workflow steps to:
1. Choose CI/CD platform (GitHub Actions or Azure DevOps)
2. Configure federated credentials (OIDC, no secrets)
3. Create pipeline YAML
4. Setup deployment workflow
5. Test pipeline

Emphasize security best practices:
- Use federated credentials (OIDC) for GitHub Actions
- No secrets in code
- Managed Identity where possible

Do not show the skill intake menu - go directly to CI/CD setup workflow.
</instructions>
