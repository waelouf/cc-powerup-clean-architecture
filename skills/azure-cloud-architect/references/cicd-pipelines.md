<overview>
Comprehensive guide to CI/CD pipelines for Azure, covering Azure DevOps Pipelines and GitHub Actions. Includes best practices, patterns, security, and real-world examples for 2024-2025.
</overview>

<platform_comparison>
## Azure DevOps vs GitHub Actions

<platform name="GitHub Actions">
**Current Status:** Industry-leading CI/CD for GitHub repositories (2024-2025)

**When to use:**
- Code hosted on GitHub
- Want code-to-cloud integration
- Open source projects
- Prefer marketplace ecosystem
- Need matrix builds across multiple OS/versions

**Strengths:**
- Tight GitHub integration (PRs, issues, releases)
- Massive marketplace of pre-built actions
- Generous free tier (2,000 mins/month for private repos)
- YAML-based, version controlled
- Excellent documentation
- Matrix builds for multi-platform testing
- Self-hosted runners easy to setup
- Federated identity (no secrets needed!)

**Weaknesses:**
- Less enterprise features than Azure DevOps
- No built-in artifact versioning (use packages)
- Limited pipeline visualization
- No release pipelines (use environments instead)

**Pricing:** Free for public repos, $0.008/minute for private (after free tier)

**Best for:** GitHub-based projects, open source, modern cloud-native apps
</platform>

<platform name="Azure DevOps Pipelines">
**Current Status:** Enterprise-grade CI/CD platform (2024-2025)

**When to use:**
- Enterprise requirements (audit, compliance)
- Need classic release pipelines
- Using Azure Boards/Repos/Artifacts
- Want built-in test management
- Multi-stage approvals needed
- Existing Azure DevOps investment

**Strengths:**
- Enterprise features (audit logs, retention policies)
- Classic and YAML pipelines
- Release pipelines with stages and gates
- Built-in artifact management
- Test plan integration
- Service connections managed centrally
- Better pipeline visualization
- Hosted on-premises option (Azure DevOps Server)

**Weaknesses:**
- More complex to learn
- Heavier UI (more enterprise-focused)
- Less marketplace momentum vs GitHub
- Requires separate Azure DevOps organization

**Pricing:** Free up to 5 users, $6/user/month after (includes boards, repos, etc.)

**Best for:** Enterprises, complex release orchestration, Azure-heavy shops
</platform>
</platform_comparison>

<decision_tree>
## Choosing the Right Platform

**Code on GitHub?**
→ Use **GitHub Actions** (native integration)

**Code on Azure Repos?**
→ Use **Azure DevOps Pipelines**

**Need complex multi-stage releases with manual gates?**
→ Use **Azure DevOps** (better release pipeline features)

**Want simplicity and modern approach?**
→ Use **GitHub Actions**

**Enterprise compliance and audit requirements?**
→ Use **Azure DevOps** (more enterprise features)

**Open source project?**
→ Use **GitHub Actions** (free unlimited minutes)

**Can use either?**
→ **GitHub Actions** for new projects (simpler, better ecosystem)
→ **Azure DevOps** if already invested in Azure DevOps ecosystem
</decision_tree>

<github_actions_guide>
## GitHub Actions for Azure Deployment

### Authentication: Federated Credentials (Recommended)

**No secrets needed!** Use OpenID Connect (OIDC) for secure, secret-free authentication.

**Setup:**

```bash
# 1. Create Azure AD Application
az ad app create --display-name "github-actions-myapp"

APP_ID=$(az ad app list --display-name "github-actions-myapp" --query "[0].appId" -o tsv)

# 2. Create Service Principal
az ad sp create --id $APP_ID

# 3. Assign permissions
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>

# 4. Configure federated credential for GitHub
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-federated",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<org>/<repo>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 5. Get values for GitHub secrets
TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "Add these as GitHub repository secrets:"
echo "AZURE_CLIENT_ID: $APP_ID"
echo "AZURE_TENANT_ID: $TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
```

