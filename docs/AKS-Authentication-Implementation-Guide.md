# AKS Authentication Modernization - Implementation Guide

---

## What This Guide Covers

✅ Enable Microsoft Entra ID authentication on existing AKS cluster  
✅ Enable Azure RBAC for Kubernetes authorization  
✅ Disable local accounts  
✅ Assign roles to users/groups  
✅ Azure Policy enforcement  

---

## Prerequisites

- Existing AKS cluster (version 1.27+)
- **Azure CLI 2.61.0+** (recommended: 2.80.0+)
- **kubelogin** (automatically installed/managed by Azure CLI)
- Permissions: Owner or Contributor on cluster + User Access Administrator
- Existing Entra ID groups for users
- kubectl installed

**Verify Azure CLI version:**
```bash
az version
# If older than 2.61.0, update: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

---

## Step-by-Step Implementation

### Step 1: Prepare Environment

```bash
# Set variables
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="rg-aks-prod"
CLUSTER_NAME="aks-prod-001"

# Login and set subscription
az login
az account set --subscription $SUBSCRIPTION_ID

# Get cluster resource ID
CLUSTER_ID=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query 'id' --output tsv)

echo "Cluster ID: $CLUSTER_ID"
```

---

### Step 2: Check Current Configuration

```bash
# Check current authentication configuration
az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query '{EntraID:aadProfile.managed, AzureRBAC:aadProfile.enableAzureRbac, AdminGroups:aadProfile.adminGroupObjectIDs, LocalAccountsDisabled:disableLocalAccounts, WorkloadIdentity:securityProfile.workloadIdentity.enabled, OidcIssuer:oidcIssuerProfile.enabled}'

# Expected output BEFORE enablement:
# {
#   "AdminGroups": null,
#   "AzureRBAC": null,              <- Not enabled
#   "EntraID": null,                <- Not enabled
#   "LocalAccountsDisabled": false, <- Local accounts enabled
#   "OidcIssuer": false,            <- Not enabled
#   "WorkloadIdentity": null        <- Not enabled
# }

# Expected output AFTER enablement:
# {
#   "AdminGroups": ["c156565c-7065-..."],
#   "AzureRBAC": true,              <- ✅ Enabled
#   "EntraID": true,                <- ✅ Enabled
#   "LocalAccountsDisabled": false, <- Will disable in Step 6
#   "OidcIssuer": true,             <- ✅ Enabled
#   "WorkloadIdentity": true        <- ✅ Enabled
# }
```

**Note:** Use lowercase `enableAzureRbac` (not `enableAzureRBAC`) in the query for accurate results.

---

### Step 3: Get Admin Group Object ID

```bash
# List AKS-related groups
az ad group list --query "[?contains(displayName, 'AKS') || contains(displayName, 'Platform')].{Name:displayName, ObjectId:id}" --output table

# Store admin group ID
ADMIN_GROUP_ID="paste-admin-group-object-id-here"
```

---

### Step 4: Enable Entra ID Integration and Assign Roles

#### 4.1: Enable Entra ID, Azure RBAC, and Workload Identity

```bash
# Enable Entra ID + Azure RBAC + Workload Identity
az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --enable-aad \
    --enable-azure-rbac \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --aad-admin-group-object-ids $ADMIN_GROUP_ID

# Takes 5-10 minutes
```

**What was enabled:**
- **Entra ID**: Users authenticate with their Microsoft Entra ID accounts
- **Azure RBAC**: Authorization managed through Azure roles
- **OIDC Issuer**: Required for workload identity federation
- **Workload Identity**: Allows pods to authenticate to Azure services without secrets

#### 4.2: Assign Admin Group Roles (Cluster-Wide Access)

```bash
# Get the cluster resource ID
CLUSTER_ID=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query id -o tsv)

# Assign Cluster User Role (required to get credentials)
az role assignment create \
    --assignee $ADMIN_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $CLUSTER_ID

# Assign RBAC Cluster Admin (full cluster access)
az role assignment create \
    --assignee $ADMIN_GROUP_ID \
    --role "Azure Kubernetes Service RBAC Cluster Admin" \
    --scope $CLUSTER_ID

# Wait 60 seconds for role assignments to propagate
echo "Waiting for role assignments to propagate..."
sleep 60
```

#### 4.3: Assign Developer/User Group Roles (Choose Access Level)

First, get the user group Object ID:

```bash
# List user groups
az ad group list --query "[?contains(displayName, 'Dev') || contains(displayName, 'Developer') || contains(displayName, 'Team')].{Name:displayName, ObjectId:id}" --output table

