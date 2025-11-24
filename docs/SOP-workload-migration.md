# Workload Migration SOP: Public to Private AKS Cluster

**Document Type**: Standard Operating Procedure (SOP)  
**Project**: Azure Kubernetes Service (AKS) - Workload Migration  
**Version**: 1.0  
**Date**: November 23, 2025  
**Status**: Active  
**Review Cycle**: Quarterly

---

**📌 [← Back to Main Documentation](private-aks-cluster-guide.md)**

---

## Related Documentation

This migration guide is part of the private AKS deployment documentation set:
- **[Private AKS Guide](private-aks-cluster-guide.md)** - Documentation overview and quick start
- **[Provision Private AKS Cluster](SOP-provision-private-aks-cluster.md)** - Complete deployment guide  
- **[Azure Policy Guide](Azure-Policy-Guide.md)** - Governance and compliance policies

> **⚠️ Important**: Before migrating, you must:
> 1. Deploy Azure policies from the **[Policy Guide](Azure-Policy-Guide.md)**
> 2. Build a new private cluster using **[Provision Private AKS SOP](SOP-provision-private-aks-cluster.md)**
> 3. Then follow this migration guide to move workloads

---

## Overview

This SOP provides step-by-step instructions for migrating application workloads from an **existing public AKS cluster** to a **new private AKS cluster** using a **blue/green deployment strategy**.

**Key Principle**: Azure does **NOT support in-place conversion** of public clusters to private clusters. This migration builds a new private cluster and moves workloads with minimal downtime.

---

## ⚠️ Prerequisites: Shell Environment

**IMPORTANT**: This SOP uses **bash shell syntax** for all commands. Ensure you are using:
- **Linux**: Native bash terminal
- **macOS**: Native Terminal or iTerm2 (bash/zsh)
- **Windows**: Use **WSL2 (Windows Subsystem for Linux)** or **Git Bash**

**Do NOT use PowerShell** - command syntax is incompatible.

---

## Migration Strategy: Blue/Green Approach

```
┌─────────────────────────────────────────────────────────────┐
│ BLUE ENVIRONMENT (Current)                                  │
│ ┌─────────────────────────────────────────────────────┐    │
│ │  Public AKS Cluster                                  │    │
│ │  - Public API endpoint                               │    │
│ │  - Production workloads (ACTIVE)                     │    │
│ │  - Current DNS points here                           │    │
│ └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ MIGRATION PROCESS
                          │ (Phases 1-6)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ GREEN ENVIRONMENT (New)                                      │
│ ┌─────────────────────────────────────────────────────┐    │
│ │  Private AKS Cluster                                 │    │
│ │  - Private API endpoint only                         │    │
│ │  - Migrated workloads (TESTING)                      │    │
│ │  - New DNS will point here after cutover             │    │
│ └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ CUTOVER (Phase 7)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ GREEN ENVIRONMENT (Active)                                   │
│ Production traffic now flows to private cluster              │
│ Blue environment kept for 48h rollback window                │
└─────────────────────────────────────────────────────────────┘
```

**Benefits**:
- ✅ Designed for data preservation - stateful resources exported/imported
- ✅ Minimal downtime - DNS/traffic cutover only
- ✅ Fast rollback - revert DNS if issues occur
- ✅ Parallel testing - validate green before cutover
- ✅ Safe decommission - keep blue for rollback period

---

## Migration Timeline

| Phase | Activity | Duration | Downtime |
|-------|----------|----------|----------|
| **Pre-Migration** | Assessment, planning, prerequisites | 2-4 hours | None |
| **Phase 1** | Export cluster configuration | 30 min | None |
| **Phase 2** | Export workload resources | 45 min | None |
| **Phase 3** | Build new private cluster | 90-105 min | None |
| **Phase 4** | Import workload resources | 60 min | None |
| **Phase 5** | Validation and testing | 2-4 hours | None |
| **Phase 6** | DNS/Traffic cutover | 15 min | **5-15 min** |
| **Phase 7** | Post-cutover validation | 30 min | None |
| **Phase 8** | Decommission old cluster | 15 min | None |

**Total Time**: 8-12 hours  
**Downtime Window**: 5-15 minutes (DNS propagation + health checks)

---

## Pre-Migration: Assessment and Planning

### Step 1: Inventory Current Cluster

**1.1 Connect to Existing Public Cluster**

```bash
# Set context to existing public cluster
export OLD_CLUSTER_NAME="aks-public-prod"
export OLD_RG="rg-aks-public-prod"
export OLD_SUBSCRIPTION="<your-subscription-id>"

az account set --subscription $OLD_SUBSCRIPTION
az aks get-credentials --resource-group $OLD_RG --name $OLD_CLUSTER_NAME --overwrite-existing

# Verify connection
kubectl cluster-info
kubectl get nodes
```

**1.2 Document Cluster Configuration**

```bash
# Get cluster details
az aks show --resource-group $OLD_RG --name $OLD_CLUSTER_NAME --output json > cluster-config-backup.json

# Document key settings
echo "=== CLUSTER CONFIGURATION ===" > migration-inventory.txt
echo "Kubernetes Version: $(kubectl version --short | grep Server)" >> migration-inventory.txt
echo "Node Pools: $(az aks nodepool list --cluster-name $OLD_CLUSTER_NAME --resource-group $OLD_RG --query '[].{name:name, vmSize:vmSize, count:count}' -o table)" >> migration-inventory.txt
echo "Network Plugin: $(az aks show -g $OLD_RG -n $OLD_CLUSTER_NAME --query 'networkProfile.networkPlugin' -o tsv)" >> migration-inventory.txt
echo "Network Policy: $(az aks show -g $OLD_RG -n $OLD_CLUSTER_NAME --query 'networkProfile.networkPolicy' -o tsv)" >> migration-inventory.txt
```

