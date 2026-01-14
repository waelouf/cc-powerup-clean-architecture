# Workflow: Setup CI/CD Pipeline

<required_reading>
**Read these reference files NOW before setting up CI/CD:**
1. references/cicd-pipelines.md
2. references/deployment-strategies.md
3. references/security-identity.md
</required_reading>

<process>
## Step 1: Choose CI/CD Platform

Ask user preference:
- **Azure DevOps**: Azure-native, tight integration, enterprise features
- **GitHub Actions**: Code-to-cloud, simpler, great for GitHub repos

Both are excellent. Choice depends on where code is hosted and team preference.

## Step 2: Setup Service Principal or Managed Identity

Azure deployments from CI/CD need authentication.

### Option A: Federated Credentials (Recommended - No secrets!)

**For GitHub Actions:**
```bash
# Create Azure AD Application
az ad app create --display-name "github-actions-deploy"

# Get App ID
APP_ID=$(az ad app list --display-name "github-actions-deploy" --query "[0].appId" -o tsv)

# Create Service Principal
az ad sp create --id $APP_ID

# Get SP Object ID
SP_OBJECT_ID=$(az ad sp show --id $APP_ID --query id -o tsv)

# Assign Contributor role to resource group
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>

# Create federated credential for GitHub
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-federated",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<org>/<repo>:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Get Tenant and Subscription IDs
TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "AZURE_CLIENT_ID: $APP_ID"
echo "AZURE_TENANT_ID: $TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
```

Add these as GitHub repository secrets (Settings → Secrets → Actions):
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

**For Azure DevOps:**
Create Service Connection in Azure DevOps:
1. Project Settings → Service connections → New service connection
2. Choose "Azure Resource Manager"
3. Select "Workload Identity federation (automatic)"
4. Name: `azure-prod-connection`

## Step 3: Create Pipeline Definition

### For GitHub Actions:

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Azure

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

env:
  AZURE_WEBAPP_NAME: myapp-prod-eastus-app-001
  RESOURCE_GROUP: myapp-prod-eastus-rg

permissions:
  id-token: write  # Required for federated auth
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-build
          path: dist/

  deploy-infrastructure:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (Federated)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Bicep
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

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          package: dist/

      - name: Verify deployment
        run: |
          echo "Waiting for app to start..."
          sleep 30
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}.azurewebsites.net/health || exit 1
```

### For Azure DevOps:

Create `azure-pipelines.yml`:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscription: 'azure-prod-connection'
  resourceGroup: 'myapp-prod-eastus-rg'
  webAppName: 'myapp-prod-eastus-app-001'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm test
            displayName: 'Run tests'

          - script: npm run build
            displayName: 'Build application'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'dist'
              ArtifactName: 'app-build'
            displayName: 'Publish build artifact'

  - stage: DeployInfrastructure
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployInfra
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: AzureResourceManagerTemplateDeployment@3
                  inputs:
                    deploymentScope: 'Resource Group'
                    azureResourceManagerConnection: '$(azureSubscription)'
                    subscriptionId: '$(subscriptionId)'
                    action: 'Create Or Update Resource Group'
                    resourceGroupName: '$(resourceGroup)'
                    location: 'East US'
                    templateLocation: 'Linked artifact'
                    csmFile: 'infrastructure/main.bicep'
                    csmParametersFile: 'infrastructure/prod.parameters.json'
                    deploymentMode: 'Incremental'
                  displayName: 'Deploy Bicep template'

  - stage: DeployApplication
    dependsOn: DeployInfrastructure
    jobs:
      - deployment: DeployApp
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
                    package: '$(Pipeline.Workspace)/app-build'
                  displayName: 'Deploy to Azure Web App'

                - script: |
                    sleep 30
                    curl -f https://$(webAppName).azurewebsites.net/health || exit 1
                  displayName: 'Verify deployment'
```

## Step 4: Add Environment Protection

### GitHub Actions:
1. Go to Settings → Environments → New environment
2. Name: `production`
3. Add protection rules:
   - Required reviewers (at least 1)
   - Wait timer: 5 minutes (optional)
   - Deployment branches: Only `main`

### Azure DevOps:
1. Pipelines → Environments → production → Approvals and checks
2. Add "Approvals" → Select approvers
3. Add "Branch control" → Allow only `main`

## Step 5: Implement Deployment Strategies

### Blue-Green Deployment (App Service Slots):