# Store user group ID
USER_GROUP_ID="paste-user-group-object-id-here"
```

**Option A: Cluster-Wide Read-Only Access (for security/audit teams)**

```bash
# Assign Cluster User Role (required to get credentials)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $CLUSTER_ID

# Assign RBAC Reader (read-only across entire cluster)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service RBAC Reader" \
    --scope $CLUSTER_ID
```

**Option B: Namespace-Scoped Write Access (recommended for application teams)**

```bash
# Create namespace for the team
NAMESPACE="team-app"
kubectl create namespace $NAMESPACE

# Verify namespace creation
kubectl get namespace $NAMESPACE
# Expected output:
# NAME       STATUS   AGE
# team-app   Active   5s

# Get namespace resource ID
NAMESPACE_ID="${CLUSTER_ID}/namespaces/${NAMESPACE}"

# Assign Cluster User Role (required to get credentials)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $CLUSTER_ID

# Assign RBAC Writer (write access to specific namespace only)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service RBAC Writer" \
    --scope $NAMESPACE_ID
```

**Option C: Cluster-Wide Admin Access (for additional admin groups)**

```bash
# Assign Cluster User Role (required to get credentials)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope $CLUSTER_ID

# Assign RBAC Cluster Admin (full cluster access)
az role assignment create \
    --assignee $USER_GROUP_ID \
    --role "Azure Kubernetes Service RBAC Cluster Admin" \
    --scope $CLUSTER_ID
```

**Role Assignment Summary:**

| Option | Use Case | Cluster User Role | RBAC Role | Scope |
|--------|----------|------------------|-----------|-------|
| A | Security/Audit teams | ✅ Cluster | RBAC Reader | Cluster |
| B | App team developers | ✅ Cluster | RBAC Writer | Namespace |
| C | Platform/Operations admins | ✅ Cluster | RBAC Cluster Admin | Cluster |

**Verify Role Assignments:**

```bash
# List all role assignments for the user group
az role assignment list \
    --assignee $USER_GROUP_ID \
    --scope $CLUSTER_ID \
    --output table

# Expected output shows both roles:
# Principal               Role                                        Scope
# ----------------------  ------------------------------------------  ---------------
# AKS-tenant1-developers  Azure Kubernetes Service Cluster User Role  /subscriptions/.../aks-prod-001
# AKS-tenant1-developers  Azure Kubernetes Service RBAC Reader        /subscriptions/.../aks-prod-001
```

---

### Step 5: Test Entra ID Authentication

```bash
# Clear kubectl cache to force fresh authentication
rm -rf ~/.kube/cache

# Download credentials (no --admin flag)
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --overwrite-existing

# Test access (will authenticate using Entra ID via kubelogin)
kubectl get nodes

# You should see the node list
# Note: kubelogin uses your existing Azure CLI login (azurecli mode)
# If you were not logged in via Azure CLI, you would see a device code prompt
```

**How Entra ID authentication works:**
- `kubelogin` is automatically installed by `az aks get-credentials`
- Uses `azurecli` mode by default (reuses your `az login` session)
- No interactive prompt needed if already logged in via Azure CLI
- If Azure CLI session expires, you'll be prompted to authenticate

**✅ If authentication works, proceed to Step 6**  
**❌ If fails, DO NOT disable local accounts yet**

---

### Step 6: Disable Local Accounts

⚠️ **WARNING**: Only do this after confirming Entra ID authentication works in Step 5

```bash
# Disable local accounts
az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --disable-local-accounts

# Takes 5-10 minutes
```

---

### Step 7: Rotate Cluster Certificates (If Needed)

⚠️ **Only required if users previously authenticated with local accounts**

**Check if rotation is needed:**
- **Existing cluster** where users may have used `--admin` credentials → **Rotate certificates**
- **New cluster** or cluster where local accounts were never used → **Skip this step**

```bash
# Rotate certificates (causes up to 30 min CLUSTER downtime)
az aks rotate-certs \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME

# Confirm when prompted
# Wait for completion (up to 30 minutes)
```

**Why rotate?** To revoke any certificates users obtained via local admin accounts before they were disabled.

**⚠️ CRITICAL WARNING**: This operation:
- **Recreates ALL nodes, virtual machine scale sets, and disks**
- **Causes up to 30 minutes of FULL CLUSTER downtime** (workloads will be unavailable)
- **MUST be planned during maintenance window**
- Cannot be cancelled once started

**After rotation completes:**

```bash
# Check if rotation is complete
az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query provisioningState -o tsv
# Wait until it shows "Succeeded"