**1.3 Inventory Namespaces and Workloads**

```bash
# List all namespaces (exclude system namespaces)
kubectl get namespaces -o json | jq -r '.items[] | select(.metadata.name | test("^(default|kube-.*|gatekeeper-system)$") | not) | .metadata.name' > namespaces.txt

echo "Application Namespaces:" >> migration-inventory.txt
cat namespaces.txt >> migration-inventory.txt

# Count resources per namespace
echo -e "\n=== RESOURCE COUNTS ===" >> migration-inventory.txt
for ns in $(cat namespaces.txt); do
  echo "Namespace: $ns" >> migration-inventory.txt
  echo "  Deployments: $(kubectl get deployments -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  StatefulSets: $(kubectl get statefulsets -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  DaemonSets: $(kubectl get daemonsets -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  Services: $(kubectl get services -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  Ingresses: $(kubectl get ingresses -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  ConfigMaps: $(kubectl get configmaps -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  Secrets: $(kubectl get secrets -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
  echo "  PVCs: $(kubectl get pvc -n $ns --no-headers 2>/dev/null | wc -l)" >> migration-inventory.txt
done

# Review inventory
cat migration-inventory.txt
```

**1.4 Identify External Dependencies**

```bash
# List LoadBalancer services (will need DNS updates)
echo -e "\n=== EXTERNAL SERVICES ===" >> migration-inventory.txt
kubectl get services --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.type=="LoadBalancer") | "\(.metadata.namespace)/\(.metadata.name): \(.status.loadBalancer.ingress[0].ip // "pending")"' >> migration-inventory.txt

# List Ingress resources (will need DNS updates)
echo -e "\n=== INGRESS RESOURCES ===" >> migration-inventory.txt
kubectl get ingress --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.rules[].host)"' >> migration-inventory.txt

# List PersistentVolumeClaims (stateful data to migrate)
echo -e "\n=== PERSISTENT VOLUMES ===" >> migration-inventory.txt
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.volumeName) (\(.spec.resources.requests.storage))"' >> migration-inventory.txt
```

**1.5 Document Azure Integrations**

```bash
# Check for Pod Identity usage
echo -e "\n=== POD IDENTITY ===" >> migration-inventory.txt
kubectl get azureidentity --all-namespaces 2>/dev/null >> migration-inventory.txt || echo "No Pod Identity found" >> migration-inventory.txt

# Check for CSI driver usage
echo -e "\n=== CSI DRIVERS ===" >> migration-inventory.txt
kubectl get csidriver >> migration-inventory.txt

# Check for Azure Key Vault secrets
kubectl get secretproviderclass --all-namespaces 2>/dev/null >> migration-inventory.txt || echo "No SecretProviderClass found" >> migration-inventory.txt
```

**1.6 Review Inventory and Plan**

```bash
# Create migration directory structure
mkdir -p migration/{exports,imports,backups,scripts}
cp migration-inventory.txt migration/

# Review inventory file
cat migration/migration-inventory.txt
```

**⚠️ DECISION POINT**: Review `migration-inventory.txt`. Identify:
- Stateful workloads requiring data migration
- External DNS records requiring updates
- Services with static IPs requiring reconfiguration
- Azure integrations requiring new managed identities

---

### Step 2: Prerequisites Checklist

**Before proceeding, ensure**:

- [ ] **New private cluster is built** (use [Scenario 1](SOP-scenario-existing-hub.md) or [Scenario 2](SOP-scenario-standalone-hub.md) SOP)
- [ ] **kubectl access to new cluster** via Bastion tunnel
- [ ] **Backup of old cluster** completed (etcd backup if applicable)
- [ ] **ACR connectivity validated** from new private cluster
- [ ] **Container images replicated** to new ACR (Scenario A only)
- [ ] **Key Vault access validated** from new private cluster
- [ ] **Change window scheduled** (communicate downtime to stakeholders)
- [ ] **Rollback plan documented** (DNS revert, traffic reroute)
- [ ] **Monitoring enabled** on new cluster (Azure Monitor, Log Analytics)

---

## Phase 1: Export Cluster Configuration

### Step 1.1: Export Cluster-Level Resources

```bash
# Create export directory
mkdir -p migration/exports/cluster-resources

# Export namespaces (excluding system namespaces)
for ns in $(cat namespaces.txt); do
  kubectl get namespace $ns -o yaml > migration/exports/cluster-resources/namespace-$ns.yaml
done

# Export cluster roles and bindings (if custom)
kubectl get clusterroles -o yaml > migration/exports/cluster-resources/clusterroles.yaml
kubectl get clusterrolebindings -o yaml > migration/exports/cluster-resources/clusterrolebindings.yaml

# Export storage classes (if custom)
kubectl get storageclasses -o yaml > migration/exports/cluster-resources/storageclasses.yaml

# Export priority classes (if custom)
kubectl get priorityclasses -o yaml > migration/exports/cluster-resources/priorityclasses.yaml

# Export CRDs (Custom Resource Definitions)
kubectl get crds -o yaml > migration/exports/cluster-resources/crds.yaml
```

---

## Phase 2: Export Workload Resources

### Step 2.1: Export Namespace Resources

**2.1.1 Create Export Script**

