# AKS Upgrade Playbook

**Document Type**: Upgrade Playbook  
**Project**: Azure Kubernetes Service (AKS) - Cluster Upgrade Operations  
**Version**: 1.0  
**Date**: November 23, 2025  
**Status**: Active  
**Review Cycle**: Quarterly

---

**📌 [← Back to Main Documentation](README.md)**

---

## Related Documentation

This upgrade playbook is part of the AKS enterprise enablement documentation set:
- **[Main Documentation Hub](README.md)** - Overview of all deliverables
- **[Private AKS Cluster Guide](private-aks-cluster-guide.md)** - Private cluster deployment
- **[Azure Policy Guide](Azure-Policy-Guide.md)** - Governance and compliance policies

---

## Overview

This playbook provides comprehensive guidance for upgrading Azure Kubernetes Service (AKS) clusters, including Kubernetes version upgrades, node pool OS upgrades, and governance-driven automation strategies.

**Key Principles**:
- ✅ **Test-first approach**: Validate upgrades in non-production before production
- ✅ **Communication-driven**: Notify application teams before upgrades
- ✅ **Governance-enabled**: Policy-driven upgrade cadence with manual override options
- ✅ **Rollback-ready**: Clear procedures for reverting failed upgrades

---

## 🎯 Upgrade Strategy Overview

### Manual vs Automatic Upgrades

| Aspect | Manual Upgrades | Automatic Upgrades |
|--------|----------------|-------------------|
| **Control** | Full control over timing | Scheduled within maintenance windows |
| **Validation** | Pre-upgrade testing in non-prod required | Post-upgrade validation + pre-testing in non-prod |
| **Communication** | Team coordination for each upgrade | One-time setup, ongoing notifications |
| **Best For** | Minor version upgrades (1.29→1.30) for stateful workloads | Security patches (1.29.0→1.29.9), non-prod, stateless workloads |
| **Rollback** | Controlled rollback possible | Limited rollback options |
| **Requirements** | Manual scheduling and execution | Pod Disruption Budgets (PDBs) + 4+ hour maintenance windows |

### When to Use Each Approach

**Use Manual Upgrades for Minor Versions When**:
- **Production clusters with stateful workloads** (databases, persistent storage)
- Multiple application teams with different maintenance windows
- Complex dependencies between services
- Regulatory compliance requires change approval
- **Critical**: Databases require validation before Kubernetes minor version upgrades

