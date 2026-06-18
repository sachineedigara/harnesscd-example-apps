# Azure Container Apps Deployment Pipeline

## Pipeline Overview

This pipeline demonstrates deploying containerized applications to Azure Container Apps using Harness CD.

**What is Azure Container Apps?**

Azure Container Apps is a fully managed serverless container service that enables you to run microservices and containerized applications without managing infrastructure. It automatically handles container orchestration, scaling, and traffic management.

**Key Features:**
- Deploy an application to Azure Container Apps
- Manage traffic shifting between revisions for blue-green or canary deployments
- Built-in rollback mechanisms for deployment failures
- Auto-scaling based on HTTP traffic, CPU, or custom metrics

---

## Key Files and Their Purpose

- [**`CONNECTOR_SETUP.md`**](./CONNECTOR_SETUP.md): **START HERE** - Complete guide to setting up Azure connector with automated scripts.
- [**`harness-cd-pipeline/pipeline.yaml`**](./harness-cd-pipeline/pipeline.yaml): Harness pipeline YAML for Azure Container Apps deployment.
- [**`harness-cd-pipeline/service.yaml`**](./harness-cd-pipeline/service.yaml): Harness service definition with artifact source configuration.
- [**`harness-cd-pipeline/environment.yaml`**](./harness-cd-pipeline/environment.yaml): YAML defining the deployment environment.
- [**`harness-cd-pipeline/infrastructureDefinition.yaml`**](./harness-cd-pipeline/infrastructureDefinition.yaml): Infrastructure configuration linking to Azure resources.
- [**`harness-cd-pipeline/manifest.yaml`**](./harness-cd-pipeline/manifest.yaml): Azure Container Apps configuration manifest.

---

## Prerequisites

Before running this pipeline, ensure you have:

1. **Azure Subscription** with Azure Container Apps enabled
2. **Azure Connector** configured in Harness with proper permissions (Contributor role)
3. **Azure Resources**:
   - Resource Group
   - Container Apps Managed Environment
4. **Kubernetes Delegate** (for running containerized step groups)
5. **Container Image** available in a registry (Docker Hub, ACR, etc.)

---

## Setup Guide

### Part 1: Azure Setup (Creating Resources and Credentials)

#### Option A: Automated Setup with Azure CLI (Recommended)

If you have Azure CLI installed, you can create all resources and credentials automatically.

**Step 1: Login to Azure**
```bash
az login
```

**Step 2: Create Resources and Service Principal**

Save this script as `setup-azure-aca.sh`:

```bash
#!/bin/bash
# Azure Container Apps Setup Script

set -e

# Configuration
RESOURCE_GROUP="aca-harness-rg"
LOCATION="eastus"
ENV_NAME="aca-harness-env"
SP_NAME="harness-aca-connector"

# Get subscription info
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

echo "Using subscription: $SUBSCRIPTION_ID"

# Create resource group
echo "Creating resource group..."
az group create \
    --name "$RESOURCE_GROUP" \
    --location "$LOCATION"

# Install Container Apps extension
echo "Installing Container Apps extension..."
az extension add --name containerapp --upgrade

# Register providers
echo "Registering Azure providers..."
az provider register --namespace Microsoft.App --wait
az provider register --namespace Microsoft.OperationalInsights --wait

# Create Container Apps environment
echo "Creating Container Apps environment..."
az containerapp env create \
    --name "$ENV_NAME" \
    --resource-group "$RESOURCE_GROUP" \
    --location "$LOCATION"

# Create service principal
echo "Creating service principal..."
SP_OUTPUT=$(az ad sp create-for-rbac \
    --name "$SP_NAME" \
    --role Contributor \
    --scopes /subscriptions/$SUBSCRIPTION_ID \
    --output json)

APP_ID=$(echo $SP_OUTPUT | jq -r '.appId')
APP_SECRET=$(echo $SP_OUTPUT | jq -r '.password')

# Display credentials
echo ""
echo "=========================================="
echo "✅ SETUP COMPLETE!"
echo "=========================================="
echo ""
echo "Azure Resources Created:"
echo "  Resource Group:      $RESOURCE_GROUP"
echo "  Location:            $LOCATION"
echo "  Container App Env:   $ENV_NAME"
echo "  Subscription ID:     $SUBSCRIPTION_ID"
echo ""
echo "Harness Connector Credentials:"
echo "  Application (Client) ID: $APP_ID"
echo "  Directory (Tenant) ID:   $TENANT_ID"
echo "  Client Secret:           $APP_SECRET"
echo ""
echo "⚠️  SAVE THESE VALUES - you'll need them for Harness!"
```

