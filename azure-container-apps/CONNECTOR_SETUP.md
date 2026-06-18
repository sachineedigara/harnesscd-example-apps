# Azure Connector Setup Guide

This guide walks you through creating an Azure connector in Harness for Azure Container Apps deployments.

## Prerequisites

- Azure subscription (can use [Free Trial](https://azure.microsoft.com/free/) with $200 credit)
- Azure CLI installed (for automated setup) OR Azure Portal access (for manual setup)
- Harness account with project access

---

## Quick Start (Automated Setup)

### 1. Install Azure CLI (if not already installed)

**macOS:**
```bash
brew install azure-cli
```

**Other platforms:** https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

### 2. Login to Azure

```bash
az login
```

This opens a browser window for authentication.

### 3. Run the Setup Script

Save this as `setup-azure-connector.sh`:

```bash
#!/bin/bash
# Automated Azure Connector Setup for Harness

set -e

echo "Azure Container Apps Connector Setup"
echo "====================================="

# Configuration (customize these)
RESOURCE_GROUP="${RESOURCE_GROUP:-aca-harness-rg}"
LOCATION="${LOCATION:-eastus}"
ENV_NAME="${ENV_NAME:-aca-harness-env}"
SP_NAME="${SP_NAME:-harness-aca-connector}"

# Get current subscription
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
SUBSCRIPTION_NAME=$(az account show --query name -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

echo ""
echo "Current Subscription:"
echo "  Name: $SUBSCRIPTION_NAME"
echo "  ID:   $SUBSCRIPTION_ID"
echo ""

# Create resource group
echo "Step 1/5: Creating resource group..."
az group create \
    --name "$RESOURCE_GROUP" \
    --location "$LOCATION" \
    --output none

echo "✅ Resource group created: $RESOURCE_GROUP"

# Install Container Apps extension
echo ""
echo "Step 2/5: Setting up Container Apps..."
az extension add --name containerapp --upgrade --only-show-errors

# Register providers
az provider register --namespace Microsoft.App --wait
az provider register --namespace Microsoft.OperationalInsights --wait

echo "✅ Azure providers registered"

# Create Container Apps environment
echo ""
echo "Step 3/5: Creating Container Apps environment..."
az containerapp env create \
    --name "$ENV_NAME" \
    --resource-group "$RESOURCE_GROUP" \
    --location "$LOCATION" \
    --output none

echo "✅ Container Apps environment created: $ENV_NAME"

# Create service principal
echo ""
echo "Step 4/5: Creating service principal for Harness..."

# Check if SP already exists
EXISTING_SP=$(az ad sp list --display-name "$SP_NAME" --query "[0].appId" -o tsv 2>/dev/null || echo "")

if [ ! -z "$EXISTING_SP" ]; then
    echo "⚠️  Service principal '$SP_NAME' already exists"
    echo "   Deleting existing SP and creating new one..."
    az ad sp delete --id "$EXISTING_SP"
    sleep 5
fi

SP_OUTPUT=$(az ad sp create-for-rbac \
    --name "$SP_NAME" \
    --role Contributor \
    --scopes /subscriptions/$SUBSCRIPTION_ID \
    --output json)

APP_ID=$(echo $SP_OUTPUT | jq -r '.appId')
APP_SECRET=$(echo $SP_OUTPUT | jq -r '.password')

echo "✅ Service principal created with Contributor role"

# Save credentials to file
CREDS_FILE="azure-connector-credentials.txt"
cat > "$CREDS_FILE" << EOF
Azure Connector Credentials for Harness
========================================

Azure Resources:
  Subscription ID:     $SUBSCRIPTION_ID
  Resource Group:      $RESOURCE_GROUP
  Location:            $LOCATION
  Environment Name:    $ENV_NAME

Harness Connector Credentials:
  Application (Client) ID: $APP_ID
  Directory (Tenant) ID:   $TENANT_ID
  Client Secret:           $APP_SECRET

Infrastructure YAML Values:
  connectorRef: <your-connector-name>
  subscriptionId: $SUBSCRIPTION_ID
  resourceGroup: $RESOURCE_GROUP
  managedEnvironment: $ENV_NAME

Next Steps:
1. Go to Harness → Connectors → New Connector → Azure
2. Use the credentials above to configure the connector
3. Update your infrastructure definition with the values above

⚠️  IMPORTANT: Save this file securely!
The client secret cannot be retrieved again.
EOF

# Display summary
echo ""
echo "=========================================="
echo "✅ SETUP COMPLETE!"
echo "=========================================="
echo ""
echo "📋 Credentials saved to: $CREDS_FILE"
echo ""
echo "Copy these values for Harness:"
echo ""
echo "┌────────────────────────────────────────────────────────────┐"
echo "│ Application (Client) ID:                                   │"
echo "│ $APP_ID"
echo "│                                                            │"
echo "│ Directory (Tenant) ID:                                     │"
echo "│ $TENANT_ID"
echo "│                                                            │"
echo "│ Client Secret:                                             │"
echo "│ $APP_SECRET"
echo "│                                                            │"
echo "│ Subscription ID:                                           │"
echo "│ $SUBSCRIPTION_ID"
echo "│                                                            │"
echo "│ Resource Group:                                            │"
echo "│ $RESOURCE_GROUP"
echo "│                                                            │"
echo "│ Managed Environment:                                       │"
echo "│ $ENV_NAME"
echo "└────────────────────────────────────────────────────────────┘"
echo ""
echo "⚠️  SAVE THESE VALUES - the client secret cannot be retrieved again!"
echo ""
echo "Next: Create Harness connector using these credentials"
echo ""
```

**Run it:**
```bash
chmod +x setup-azure-connector.sh
./setup-azure-connector.sh
```

The script will:
1. Create Azure resource group
2. Set up Container Apps environment
3. Create service principal with proper permissions
4. Display all credentials needed for Harness
5. Save credentials to `azure-connector-credentials.txt`

---

## Manual Setup (Azure Portal)

If you prefer not to use the script:

### Part 1: Create Service Principal

#### Step 1: Create App Registration

1. Open **Azure Portal**: https://portal.azure.com
2. Navigate to **Microsoft Entra ID** (formerly Azure Active Directory)
3. Click **App registrations** in the left menu
4. Click **+ New registration**
5. Fill in:
   - **Name**: `harness-aca-connector`
   - **Supported account types**: Accounts in this organizational directory only (Single tenant)
   - **Redirect URI**: Leave blank
6. Click **Register**

#### Step 2: Note Application and Tenant IDs

From the app registration **Overview** page, copy:
- **Application (client) ID**: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- **Directory (tenant) ID**: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

#### Step 3: Create Client Secret

1. In your app registration, click **Certificates & secrets** (left menu)
2. Click **Client secrets** tab
3. Click **+ New client secret**
4. Fill in:
   - **Description**: `harness-connector-secret`
   - **Expires**: 24 months (recommended)
5. Click **Add**
6. **IMMEDIATELY COPY THE VALUE** (shown once only!)
   - It looks like: `abc123~DEF456.ghi789...`

⚠️ **Critical**: You can never see this value again. If you lose it, you must create a new secret.

#### Step 4: Assign Permissions to Subscription

1. Go to **Subscriptions** (use search bar)
2. Click on your subscription
3. Click **Access control (IAM)** in the left menu
4. Click **+ Add** → **Add role assignment**
5. **Role tab:**
   - Search and select: **Contributor**
   - Click **Next**
6. **Members tab:**
   - **Assign access to**: User, group, or service principal
   - Click **+ Select members**
   - Search for: `harness-aca-connector`
   - Select it
   - Click **Select**
   - Click **Next**
7. **Review + assign tab:**
   - Click **Review + assign**

✅ Service principal now has Contributor access!

### Part 2: Create Azure Resources

#### Create Resource Group

1. Search for **Resource groups**
2. Click **+ Create**
3. Fill in:
   - **Subscription**: Your subscription
   - **Resource group**: `aca-harness-rg`
   - **Region**: East US (or your preference)
4. Click **Review + create** → **Create**

#### Create Container Apps Environment

**Option A: Via Portal**
1. Search for **Container Apps**
2. Click **Container Apps Environments** (left menu)
3. Click **+ Create**
4. Fill in:
   - **Subscription**: Your subscription
   - **Resource group**: `aca-harness-rg`
   - **Environment name**: `aca-harness-env`
   - **Region**: East US (same as resource group)
5. Click **Review + create** → **Create**

**Option B: Via CLI (faster)**
```bash
az containerapp env create \
    --name aca-harness-env \
    --resource-group aca-harness-rg \
    --location eastus
```

---

## Create Harness Connector

Now use the credentials to create the connector in Harness.

### Step 1: Navigate to Connectors

1. Open **Harness UI**: https://app.harness.io
2. Go to your project
3. Click **Project Setup** → **Connectors**
4. Click **+ New Connector**
5. Select **Cloud Providers** → **Azure**

### Step 2: Overview Tab

- **Name**: `azure-container-apps-connector`
- **Description**: `Azure connector for Container Apps deployments`
- Click **Continue**

### Step 3: Credentials Tab

1. **Select**: `Specify credentials here`
2. **Environment**: `Azure Global`
3. **Application (Client) ID**: Paste the Application ID from Azure
4. **Tenant ID**: Paste the Tenant ID from Azure
5. **Authentication**: Select `Secret`

### Step 4: Create Secret in Harness

1. Click **Create or Select a Secret**
2. Click **+ New Secret Text**
3. Fill in:
   - **Secret Name**: `azure-aca-client-secret`
   - **Secret Value**: Paste the Client Secret VALUE from Azure
   - ⚠️ Make sure there are no spaces before/after the value!
4. Click **Save**
5. The secret should be auto-selected
6. Click **Continue**

### Step 5: Connectivity Mode

**Option A: Connect through Harness Platform** (Recommended for getting started)
- Select this option
- No delegate required
- Click **Save and Continue**

**Option B: Connect through a Harness Delegate** (For production)
- Select this option
- Choose: "Use any available Delegate" OR "Only use Delegates with all of the following tags"
- Select/enter delegate tags
- Click **Save and Continue**

### Step 6: Test Connection

- Harness will test the Azure connection
- This verifies:
  - Credentials are correct
  - Service principal has proper permissions
  - Azure subscription is accessible

**If successful:**
- ✅ You'll see "Verification successful"
- Click **Finish**

**If failed:** See [Troubleshooting](#troubleshooting) below

---

## Verify Connector Setup

### Test with Azure CLI

Verify the service principal works:

```bash
# Get your service principal credentials
APP_ID="<your-app-id>"
CLIENT_SECRET="<your-client-secret>"
TENANT_ID="<your-tenant-id>"
SUBSCRIPTION_ID="<your-subscription-id>"

# Login as service principal
az login --service-principal \
    --username "$APP_ID" \
    --password "$CLIENT_SECRET" \
    --tenant "$TENANT_ID"

# Set subscription
az account set --subscription "$SUBSCRIPTION_ID"

# Test listing resources
az resource list --output table
```

If this works, your connector should work in Harness!

---

## Troubleshooting

### Issue: "AuthorizationFailed - does not have authorization"

**Symptoms:**
```
The client does not have authorization to perform action 
'Microsoft.Authorization/roleAssignments/write'
```

**Cause**: You don't have permissions to assign roles.

**Solution:**
- Ask an Azure administrator to assign the Contributor role to your service principal
- Or use the automated script which requires fewer permissions

### Issue: "No subscriptions found"

**Symptoms:**
```
az login succeeds but no subscriptions are visible
```

**Cause**: Service principal has no role assignments yet.

**Solution:**
```bash
# Verify role assignment
az role assignment list \
    --assignee <APP_ID> \
    --output table

# If empty, assign Contributor role
az role assignment create \
    --role Contributor \
    --assignee <APP_ID> \
    --scope /subscriptions/<SUBSCRIPTION_ID>

# Wait 2-3 minutes for Azure to propagate
```

### Issue: "Invalid client secret" or "No value present"

**Symptoms:**
- Harness test connection fails
- Error mentions authentication or credentials

**Causes:**
1. Client secret has spaces before/after the value
2. Wrong secret was copied (Secret ID instead of Value)
3. Secret expired

**Solution:**
1. **Re-create the secret in Azure:**
   - Go to App registration → Certificates & secrets
   - Delete old secret
   - Create new secret
   - Copy the VALUE (not the ID!)

2. **Update Harness secret:**
   - Go to Project Settings → Secrets
   - Find `azure-aca-client-secret`
   - Edit and paste new value
   - Make sure no spaces!

3. **Test connector again**

### Issue: "Container Apps environment not found"

**Symptoms:**
```
The managed environment 'aca-harness-env' was not found
```

**Causes:**
1. Environment doesn't exist
2. Wrong resource group
3. Wrong subscription
4. Typo in environment name

**Solution:**
```bash
# List environments
az containerapp env list \
    --resource-group aca-harness-rg \
    --output table

# Verify the exact name and use it in your infrastructure YAML
```

### Issue: Test connection times out

**Cause**: Network or Azure API issues

**Solution:**
- Wait a few minutes and retry
- Check Azure status: https://status.azure.com
- Verify your network allows access to Azure endpoints

---

## Cost Information

### Free Tier
Azure Container Apps includes:
- **180,000 vCPU-seconds** per month (free)
- **360,000 GB-seconds** per month (free)
- Equals ~200 hours of 0.25vCPU + 0.5GB app

### Beyond Free Tier
- **vCPU**: $0.000012 per vCPU-second (~$0.0432 per vCPU-hour)
- **Memory**: $0.000002 per GB-second (~$0.0072 per GB-hour)

**Example cost for 0.25 vCPU + 0.5 GB:**
- Testing (< 200 hours/month): **$0** (free tier)
- 24/7 production (720 hours/month): ~$10/month

### Cost Management
```bash
# Set spending alert
az consumption budget create \
    --subscription-id <SUBSCRIPTION_ID> \
    --budget-name aca-budget \
    --amount 10 \
    --time-grain Monthly
```

---

## Security Best Practices

1. **Use short-lived secrets**: Set client secret expiry to 6-12 months, not 24 months
2. **Rotate secrets regularly**: Before they expire
3. **Principle of least privilege**: Only grant Contributor at subscription level if needed
4. **Use separate service principals**: One per environment (dev, staging, prod)
5. **Audit access**: Regularly review who has access via Azure IAM

---

## Cleanup (When Done Testing)

### Delete Resources
```bash
# Delete resource group (deletes everything inside)
az group delete --name aca-harness-rg --yes --no-wait

# Delete service principal
az ad sp delete --id <APP_ID>
```

### Remove Harness Connector
1. Go to Project Setup → Connectors
2. Find your Azure connector
3. Click three dots → Delete
4. Confirm deletion

---

## Next Steps

After creating the connector:

1. **Update infrastructure definition** with your values
2. **Upload manifest** to Harness File Store
3. **Create service** using provided YAML
4. **Create environment** using provided YAML
5. **Create pipeline** and run your first deployment!

See [Description.md](./Description.md) for complete deployment guide.

---

## Additional Resources

- [Harness Azure Connector Docs](https://developer.harness.io/docs/platform/connectors/cloud-providers/add-a-microsoft-azure-connector)
- [Azure Service Principal Docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)
- [Azure Container Apps Docs](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Azure RBAC Docs](https://docs.microsoft.com/en-us/azure/role-based-access-control/)

---

## Support

If you encounter issues:
1. Check the [Troubleshooting](#troubleshooting) section
2. Verify all prerequisites are met
3. Test credentials with Azure CLI first
4. Contact Harness support with connector test logs