**Add to GitHub:**
Settings → Secrets and variables → Actions → New repository secret

### Complete GitHub Actions Workflow

`.github/workflows/deploy-azure.yml`:

```yaml
name: Deploy to Azure

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger

env:
  AZURE_WEBAPP_NAME: myapp-prod-eastus-app-001
  RESOURCE_GROUP: myapp-prod-eastus-rg
  NODE_VERSION: '20'

# Required for federated auth
permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Run security audit
        run: npm audit --audit-level=high

      - name: Build application
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-build
          path: dist/
          retention-days: 7

  deploy-infrastructure:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Azure Login (Federated)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Validate Bicep
        run: az bicep build --file infrastructure/main.bicep

      - name: What-If Analysis
        run: |
          az deployment group what-if \
            --name "deploy-$(date +%Y%m%d-%H%M%S)" \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/prod.parameters.json

      - name: Deploy Infrastructure
        run: |
          az deployment group create \
            --name "deploy-$(date +%Y%m%d-%H%M%S)" \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --template-file infrastructure/main.bicep \
            --parameters infrastructure/prod.parameters.json

  deploy-app:
    runs-on: ubuntu-latest
    needs: [build, deploy-infrastructure]
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: app-build
          path: dist/

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to staging slot
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          slot-name: staging
          package: dist/

      - name: Test staging slot
        run: |
          echo "Waiting for app to start..."
          sleep 30

          # Health check
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}-staging.azurewebsites.net/health || exit 1

          # Smoke tests
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}-staging.azurewebsites.net/ || exit 1

      - name: Swap to production
        run: |
          az webapp deployment slot swap \
            --name ${{ env.AZURE_WEBAPP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --slot staging \
            --target-slot production

      - name: Verify production
        run: |
          sleep 30
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}.azurewebsites.net/health || exit 1

      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, rolling back..."
          az webapp deployment slot swap \
            --name ${{ env.AZURE_WEBAPP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --slot production \
            --target-slot staging

  notify:
    runs-on: ubuntu-latest
    needs: [deploy-app]
    if: always()

    steps:
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ needs.deploy-app.result }}: ${{ github.repository }} to production",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment ${{ needs.deploy-app.result }}*\n*Repository:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Commit:* ${{ github.sha }}\n*Author:* ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Matrix Builds

Test across multiple Node versions:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - run: npm ci
      - run: npm test
```

### Reusable Workflows

`.github/workflows/deploy-template.yml`:

```yaml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      app-name:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true
      AZURE_TENANT_ID:
        required: true
      AZURE_SUBSCRIPTION_ID:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy
        run: |
          az webapp deployment source config-zip \
            --name ${{ inputs.app-name }} \
            --resource-group ${{ inputs.environment }}-rg \
            --src app.zip
```

**Use the template:**

```yaml
name: Deploy to Environments

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: dev
      app-name: myapp-dev-app-001
    secrets: inherit

  deploy-prod:
    uses: ./.github/workflows/deploy-template.yml
    needs: deploy-dev
    with:
      environment: prod
      app-name: myapp-prod-app-001
    secrets: inherit
```

### Environment Protection

1. Settings → Environments → New environment
2. Name: `production`
3. Protection rules:
   - **Required reviewers:** Select 1+ people
   - **Wait timer:** 5 minutes (optional cooling-off period)
   - **Deployment branches:** Only `main`
   - **Environment secrets:** Add environment-specific secrets

### Secrets Hierarchy

- **Repository secrets:** Available to all workflows
- **Environment secrets:** Only available when deploying to that environment
- **Organization secrets:** Shared across repos in org

### Composite Actions

Create reusable action: `.github/actions/azure-deploy/action.yml`:

```yaml
name: 'Azure Deploy'
description: 'Deploy to Azure Web App'
inputs:
  app-name:
    description: 'Azure App Service name'
    required: true
  package:
    description: 'Path to deployment package'
    required: true
    default: 'dist/'

runs:
  using: 'composite'
  steps:
    - name: Deploy to Azure
      uses: azure/webapps-deploy@v3
      with:
        app-name: ${{ inputs.app-name }}
        package: ${{ inputs.package }}

    - name: Verify deployment
      shell: bash
      run: |
        sleep 30
        curl -f https://${{ inputs.app-name }}.azurewebsites.net/health || exit 1
```

**Use it:**

```yaml
- name: Deploy to production
  uses: ./.github/actions/azure-deploy
  with:
    app-name: myapp-prod-app-001
    package: dist/
```

### Caching

Speed up builds with caching:

```yaml
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Cache Bicep modules
  uses: actions/cache@v4
  with:
    path: ~/.bicep
    key: bicep-${{ hashFiles('infrastructure/**/*.bicep') }}
```
</github_actions_guide>

<azure_devops_guide>
## Azure DevOps Pipelines

### YAML Pipeline Structure

`azure-pipelines.yml`:

```yaml
# Trigger on main branch
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - 'src/**'
      - 'infrastructure/**'

# PR validation
pr:
  branches:
    include:
      - main

# Pipeline variables
variables:
  - group: prod-variables  # Variable group from Library
  - name: azureSubscription
    value: 'azure-prod-connection'
  - name: resourceGroup
    value: 'myapp-prod-eastus-rg'
  - name: webAppName
    value: 'myapp-prod-eastus-app-001'

# Build on latest Ubuntu
pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    displayName: 'Build Application'
    jobs:
      - job: BuildJob
        displayName: 'Build and Test'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run lint
            displayName: 'Run linter'

          - script: npm test
            displayName: 'Run tests'

          - task: PublishTestResults@2
            condition: succeededOrFailed()
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/test-results.xml'
            displayName: 'Publish test results'

          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '**/coverage/cobertura-coverage.xml'
            displayName: 'Publish coverage'

          - script: npm run build
            displayName: 'Build application'

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: 'dist'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'
            displayName: 'Archive build'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'app-build'
            displayName: 'Publish artifact'

  - stage: DeployInfrastructure
    displayName: 'Deploy Infrastructure'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployInfra
        displayName: 'Deploy Bicep Templates'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az bicep build --file infrastructure/main.bicep
                  displayName: 'Validate Bicep'

                - task: AzureResourceManagerTemplateDeployment@3
                  inputs:
                    deploymentScope: 'Resource Group'
                    azureResourceManagerConnection: '$(azureSubscription)'
                    action: 'Create Or Update Resource Group'
                    resourceGroupName: '$(resourceGroup)'
                    location: 'East US'
                    templateLocation: 'Linked artifact'
                    csmFile: 'infrastructure/main.bicep'
                    csmParametersFile: 'infrastructure/prod.parameters.json'
                    deploymentMode: 'Incremental'
                    deploymentOutputs: 'infraOutputs'
                  displayName: 'Deploy Bicep'

  - stage: DeployApplication
    displayName: 'Deploy Application'
    dependsOn: DeployInfrastructure
    jobs:
      - deployment: DeployApp
        displayName: 'Deploy to Azure'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: app-build

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    appType: 'webAppLinux'
                    appName: '$(webAppName)'
                    deployToSlotOrASE: true
                    resourceGroupName: '$(resourceGroup)'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/app-build/app.zip'
                  displayName: 'Deploy to staging slot'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Test staging slot
                      sleep 30
                      curl -f https://$(webAppName)-staging.azurewebsites.net/health || exit 1
                  displayName: 'Test staging slot'

                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    action: 'Swap Slots'
                    webAppName: '$(webAppName)'
                    resourceGroupName: '$(resourceGroup)'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
                  displayName: 'Swap to production'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      sleep 30
                      curl -f https://$(webAppName).azurewebsites.net/health || exit 1
                  displayName: 'Verify production'
```