```bash
cat > migration/scripts/export-namespace.sh <<'EOF'
#!/bin/bash
# Export all resources from a namespace

NAMESPACE=$1
EXPORT_DIR="migration/exports/namespaces/$NAMESPACE"

if [ -z "$NAMESPACE" ]; then
  echo "Usage: $0 <namespace>"
  exit 1
fi

echo "Exporting namespace: $NAMESPACE"
mkdir -p $EXPORT_DIR

# Export resource types
RESOURCE_TYPES=(
  "configmaps"
  "secrets"
  "services"
  "deployments"
  "statefulsets"
  "daemonsets"
  "replicasets"
  "jobs"
  "cronjobs"
  "ingresses"
  "persistentvolumeclaims"
  "serviceaccounts"
  "roles"
  "rolebindings"
  "networkpolicies"
  "horizontalpodautoscalers"
)

for resource in "${RESOURCE_TYPES[@]}"; do
  echo "  Exporting $resource..."
  kubectl get $resource -n $NAMESPACE -o yaml > $EXPORT_DIR/$resource.yaml 2>/dev/null
  
  # Check if file has content (more than just empty list)
  if [ $(wc -l < $EXPORT_DIR/$resource.yaml) -lt 5 ]; then
    rm $EXPORT_DIR/$resource.yaml
  fi
done

echo "Export complete: $EXPORT_DIR"
EOF

chmod +x migration/scripts/export-namespace.sh
```

**2.1.2 Export All Application Namespaces**

```bash
# Export each namespace
for ns in $(cat namespaces.txt); do
  echo "Exporting namespace: $ns"
  ./migration/scripts/export-namespace.sh $ns
done

# Verify exports
echo "=== EXPORT SUMMARY ==="
for ns in $(cat namespaces.txt); do
  echo "Namespace: $ns"
  ls -lh migration/exports/namespaces/$ns/ | tail -n +2
done
```

---

### Step 2.2: Export Secrets to Azure Key Vault (Recommended)

**Why**: Secrets in YAML are base64-encoded (not encrypted). Store in Key Vault for secure transfer.

```bash
# Set Key Vault name
export KEY_VAULT_NAME="kv-aks-migration"

# Extract secrets and store in Key Vault
for ns in $(cat namespaces.txt); do
  echo "Processing secrets in namespace: $ns"
  
  # Get all secret names (exclude service account tokens and helm secrets)
  SECRET_NAMES=$(kubectl get secrets -n $ns -o json | \
    jq -r '.items[] | select(.type != "kubernetes.io/service-account-token" and .type != "helm.sh/release.v1") | .metadata.name')
  
  for secret in $SECRET_NAMES; do
    echo "  Exporting secret: $secret"
    
    # Get secret data
    kubectl get secret $secret -n $ns -o json | \
      jq -r '.data | to_entries[] | "\(.key)=\(.value | @base64d)"' | \
      while IFS='=' read -r key value; do
        # Store in Key Vault with namespace prefix
        SECRET_KV_NAME="${ns}-${secret}-${key}"
        SECRET_KV_NAME=$(echo $SECRET_KV_NAME | tr '.' '-' | tr '_' '-')  # KV naming rules
        
        az keyvault secret set \
          --vault-name $KEY_VAULT_NAME \
          --name $SECRET_KV_NAME \
          --value "$value" \
          --tags namespace=$ns secret=$secret key=$key \
          --output none
        
        echo "    Stored: $SECRET_KV_NAME"
      done
  done
done

echo "All secrets backed up to Key Vault: $KEY_VAULT_NAME"
```

**Alternative**: If not using Key Vault, ensure exported YAML files are stored securely and encrypted at rest.

---

### Step 2.3: Backup Persistent Volume Data

**⚠️ CRITICAL**: For stateful workloads (databases, file storage), back up persistent volume data.

**Option A: Azure Disk Snapshot (Recommended)**

```bash
# List all PVCs and their backing Azure Disks
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace),\(.metadata.name),\(.spec.volumeName)"' | \
  while IFS=',' read -r ns pvc pv; do
    echo "Processing PVC: $ns/$pvc (PV: $pv)"
    
    # Get Azure Disk ID from PV
    DISK_ID=$(kubectl get pv $pv -o json | jq -r '.spec.azureDisk.diskURI // .spec.csi.volumeHandle')
    
    if [ ! -z "$DISK_ID" ] && [ "$DISK_ID" != "null" ]; then
      echo "  Azure Disk: $DISK_ID"
      
      # Create snapshot
      SNAPSHOT_NAME="snapshot-migration-$ns-$pvc-$(date +%Y%m%d-%H%M%S)"
      
      az snapshot create \
        --resource-group $OLD_RG \
        --name $SNAPSHOT_NAME \
        --source $DISK_ID \
        --tags "namespace=$ns" "pvc=$pvc" "migration=true"
      
      echo "  Snapshot created: $SNAPSHOT_NAME"
      echo "$ns,$pvc,$pv,$DISK_ID,$SNAPSHOT_NAME" >> migration/pvc-snapshots.csv
    else
      echo "  WARNING: Could not find Azure Disk ID for PV $pv"
    fi
  done

echo "Snapshot inventory saved to: migration/pvc-snapshots.csv"
cat migration/pvc-snapshots.csv
```

**Option B: Application-Level Backup (Databases)**

For databases (PostgreSQL, MongoDB, etc.), use native backup tools:

```bash
# Example: PostgreSQL backup
kubectl exec -n production postgres-0 -- pg_dump -U postgres mydb > migration/backups/mydb-$(date +%Y%m%d).sql

# Example: MongoDB backup
kubectl exec -n production mongo-0 -- mongodump --out /tmp/backup
kubectl cp production/mongo-0:/tmp/backup migration/backups/mongodb-$(date +%Y%m%d)
```

---

### Step 2.4: Document Current DNS Configuration