**Step 3: Run the Script**
```bash
chmod +x setup-azure-aca.sh
./setup-azure-aca.sh
```

**Step 4: Save the Output**

The script will output three important values:
- **Application (Client) ID**: e.g., `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee`
- **Directory (Tenant) ID**: e.g., `11111111-2222-3333-4444-555555555555`
- **Client Secret**: e.g., `ABC123~XXXXXXXXXXXXXXXXXXXXXXXXX`

⚠️ **Important**: Save these values immediately! The client secret cannot be retrieved again.

---

#### Option B: Manual Setup via Azure Portal

If you prefer using the Azure Portal:

**Step 1: Create App Registration (Service Principal)**

1. Go to **Azure Portal** → **Microsoft Entra ID** (formerly Azure AD)
2. Click **App registrations** → **+ New registration**
3. Fill in:
   - **Name**: `harness-aca-connector`
   - **Supported account types**: Accounts in this organizational directory only
   - Click **Register**
4. **Copy the Application (client) ID and Directory (tenant) ID** from the Overview page

**Step 2: Create Client Secret**

1. In the app registration, go to **Certificates & secrets**
2. Click **+ New client secret**
3. Fill in:
   - **Description**: `harness-connector-secret`
   - **Expires**: 24 months (recommended)
4. Click **Add**
5. **Copy the Value immediately** (you can only see it once!)

**Step 3: Assign Permissions**

1. Go to **Subscriptions** → Select your subscription
2. Click **Access control (IAM)**
3. Click **+ Add** → **Add role assignment**
4. Select **Contributor** role
5. Click **Next**
6. Click **+ Select members**
7. Search for `harness-aca-connector` and select it
8. Click **Review + assign**

**Step 4: Create Azure Resources**

1. **Create Resource Group:**
   - Search for "Resource groups" → **+ Create**
   - Name: `aca-harness-rg`
   - Region: Choose your preferred region
   - Click **Review + create** → **Create**

2. **Create Container Apps Environment:**
   - Search for "Container Apps" → **+ Create**
   - Or use Azure CLI:
     ```bash
     az containerapp env create \
         --name aca-harness-env \
         --resource-group aca-harness-rg \
         --location eastus
     ```

---

### Part 2: Harness Connector Setup

Now create the Azure connector in Harness using the credentials from Part 1.

**Step 1: Navigate to Connectors**

Go to: **Harness UI** → **Project Setup** → **Connectors** → **+ New Connector** → **Cloud Providers** → **Azure**

**Step 2: Configure the Connector**

**Overview Tab:**
- **Name**: `azure-container-apps-connector`
- **Description**: `Azure connector for Container Apps deployments`
- Click **Continue**

**Credentials Tab:**
- **Select**: `Specify credentials here`
- **Environment**: `Azure Global`
- **Application (Client) ID**: Paste the Application ID from Part 1
- **Tenant ID**: Paste the Tenant ID from Part 1
- **Authentication**: Select `Secret`

**Step 3: Create Secret**

- Click **Create or Select a Secret**
- Click **+ New Secret Text**
  - **Secret Name**: `azure-aca-client-secret`
  - **Secret Value**: Paste the Client Secret from Part 1
  - Click **Save**
- The secret should now be selected
- Click **Continue**