### Service Connections

**Setup (using Workload Identity Federation - no secrets!):**

1. Project Settings → Service connections
2. New service connection → Azure Resource Manager
3. Authentication method: **Workload Identity federation (automatic)**
4. Scope: Subscription or Resource Group
5. Name: `azure-prod-connection`
6. Save

Azure DevOps automatically:
- Creates Azure AD application
- Configures federated credential
- Assigns permissions
- No secrets stored!

### Variable Groups

**Store shared variables:**

Pipelines → Library → Variable Groups → New

```
Group: prod-variables

Variables:
- resourceGroup: myapp-prod-eastus-rg
- location: eastus
- environment: prod

Secrets (lock icon):
- instrumentationKey: <from Key Vault>
```

**Link to Key Vault:**

1. Link secrets from Azure Key Vault
2. Select subscription and Key Vault
3. Authorize pipeline
4. Select secrets to sync

### Environments

**Setup approvals:**

Pipelines → Environments → production → Approvals and checks

Add:
- **Approvals:** Require 1+ people to approve
- **Branch control:** Only `main` can deploy
- **Business hours:** Only deploy 9 AM - 5 PM
- **Invoke Azure Function:** Custom validation logic

### Templates

Reusable pipeline template: `templates/build-template.yml`:

```yaml
parameters:
  - name: nodeVersion
    type: string
    default: '20.x'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: ${{ parameters.nodeVersion }}

  - script: npm ci
  - script: npm test
  - script: npm run build
```

**Use template:**

```yaml
jobs:
  - job: Build
    steps:
      - template: templates/build-template.yml
        parameters:
          nodeVersion: '20.x'
```
</azure_devops_guide>

<best_practices>
## CI/CD Best Practices (2024-2025)

### 1. Security

**Use federated credentials (no secrets):**
- GitHub Actions: OIDC with Azure
- Azure DevOps: Workload Identity Federation

**Scan for vulnerabilities:**
```yaml
- name: Security scan
  run: |
    npm audit --audit-level=high
    # Fail build if high/critical vulnerabilities
```

**Never commit secrets:**
- Use GitHub/Azure DevOps secrets
- Reference Key Vault for runtime secrets
- Scan for secrets in code (use GitGuardian, TruffleHog)

### 2. Testing

**Run tests in pipeline:**
```yaml
- name: Unit tests
  run: npm test

- name: Integration tests
  run: npm run test:integration

- name: E2E tests
  run: npm run test:e2e
```

**Code coverage gates:**
```yaml
- name: Check coverage
  run: |
    npm run test:coverage
    COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "Coverage $COVERAGE% below 80%"
      exit 1
    fi
```

### 3. Quality Gates

**Pre-deployment validation:**
```yaml
- name: Lint code
  run: npm run lint

- name: Type check
  run: npm run type-check

- name: Security audit
  run: npm audit --audit-level=moderate

- name: Bundle size check
  run: |
    SIZE=$(du -sh dist | cut -f1)
    echo "Bundle size: $SIZE"
    # Add size limit check
```

### 4. Deployment Strategies

**Blue-Green (using slots):**
1. Deploy to staging slot
2. Test staging
3. Swap to production
4. Rollback if needed (swap back)

**Canary:**
```yaml
- name: Deploy canary (10%)
  run: |
    az webapp traffic-routing set \
      --name myapp-prod-app-001 \
      --resource-group myapp-prod-rg \
      --distribution staging=10

- name: Monitor metrics
  run: # Check error rates, latency

- name: Promote to 100%
  run: |
    az webapp deployment slot swap \
      --name myapp-prod-app-001 \
      --resource-group myapp-prod-rg \
      --slot staging
```

### 5. Branch Protection

**GitHub:**
Settings → Branches → Add rule for `main`:
- Require pull request reviews (1+)
- Require status checks (build, test, lint)
- Require branches up to date
- Include administrators