> **⚠️ Important for Stateful Workloads**: Even with manual minor version upgrades, **enable `patch` auto-upgrade** to receive security patches automatically. See [Stateful workload upgrade patterns](https://learn.microsoft.com/en-us/azure/aks/stateful-workload-upgrades).

**Use Automatic Upgrades (`patch` or `stable`) When**:
- **Production clusters** - Use `patch` for security-only updates or `stable` for N-1 version stability
- Non-production environments (dev, test, staging)
- Stateless workloads with proper Pod Disruption Budgets (PDBs)
- Single-team ownership with aligned schedules
- Need to stay current with security patches

**Critical Requirements for Auto-Upgrades**:
- ✅ Configure **Pod Disruption Budgets (PDBs)** for all deployments
- ✅ Set **4+ hour maintenance windows** (Microsoft requirement)
- ✅ Test in non-prod with same upgrade channel first
- ✅ For stateful workloads: Review [Stateful upgrade patterns](https://learn.microsoft.com/en-us/azure/aks/stateful-workload-upgrades)

---

## 📋 Pre-Upgrade Checklist

Complete this checklist **before** upgrading any AKS cluster:

### 1. Capacity & Resource Checks

```bash
# Check current cluster version
CLUSTER_NAME="aks-prod-001"
RESOURCE_GROUP="rg-aks-prod"

az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "{kubernetesVersion:kubernetesVersion, currentNodePoolVersions:agentPoolProfiles[].orchestratorVersion}" \
  -o json

# Check available upgrade versions
az aks get-upgrades --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP -o table

# Check node pool capacity
kubectl top nodes
kubectl describe nodes | grep -i "allocatable:\|allocated resources:"

# Check pending PVCs and storage
kubectl get pvc --all-namespaces
kubectl get pv
```

### 2. Backup Validation

```bash
# Verify Azure Backup for AKS is configured
az backup container list --resource-group $RESOURCE_GROUP --vault-name <backup-vault-name> -o table

# Backup cluster configuration (manual)
kubectl get all --all-namespaces -o yaml > cluster-backup-$(date +%Y%m%d).yaml
kubectl get pvc --all-namespaces -o yaml > pvc-backup-$(date +%Y%m%d).yaml

# Export Helm releases
helm list --all-namespaces -o yaml > helm-releases-$(date +%Y%m%d).yaml
```

### 3. Application Health Checks

```bash
# Check pod health
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Check persistent volumes
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name

# Verify application endpoints
kubectl get ingress --all-namespaces
kubectl get svc --all-namespaces --field-selector spec.type=LoadBalancer

# Test application functionality
# (Run application-specific health checks here)
```

### 4. Communication & Approval

- [ ] Notify application teams 7 days before upgrade
- [ ] Schedule upgrade during approved maintenance window
- [ ] Obtain change approval from relevant stakeholders
- [ ] Confirm rollback plan with operations team
- [ ] Document expected downtime (if any)

---

## 🔍 Pre-Upgrade Validation Script

Run this comprehensive script before upgrading any cluster:

```bash
#!/bin/bash
# pre-upgrade-validation.sh

CLUSTER_NAME="${1:-aks-prod-001}"
RESOURCE_GROUP="${2:-rg-aks-prod}"
TARGET_VERSION="${3}"

echo "=========================================="
echo "AKS Pre-Upgrade Validation"
echo "=========================================="
echo "Cluster: $CLUSTER_NAME"
echo "Resource Group: $RESOURCE_GROUP"
echo "Target Version: ${TARGET_VERSION:-TBD}"
echo "Date: $(date)"
echo "=========================================="
echo ""

# Initialize counters
WARNINGS=0
ERRORS=0

# 1. Check cluster accessibility
echo "[1/12] Checking cluster accessibility..."
if az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP &>/dev/null; then
  echo "✅ Cluster accessible via Azure CLI"
else
  echo "❌ Cannot access cluster via Azure CLI"
  ERRORS=$((ERRORS + 1))
fi

if kubectl cluster-info &>/dev/null; then
  echo "✅ Cluster accessible via kubectl"
else
  echo "❌ Cannot access cluster via kubectl"
  ERRORS=$((ERRORS + 1))
fi
echo ""

# 2. Check current and available versions
echo "[2/12] Checking Kubernetes versions..."
CURRENT_VERSION=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query kubernetesVersion -o tsv)
echo "Current version: $CURRENT_VERSION"

echo "Available upgrade versions:"
az aks get-upgrades --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP -o table
echo ""

# 3. Check cluster state
echo "[3/12] Checking cluster provisioning state..."
PROVISIONING_STATE=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query provisioningState -o tsv)
if [ "$PROVISIONING_STATE" == "Succeeded" ]; then
  echo "✅ Cluster state: $PROVISIONING_STATE"
else
  echo "⚠️  Cluster state: $PROVISIONING_STATE (should be 'Succeeded')"
  WARNINGS=$((WARNINGS + 1))
fi
echo ""

# 4. Check node health
echo "[4/12] Checking node health..."
TOTAL_NODES=$(kubectl get nodes --no-headers | wc -l)
READY_NODES=$(kubectl get nodes --no-headers | grep " Ready" | wc -l)
NOT_READY=$(kubectl get nodes --no-headers | grep -v " Ready" | wc -l)

echo "Total nodes: $TOTAL_NODES"
echo "Ready nodes: $READY_NODES"

if [ "$NOT_READY" -eq 0 ]; then
  echo "✅ All nodes are Ready"
else
  echo "❌ $NOT_READY nodes are NOT Ready:"
  kubectl get nodes | grep -v " Ready"
  ERRORS=$((ERRORS + 1))
fi
echo ""

# 5. Check pod health
echo "[5/12] Checking pod health..."
TOTAL_PODS=$(kubectl get pods --all-namespaces --no-headers | wc -l)
RUNNING_PODS=$(kubectl get pods --all-namespaces --no-headers | grep "Running\|Completed" | wc -l)
UNHEALTHY_PODS=$(kubectl get pods --all-namespaces --no-headers | grep -v "Running\|Completed" | wc -l)

echo "Total pods: $TOTAL_PODS"
echo "Healthy pods: $RUNNING_PODS"

if [ "$UNHEALTHY_PODS" -eq 0 ]; then
  echo "✅ All pods are healthy"
else
  echo "⚠️  $UNHEALTHY_PODS pods are not healthy:"
  kubectl get pods --all-namespaces | grep -v "Running\|Completed"
  WARNINGS=$((WARNINGS + 1))
fi
echo ""

# 6. Check system pods (critical for cluster operation)
echo "[6/12] Checking system pods (kube-system)..."
SYSTEM_PODS_NOT_READY=$(kubectl get pods -n kube-system --no-headers | grep -v "Running\|Completed" | wc -l)
if [ "$SYSTEM_PODS_NOT_READY" -eq 0 ]; then
  echo "✅ All kube-system pods are healthy"
else
  echo "❌ System pods not healthy:"
  kubectl get pods -n kube-system | grep -v "Running\|Completed"
  ERRORS=$((ERRORS + 1))
fi
echo ""

# 7. Check PVC and PV status
echo "[7/12] Checking Persistent Volumes..."
TOTAL_PVC=$(kubectl get pvc --all-namespaces --no-headers 2>/dev/null | wc -l)
BOUND_PVC=$(kubectl get pvc --all-namespaces --no-headers 2>/dev/null | grep "Bound" | wc -l)
UNBOUND_PVC=$(kubectl get pvc --all-namespaces --no-headers 2>/dev/null | grep -v "Bound" | wc -l)

echo "Total PVCs: $TOTAL_PVC"
if [ "$TOTAL_PVC" -gt 0 ]; then
  echo "Bound PVCs: $BOUND_PVC"
  if [ "$UNBOUND_PVC" -gt 0 ]; then
    echo "⚠️  $UNBOUND_PVC PVCs are not Bound:"
    kubectl get pvc --all-namespaces | grep -v "Bound"
    WARNINGS=$((WARNINGS + 1))
  else
    echo "✅ All PVCs are Bound"
  fi
else
  echo "ℹ️  No PVCs found in cluster"
fi
echo ""

# 8. Check node pool versions
echo "[8/12] Checking node pool versions..."
echo "Node pool versions:"
az aks nodepool list --cluster-name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name,K8sVersion:orchestratorVersion,Count:count,VMSize:vmSize}" -o table

# Check for version skew
NODE_VERSIONS=$(kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}' | tr ' ' '\n' | sort -u)
UNIQUE_VERSIONS=$(echo "$NODE_VERSIONS" | wc -l)
if [ "$UNIQUE_VERSIONS" -eq 1 ]; then
  echo "✅ All nodes running same Kubernetes version"
else
  echo "⚠️  Nodes running different versions (version skew detected):"
  echo "$NODE_VERSIONS"
  WARNINGS=$((WARNINGS + 1))
fi
echo ""

# 9. Check for deprecated APIs
echo "[9/12] Checking for deprecated API usage..."
DEPRECATED_APIS=$(kubectl get --raw /apis 2>/dev/null | grep -i "deprecated" || echo "")
if [ -z "$DEPRECATED_APIS" ]; then
  echo "✅ No deprecated APIs detected"
else
  echo "⚠️  Deprecated APIs found - review application manifests"
  WARNINGS=$((WARNINGS + 1))
fi
echo ""

# 10. Check cluster resource capacity
echo "[10/12] Checking cluster resource capacity..."
echo "Node resource allocation:"
kubectl top nodes 2>/dev/null || echo "⚠️  Metrics server not available"
echo ""

# 11. Check for Pod Disruption Budgets
echo "[11/12] Checking Pod Disruption Budgets (PDBs)..."
PDB_COUNT=$(kubectl get pdb --all-namespaces --no-headers 2>/dev/null | wc -l)
if [ "$PDB_COUNT" -gt 0 ]; then
  echo "✅ Found $PDB_COUNT Pod Disruption Budget(s)"
  kubectl get pdb --all-namespaces
else
  echo "⚠️  No Pod Disruption Budgets found - consider adding PDBs for critical workloads"
  WARNINGS=$((WARNINGS + 1))
fi
echo ""

# 12. Check auto-upgrade configuration
echo "[12/12] Checking auto-upgrade configuration..."
AUTO_UPGRADE=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "autoUpgradeProfile.upgradeChannel" -o tsv)
if [ "$AUTO_UPGRADE" != "None" ] && [ "$AUTO_UPGRADE" != "" ]; then
  echo "✅ Auto-upgrade enabled: $AUTO_UPGRADE"
else
  echo "ℹ️  Auto-upgrade disabled (manual upgrades required)"
fi
echo ""

# Summary
echo "=========================================="
echo "Pre-Upgrade Validation Summary"
echo "=========================================="
echo "Errors: $ERRORS"
echo "Warnings: $WARNINGS"
echo ""

if [ "$ERRORS" -gt 0 ]; then
  echo "❌ VALIDATION FAILED - Fix errors before upgrading"
  exit 1
elif [ "$WARNINGS" -gt 0 ]; then
  echo "⚠️  VALIDATION PASSED WITH WARNINGS - Review warnings before proceeding"
  exit 0
else
  echo "✅ VALIDATION PASSED - Cluster is ready for upgrade"
  exit 0
fi
```

**Usage**:
```bash
# Make script executable
chmod +x pre-upgrade-validation.sh

# Run validation
./pre-upgrade-validation.sh aks-prod-001 rg-aks-prod 1.29.9

# Check exit code
if [ $? -eq 0 ]; then
  echo "Proceeding with upgrade..."
else
  echo "Fix issues before upgrading"
fi
```

---

## 🚀 Upgrade Procedures

### Option 1: Manual Kubernetes Version Upgrade

**Use Case**: Production clusters requiring controlled upgrades

#### Step 1: Validate Target Version

```bash
# List available upgrade versions
az aks get-upgrades --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP -o table

# Check Kubernetes version support policy
# Microsoft supports N-2 versions (e.g., if latest is 1.30, 1.28 and 1.29 are supported)
```

#### Step 2: Test in Non-Production First

```bash
# Upgrade staging/test cluster first
TEST_CLUSTER="aks-test-001"
TARGET_VERSION="1.29.9"

az aks upgrade \
  --name $TEST_CLUSTER \
  --resource-group $RESOURCE_GROUP \
  --kubernetes-version $TARGET_VERSION \
  --yes

# Wait for upgrade to complete (20-30 minutes)
az aks show --name $TEST_CLUSTER --resource-group $RESOURCE_GROUP \
  --query "kubernetesVersion" -o tsv
```

#### Step 3: Validate Test Cluster

```bash
# Switch to test cluster
az aks get-credentials --name $TEST_CLUSTER --resource-group $RESOURCE_GROUP --overwrite-existing

# Run validation tests
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl top nodes

# Run application smoke tests
# (Application-specific validation)

# Check for deprecation warnings
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found --all-namespaces
```

#### Step 4: Upgrade Production Cluster (Control Plane)

```bash
# Upgrade control plane only (no node pool upgrade yet)
az aks upgrade \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --kubernetes-version $TARGET_VERSION \
  --control-plane-only \
  --yes

# Monitor upgrade progress
az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "kubernetesVersion" -o tsv
```

#### Step 5: Upgrade Node Pools (One at a Time)

```bash
# List node pools
az aks nodepool list --cluster-name $CLUSTER_NAME --resource-group $RESOURCE_GROUP -o table

# Upgrade first node pool (system pool)
NODEPOOL_NAME="system"

az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name $NODEPOOL_NAME \
  --kubernetes-version $TARGET_VERSION \
  --yes

# Wait for system pool upgrade (monitor with kubectl)
kubectl get nodes -o wide --watch

# Upgrade remaining node pools one by one
NODEPOOL_NAME="userpool01"

az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name $NODEPOOL_NAME \
  --kubernetes-version $TARGET_VERSION \
  --yes
```

#### Step 6: Post-Upgrade Validation

```bash
# Verify all nodes upgraded
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

# Check pod health
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Verify application functionality
# (Run application smoke tests)

# Check for any admission webhook errors
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

---

### Option 2: Automatic Upgrade with Maintenance Windows

**Use Case**: Non-production clusters or production with flexible maintenance windows

#### Step 1: Configure Auto-Upgrade Channel

```bash
# Set auto-upgrade channel
# Options: rapid, stable, patch, node-image, none

az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --auto-upgrade-channel patch
```

**Auto-Upgrade Channels Explained**:
- **`rapid`**: Upgrade to latest supported version immediately after release
- **`stable`**: Upgrade to N-1 version (one version behind latest)
- **`patch`** ⭐ **RECOMMENDED**: Automatically apply patch updates within current minor version (e.g., 1.29.0 → 1.29.9)
- **`node-image`**: Auto-upgrade node OS images only (not Kubernetes version)
- **`none`**: Disable automatic upgrades (manual only)

> **💡 Best Practice**: Use `patch` channel for production clusters to automatically receive security patches while maintaining control over minor version upgrades (e.g., 1.29 → 1.30).

#### Step 2: Configure Maintenance Window

```bash
# Create maintenance window configuration
cat <<EOF > maintenance-config.json
{
  "timeInWeek": [
    {
      "day": "Saturday",
      "hourSlots": [2, 3, 4]
    },
    {
      "day": "Sunday",
      "hourSlots": [2, 3, 4]
    }
  ],
  "notAllowedTime": [
    {
      "start": "2025-12-20T00:00:00Z",
      "end": "2026-01-05T00:00:00Z"
    }
  ]
}
EOF

# Apply maintenance configuration
az aks maintenanceconfiguration add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name aksManagedAutoUpgradeSchedule \
  --config-file maintenance-config.json
```

#### Step 3: Enable Planned Maintenance Notifications

```bash
# Configure Azure Monitor alerts for maintenance events
az monitor metrics alert create \
  --name "AKS-Upgrade-Starting" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/<subscription-id>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerService/managedClusters/$CLUSTER_NAME" \
  --condition "count NaN" \
  --description "Alert when AKS cluster upgrade begins" \
  --action <action-group-id>
```

---

### Option 3: Node Pool OS SKU Upgrade (Ubuntu 22.04 → 24.04)

**Use Case**: Upgrade node pool OS version without changing Kubernetes version

#### Step 1: Check Current OS SKU

```bash
# Check current OS SKU per node pool
az aks nodepool list --cluster-name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name, OsSku:osSku, OsType:osType}" -o table
```

#### Step 2: Create New Node Pool with Updated OS SKU

```bash
# Create new node pool with Ubuntu 24.04
NEW_NODEPOOL="userpool02"

az aks nodepool add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name $NEW_NODEPOOL \
  --node-count 3 \
  --os-sku Ubuntu \
  --os-type Linux \
  --mode User \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10

# Wait for new node pool to be ready
kubectl get nodes -l agentpool=$NEW_NODEPOOL
```

#### Step 3: Drain Workloads from Old Node Pool

```bash
# Cordon old nodes (prevent new pods from scheduling)
OLD_NODEPOOL="userpool01"
kubectl cordon -l agentpool=$OLD_NODEPOOL

# Drain pods from old nodes (gracefully migrate)
for node in $(kubectl get nodes -l agentpool=$OLD_NODEPOOL -o name); do
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --timeout=300s
done

# Verify workloads moved to new node pool
kubectl get pods --all-namespaces -o wide | grep $NEW_NODEPOOL
```

#### Step 4: Delete Old Node Pool

```bash
# Delete old node pool
az aks nodepool delete \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name $OLD_NODEPOOL \
  --yes
```

---

## 🔄 Rollback Procedures

### Scenario 1: Control Plane Upgrade Failed

**Symptom**: Control plane upgrade fails or cluster becomes unresponsive

```bash
# Check cluster state
az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "{provisioningState:provisioningState, powerState:powerState}" -o json

# Contact Azure Support immediately - control plane rollback requires support ticket
# Provide cluster details and error logs
```

### Scenario 2: Node Pool Upgrade Failed (Pods Not Starting)

**Symptom**: Pods fail to start on upgraded nodes due to compatibility issues

```bash
# Option A: Rollback node pool to previous version (if supported)
PREVIOUS_VERSION="1.28.13"

az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name $NODEPOOL_NAME \
  --kubernetes-version $PREVIOUS_VERSION \
  --yes

# Option B: Create new node pool with previous version
az aks nodepool add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name ${NODEPOOL_NAME}-rollback \
  --kubernetes-version $PREVIOUS_VERSION \
  --node-count 3 \
  --zones 1 2 3

# Drain and delete failed node pool
kubectl drain -l agentpool=$NODEPOOL_NAME --ignore-daemonsets --delete-emptydir-data
az aks nodepool delete --cluster-name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --name $NODEPOOL_NAME --yes
```

### Scenario 3: Application Compatibility Issues Post-Upgrade

**Symptom**: Applications work but exhibit errors or degraded performance

```bash
# Identify problematic workloads
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -50
kubectl logs -n <namespace> <pod-name> --previous

# Rollback specific deployment to previous image
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# If cluster-wide issue, consider node pool recreation with previous version (see Scenario 2)
```

---

## 📜 Policy-Driven Upgrade Governance

### 1. Enforce Minimum Kubernetes Version (Built-in Policy)

**Use Case**: Ensure all clusters stay within supported version window (N-2)

**Policy Type**: ✅ **Built-in Azure Policy**

```bash
# Use built-in policy for Kubernetes version compliance
# Policy: "Kubernetes Services should be upgraded to a non-vulnerable Kubernetes version"
# Policy ID: /providers/Microsoft.Authorization/policyDefinitions/fb893a29-21bb-418c-a157-e99480ec364c

az policy assignment create \
  --name "audit-aks-version-currency" \
  --display-name "Audit AKS Kubernetes Version Currency" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/fb893a29-21bb-418c-a157-e99480ec364c" \
  --scope "/providers/Microsoft.Management/managementGroups/<mg-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

> **💡 Note**: This built-in policy audits clusters running versions outside the N-2 support window. See [Azure Policy Guide](Azure-Policy-Guide.md) for this policy (Policy #12).

---

### 2. Enforce Auto-Upgrade Configuration (Custom Policy)

**Use Case**: Audit or deny clusters without auto-upgrade enabled

**Policy Type**: 🔧 **Custom Azure Policy** (no built-in equivalent)

```bash
# Create custom policy to require auto-upgrade enabled
cat <<EOF > policy-require-auto-upgrade.json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ContainerService/managedClusters"
        },
        {
          "anyOf": [
            {
              "field": "Microsoft.ContainerService/managedClusters/autoUpgradeProfile.upgradeChannel",
              "exists": false
            },
            {
              "field": "Microsoft.ContainerService/managedClusters/autoUpgradeProfile.upgradeChannel",
              "equals": "none"
            }
          ]
        }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  },
  "parameters": {
    "effect": {
      "type": "String",
      "allowedValues": ["Audit", "Deny", "Disabled"],
      "defaultValue": "Audit"
    }
  }
}
EOF

# Create custom policy definition at management group
az policy definition create \
  --name "require-aks-auto-upgrade" \
  --display-name "Require AKS Auto-Upgrade Enabled" \
  --description "Audits or denies AKS clusters without auto-upgrade configured (recommended: patch channel)" \
  --rules policy-require-auto-upgrade.json \
  --mode Indexed \
  --management-group <mg-name>

# Assign policy (start with Audit mode)
az policy assignment create \
  --name "require-auto-upgrade" \
  --display-name "Require AKS Auto-Upgrade Configuration" \
  --policy "require-aks-auto-upgrade" \
  --scope "/providers/Microsoft.Management/managementGroups/<mg-name>" \
  --params '{"effect": {"value": "Audit"}}'
```

> **⚠️ Important**: This policy audits clusters with `upgradeChannel: none`. After validating compliance, switch to `Deny` mode to prevent new clusters without auto-upgrade.

---

## 📧 Communication Templates

### Template 1: 7-Day Advance Notice (Manual Upgrade)

**Subject**: [ACTION REQUIRED] AKS Cluster Upgrade Scheduled - {{ Cluster Name }} - {{ Date }}

**Body**:
```
Hello Team,

This is to notify you that a Kubernetes upgrade has been scheduled for the following AKS cluster:

Cluster Name: {{ Cluster Name }}
Current Version: {{ Current K8s Version }}
Target Version: {{ Target K8s Version }}
Scheduled Date: {{ Upgrade Date }}
Maintenance Window: {{ Start Time }} - {{ End Time }} ({{ Timezone }})
Expected Downtime: {{ Downtime Estimate or "Minimal downtime expected" }}

Pre-Upgrade Checklist:
☐ Review Kubernetes {{ Target Version }} release notes
☐ Test applications in staging cluster
☐ Verify pod disruption budgets are configured
☐ Confirm backup procedures are current

Post-Upgrade:
- Automated validation tests will run immediately after upgrade
- Application teams should perform smoke tests within 1 hour
- Rollback window: {{ Rollback Window }}

Questions or Concerns:
Contact: {{ Operations Team Contact }}
Incident Channel: {{ Slack/Teams Channel }}

Thank you,
{{ Your Name }}
AKS Operations Team
```

### Template 2: Auto-Upgrade Notification (30-Day Notice)

**Subject**: [NOTIFICATION] Automatic AKS Upgrade Enabled - {{ Cluster Name }}

**Body**:
```
Hello Team,

Automatic upgrades have been enabled for the following cluster:

Cluster Name: {{ Cluster Name }}
Auto-Upgrade Channel: {{ stable/patch/node-image }}
Maintenance Window: {{ Days }} at {{ Hours }} UTC
Next Expected Upgrade: {{ Estimated Date Range }}

What This Means:
- Cluster will automatically upgrade to {{ Channel Description }}
- Upgrades occur only during configured maintenance windows
- Azure typically notifies 48 hours before scheduled upgrade

Application Team Actions Required:
☐ Ensure your applications are compatible with N-1 Kubernetes versions
☐ Configure Pod Disruption Budgets for all critical deployments
☐ Subscribe to AKS release notes: https://github.com/Azure/AKS/releases
☐ Test application compatibility in staging after each upgrade

Monitoring & Alerts:
- Upgrade start/completion notifications: {{ Alert Channel }}
- Post-upgrade health checks: Automated via Azure Monitor

Opt-Out Process (if needed):
Contact {{ Operations Team }} with business justification for manual upgrade override.

Questions:
{{ Operations Team Contact }}

Thank you,
{{ Your Name }}
AKS Operations Team
```

---

## 📊 Upgrade Impact Analysis Template

Use this template to document upgrade impact before proceeding:

```markdown
# AKS Upgrade Impact Analysis

## Cluster Details
- **Cluster Name**: {{ Cluster Name }}
- **Current Version**: {{ Current Version }}
- **Target Version**: {{ Target Version }}
- **Upgrade Date**: {{ Scheduled Date }}

## Application Inventory
| Application | Owner Team | Criticality | Stateful? | Test Status | Approval |
|-------------|-----------|-------------|-----------|-------------|----------|
| app-1 | Team A | High | Yes | ✅ Passed | ✅ Approved |
| app-2 | Team B | Medium | No | ⏳ Pending | ⏳ Pending |
| app-3 | Team C | Low | No | ✅ Passed | ✅ Approved |

## Breaking Changes (Kubernetes {{ Target Version }})
- [ ] API deprecations affecting deployed resources
- [ ] Admission webhook changes
- [ ] RBAC policy changes
- [ ] Network policy changes

## Risk Assessment
**Risk Level**: {{ Low/Medium/High }}

**Mitigation**:
- {{ Mitigation Strategy 1 }}
- {{ Mitigation Strategy 2 }}

## Rollback Plan
**Trigger Conditions**:
- {{ Condition 1 }}
- {{ Condition 2 }}

**Rollback Steps**:
1. {{ Step 1 }}
2. {{ Step 2 }}

## Approvals
- [ ] Application Team Lead: {{ Name }}
- [ ] Operations Manager: {{ Name }}
- [ ] Change Advisory Board (if required)
```

---

## 🔍 Post-Upgrade Validation Checklist

Run these checks after every upgrade:

```bash
#!/bin/bash
# post-upgrade-validation.sh

CLUSTER_NAME="aks-prod-001"
RESOURCE_GROUP="rg-aks-prod"
TARGET_VERSION="1.29.9"

echo "=== AKS Post-Upgrade Validation ==="
echo "Target Version: $TARGET_VERSION"
echo ""

# 1. Verify control plane version
echo "[1/10] Checking control plane version..."
CONTROL_PLANE_VERSION=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query kubernetesVersion -o tsv)
if [ "$CONTROL_PLANE_VERSION" == "$TARGET_VERSION" ]; then
  echo "✅ Control plane upgraded successfully: $CONTROL_PLANE_VERSION"
else
  echo "❌ Control plane version mismatch: Expected $TARGET_VERSION, Got $CONTROL_PLANE_VERSION"
fi

# 2. Verify all node pool versions
echo "[2/10] Checking node pool versions..."
az aks nodepool list --cluster-name $CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name,Version:orchestratorVersion}" -o table

# 3. Check node readiness
echo "[3/10] Checking node readiness..."
NOT_READY=$(kubectl get nodes --no-headers | grep -v " Ready" | wc -l)
if [ "$NOT_READY" -eq 0 ]; then
  echo "✅ All nodes are Ready"
else
  echo "⚠️  $NOT_READY nodes are not Ready"
  kubectl get nodes
fi

# 4. Check pod health
echo "[4/10] Checking pod health..."
UNHEALTHY_PODS=$(kubectl get pods --all-namespaces --no-headers | grep -v "Running\|Completed" | wc -l)
if [ "$UNHEALTHY_PODS" -eq 0 ]; then
  echo "✅ All pods are healthy"
else
  echo "⚠️  $UNHEALTHY_PODS pods are not healthy"
  kubectl get pods --all-namespaces | grep -v "Running\|Completed"
fi

# 5. Check system pods (kube-system namespace)
echo "[5/10] Checking system pods..."
kubectl get pods -n kube-system -o wide

# 6. Verify Persistent Volume Claims
echo "[6/10] Checking PVC status..."
FAILED_PVC=$(kubectl get pvc --all-namespaces --no-headers | grep -v "Bound" | wc -l)
if [ "$FAILED_PVC" -eq 0 ]; then
  echo "✅ All PVCs are Bound"
else
  echo "⚠️  $FAILED_PVC PVCs are not Bound"
  kubectl get pvc --all-namespaces | grep -v "Bound"
fi

# 7. Check API server responsiveness
echo "[7/10] Testing API server responsiveness..."
kubectl cluster-info
kubectl get --raw /healthz

# 8. Verify ingress controllers
echo "[8/10] Checking ingress controllers..."
kubectl get pods -n ingress-nginx -o wide 2>/dev/null || echo "No ingress-nginx namespace found"

# 9. Check for deprecated API warnings
echo "[9/10] Checking for deprecated APIs..."
kubectl get --warnings-as-errors=true all --all-namespaces 2>&1 | grep -i "deprecated" || echo "✅ No deprecated API warnings"

# 10. Validate Azure Policy compliance
echo "[10/10] Checking Azure Policy compliance..."
kubectl get constrainttemplates 2>/dev/null || echo "Azure Policy add-on not detected"

echo ""
echo "=== Validation Complete ==="
echo "Review any warnings above before closing upgrade ticket."
```

---

## 📚 Additional Resources

### Microsoft Documentation
- [AKS Upgrade Documentation](https://learn.microsoft.com/en-us/azure/aks/upgrade-cluster)
- [Kubernetes Version Support Policy](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)
- [AKS Release Notes](https://github.com/Azure/AKS/releases)

### Kubernetes Upstream
- [Kubernetes Release Notes](https://kubernetes.io/releases/)
- [API Deprecation Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)

---

## 📋 Upgrade Decision Matrix

Use this matrix to determine the appropriate upgrade strategy based on Microsoft best practices:

| Cluster Type | Workload | Team Structure | Recommended Strategy | Rationale |
|--------------|----------|----------------|---------------------|-----------|
| Production | Stateful | Multi-team | **Manual** minor + **Auto (patch)** ⭐ | Manual control for minor versions (1.29→1.30), auto security patches within minor version |
| Production | Stateless | Single-team | **Auto (stable)** or **Auto (patch)** | `stable` for maturity (N-1), or `patch` for tighter control |
| Non-Prod | Any | Any | **Auto (stable)** | Stay current with N-1 version for pre-prod validation |
| Development | Any | Any | **Auto (rapid)** | Stay current with latest features, fast iteration |
| Staging | Stateful | Multi-team | **Auto (stable)** or **Auto (patch)** | Mirror production strategy, use same channel as prod |

> **⭐ Microsoft Recommendation**: 
> - **Production stateful workloads**: Use `patch` channel for automatic security updates (e.g., 1.29.0 → 1.29.9) while **manually upgrading minor versions** (e.g., 1.29 → 1.30) after validation. See [Stateful workload upgrade patterns](https://learn.microsoft.com/en-us/azure/aks/stateful-workload-upgrades).
> - **Production environments**: Microsoft recommends `stable` channel for "stability and version maturity" or `patch` for tighter control. Both require **4+ hour maintenance windows**.
> - **Always use Pod Disruption Budgets (PDBs)** for stateful workloads to prevent data loss during upgrades.

---

## ✅ Success Criteria

An upgrade is considered successful when:

- ✅ Control plane upgraded to target version
- ✅ All node pools upgraded to target version
- ✅ All nodes in Ready state
- ✅ All pods in Running or Completed state
- ✅ PVCs remain Bound
- ✅ Application smoke tests pass
- ✅ No Azure Policy violations introduced
- ✅ API server responsive (sub-second response times)
- ✅ Monitoring dashboards show normal metrics

---

## 📄 Document History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-11-23 | Initial release | AKS Enablement Team |

---

**📌 [← Back to Main Documentation](README.md)**