**Step 4: Select Connectivity Mode**

- Choose: **Connect through Harness Platform** (simpler)
  - OR **Connect through a Harness Delegate** (if you have a delegate in Azure)
- Click **Save and Continue**

**Step 5: Test Connection**

- Harness will test the connection
- If successful, click **Finish**

✅ **Azure connector is now ready!**

---

### Part 3: Update Infrastructure Definition

After creating the connector, update your infrastructure YAML:

```yaml
infrastructureDefinition:
  name: Azure Container Apps Infrastructure
  identifier: azure_container_apps_infrastructure
  spec:
    connectorRef: azure_container_apps_connector  # Your connector identifier
    subscriptionId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # Your subscription ID
    resourceGroup: aca-harness-rg  # Your resource group
    managedEnvironment: aca-harness-env  # Your environment name
```

---

### Troubleshooting Connector Setup

#### Issue: "AuthorizationFailed - does not have authorization"

**Cause**: Service principal doesn't have the Contributor role assigned.

**Solution**: 
```bash
# Assign Contributor role via Azure CLI
az role assignment create \
    --role Contributor \
    --assignee <YOUR_APP_ID> \
    --scope /subscriptions/<YOUR_SUBSCRIPTION_ID>
```

#### Issue: "No subscriptions found"

**Cause**: Service principal has no role assignments.

**Solution**: 
- Verify role assignment in Azure Portal: Subscriptions → Access control (IAM)
- Wait 5-10 minutes for Azure propagation
- Retry the connector test

#### Issue: "Invalid client secret"

**Cause**: Client secret is incorrect or expired.

**Solution**:
- Generate a new client secret in Azure Portal
- Update the Harness secret with the new value

#### Issue: Test connection fails with "No value present"

**Cause**: Secret wasn't created properly in Harness.

**Solution**:
- Go to Project Settings → Secrets
- Verify `azure-aca-client-secret` exists
- Re-create if needed (make sure no extra spaces in the secret value)
- Update connector to reference the correct secret

---

## Pipeline Stages

### Stage: Deploy Container App

**Objective**: Deploy a containerized application to Azure Container Apps with traffic management.

**Steps:**

1. **Download Manifests**
   - Fetches the container app configuration from Harness File Store

2. **Prepare Rollback Data**
   - Captures current revision state for potential rollback
   - Stores revision metadata for recovery

3. **Deploy**
   - Creates or updates the Azure Container App
   - Generates a new revision with the specified artifact
   - Optionally skips traffic shift for manual control

4. **Traffic Shift**
   - Routes traffic to the new revision
   - Supports weighted traffic distribution
   - Enables blue-green or canary deployment patterns

**Rollback Steps:**
- Automatically triggers on deployment failure
- Reverts traffic to the previous stable revision
- Maintains application availability during failures

---

## Pipeline Inputs

When running the pipeline, you'll need to provide:

**Artifact Configuration:**
- **Container Image**: Docker image path (e.g., `library/nginx`)
- **Image Tag**: Version to deploy (e.g., `alpine`)
- **Container Registry Connector**: Harness connector to pull the image

**Infrastructure:**
- **Azure Connector**: Authentication to Azure subscription
- **Subscription ID**: Azure subscription identifier
- **Resource Group**: Azure resource group name
- **Managed Environment**: Container Apps environment name

**Step Group Infrastructure:**
- **Kubernetes Connector**: For running containerized steps
- **Namespace**: Kubernetes namespace (typically `default`)

---

## Azure Container Apps Manifest Structure

The manifest file defines the container app configuration:

```yaml
name: my-container-app
properties:
  configuration:
    activeRevisionsMode: Single  # Or 'Multiple' for traffic splitting
    ingress:
      external: true              # Public internet access
      targetPort: 80              # Container port
      traffic:
        - latestRevision: true
          weight: 100
  template:
    containers:
      - name: my-app
        image: <+artifact.image>  # Harness expression for artifact
        resources:
          cpu: 0.25                # vCPU cores
          memory: 0.5Gi            # Memory allocation
```