**Azure DevOps:**
Repos → Branches → main → Branch policies:
- Require reviewers (1+)
- Check for linked work items
- Build validation (required)
- Comment resolution required

### 6. Artifact Management

**Version artifacts:**
```yaml
- name: Version artifact
  run: |
    VERSION=$(cat package.json | jq -r '.version')
    BUILD_NUMBER=${{ github.run_number }}
    echo "VERSION=$VERSION-$BUILD_NUMBER" >> $GITHUB_ENV

- name: Tag artifact
  run: |
    mv app.zip app-${{ env.VERSION }}.zip
```

**Retention:**
- Keep artifacts 30 days for dev
- Keep 90 days for prod
- Tag releases for long-term retention

### 7. Notifications

**GitHub Actions (Slack):**
```yaml
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
    payload: '{"text": "Build failed!"}'
```

**Azure DevOps:**
Project Settings → Notifications → New subscription

### 8. Monitoring Deployments

**Track deployments in Application Insights:**
```yaml
- name: Create deployment annotation
  run: |
    az rest --method put \
      --uri "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/microsoft.insights/components/$APP_INSIGHTS/Annotations?api-version=2015-05-01" \
      --body '{
        "AnnotationName": "Deployment",
        "Category": "Deployment",
        "EventTime": "'$(date -u +%Y-%m-%dT%H:%M:%S)'",
        "Properties": {
          "ReleaseName": "${{ github.sha }}",
          "BuildNumber": "${{ github.run_number }}"
        }
      }'
```

### 9. Infrastructure as Code in Pipeline

**Validate before deploy:**
```yaml
- name: Validate IaC
  run: |
    az bicep build --file infrastructure/main.bicep
    az deployment group what-if \
      --resource-group $RESOURCE_GROUP \
      --template-file infrastructure/main.bicep
```

**Separate infrastructure and app deployments:**
- Infrastructure changes: Require approval
- App deployments: Automated after tests pass

### 10. Rollback Strategy

**Automatic rollback on failure:**
```yaml
- name: Deploy
  id: deploy
  run: # deployment command

- name: Verify
  id: verify
  run: curl -f https://app.com/health

- name: Rollback on failure
  if: failure() && steps.deploy.outcome == 'success'
  run: |
    az webapp deployment slot swap \
      --slot production \
      --target-slot staging
```

**Keep previous version accessible:**
- Keep staging slot with previous version
- Tag container images with version
- Maintain deployment history
</best_practices>

<anti_patterns>
**Avoid:**

<anti_pattern name="No Tests in Pipeline">
**Problem:** Deploying without running automated tests

**Why bad:** Bugs reach production, quality deteriorates

**Instead:** Run unit, integration, E2E tests in pipeline. Block deployment on test failures.
</anti_pattern>

<anti_pattern name="Secrets in Code">
**Problem:** Hardcoded API keys, passwords in repository

**Why bad:** Security breach, secrets exposed in Git history

**Instead:** Use GitHub/Azure DevOps secrets, reference Key Vault at runtime
</anti_pattern>

<anti_pattern name="Direct Production Deploy">
**Problem:** Deploying straight to production without staging

**Why bad:** No testing in production-like environment, no rollback strategy

**Instead:** Use deployment slots, deploy to staging first, swap to production
</anti_pattern>

<anti_pattern name="Manual Approvals for Everything">
**Problem:** Requiring manual approval for every deployment

**Why bad:** Slows down delivery, doesn't add value for low-risk changes

**Instead:** Automate dev/staging, require approval only for production
</anti_pattern>

<anti_pattern name="No Deployment Verification">
**Problem:** Deploy and assume it worked

**Why bad:** Silent failures, degraded service unnoticed

**Instead:** Test health endpoints, check metrics, verify functionality post-deploy
</anti_pattern>
</anti_patterns>
