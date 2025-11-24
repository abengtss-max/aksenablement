# SOP: Provision Private AKS Cluster with Hub-Spoke Architecture

**Document Version:** 1.0  
**Last Updated:** November 23, 2025  

---

## Table of Contents

- [SOP: Provision Private AKS Cluster with Hub-Spoke Architecture](#sop-provision-private-aks-cluster-with-hub-spoke-architecture)
  - [Table of Contents](#table-of-contents)
  - [1. Overview](#1-overview)
    - [1.1 Purpose](#11-purpose)
    - [1.2 Scope](#12-scope)
    - [1.3 Architecture Overview](#13-architecture-overview)
  - [2. Prerequisites](#2-prerequisites)
    - [2.1 Required Permissions](#21-required-permissions)
    - [2.2 Required Tools](#22-required-tools)
    - [2.3 Network Planning](#23-network-planning)
  - [3. Hub Infrastructure Deployment](#3-hub-infrastructure-deployment)
    - [3.1 Prepare Environment Variables](#31-prepare-environment-variables)
    - [3.2 Create Hub Resource Group](#32-create-hub-resource-group)
    - [3.3 Create Hub Network Security Groups](#33-create-hub-network-security-groups)
    - [3.4 Create Hub Virtual Network](#34-create-hub-virtual-network)
    - [3.5 Deploy Azure Bastion](#35-deploy-azure-bastion)
    - [3.6 Deploy Azure Firewall](#36-deploy-azure-firewall)
    - [3.7 Configure Firewall Rules](#37-configure-firewall-rules)
  - [4. Spoke Infrastructure Deployment](#4-spoke-infrastructure-deployment)
    - [4.1 Create Spoke Resource Group](#41-create-spoke-resource-group)
    - [4.2 Create Spoke Network Security Groups](#42-create-spoke-network-security-groups)
    - [4.3 Create Spoke Virtual Network](#43-create-spoke-virtual-network)
    - [4.4 Create VNet Peering](#44-create-vnet-peering)
    - [4.5 Configure User-Defined Routing](#45-configure-user-defined-routing)
  - [5. Private AKS Cluster Deployment](#5-private-aks-cluster-deployment)
    - [5.1 Create Managed Identity](#51-create-managed-identity)
    - [5.2 Assign RBAC Permissions](#52-assign-rbac-permissions)
    - [5.3 Deploy Private AKS Cluster](#53-deploy-private-aks-cluster)
    - [5.4 Add User Node Pool](#54-add-user-node-pool)
    - [5.5 Configure DNS for Private Cluster](#55-configure-dns-for-private-cluster)
  - [6. Azure Container Registry Deployment](#6-azure-container-registry-deployment)
    - [6.1 Create Private ACR](#61-create-private-acr)
    - [6.2 Configure Private Endpoint for ACR](#62-configure-private-endpoint-for-acr)
    - [6.3 Configure DNS for ACR](#63-configure-dns-for-acr)
    - [6.4 Attach ACR to AKS](#64-attach-acr-to-aks)
  - [7. Connect to Private AKS](#7-connect-to-private-aks)
    - [7.1 Access Methods Overview](#71-access-methods-overview)
    - [7.2 Option A: Using az aks command invoke (No VM Required)](#72-option-a-using-az-aks-command-invoke-no-vm-required)
    - [7.3 Option B: Using Azure Bastion with Temporary VM](#73-option-b-using-azure-bastion-with-temporary-vm)
    - [7.4 Recommendation](#74-recommendation)
  - [8. Validation and Testing](#8-validation-and-testing)
    - [8.1 Validate Hub Infrastructure](#81-validate-hub-infrastructure)
    - [8.2 Validate Spoke Infrastructure](#82-validate-spoke-infrastructure)
    - [8.3 Validate AKS Cluster](#83-validate-aks-cluster)
    - [8.4 Validate ACR Integration](#84-validate-acr-integration)
    - [8.5 Test Private Connectivity](#85-test-private-connectivity)
  - [9. Troubleshooting](#9-troubleshooting)
    - [9.1 Common Issues](#91-common-issues)
    - [9.2 Diagnostic Commands](#92-diagnostic-commands)
  - [10. Cleanup Procedures](#10-cleanup-procedures)

---

## Related Documentation

This deployment guide is part of the private AKS documentation set:
- **[Private AKS Guide](private-aks-cluster-guide.md)** - Documentation overview and quick start
- **[Azure Policy Guide](Azure-Policy-Guide.md)** - Required governance policies (deploy first)
- **[Workload Migration](SOP-workload-migration.md)** - Migrate workloads from public to private AKS

> **👉 Deployment Order**:
> 1. Deploy Azure policies: **[Policy Guide](Azure-Policy-Guide.md)**
> 2. Follow this guide to provision private AKS cluster
> 3. If migrating workloads: **[Migration SOP](SOP-workload-migration.md)**

---

## 1. Overview

### 1.1 Purpose

This Standard Operating Procedure (SOP) provides step-by-step instructions for deploying an enterprise-grade private Azure Kubernetes Service (AKS) cluster using a hub-spoke network architecture with Azure Firewall for egress traffic control.

### 1.2 Scope

This document covers:
- Hub-spoke network topology with VNet peering
- Private AKS cluster with private API endpoint only
- Azure Firewall for egress traffic inspection
- Private Azure Container Registry with private endpoints
- Azure Bastion for secure remote access (no jumpbox VMs required)
- Private DNS zones for name resolution

### 1.3 Architecture Overview

**Key Components:**
- **Hub VNet**: Contains Azure Firewall and Azure Bastion
- **Spoke VNet**: Contains private AKS cluster, ACR private endpoints, and load balancers
- **Network Security Groups**: Control traffic between subnets
- **User-Defined Routes**: Force tunnel all egress traffic through Azure Firewall
- **Private DNS Zones**: Enable name resolution for private endpoints

---

## 2. Prerequisites

### 2.1 Required Permissions

Before starting, verify you have the following Azure RBAC roles:
- **Contributor** role on the subscription
- **User Access Administrator** role on the subscription (for identity assignments)

**Verify permissions:**

```bash
# Get your user ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Check role assignments
az role assignment list \
    --assignee $USER_ID \
    --scope /subscriptions/<SUBSCRIPTION_ID> \
    --query "[].{role:roleDefinitionName, scope:scope}" \
    -o table
```

### 2.2 Required Tools

Ensure the following tools are installed:
- Azure CLI (version 2.50.0 or later)
- kubectl (version 1.27 or later)
- jq (for JSON parsing)

**Verify installations:**

```bash
az version
kubectl version --client
jq --version
```

### 2.3 Network Planning

**Recommended IP Address Ranges:**

| Component | Address Range | Purpose |
|-----------|--------------|---------|
| Hub VNet | 10.0.0.0/22 | Hub network |
| Firewall Subnet | 10.0.0.0/26 | Azure Firewall |
| Bastion Subnet | 10.0.1.0/26 | Azure Bastion |
| Management Subnet | 10.0.2.0/24 | Temporary VMs |
| Spoke VNet | 10.1.0.0/20 | Spoke network |
| AKS Subnet | 10.1.0.0/23 | AKS nodes |
| Private Endpoints | 10.1.2.0/28 | ACR, Key Vault |
| Load Balancer | 10.1.3.0/28 | Internal LB |

---

## 3. Hub Infrastructure Deployment

### 3.1 Prepare Environment Variables

Set environment variables for resource names and locations:

```bash
# Resource naming
HUB_RG="rg-hub"
SPOKE_RG="rg-spoke"
LOCATION="swedencentral"
PROJECT_PREFIX="<your-org-prefix>"

# Network address spaces
HUB_VNET_PREFIX="10.0.0.0/22"
HUB_FIREWALL_SUBNET_PREFIX="10.0.0.0/26"
HUB_BASTION_SUBNET_PREFIX="10.0.1.0/26"
HUB_MGMT_SUBNET_PREFIX="10.0.2.0/24"

SPOKE_VNET_PREFIX="10.1.0.0/20"
AKS_SUBNET_PREFIX="10.1.0.0/23"
PRIVATE_LINK_SUBNET_PREFIX="10.1.2.0/28"
LOADBALANCER_SUBNET_PREFIX="10.1.3.0/28"

# Resource names
HUB_VNET_NAME="vnet-${PROJECT_PREFIX}-hub"
SPOKE_VNET_NAME="vnet-${PROJECT_PREFIX}-spoke"
FIREWALL_NAME="fw-${PROJECT_PREFIX}-hub"
BASTION_NAME="bastion-${PROJECT_PREFIX}-hub"
AKS_CLUSTER_NAME="aks-${PROJECT_PREFIX}-private"
ACR_NAME="acr${PROJECT_PREFIX}$(date +%s | tail -c 5)"  # Ensure global uniqueness
AKS_IDENTITY_NAME="id-${PROJECT_PREFIX}-aks"
ROUTE_TABLE_NAME="rt-${PROJECT_PREFIX}-spoke"

# NSG names
BASTION_NSG_NAME="nsg-bastion"
MGMT_NSG_NAME="nsg-management"
AKS_NSG_NAME="nsg-aks"
ENDPOINTS_NSG_NAME="nsg-endpoints"
LOADBALANCER_NSG_NAME="nsg-loadbalancer"

# Subnet names
FW_SUBNET_NAME="AzureFirewallSubnet"
BASTION_SUBNET_NAME="AzureBastionSubnet"
MGMT_SUBNET_NAME="snet-management"
AKS_SUBNET_NAME="snet-aks-nodes"
ENDPOINTS_SUBNET_NAME="snet-private-endpoints"
LOADBALANCER_SUBNET_NAME="snet-loadbalancer"
```

### 3.2 Create Hub Resource Group

```bash
az group create \
    --name $HUB_RG \
    --location $LOCATION
```

**Expected Output:**
```json
{
  "id": "/subscriptions/.../resourceGroups/rg-hub",
  "location": "swedencentral",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

### 3.3 Create Hub Network Security Groups

#### 3.3.1 Create Bastion NSG

```bash
az network nsg create \
    --resource-group $HUB_RG \
    --name $BASTION_NSG_NAME \
    --location $LOCATION
```

#### 3.3.2 Configure Bastion Inbound Rules

Azure Bastion requires specific inbound rules. For details, see: [Azure Bastion NSG requirements](https://learn.microsoft.com/en-us/azure/bastion/bastion-nsg)

```bash
# Allow HTTPS from Internet
az network nsg rule create \
    --name AllowHttpsInbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 120 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol TCP \
    --direction Inbound \
    --source-address-prefixes "Internet" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges "443"

# Allow Gateway Manager
az network nsg rule create \
    --name AllowGatewayManagerInbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 130 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol TCP \
    --direction Inbound \
    --source-address-prefixes "GatewayManager" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges "443"

# Allow Azure Load Balancer
az network nsg rule create \
    --name AllowAzureLoadBalancerInbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 140 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol TCP \
    --direction Inbound \
    --source-address-prefixes "AzureLoadBalancer" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges "443"

# Allow Bastion Host Communication
az network nsg rule create \
    --name AllowBastionHostCommunication \
    --nsg-name $BASTION_NSG_NAME \
    --priority 150 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol "*" \
    --direction Inbound \
    --source-address-prefixes "VirtualNetwork" \
    --source-port-ranges "*" \
    --destination-address-prefixes "VirtualNetwork" \
    --destination-port-ranges 8080 5701
```

#### 3.3.3 Configure Bastion Outbound Rules

```bash
# Allow SSH/RDP to VNet
az network nsg rule create \
    --name AllowSshRdpOutbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 100 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol "*" \
    --direction Outbound \
    --source-address-prefixes "*" \
    --source-port-ranges "*" \
    --destination-address-prefixes "VirtualNetwork" \
    --destination-port-ranges 22 3389

# Allow Azure Cloud
az network nsg rule create \
    --name AllowAzureCloudOutbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 110 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol TCP \
    --direction Outbound \
    --source-address-prefixes "*" \
    --source-port-ranges "*" \
    --destination-address-prefixes "AzureCloud" \
    --destination-port-ranges 443

# Allow Bastion Communication
az network nsg rule create \
    --name AllowBastionCommunication \
    --nsg-name $BASTION_NSG_NAME \
    --priority 120 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol "*" \
    --direction Outbound \
    --source-address-prefixes "VirtualNetwork" \
    --source-port-ranges "*" \
    --destination-address-prefixes "VirtualNetwork" \
    --destination-port-ranges 8080 5701

# Allow HTTP to Internet
az network nsg rule create \
    --name AllowHttpOutbound \
    --nsg-name $BASTION_NSG_NAME \
    --priority 130 \
    --resource-group $HUB_RG \
    --access Allow \
    --protocol "*" \
    --direction Outbound \
    --source-address-prefixes "*" \
    --source-port-ranges "*" \
    --destination-address-prefixes "Internet" \
    --destination-port-ranges 80
```

#### 3.3.4 Create Management Subnet NSG

```bash
az network nsg create \
    --resource-group $HUB_RG \
    --name $MGMT_NSG_NAME \
    --location $LOCATION
```

### 3.4 Create Hub Virtual Network

#### 3.4.1 Create Hub VNet with Bastion Subnet

```bash
az network vnet create \
    --resource-group $HUB_RG \
    --name $HUB_VNET_NAME \
    --address-prefixes $HUB_VNET_PREFIX \
    --subnet-name $BASTION_SUBNET_NAME \
    --subnet-prefixes $HUB_BASTION_SUBNET_PREFIX \
    --network-security-group $BASTION_NSG_NAME
```

#### 3.4.2 Create Firewall Subnet

> **Note:** Azure Firewall requires a dedicated subnet named `AzureFirewallSubnet` with minimum size /26.

```bash
az network vnet subnet create \
    --resource-group $HUB_RG \
    --vnet-name $HUB_VNET_NAME \
    --name $FW_SUBNET_NAME \
    --address-prefixes $HUB_FIREWALL_SUBNET_PREFIX
```

#### 3.4.3 Create Management Subnet

```bash
az network vnet subnet create \
    --resource-group $HUB_RG \
    --vnet-name $HUB_VNET_NAME \
    --name $MGMT_SUBNET_NAME \
    --address-prefixes $HUB_MGMT_SUBNET_PREFIX \
    --network-security-group $MGMT_NSG_NAME
```

### 3.5 Deploy Azure Bastion

#### 3.5.1 Create Public IP for Bastion

```bash
az network public-ip create \
    --resource-group $HUB_RG \
    --name pip-bastion \
    --sku Standard \
    --allocation-method Static \
    --location $LOCATION
```

#### 3.5.2 Deploy Azure Bastion (Standard SKU)

> **Note:** Standard SKU enables features like native client support and IP-based connections required for tunneling.

```bash
az network bastion create \
    --resource-group $HUB_RG \
    --name $BASTION_NAME \
    --public-ip-address pip-bastion \
    --vnet-name $HUB_VNET_NAME \
    --location $LOCATION \
    --sku Standard \
    --enable-tunneling true
```

> **Note:** Bastion deployment takes approximately 10-15 minutes.

### 3.6 Deploy Azure Firewall

#### 3.6.1 Create Azure Firewall (Premium SKU)

```bash
az network firewall create \
    --resource-group $HUB_RG \
    --name $FIREWALL_NAME \
    --location $LOCATION \
    --vnet-name $HUB_VNET_NAME \
    --enable-dns-proxy true
```

#### 3.6.2 Create Public IP for Firewall

```bash
az network public-ip create \
    --name pip-firewall \
    --resource-group $HUB_RG \
    --location $LOCATION \
    --allocation-method Static \
    --sku Standard
```

#### 3.6.3 Associate Public IP with Firewall

```bash
az network firewall ip-config create \
    --firewall-name $FIREWALL_NAME \
    --name FW-config \
    --public-ip-address pip-firewall \
    --resource-group $HUB_RG \
    --vnet-name $HUB_VNET_NAME
```

#### 3.6.4 Update Firewall Configuration

```bash
az network firewall update \
    --name $FIREWALL_NAME \
    --resource-group $HUB_RG
```

#### 3.6.5 Get Firewall Private IP

```bash
FIREWALL_PRIVATE_IP=$(az network firewall show \
    --resource-group $HUB_RG \
    --name $FIREWALL_NAME \
    --query "ipConfigurations[0].privateIPAddress" \
    -o tsv)

echo "Firewall Private IP: $FIREWALL_PRIVATE_IP"
```

### 3.7 Configure Firewall Rules

#### 3.7.1 Create Network Rules for AKS

> **Reference:** [AKS egress requirements](https://learn.microsoft.com/en-us/azure/aks/limit-egress-traffic)

```bash
# Allow AKS API UDP traffic
az network firewall network-rule create \
    --resource-group $HUB_RG \
    --firewall-name $FIREWALL_NAME \
    --collection-name aks-required \
    --name apiudp \
    --protocols UDP \
    --source-addresses "*" \
    --destination-addresses "AzureCloud.$LOCATION" \
    --destination-ports 1194 \
    --action Allow \
    --priority 100

# Allow AKS API TCP traffic
az network firewall network-rule create \
    --resource-group $HUB_RG \
    --firewall-name $FIREWALL_NAME \
    --collection-name aks-required \
    --name apitcp \
    --protocols TCP \
    --source-addresses "*" \
    --destination-addresses "AzureCloud.$LOCATION" \
    --destination-ports 9000

# Allow NTP
az network firewall network-rule create \
    --resource-group $HUB_RG \
    --firewall-name $FIREWALL_NAME \
    --collection-name aks-required \
    --name time \
    --protocols UDP \
    --source-addresses "*" \
    --destination-fqdns ntp.ubuntu.com \
    --destination-ports 123
```

#### 3.7.2 Create Application Rules for AKS

```bash
az network firewall application-rule create \
    --resource-group $HUB_RG \
    --firewall-name $FIREWALL_NAME \
    --collection-name aks-fqdns \
    --name allow-aks-fqdns \
    --source-addresses "*" \
    --protocols "http=80" "https=443" \
    --fqdn-tags "AzureKubernetesService" \
    --action Allow \
    --priority 100
```

---

## 4. Spoke Infrastructure Deployment

### 4.1 Create Spoke Resource Group

```bash
az group create \
    --name $SPOKE_RG \
    --location $LOCATION
```

### 4.2 Create Spoke Network Security Groups

#### 4.2.1 Create AKS NSG

```bash
az network nsg create \
    --resource-group $SPOKE_RG \
    --name $AKS_NSG_NAME \
    --location $LOCATION
```

#### 4.2.2 Create Private Endpoints NSG

```bash
az network nsg create \
    --resource-group $SPOKE_RG \
    --name $ENDPOINTS_NSG_NAME \
    --location $LOCATION
```

#### 4.2.3 Create Load Balancer NSG

```bash
az network nsg create \
    --resource-group $SPOKE_RG \
    --name $LOADBALANCER_NSG_NAME \
    --location $LOCATION
```

### 4.3 Create Spoke Virtual Network

#### 4.3.1 Create Spoke VNet with AKS Subnet

```bash
az network vnet create \
    --resource-group $SPOKE_RG \
    --name $SPOKE_VNET_NAME \
    --address-prefixes $SPOKE_VNET_PREFIX \
    --subnet-name $AKS_SUBNET_NAME \
    --subnet-prefixes $AKS_SUBNET_PREFIX \
    --network-security-group $AKS_NSG_NAME
```

#### 4.3.2 Create Private Endpoints Subnet

```bash
az network vnet subnet create \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --name $ENDPOINTS_SUBNET_NAME \
    --address-prefixes $PRIVATE_LINK_SUBNET_PREFIX \
    --network-security-group $ENDPOINTS_NSG_NAME \
    --private-endpoint-network-policies Disabled
```

#### 4.3.3 Create Load Balancer Subnet

```bash
az network vnet subnet create \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --name $LOADBALANCER_SUBNET_NAME \
    --address-prefixes $LOADBALANCER_SUBNET_PREFIX \
    --network-security-group $LOADBALANCER_NSG_NAME
```

### 4.4 Create VNet Peering

#### 4.4.1 Get VNet Resource IDs

```bash
HUB_VNET_ID=$(az network vnet show \
    --resource-group $HUB_RG \
    --name $HUB_VNET_NAME \
    --query id \
    -o tsv)

SPOKE_VNET_ID=$(az network vnet show \
    --resource-group $SPOKE_RG \
    --name $SPOKE_VNET_NAME \
    --query id \
    -o tsv)
```

#### 4.4.2 Create Hub-to-Spoke Peering

```bash
az network vnet peering create \
    --resource-group $HUB_RG \
    --name hub-to-spoke \
    --vnet-name $HUB_VNET_NAME \
    --remote-vnet $SPOKE_VNET_ID \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

#### 4.4.3 Create Spoke-to-Hub Peering

```bash
az network vnet peering create \
    --resource-group $SPOKE_RG \
    --name spoke-to-hub \
    --vnet-name $SPOKE_VNET_NAME \
    --remote-vnet $HUB_VNET_ID \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

### 4.5 Configure User-Defined Routing

#### 4.5.1 Create Route Table

```bash
az network route-table create \
    --resource-group $SPOKE_RG \
    --name $ROUTE_TABLE_NAME
```

#### 4.5.2 Create Default Route to Firewall

```bash
az network route-table route create \
    --resource-group $SPOKE_RG \
    --name default-route \
    --route-table-name $ROUTE_TABLE_NAME \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address $FIREWALL_PRIVATE_IP
```

#### 4.5.3 Associate Route Table with AKS Subnet

```bash
az network vnet subnet update \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --name $AKS_SUBNET_NAME \
    --route-table $ROUTE_TABLE_NAME
```

---

## 5. Private AKS Cluster Deployment

### 5.1 Create Managed Identity

```bash
az identity create \
    --resource-group $SPOKE_RG \
    --name $AKS_IDENTITY_NAME
```

### 5.2 Assign RBAC Permissions

#### 5.2.1 Get Identity Details

```bash
IDENTITY_ID=$(az identity show \
    --resource-group $SPOKE_RG \
    --name $AKS_IDENTITY_NAME \
    --query id \
    -o tsv)

PRINCIPAL_ID=$(az identity show \
    --resource-group $SPOKE_RG \
    --name $AKS_IDENTITY_NAME \
    --query principalId \
    -o tsv)
```

#### 5.2.2 Assign Permissions to Route Table

```bash
RT_SCOPE=$(az network route-table show \
    --resource-group $SPOKE_RG \
    --name $ROUTE_TABLE_NAME \
    --query id \
    -o tsv)

az role assignment create \
    --assignee $PRINCIPAL_ID \
    --scope $RT_SCOPE \
    --role "Network Contributor"
```

#### 5.2.3 Assign Permissions to Load Balancer Subnet

```bash
LB_SUBNET_SCOPE=$(az network vnet subnet list \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --query "[?name=='$LOADBALANCER_SUBNET_NAME'].id" \
    -o tsv)

az role assignment create \
    --assignee $PRINCIPAL_ID \
    --scope $LB_SUBNET_SCOPE \
    --role "Network Contributor"
```

### 5.3 Deploy Private AKS Cluster

#### 5.3.1 Get AKS Subnet ID

```bash
AKS_SUBNET_SCOPE=$(az network vnet subnet list \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --query "[?name=='$AKS_SUBNET_NAME'].id" \
    -o tsv)
```

#### 5.3.2 Create Private AKS Cluster

```bash
az aks create \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --location $LOCATION \
    --node-count 2 \
    --node-vm-size Standard_D2s_v5 \
    --vnet-subnet-id $AKS_SUBNET_SCOPE \
    --enable-private-cluster \
    --outbound-type userDefinedRouting \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --generate-ssh-keys \
    --assign-identity $IDENTITY_ID \
    --network-plugin azure \
    --network-policy azure \
    --disable-public-fqdn \
    --zones 1 2 3 \
    --no-wait
```

> **Note:** AKS cluster creation takes approximately 10-15 minutes. Remove `--no-wait` to wait for completion.

### 5.4 Add User Node Pool

```bash
az aks nodepool add \
    --resource-group $SPOKE_RG \
    --cluster-name $AKS_CLUSTER_NAME \
    --name userpool \
    --node-count 3 \
    --mode User \
    --zones 1 2 3 \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 5
```

### 5.5 Configure DNS for Private Cluster

#### 5.5.1 Get AKS Node Resource Group

```bash
NODE_GROUP=$(az aks show \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --query nodeResourceGroup \
    -o tsv)
```

#### 5.5.2 Get Private DNS Zone Name

```bash
DNS_ZONE_NAME=$(az network private-dns zone list \
    --resource-group $NODE_GROUP \
    --query "[0].name" \
    -o tsv)

echo "AKS Private DNS Zone: $DNS_ZONE_NAME"
```

#### 5.5.3 Link Private DNS Zone to Hub VNet

```bash
az network private-dns link vnet create \
    --name hub-dns-link \
    --registration-enabled false \
    --resource-group $NODE_GROUP \
    --virtual-network $HUB_VNET_ID \
    --zone-name $DNS_ZONE_NAME
```

---

## 6. Azure Container Registry Deployment

### 6.1 Create Private ACR

```bash
az acr create \
    --resource-group $SPOKE_RG \
    --name $ACR_NAME \
    --sku Premium \
    --admin-enabled false \
    --location $LOCATION \
    --allow-trusted-services false \
    --public-network-enabled false
```

> **Important:** ACR names must be globally unique (3-50 alphanumeric characters).

### 6.2 Configure Private Endpoint for ACR

#### 6.2.1 Create Private DNS Zone for ACR

```bash
az network private-dns zone create \
    --resource-group $SPOKE_RG \
    --name "privatelink.azurecr.io"
```

#### 6.2.2 Link DNS Zone to Spoke VNet

```bash
az network private-dns link vnet create \
    --resource-group $SPOKE_RG \
    --zone-name "privatelink.azurecr.io" \
    --name acr-spoke-link \
    --virtual-network $SPOKE_VNET_NAME \
    --registration-enabled false
```

#### 6.2.3 Link DNS Zone to Hub VNet

```bash
az network private-dns link vnet create \
    --resource-group $SPOKE_RG \
    --zone-name "privatelink.azurecr.io" \
    --name acr-hub-link \
    --virtual-network $HUB_VNET_ID \
    --registration-enabled false
```

#### 6.2.4 Create Private Endpoint for ACR

```bash
REGISTRY_ID=$(az acr show \
    --name $ACR_NAME \
    --query 'id' \
    -o tsv)

az network private-endpoint create \
    --name pe-acr \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --subnet $ENDPOINTS_SUBNET_NAME \
    --private-connection-resource-id $REGISTRY_ID \
    --group-ids registry \
    --connection-name acr-private-connection
```

### 6.3 Configure DNS for ACR

#### 6.3.1 Get Private Endpoint Network Interface

```bash
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
    --name pe-acr \
    --resource-group $SPOKE_RG \
    --query 'networkInterfaces[0].id' \
    -o tsv)
```

#### 6.3.2 Get ACR Private IP Addresses

```bash
# Display IP configurations
az network nic show --ids $NETWORK_INTERFACE_ID \
    --query 'ipConfigurations[].{Name:name, IP:privateIPAddress, FQDN:privateLinkConnectionProperties.fqdns}' \
    -o table
```

> **Note:** Save the IP addresses for the next steps. You should see two entries:
> - Control plane: `<acrname>.azurecr.io`
> - Data plane: `<acrname>.<region>.data.azurecr.io`

#### 6.3.3 Create DNS A Records

**Control plane record:**

```bash
# Extract control plane IP
CONTROL_IP=$(az network nic show --ids $NETWORK_INTERFACE_ID \
    --query "ipConfigurations[?contains(privateLinkConnectionProperties.fqdns[0], '$ACR_NAME.azurecr.io')].privateIPAddress" \
    -o tsv | head -n 1)

# Create A record
az network private-dns record-set a create \
    --name $ACR_NAME \
    --zone-name privatelink.azurecr.io \
    --resource-group $SPOKE_RG

az network private-dns record-set a add-record \
    --record-set-name $ACR_NAME \
    --zone-name privatelink.azurecr.io \
    --resource-group $SPOKE_RG \
    --ipv4-address $CONTROL_IP
```

**Data plane record:**

```bash
# Extract data plane IP
DATA_IP=$(az network nic show --ids $NETWORK_INTERFACE_ID \
    --query "ipConfigurations[?contains(privateLinkConnectionProperties.fqdns[0], 'data.azurecr.io')].privateIPAddress" \
    -o tsv | head -n 1)

# Create A record
az network private-dns record-set a create \
    --name "$ACR_NAME.$LOCATION.data" \
    --zone-name privatelink.azurecr.io \
    --resource-group $SPOKE_RG

az network private-dns record-set a add-record \
    --record-set-name "$ACR_NAME.$LOCATION.data" \
    --zone-name privatelink.azurecr.io \
    --resource-group $SPOKE_RG \
    --ipv4-address $DATA_IP
```

### 6.4 Attach ACR to AKS

```bash
az aks update \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --attach-acr $ACR_NAME
```

---

## 7. Connect to Private AKS

### 7.1 Access Methods Overview

For private AKS clusters, you have two primary access methods:

**Option A: `az aks command invoke` (Recommended)**
- No VMs required
- Execute kubectl commands directly from your local machine
- Uses Azure RBAC for authentication
- Ideal for ad-hoc cluster management

**Option B: Azure Bastion Tunneling**
- Requires temporary management VM
- Full kubectl access from within Azure network
- Useful for prolonged administrative sessions

### 7.2 Option A: Using `az aks command invoke` (No VM Required)

#### 7.2.1 Run kubectl Commands Directly

```bash
# Get cluster nodes
az aks command invoke \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --command "kubectl get nodes"

# Get all pods
az aks command invoke \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --command "kubectl get pods -A"

# Deploy application
az aks command invoke \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --command "kubectl apply -f deployment.yaml" \
    --file deployment.yaml
```

**Benefits:**
- Zero infrastructure overhead
- No VMs to patch or maintain
- Commands execute securely through Azure control plane

### 7.3 Option B: Using Azure Bastion with Temporary VM

> **Note:** Only use this option if you need interactive shell access or prolonged administrative sessions.

#### 7.3.1 Install Azure CLI Bastion Extension

```bash
az extension add --name bastion
```

#### 7.3.2 Create Temporary Management VM

> **Note:** This VM is only needed temporarily. Delete it after completing administrative tasks.

```bash
# Create VM for initial setup
az vm create \
    --resource-group $HUB_RG \
    --name vm-mgmt-temp \
    --image Ubuntu2204 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --vnet-name $HUB_VNET_NAME \
    --subnet $MGMT_SUBNET_NAME \
    --size Standard_B2s \
    --public-ip-address "" \
    --nsg ""
```

#### 7.3.3 Connect via Bastion Tunnel

**Get VM Private IP:**

```bash
VM_PRIVATE_IP=$(az vm show \
    --resource-group $HUB_RG \
    --name vm-mgmt-temp \
    --show-details \
    --query privateIps \
    -o tsv)

echo "VM Private IP: $VM_PRIVATE_IP"
```

**Create SSH Tunnel via Bastion:**

```bash
# Create tunnel (runs in background)
az network bastion tunnel \
    --name $BASTION_NAME \
    --resource-group $HUB_RG \
    --target-resource-id $(az vm show -g $HUB_RG -n vm-mgmt-temp --query id -o tsv) \
    --resource-port 22 \
    --port 2222 &
```

> **Note:** This command opens a tunnel on local port 2222. Keep this terminal window open.

**Connect to VM via Tunnel:**

Open a **new terminal window** and connect:

```bash
ssh azureuser@127.0.0.1 -p 2222
```

#### 7.3.4 Install kubectl on Management VM

Connect to the management VM and install tools:

```bash
# Update package list
sudo apt update

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl
sudo az aks install-cli

# Login to Azure
az login
```

#### 7.3.5 Get AKS Credentials

```bash
# Set environment variables
SPOKE_RG="rg-spoke"
AKS_CLUSTER_NAME="<your-cluster-name>"

# Get AKS credentials
az aks get-credentials \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --admin
```

#### 7.3.6 Verify AKS Access

```bash
kubectl get nodes
kubectl get pods -A
```

**Expected Output:**

```bash
NAME                                STATUS   ROLES   AGE     VERSION
aks-nodepool1-12345678-vmss000000   Ready    agent   45m     v1.27.7
aks-nodepool1-12345678-vmss000001   Ready    agent   45m     v1.27.7
aks-userpool-87654321-vmss000000    Ready    agent   30m     v1.27.7
aks-userpool-87654321-vmss000001    Ready    agent   30m     v1.27.7
aks-userpool-87654321-vmss000002    Ready    agent   30m     v1.27.7
```

### 7.4 Recommendation

**For most scenarios, use Option A (`az aks command invoke`)** as it requires no infrastructure and provides secure, direct access to your private AKS cluster. Only use Option B if you need prolonged interactive shell sessions.

---

## 8. Validation and Testing

### 8.1 Validate Hub Infrastructure

#### 8.1.1 Check Hub VNet

```bash
az network vnet show \
    --resource-group $HUB_RG \
    --name $HUB_VNET_NAME \
    --query "{Name:name, Subnets:subnets[].name, AddressSpace:addressSpace.addressPrefixes[0]}" \
    -o table
```

#### 8.1.2 Check Azure Bastion

```bash
az network bastion show \
    --resource-group $HUB_RG \
    --name $BASTION_NAME \
    --query "{Name:name, SKU:sku.name, Tunneling:enableTunneling, State:provisioningState}" \
    -o table
```

#### 8.1.3 Check Azure Firewall

```bash
az network firewall show \
    --resource-group $HUB_RG \
    --name $FIREWALL_NAME \
    --query "{Name:name, PrivateIP:ipConfigurations[0].privateIPAddress, DNSProxy:dnsSettings.enableProxy}" \
    -o table
```

### 8.2 Validate Spoke Infrastructure

#### 8.2.1 Check Spoke VNet

```bash
az network vnet show \
    --resource-group $SPOKE_RG \
    --name $SPOKE_VNET_NAME \
    --query "{Name:name, Subnets:subnets[].name, AddressSpace:addressSpace.addressPrefixes[0]}" \
    -o table
```

#### 8.2.2 Check VNet Peering

```bash
az network vnet peering list \
    --resource-group $SPOKE_RG \
    --vnet-name $SPOKE_VNET_NAME \
    --query "[].{Name:name, State:peeringState, RemoteVNet:remoteVirtualNetwork.id}" \
    -o table
```

#### 8.2.3 Check Route Table

```bash
az network route-table route list \
    --resource-group $SPOKE_RG \
    --route-table-name $ROUTE_TABLE_NAME \
    --query "[].{Name:name, Prefix:addressPrefix, NextHop:nextHopType, NextHopIP:nextHopIpAddress}" \
    -o table
```

### 8.3 Validate AKS Cluster

#### 8.3.1 Check Cluster Status

```bash
az aks show \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --query "{Name:name, PrivateCluster:apiServerAccessProfile.enablePrivateCluster, OutboundType:networkProfile.outboundType, State:provisioningState}" \
    -o table
```

#### 8.3.2 Check Node Pools

```bash
az aks nodepool list \
    --resource-group $SPOKE_RG \
    --cluster-name $AKS_CLUSTER_NAME \
    --query "[].{Name:name, Mode:mode, Count:count, VMSize:vmSize, Zones:availabilityZones}" \
    -o table
```

#### 8.3.3 Check Nodes (via kubectl)

```bash
kubectl get nodes -o wide
```

### 8.4 Validate ACR Integration

#### 8.4.1 Check ACR Status

```bash
az acr show \
    --name $ACR_NAME \
    --query "{Name:name, LoginServer:loginServer, PublicAccess:publicNetworkAccess, SKU:sku.name}" \
    -o table
```

#### 8.4.2 Check Private Endpoint

```bash
az network private-endpoint show \
    --resource-group $SPOKE_RG \
    --name pe-acr \
    --query "{Name:name, State:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}" \
    -o table
```

#### 8.4.3 Check DNS Resolution (from management VM)

```bash
# SSH into management VM first
dig $ACR_NAME.azurecr.io
```

Expected output should show private IP in the 10.1.2.0/28 range.

### 8.5 Test Private Connectivity

#### 8.5.1 Test ACR Push/Pull

From the management VM:

```bash
# Login to ACR
az acr login --name $ACR_NAME

# Pull a test image
docker pull mcr.microsoft.com/hello-world:latest

# Tag and push to ACR
docker tag mcr.microsoft.com/hello-world:latest $ACR_NAME.azurecr.io/hello-world:latest
docker push $ACR_NAME.azurecr.io/hello-world:latest
```

#### 8.5.2 Deploy Test Pod to AKS

Create test pod manifest:

```bash
cat <<EOF > test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test
spec:
  containers:
  - name: hello-world
    image: $ACR_NAME.azurecr.io/hello-world:latest
    ports:
    - containerPort: 80
EOF
```

Deploy and verify:

```bash
kubectl apply -f test-pod.yaml
kubectl get pod test-pod
kubectl describe pod test-pod
```

#### 8.5.3 Test Internal Load Balancer

Create service manifest:

```bash
cat <<EOF > test-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: test-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "$LOADBALANCER_SUBNET_NAME"
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: test
EOF
```

Deploy and verify:

```bash
kubectl apply -f test-service.yaml
kubectl get svc test-service

# Wait for EXTERNAL-IP (private IP)
kubectl get svc test-service -w
```

Test connectivity from management VM:

```bash
LB_IP=$(kubectl get svc test-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$LB_IP
```

---

## 9. Troubleshooting

### 9.1 Common Issues

#### Issue 1: Cannot Connect to AKS API Server

**Symptoms:**
```bash
Unable to connect to the server: dial tcp: lookup <fqdn> on <ip>:53: no such host
```

**Resolution:**
1. Verify Private DNS zone link to Hub VNet:
```bash
az network private-dns link vnet list \
    --resource-group $NODE_GROUP \
    --zone-name $DNS_ZONE_NAME \
    -o table
```

2. Check DNS resolution from management VM:
```bash
nslookup <aks-fqdn>
```

3. Ensure management VM is in hub VNet or peered network.

#### Issue 2: ACR Pull Failures

**Symptoms:**
```bash
Failed to pull image: rpc error: code = Unknown desc = failed to pull and unpack image
```

**Resolution:**
1. Verify ACR is attached to AKS:
```bash
az aks show -g $SPOKE_RG -n $AKS_CLUSTER_NAME --query servicePrincipalProfile.clientId -o tsv
```

2. Check private endpoint DNS:
```bash
dig $ACR_NAME.azurecr.io
```

3. Verify RBAC assignments:
```bash
az role assignment list --scope $REGISTRY_ID -o table
```

#### Issue 3: Firewall Blocking Traffic

**Symptoms:**
- Pods stuck in `ContainerCreating`
- Timeout errors in pod logs

**Resolution:**
1. Check firewall logs:
```bash
az monitor diagnostic-settings create \
    --name firewall-logs \
    --resource $(az network firewall show -g $HUB_RG -n $FIREWALL_NAME --query id -o tsv) \
    --logs '[{"category":"AzureFirewallApplicationRule","enabled":true},{"category":"AzureFirewallNetworkRule","enabled":true}]' \
    --workspace <log-analytics-workspace-id>
```

2. Review network rules:
```bash
az network firewall network-rule list \
    -g $HUB_RG \
    -f $FIREWALL_NAME \
    --collection-name aks-required \
    -o table
```

### 9.2 Diagnostic Commands

#### Get AKS Cluster Details

```bash
az aks show \
    --resource-group $SPOKE_RG \
    --name $AKS_CLUSTER_NAME \
    --output json > aks-config.json
```

#### Get Network Topology

```bash
# Hub resources
az network vnet show -g $HUB_RG -n $HUB_VNET_NAME

# Spoke resources
az network vnet show -g $SPOKE_RG -n $SPOKE_VNET_NAME

# Peering status
az network vnet peering list -g $SPOKE_RG --vnet-name $SPOKE_VNET_NAME
```

#### Check AKS Node Pool Status

```bash
az aks nodepool list \
    --resource-group $SPOKE_RG \
    --cluster-name $AKS_CLUSTER_NAME \
    --output table
```

#### Check Kubernetes Events

```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

---

## 10. Cleanup Procedures

### 10.1 Delete Test Resources

```bash
# Delete test pod and service
kubectl delete -f test-pod.yaml
kubectl delete -f test-service.yaml
```

### 10.2 Delete Management VM

```bash
az vm delete \
    --resource-group $HUB_RG \
    --name vm-mgmt-temp \
    --yes \
    --no-wait
```

### 10.3 Delete All Infrastructure (Optional)

> **Warning:** This will delete all resources created in this SOP.

```bash
# Delete spoke resources (includes AKS)
az group delete \
    --name $SPOKE_RG \
    --yes \
    --no-wait

# Delete hub resources
az group delete \
    --name $HUB_RG \
    --yes \
    --no-wait

# Delete AKS node resource group (if not auto-deleted)
az group delete \
    --name $NODE_GROUP \
    --yes \
    --no-wait
```

---

## Appendix A: Network Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                            │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ HTTPS/SSH
                   ▼
        ┌──────────────────────┐
        │   Azure Bastion      │
        │   (Standard SKU)     │
        │   - Tunneling: Yes   │
        └──────────┬───────────┘
                   │
    ┌──────────────┼──────────────┐
    │         Hub VNet             │
    │      10.0.0.0/22             │
    │  ┌────────────────────┐     │
    │  │ Azure Firewall     │     │
    │  │ 10.0.0.4           │     │
    │  └─────────┬──────────┘     │
    └────────────┼─────────────────┘
                 │ VNet Peering
    ┌────────────┼─────────────────┐
    │        Spoke VNet            │
    │       10.1.0.0/20            │
    │  ┌──────────────────────┐   │
    │  │   Private AKS        │   │
    │  │   10.1.0.0/23        │   │
    │  │   - API: Private     │   │
    │  │   - Egress: UDR      │   │
    │  └──────────────────────┘   │
    │  ┌──────────────────────┐   │
    │  │   Private ACR        │   │
    │  │   10.1.2.4           │   │
    │  └──────────────────────┘   │
    │  ┌──────────────────────┐   │
    │  │ Internal LB          │   │
    │  │   10.1.3.4           │   │
    │  └──────────────────────┘   │
    └──────────────────────────────┘
```

---

## Appendix B: Required Azure Providers

Ensure the following resource providers are registered:

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.ManagedIdentity

# Check registration status
az provider show -n Microsoft.ContainerService --query registrationState
```

---

## Appendix C: Cost Estimation

**Approximate monthly costs (Sweden Central):**

| Resource | SKU | Quantity | Monthly Cost (USD) |
|----------|-----|----------|-------------------|
| AKS Control Plane | Free | 1 | $0 |
| AKS System Nodes | Standard_D2s_v5 | 2 | ~$140 |
| AKS User Nodes | Standard_D2s_v5 | 3 | ~$210 |
| Azure Firewall | Premium | 1 | ~$625 |
| Azure Bastion | Standard | 1 | ~$140 |
| ACR | Premium | 1 | ~$500 |
| Managed Disks | P10 (128GB) | 5 | ~$40 |
| **Total** | | | **~$1,655/month** |

> **Note:** Costs vary by region and usage. Use [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for accurate estimates.

---

## Appendix D: Security Recommendations

1. **Enable Azure Policy for AKS**
   - Implement pod security policies
   - Enforce network policies
   - Scan container images

2. **Configure Azure Monitor**
   - Enable Container Insights
   - Configure log analytics
   - Set up alerts

3. **Implement Least Privilege**
   - Use Azure RBAC for AKS access
   - Implement Kubernetes RBAC
   - Rotate credentials regularly

4. **Network Security**
   - Keep NSG rules minimal
   - Review firewall rules periodically
   - Enable flow logs for diagnostics

5. **Backup and Disaster Recovery**
   - Enable AKS backup (Azure Backup for AKS)
   - Replicate ACR across regions
   - Document recovery procedures

---

**Document Control:**
- **Version:** 1.0
- **Last Review:** November 23, 2025
- **Next Review:** February 23, 2026
- **Classification:** Internal Use

---

**End of Document**