```bash
# Export DNS records for LoadBalancer services
echo "=== DNS RECORDS TO UPDATE ===" > migration/dns-records.txt

kubectl get services --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.type=="LoadBalancer") | "\(.metadata.namespace),\(.metadata.name),\(.status.loadBalancer.ingress[0].ip)"' | \
  while IFS=',' read -r ns svc ip; do
    echo "Service: $ns/$svc" >> migration/dns-records.txt
    echo "  Current IP: $ip" >> migration/dns-records.txt
    echo "  Action: Update DNS A record to new IP after migration" >> migration/dns-records.txt
    echo "" >> migration/dns-records.txt
  done

# Export Ingress hostnames
kubectl get ingress --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace),\(.metadata.name),\(.spec.rules[].host)"' | \
  while IFS=',' read -r ns ing host; do
    echo "Ingress: $ns/$ing" >> migration/dns-records.txt
    echo "  Hostname: $host" >> migration/dns-records.txt
    echo "  Action: Update DNS CNAME/A record to new ingress controller IP" >> migration/dns-records.txt
    echo "" >> migration/dns-records.txt
  done

cat migration/dns-records.txt
```

---

## Phase 3: Build New Private AKS Cluster

**⚠️ Before Building**: Ensure Azure policies from the **[Policy Guide](Azure-Policy-Guide.md)** are deployed at the management group or subscription level. This helps prevent accidental creation of non-compliant resources.

**Build your new private cluster**:

Follow the **[Provision Private AKS Cluster SOP](SOP-provision-private-aks-cluster.md)** to deploy the complete hub-spoke architecture with private AKS cluster.

**Key Configuration Matching**:

When building the new cluster using the [Provision SOP](SOP-provision-private-aks-cluster.md), ensure these match the old cluster:
- Kubernetes version (same major.minor version)
- Node pool VM sizes (same or larger)
- Network plugin (Azure CNI)
- Network policy (if enabled on old cluster)
- Monitoring add-ons (Container Insights)

**⏱️ Time Required**: 90-105 minutes (depending on scenario)

**After cluster is deployed**:

```bash
# Set environment variables for new cluster
export NEW_CLUSTER_NAME="aks-private-prod"
export NEW_RG="rg-aks-private-prod"
export NEW_SUBSCRIPTION="<your-subscription-id>"

# Get credentials (via Bastion tunnel)
az account set --subscription $NEW_SUBSCRIPTION
az aks get-credentials --resource-group $NEW_RG --name $NEW_CLUSTER_NAME --overwrite-existing

# Verify connection
kubectl cluster-info
kubectl get nodes
```

---

## Phase 3.5: Replicate Container Images to New ACR (Scenario A)

**⚠️ IMPORTANT**: If your old and new clusters use **different Azure Container Registries**, you must replicate images.

**Skip this step if**: Both clusters share the same ACR (Scenario B - shared services).

### Step 3.5.1: Identify Required Images

```bash
# Extract all unique image references from exported workloads
grep -h "image:" migration/exports/namespaces/*/*.yaml | \
  sed 's/.*image: *//g' | \
  sed 's/["'\'']*//g' | \
  sort -u > migration/required-images.txt

# Filter only images from old ACR (not public registries)
export OLD_ACR_NAME="acrpublicprod"
grep "${OLD_ACR_NAME}.azurecr.io" migration/required-images.txt > migration/acr-images-to-replicate.txt

echo "Images to replicate:"
cat migration/acr-images-to-replicate.txt
```

### Step 3.5.2: Replicate Images to New ACR

```bash
export NEW_ACR_NAME="acrprivateprod"

# Login to both ACRs
az acr login --name $OLD_ACR_NAME
az acr login --name $NEW_ACR_NAME

# Replicate each image
while IFS= read -r full_image; do
  echo "Replicating: $full_image"
  
  # Extract image name and tag
  image_with_tag=$(echo $full_image | sed "s/${OLD_ACR_NAME}.azurecr.io\///")
  
  # Use Azure CLI import (recommended - no local pull needed)
  az acr import \
    --name $NEW_ACR_NAME \
    --source $full_image \
    --image $image_with_tag \
    --resource-group $NEW_RG \
    --force
  
  echo "  ✓ Replicated to ${NEW_ACR_NAME}.azurecr.io/${image_with_tag}"
done < migration/acr-images-to-replicate.txt

echo "Image replication complete!"
```

### Step 3.5.3: Update Image References in Manifests

```bash
# Update all YAML files to point to new ACR
find migration/exports/namespaces -name "*.yaml" -type f -exec sed -i \
  "s/${OLD_ACR_NAME}.azurecr.io/${NEW_ACR_NAME}.azurecr.io/g" {} \;

echo "Updated image references from $OLD_ACR_NAME to $NEW_ACR_NAME"

# Verify changes
echo "Verifying updated images:"
grep -h "image:" migration/exports/namespaces/*/*.yaml | grep "azurecr.io" | head -10
```

**⏱️ Time Required**: 15-30 minutes (depending on image count and size)

---

## Phase 4: Import Workload Resources

### Step 4.1: Import Cluster-Level Resources

```bash
# Switch to new cluster context
kubectl config use-context $NEW_CLUSTER_NAME

# Import namespaces first
for ns in $(cat namespaces.txt); do
  echo "Creating namespace: $ns"
  kubectl apply -f migration/exports/cluster-resources/namespace-$ns.yaml
done

# Import CRDs (if any)
if [ -f migration/exports/cluster-resources/crds.yaml ]; then
  echo "Importing CRDs..."
  kubectl apply -f migration/exports/cluster-resources/crds.yaml
fi

# Import custom storage classes (if any)
if [ -f migration/exports/cluster-resources/storageclasses.yaml ]; then
  echo "Importing storage classes..."
  kubectl apply -f migration/exports/cluster-resources/storageclasses.yaml
fi

# Import cluster roles (review before applying - may conflict with AKS defaults)
# kubectl apply -f migration/exports/cluster-resources/clusterroles.yaml
# kubectl apply -f migration/exports/cluster-resources/clusterrolebindings.yaml
```

---

### Step 4.2: Clean Exported YAMLs (Remove Cluster-Specific Fields)

**Create cleanup script**:

```bash
cat > migration/scripts/clean-yaml.sh <<'EOF'
#!/bin/bash
# Remove cluster-specific fields from exported YAML

INPUT_FILE=$1
OUTPUT_FILE=$2

if [ -z "$INPUT_FILE" ] || [ -z "$OUTPUT_FILE" ]; then
  echo "Usage: $0 <input-file> <output-file>"
  exit 1
fi

# Use yq to remove cluster-specific fields
yq eval 'del(.items[].metadata.uid,
             .items[].metadata.resourceVersion,
             .items[].metadata.selfLink,
             .items[].metadata.creationTimestamp,
             .items[].metadata.generation,
             .items[].metadata.managedFields,
             .items[].status)' $INPUT_FILE > $OUTPUT_FILE

echo "Cleaned: $INPUT_FILE -> $OUTPUT_FILE"
EOF

chmod +x migration/scripts/clean-yaml.sh
```

**Clean all exported resources**:

```bash
# Install yq if not available
# Linux: wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq
# macOS: brew install yq
# Windows (WSL2): Same as Linux

# Clean namespace exports
for ns in $(cat namespaces.txt); do
  echo "Cleaning exports for namespace: $ns"
  mkdir -p migration/imports/namespaces/$ns
  
  for file in migration/exports/namespaces/$ns/*.yaml; do
    if [ -f "$file" ]; then
      filename=$(basename $file)
      ./migration/scripts/clean-yaml.sh $file migration/imports/namespaces/$ns/$filename
    fi
  done
done
```

---

### Step 4.3: Import Secrets from Key Vault

```bash
# Recreate secrets in new cluster from Key Vault
for ns in $(cat namespaces.txt); do
  echo "Restoring secrets for namespace: $ns"
  
  # List all secrets for this namespace in Key Vault
  az keyvault secret list --vault-name $KEY_VAULT_NAME --query "[?tags.namespace=='$ns'].name" -o tsv | \
    while read -r kv_secret_name; do
      # Get secret metadata
      SECRET_NAME=$(az keyvault secret show --vault-name $KEY_VAULT_NAME --name $kv_secret_name --query "tags.secret" -o tsv)
      KEY_NAME=$(az keyvault secret show --vault-name $KEY_VAULT_NAME --name $kv_secret_name --query "tags.key" -o tsv)
      SECRET_VALUE=$(az keyvault secret show --vault-name $KEY_VAULT_NAME --name $kv_secret_name --query "value" -o tsv)
      
      # Create or update Kubernetes secret
      kubectl create secret generic $SECRET_NAME \
        --from-literal=$KEY_NAME="$SECRET_VALUE" \
        --namespace $ns \
        --dry-run=client -o yaml | kubectl apply -f -
      
      echo "  Restored: $ns/$SECRET_NAME (key: $KEY_NAME)"
    done
done
```

**Alternative**: If secrets were exported to YAML (not Key Vault):

```bash
for ns in $(cat namespaces.txt); do
  if [ -f migration/imports/namespaces/$ns/secrets.yaml ]; then
    echo "Importing secrets for namespace: $ns"
    kubectl apply -f migration/imports/namespaces/$ns/secrets.yaml
  fi
done
```

---

### Step 4.4: Import ConfigMaps

```bash
for ns in $(cat namespaces.txt); do
  if [ -f migration/imports/namespaces/$ns/configmaps.yaml ]; then
    echo "Importing configmaps for namespace: $ns"
    kubectl apply -f migration/imports/namespaces/$ns/configmaps.yaml
  fi
done

# Verify
kubectl get configmaps --all-namespaces | grep -v "kube-"
```

---

### Step 4.5: Restore Persistent Volume Data

**Option A: Restore from Azure Disk Snapshots**

```bash
# For each PVC, create new disk from snapshot and PVC
cat migration/pvc-snapshots.csv | while IFS=',' read -r ns pvc pv disk_id snapshot_name; do
  echo "Restoring PVC: $ns/$pvc from snapshot $snapshot_name"
  
  # Get PVC size from old cluster
  PVC_SIZE=$(kubectl get pvc $pvc -n $ns -o jsonpath='{.spec.resources.requests.storage}')
  
  # Create disk from snapshot in new resource group
  NEW_DISK_NAME="disk-$ns-$pvc-migrated"
  
  az disk create \
    --resource-group $NEW_RG \
    --name $NEW_DISK_NAME \
    --source $(az snapshot show --name $snapshot_name --resource-group $OLD_RG --query id -o tsv) \
    --size-gb $(echo $PVC_SIZE | sed 's/Gi//')
  
  # Get new disk ID
  NEW_DISK_ID=$(az disk show --resource-group $NEW_RG --name $NEW_DISK_NAME --query id -o tsv)
  
  # Create PV pointing to new disk
  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: $NEW_DISK_NAME
spec:
  capacity:
    storage: $PVC_SIZE
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: disk.csi.azure.com
    volumeHandle: $NEW_DISK_ID
    volumeAttributes:
      fsType: ext4
EOF

  # Create PVC binding to new PV
  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: $pvc
  namespace: $ns
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: $PVC_SIZE
  volumeName: $NEW_DISK_NAME
EOF

  echo "  Restored: $ns/$pvc -> $NEW_DISK_NAME"
done
```

**Option B: Restore from Application Backups**

```bash
# Example: Restore PostgreSQL database
kubectl exec -n production postgres-0 -- psql -U postgres -d mydb < migration/backups/mydb-20251123.sql

# Example: Restore MongoDB
kubectl cp migration/backups/mongodb-20251123 production/mongo-0:/tmp/backup
kubectl exec -n production mongo-0 -- mongorestore /tmp/backup
```

---

### Step 4.6: Import Workload Resources (Deployments, StatefulSets, etc.)

**⚠️ IMPORTANT**: Import in correct order to avoid errors.