# Clear old certificates and download fresh credentials
rm -rf ~/.kube/cache
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --overwrite-existing \
    --file ~/.kube/config
```

**Note:** During rotation, you'll get certificate errors like `x509: certificate signed by unknown authority`. This is normal - wait for rotation to complete, then download fresh credentials.

---

### Step 8: Verify Configuration

```bash
# Check current authentication configuration
az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query '{EntraID:aadProfile.managed, AzureRBAC:aadProfile.enableAzureRbac, AdminGroups:aadProfile.adminGroupObjectIDs, LocalAccountsDisabled:disableLocalAccounts, WorkloadIdentity:securityProfile.workloadIdentity.enabled, OidcIssuer:oidcIssuerProfile.enabled}'

# List all role assignments
az role assignment list \
    --scope $CLUSTER_ID \
    --query "[?contains(roleDefinitionName, 'Kubernetes')].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
    --output table

# Test kubectl access
kubectl get nodes
kubectl get namespaces
```

**Expected output:**
- EntraID: `true`
- AzureRBAC: `true`
- LocalAccountsDisabled: `true`
- WorkloadIdentity: `true`
- OidcIssuer: `true`
- Multiple role assignments visible for different groups

---

### Step 9: Azure Policy Compliance

Verify cluster complies with authentication policies:

```bash
# Check policy compliance
az policy state list \
    --resource $CLUSTER_ID \
    --query "[?contains(policyDefinitionName, 'local') || contains(policyDefinitionName, 'RBAC')].{Policy:policyDefinitionName, State:complianceState}" \
    --output table
```

**Expected policies (from [Azure Policy Guide](Azure-Policy-Guide.md)):**
- **Policy #8**: Azure RBAC for Kubernetes Authorization (Audit) - Should be Compliant
- **Policy #9**: Disable Local Accounts (Deny) - Should be Compliant
- **Policy #13**: Require AKS Managed Identity (Deny) - Should be Compliant

---

## Quick Reference: Azure Roles

| Role | Use Case | Scope |
|------|----------|-------|
| **Azure Kubernetes Service Cluster User Role** | Get cluster credentials | Cluster (required for all users) |
| **Azure Kubernetes Service RBAC Cluster Admin** | Full cluster management | Cluster |
| **Azure Kubernetes Service RBAC Admin** | Manage resources + roles in namespace | Cluster or Namespace |
| **Azure Kubernetes Service RBAC Writer** | Deploy and manage apps | Cluster or Namespace |
| **Azure Kubernetes Service RBAC Reader** | Read-only access | Cluster or Namespace |

---

## Troubleshooting

### Issue: "User does not have access to the resource"

**Cause**: Missing RBAC role assignment

**Fix**: Assign appropriate role (see Step 8)

---

### Issue: "Getting static credential is not allowed"

**Cause**: Local accounts disabled (expected after Step 6)

**Fix**: Don't use `--admin` flag. Use Entra ID authentication.

---

### Issue: Token expired

**Fix**: Clear cache and re-authenticate
```bash
rm -rf ~/.kube/cache/*
kubectl get nodes  # Will re-authenticate
```

---

### Issue: Conditional Access blocking

**Cause**: MFA or device compliance required

**Fix**: 
1. Check Entra ID sign-in logs
2. Complete MFA enrollment
3. Enroll device in Intune (if device compliance required)

---

## Emergency Access (Break-Glass)

If you're locked out after disabling local accounts:

```bash
# Option 1: Use command invoke (no VM required)
az aks command invoke \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --command "kubectl get nodes"

# Option 2: Use emergency access script
./scripts/emergency-access-procedure.sh \
    --cluster-name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP
```

---

## Rollback Procedure

If you need to rollback:

```bash
# Re-enable local accounts
az aks update \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --enable-local-accounts

# Get admin credentials
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --admin --overwrite-existing
```

⚠️ **Note**: Rollback should be temporary. Fix issues and re-enable Entra ID.

---

## Next Steps

After authentication is configured:

1. **Workload Identity**: Enable secret-less authentication for pods
2. **Network Policies**: Enforce namespace isolation at network level
3. **Resource Quotas**: Limit resources per namespace
4. **Monitoring**: Set up alerts for authentication failures