**Key Configuration Options:**

- **activeRevisionsMode**: 
  - `Single`: Only one revision active (standard deployment)
  - `Multiple`: Multiple revisions active (for traffic splitting)

- **ingress.external**: 
  - `true`: Accessible from public internet
  - `false`: Internal only (within VNET)

- **resources**: Define CPU and memory allocation per container

---

## Traffic Management Strategies

### Standard Deployment (100% Traffic)
```yaml
revisionTrafficDetails:
  - revisionName: latest
    trafficValue: 100
```

### Blue-Green Deployment
```yaml
# Before traffic shift
- revisionName: blue-revision
  trafficValue: 100

# After validation
- revisionName: green-revision
  trafficValue: 100
```

### Canary Deployment
```yaml
# Initial canary
- revisionName: stable-revision
  trafficValue: 90
- revisionName: canary-revision
  trafficValue: 10

# Gradual rollout
- revisionName: stable-revision
  trafficValue: 50
- revisionName: canary-revision
  trafficValue: 50
```

---

## Example Deployment Flow

1. **Initial Deployment**
   - Creates container app with revision `my-app--rev1`
   - Routes 100% traffic to `rev1`

2. **Update Deployment**
   - Deploys new version as revision `my-app--rev2`
   - Shifts traffic: `rev1` (0%) → `rev2` (100%)
   - `rev1` remains inactive for potential rollback

3. **Rollback (if needed)**
   - Detects deployment failure
   - Shifts traffic back: `rev2` (0%) → `rev1` (100%)
   - Application remains available

---

## Cost Considerations

Azure Container Apps pricing (consumption-based):
- **vCPU**: ~$0.0432 per vCPU-hour
- **Memory**: ~$0.0072 per GB-hour
- **Free Tier**: 180,000 vCPU-seconds + 360,000 GB-seconds per month

**Example cost for this template:**
- 0.25 vCPU + 0.5 GB memory
- Running 24/7: ~$10/month
- Testing (< 200 hours/month): **$0** (within free tier)

---

## Troubleshooting

### Common Issues:

**1. Connector authentication fails**
- Verify Azure connector credentials are valid
- Ensure service principal has Contributor role
- Check subscription ID is correct

**2. Managed environment not found**
- Confirm environment exists in the specified resource group
- Verify environment name matches exactly (case-sensitive)
- Check Azure region matches

**3. Traffic shift fails with "Invalid revision name"**
- Use `latest` keyword instead of literal revision name
- Or reference deploy step output: `<+pipeline.stages.deploy.spec.execution.steps.deploy_step.output.revisionName>`

**4. Rollback step fails**
- Ensure rollback data was captured successfully
- Verify previous revision still exists
- Check delegate has connectivity to Azure

---

## Documentation

- [Azure Container Apps](https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/azure/azure-container-apps)
- [Azure Connector Setup](https://developer.harness.io/docs/platform/connectors/cloud-providers/add-a-microsoft-azure-connector)
- [CD Artifact Sources](https://developer.harness.io/docs/continuous-delivery/x-platform-cd-features/services/artifact-sources/)
- [Azure Container Apps Documentation](https://learn.microsoft.com/en-us/azure/container-apps/)

---

## Conclusion

This Azure Container Apps deployment pipeline streamlines the process of deploying containerized applications to Azure's serverless container platform. By leveraging Harness CD, it provides:

- ✅ Automated deployment with rollback protection
- ✅ Flexible traffic management for advanced deployment strategies
- ✅ Infrastructure as code for repeatable deployments
- ✅ Integration with existing CI/CD workflows

The provided YAML configurations enable quick setup of services, environments, infrastructure, and pipelines, making it easy to get started with Azure Container Apps deployments.

For detailed setup instructions, refer to the [Harness Azure Container Apps documentation](https://developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/azure/azure-container-apps).