```bash
# Define import order (dependencies first)
RESOURCE_ORDER=(
  "serviceaccounts"
  "roles"
  "rolebindings"
  "persistentvolumeclaims"  # Already done in Step 4.5, but included for completeness
  "services"                 # Before deployments (pods need service endpoints)
  "deployments"
  "statefulsets"
  "daemonsets"
  "jobs"
  "cronjobs"
  "ingresses"
  "networkpolicies"
  "horizontalpodautoscalers"
)

# Import each resource type for all namespaces
for resource in "${RESOURCE_ORDER[@]}"; do
  echo "=== Importing $resource ==="
  
  for ns in $(cat namespaces.txt); do
    FILE="migration/imports/namespaces/$ns/$resource.yaml"
    
    if [ -f "$FILE" ]; then
      echo "  Namespace: $ns"
      kubectl apply -f $FILE
    fi
  done
  
  # Wait for resources to stabilize before moving to next type
  sleep 5
done
```

---

### Step 4.7: Verify Import Status

```bash
# Check all pods are running
echo "=== POD STATUS ==="
kubectl get pods --all-namespaces | grep -v "kube-system"

# Check for failed pods
FAILED_PODS=$(kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded -o json | jq -r '.items | length')

if [ $FAILED_PODS -gt 0 ]; then
  echo "WARNING: $FAILED_PODS pods are not running!"
  kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
  
  # Get details for failed pods
  kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded -o json | \
    jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"' | \
    while read -r pod; do
      echo "=== Logs for $pod ==="
      kubectl logs $pod --tail=50
    done
else
  echo "✅ All pods are running successfully"
fi

# Check services have endpoints
echo "=== SERVICES WITHOUT ENDPOINTS ==="
kubectl get endpoints --all-namespaces -o json | \
  jq -r '.items[] | select(.subsets == null or .subsets == []) | "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## Phase 5: Validation and Testing

### Step 5.1: Internal Connectivity Testing

**5.1.1 Test Pod-to-Pod Communication**

```bash
# Deploy test pod
kubectl run test-pod --image=busybox --restart=Never --command -- sleep 3600

# Test DNS resolution
kubectl exec test-pod -- nslookup kubernetes.default

# Test service connectivity (replace with your service)
kubectl exec test-pod -- wget -O- http://my-service.production.svc.cluster.local

# Cleanup
kubectl delete pod test-pod
```

**5.1.2 Test External Connectivity (Egress through Firewall)**

```bash
# Test internet connectivity (should go through hub firewall)
kubectl run curl-test --image=curlimages/curl --restart=Never --command -- sleep 3600
kubectl exec curl-test -- curl -I https://www.microsoft.com
kubectl exec curl-test -- curl -I https://mcr.microsoft.com

# Check egress IP (should be hub firewall public IP)
kubectl exec curl-test -- curl -s https://ifconfig.me

# Cleanup
kubectl delete pod curl-test
```

**5.1.3 Test ACR Connectivity**

```bash
# Test pulling image from private ACR
export ACR_NAME="acrprivateprod"

kubectl run acr-test \
  --image=$ACR_NAME.azurecr.io/nginx:latest \
  --restart=Never

# Check pod status (should pull image successfully)
kubectl get pod acr-test
kubectl describe pod acr-test

# Cleanup
kubectl delete pod acr-test
```

---

### Step 5.2: Application Health Checks

**5.2.1 Check Application Endpoints**

```bash
# Get LoadBalancer IPs for services
kubectl get services --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.type=="LoadBalancer") | "\(.metadata.namespace)/\(.metadata.name): \(.status.loadBalancer.ingress[0].ip)"'

# Test each service (replace with actual IPs)
# curl -I http://<LOADBALANCER-IP>

# Get Ingress IPs
kubectl get ingress --all-namespaces -o wide
```

**5.2.2 Run Application Smoke Tests**

```bash
# Example: Test API endpoints
# curl -X GET https://api.myapp.com/health
# curl -X GET https://api.myapp.com/version

# Example: Test database connectivity
kubectl exec -n production api-deployment-xxxxx -- nc -zv postgres-service 5432
```

**5.2.3 Monitor Application Logs**

```bash
# Tail logs for main application pods
for ns in $(cat namespaces.txt); do
  echo "=== Logs for namespace: $ns ==="
  kubectl logs -n $ns --tail=20 -l app=main-app  # Adjust label selector
done
```

---

### Step 5.3: Performance Testing (Optional but Recommended)

```bash
# Run load test against new cluster (using same tool as production)
# Example with Apache Bench
# ab -n 1000 -c 10 http://<NEW-LOADBALANCER-IP>/

# Compare response times and error rates with old cluster baseline
```

---

### Step 5.4: Validation Checklist

**Before proceeding to cutover, verify**:

- [ ] **All pods running**: No CrashLoopBackOff or ImagePullBackOff errors
- [ ] **Services have endpoints**: All services are backed by healthy pods
- [ ] **Ingress controller running**: Nginx/App Gateway ingress is healthy
- [ ] **ACR connectivity works**: Pods can pull images from private ACR
- [ ] **Key Vault access works**: Pods can read secrets via CSI driver
- [ ] **Database connectivity**: Applications can connect to databases
- [ ] **External egress works**: Pods can reach internet through firewall
- [ ] **DNS resolution works**: Pods can resolve external and internal DNS
- [ ] **Persistent data restored**: Stateful apps have correct data
- [ ] **Application smoke tests pass**: Critical user journeys work
- [ ] **Monitoring operational**: Azure Monitor showing metrics and logs
- [ ] **No Azure policy violations**: Compliance state is healthy

---

## Phase 6: DNS and Traffic Cutover

**⚠️ THIS IS THE DOWNTIME WINDOW** (5-15 minutes)

### Step 6.1: Pre-Cutover Communication

```bash
# 1 hour before cutover
# - Send notification to all stakeholders
# - Put maintenance banner on application (if applicable)
# - Stop non-critical background jobs on old cluster