Update GitHub Actions workflow:

```yaml
  deploy-app:
    runs-on: ubuntu-latest
    needs: [build, deploy-infrastructure]
    environment: production
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: app-build

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # Deploy to staging slot
      - name: Deploy to staging slot
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          slot-name: staging
          package: dist/

      # Run smoke tests on staging
      - name: Test staging slot
        run: |
          echo "Testing staging slot..."
          sleep 30
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}-staging.azurewebsites.net/health || exit 1

      # Swap staging to production
      - name: Swap slots
        run: |
          az webapp deployment slot swap \
            --name ${{ env.AZURE_WEBAPP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --slot staging \
            --target-slot production

      # Verify production
      - name: Verify production
        run: |
          sleep 30
          curl -f https://${{ env.AZURE_WEBAPP_NAME }}.azurewebsites.net/health || exit 1
```

## Step 6: Add Quality Gates

Add to pipeline:

```yaml
      - name: Run linter
        run: npm run lint

      - name: Run security scan
        run: npm audit --audit-level=high

      - name: Check code coverage
        run: |
          npm run test:coverage
          # Fail if coverage below 80%
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80%"
            exit 1
          fi
```

## Step 7: Setup Branch Policies

### GitHub:
Settings → Branches → Add branch protection rule for `main`:
- Require pull request reviews (1 approver)
- Require status checks to pass (build, tests)
- Require branches to be up to date
- Include administrators

### Azure DevOps:
Repos → Branches → main → Branch policies:
- Require minimum number of reviewers: 1
- Check for linked work items
- Check for comment resolution
- Build validation (required)

## Step 8: Configure Notifications

### GitHub Actions:
Add Slack notification step:

```yaml
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}: ${{ github.repository }} to production"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Azure DevOps:
Project Settings → Notifications → New subscription:
- Event: Release deployment completed
- Deliver to: Email/Slack/Teams

## Step 9: Setup Rollback Procedure

Document rollback in `RUNBOOK.md`:

```markdown
## Rollback Procedure

### For Slot-based Deployment:
\`\`\`bash
# Swap back to previous version
az webapp deployment slot swap \
  --name myapp-prod-eastus-app-001 \
  --resource-group myapp-prod-eastus-rg \
  --slot production \
  --target-slot staging
\`\`\`

### For Infrastructure:
\`\`\`bash
# Redeploy previous version
az deployment group create \
  --resource-group myapp-prod-eastus-rg \
  --template-file infrastructure/main.bicep \
  --parameters infrastructure/prod.parameters.json

# Or rollback specific deployment
az deployment group rollback \
  --name <deployment-name> \
  --resource-group myapp-prod-eastus-rg
\`\`\`
```

## Step 10: Test Pipeline

1. Create a test branch
2. Make a small change
3. Push and verify pipeline runs
4. Create PR and verify gates work
5. Merge to main and verify deployment
6. Check application is accessible
7. Verify monitoring shows deployment event
</process>

<anti_patterns>
**Avoid:**

- **Secrets in code**: Never commit client secrets. Use federated credentials or managed identity.

- **No tests in pipeline**: Always run tests before deployment. No tests = no CI/CD.

- **Deploying from developer machine**: All deployments must go through pipeline for auditability.

- **No approval gates**: Production deployments should require approval.

- **Directly deploying to production**: Use staging slot first, then swap.

- **No rollback plan**: Always have documented rollback procedure.

- **Weak branch protection**: Protect main branch with required reviews and status checks.

- **No quality gates**: Implement linting, security scanning, coverage checks.

- **Manual infrastructure changes**: Infrastructure changes must go through IaC pipeline.

- **Ignoring failed deployments**: Set up alerts for pipeline failures.

- **No deployment verification**: Always verify app is healthy after deployment.
</anti_patterns>

<success_criteria>
A well-configured CI/CD pipeline has:

- Federated credentials configured (no secrets)
- Pipeline defined in YAML (version controlled)
- Build job runs tests and linting
- Quality gates implemented (tests, coverage, security scans)
- Infrastructure deployment automated
- Blue-green deployment with staging slot
- Approval required for production deployment
- Branch protection on main branch
- Deployment verification step
- Rollback procedure documented
- Notifications configured
- Pipeline runs successfully on PR and main
- Application deploys and is accessible
- No manual steps required
- All changes auditable in Git history
</success_criteria>