# 15 minutes before cutover
# - Final validation of new cluster health
# - Confirm rollback plan is ready
# - Have monitoring dashboards open for both clusters
```

---

### Step 6.2: Update DNS Records

**6.2.1 Get New LoadBalancer IPs**

```bash
# Get new LoadBalancer IPs
echo "=== NEW LOADBALANCER IPS ===" > migration/new-ips.txt

kubectl get services --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.type=="LoadBalancer") | "\(.metadata.namespace),\(.metadata.name),\(.status.loadBalancer.ingress[0].ip)"' | \
  while IFS=',' read -r ns svc ip; do
    echo "Service: $ns/$svc -> $ip" >> migration/new-ips.txt
  done

cat migration/new-ips.txt
```

**6.2.2 Update DNS A Records**

```bash
# Option A: Azure DNS Zone (automated)
export DNS_ZONE="myapp.com"
export DNS_RG="rg-dns"

# Example: Update A record
OLD_IP="20.50.100.10"
NEW_IP="10.100.2.50"  # From new-ips.txt

az network dns record-set a delete \
  --resource-group $DNS_RG \
  --zone-name $DNS_ZONE \
  --name api \
  --yes

az network dns record-set a add-record \
  --resource-group $DNS_RG \
  --zone-name $DNS_ZONE \
  --record-set-name api \
  --ipv4-address $NEW_IP

# Set low TTL for fast propagation (if not already low)
az network dns record-set a update \
  --resource-group $DNS_RG \
  --zone-name $DNS_ZONE \
  --name api \
  --set ttl=60

# Option B: External DNS provider (manual)
# Update A records in your DNS provider console (GoDaddy, Cloudflare, etc.)
```

**6.2.3 Wait for DNS Propagation**

```bash
# Check DNS propagation from multiple locations
dig api.myapp.com +short

# Use online DNS checker: https://dnschecker.org/
# Verify new IP is returned globally

# Expected propagation time: 1-5 minutes (with low TTL)
```

---

### Step 6.3: Monitor Traffic Shift

**6.3.1 Watch Connection Counts**

```bash
# Monitor connections to old cluster (should drop to zero)
# kubectl top pods -n production --sort-by=cpu

# Monitor connections to new cluster (should increase)
# kubectl top pods -n production --sort-by=cpu

# Monitor Azure Load Balancer metrics (new cluster)
az monitor metrics list \
  --resource $(az network lb list --resource-group $NEW_RG --query "[0].id" -o tsv) \
  --metric "PacketCount" \
  --interval PT1M
```

**6.3.2 Check Application Logs**

```bash
# Tail logs for main application pods on new cluster
kubectl logs -n production -l app=main-app --tail=50 -f

# Watch for errors or warnings during cutover
```

---

### Step 6.4: Validate Cutover Success

```bash
# Test from external client (your laptop)
curl -I https://api.myapp.com

# Verify response is coming from new cluster
# Check response headers, application version, etc.

# Run critical user journeys
# Example: Login, browse products, place order, etc.

# Check monitoring dashboard
# - Response times within SLA
# - Error rates < 1%
# - No 5xx errors
```

---

## Phase 7: Post-Cutover Validation

### Step 7.1: Monitor for 2 Hours

```bash
# Continuously monitor for first 2 hours after cutover
watch -n 30 "kubectl get pods --all-namespaces | grep -v 'Running\|Completed'"

# Check error logs
kubectl logs -n production -l app=main-app --since=2h | grep -i error

# Monitor Azure Monitor metrics
# - CPU/Memory usage
# - Request rates
# - Error rates
# - Response times
```

---

### Step 7.2: User Acceptance Testing

```bash
# Have key stakeholders test critical workflows
# - Login/Authentication
# - Core business transactions
# - Report generation
# - Data integrity checks

# Document any issues found
```

---

### Step 7.3: Performance Comparison

```bash
# Compare metrics: Old cluster (pre-migration) vs New cluster (post-migration)

# Metrics to compare:
# - Average response time
# - P95/P99 latency
# - Error rate
# - Throughput (requests/sec)
# - Resource utilization (CPU/Memory)

# Expected: New cluster should perform equal or better
```

---

## Phase 8: Decommission Old Cluster

**⚠️ WAIT 48 HOURS** before decommissioning old cluster (rollback window)

### Step 8.1: Scale Down Old Cluster (Keep for Rollback)

```bash
# Switch to old cluster context
kubectl config use-context $OLD_CLUSTER_NAME

# Scale down all deployments (keep cluster running for rollback)
for ns in $(cat namespaces.txt); do
  echo "Scaling down deployments in namespace: $ns"
  kubectl scale deployment --all --replicas=0 -n $ns
done

# Verify no pods running
kubectl get pods --all-namespaces | grep -v "kube-system"
```

---

### Step 8.2: Delete Old Cluster (After 48h)

```bash
# Confirm migration is stable for 48 hours
# Confirm no rollback is needed
# Get approval from stakeholders

# Delete old AKS cluster
az aks delete \
  --resource-group $OLD_RG \
  --name $OLD_CLUSTER_NAME \
  --yes \
  --no-wait

# Clean up associated resources (if no longer needed)
# - LoadBalancers
# - Public IPs
# - Disks (if not snapshotted)
# - Network resources (if dedicated to old cluster)

# Keep snapshots for 30 days (compliance/disaster recovery)
```

---

## Rollback Procedure

**If issues occur during/after cutover, rollback immediately**:

### Rollback Step 1: Revert DNS

```bash
# Update DNS A records back to old cluster IPs
az network dns record-set a delete \
  --resource-group $DNS_RG \
  --zone-name $DNS_ZONE \
  --name api \
  --yes

az network dns record-set a add-record \
  --resource-group $DNS_RG \
  --zone-name $DNS_ZONE \
  --record-set-name api \
  --ipv4-address $OLD_IP  # Original IP from migration/dns-records.txt

# Wait for DNS propagation (1-5 minutes)
```

### Rollback Step 2: Scale Up Old Cluster

```bash
# If scaled down, scale back up
kubectl config use-context $OLD_CLUSTER_NAME

for ns in $(cat namespaces.txt); do
  # Scale deployments back to original replica count
  # (You should have documented original counts in migration-inventory.txt)
  kubectl scale deployment --all --replicas=3 -n $ns  # Adjust replica count
done

# Wait for pods to be ready
kubectl get pods --all-namespaces --watch
```

### Rollback Step 3: Validate

```bash
# Test from external client
curl -I https://api.myapp.com

# Verify traffic back to old cluster
# Monitor old cluster metrics
```

### Rollback Step 4: Post-Mortem

```bash
# Document what went wrong
# Analyze logs from new cluster
# Fix issues before attempting migration again
```

---

## Troubleshooting

### Issue: Pods in ImagePullBackOff

**Cause**: Cannot pull images from private ACR

**Resolution**:
```bash
# Verify ACR private endpoint is working
kubectl run test-acr --image=busybox --restart=Never -- nslookup $ACR_NAME.azurecr.io
kubectl logs test-acr

# Should resolve to private IP (10.x.x.x)

# Check managed identity has AcrPull role
az role assignment list --assignee $(az aks show -g $NEW_RG -n $NEW_CLUSTER_NAME --query identityProfile.kubeletidentity.clientId -o tsv) --scope $(az acr show -n $ACR_NAME --query id -o tsv)

# If missing, assign role
az role assignment create \
  --assignee $(az aks show -g $NEW_RG -n $NEW_CLUSTER_NAME --query identityProfile.kubeletidentity.clientId -o tsv) \
  --role AcrPull \
  --scope $(az acr show -n $ACR_NAME --query id -o tsv)
```

---

### Issue: Pods Cannot Reach Database

**Cause**: Network connectivity or DNS resolution issue

**Resolution**:
```bash
# Test DNS resolution
kubectl run test-dns --image=busybox --restart=Never -- nslookup postgres-service.production.svc.cluster.local

# Test network connectivity
kubectl run test-net --image=busybox --restart=Never -- nc -zv postgres-service.production 5432

# Check service has endpoints
kubectl get endpoints postgres-service -n production

# Check NSG rules allow traffic
# Check UDR not blocking internal traffic
```

---

### Issue: Application Shows Old Data

**Cause**: Persistent volume data not restored correctly

**Resolution**:
```bash
# Verify PVC is bound to correct PV
kubectl get pvc -n production
kubectl describe pvc my-app-data -n production

# Check PV points to correct Azure Disk (restored from snapshot)
kubectl describe pv pv-my-app-data

# If incorrect, delete PVC/PV and recreate from snapshot (see Phase 4.5)
```

---

### Issue: High Response Times After Migration

**Cause**: Resource constraints or networking overhead

**Resolution**:
```bash
# Check resource utilization
kubectl top nodes
kubectl top pods -n production

# Scale up if needed
kubectl scale deployment my-app --replicas=5 -n production

# Check network latency (private endpoints add ~2-5ms)
# This is expected for private cluster architecture

# Review Application Gateway / Ingress configuration
kubectl describe ingress my-app -n production
```

---

## Migration Checklist

### Pre-Migration
- [ ] Inventory completed (all namespaces, workloads, data)
- [ ] New private cluster built and validated
- [ ] Backup of old cluster completed (etcd, PVs, configs)
- [ ] Change window scheduled and communicated
- [ ] Rollback plan documented and tested
- [ ] Monitoring enabled on new cluster

### Phase 1: Export
- [ ] Cluster-level resources exported
- [ ] Namespace resources exported
- [ ] Secrets backed up to Key Vault
- [ ] Persistent volume snapshots created
- [ ] DNS records documented

### Phase 2: Build
- [ ] New private cluster deployed (Scenario 1 or 2)
- [ ] kubectl access via Bastion verified
- [ ] ACR connectivity validated
- [ ] Key Vault access validated

### Phase 3: Image Replication (Scenario A Only)
- [ ] Required images identified from manifests
- [ ] Images replicated from old ACR to new ACR
- [ ] Image references updated in manifests
- [ ] Image pull tests successful in new cluster

### Phase 4: Import
- [ ] Namespaces created
- [ ] Secrets restored
- [ ] ConfigMaps imported
- [ ] Persistent volumes restored
- [ ] Workload resources imported
- [ ] All pods running

### Phase 4: Validation
- [ ] Pod-to-pod connectivity verified
- [ ] External egress verified (through firewall)
- [ ] ACR image pull verified
- [ ] Database connectivity verified
- [ ] Application smoke tests passed
- [ ] No policy violations

### Phase 5: Cutover
- [ ] DNS records updated
- [ ] DNS propagation verified
- [ ] Traffic shifted to new cluster
- [ ] Application endpoints responding
- [ ] No errors in logs

### Phase 6: Post-Cutover
- [ ] 2-hour monitoring period completed
- [ ] User acceptance testing passed
- [ ] Performance metrics validated
- [ ] Stakeholder sign-off received

### Phase 7: Decommission
- [ ] 48-hour stability period completed
- [ ] Old cluster scaled down
- [ ] Approval to decommission received
- [ ] Old cluster deleted
- [ ] Resources cleaned up
- [ ] Documentation updated

---

## Support

**Questions about migration?**
- Kubernetes Migration Guide: https://kubernetes.io/docs/tasks/administer-cluster/migrate-to-new-cluster/
- Microsoft AKS Support: https://learn.microsoft.com/azure/aks/

---

**Last Updated**: November 23, 2025  
**Next Review**: February 23, 2026
