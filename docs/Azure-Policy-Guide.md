# Azure Policy for Private AKS Clusters

**Document Type**: Policy Guide  
**Project**: Azure Kubernetes Service (AKS) - Private Cluster Deployment  
**Version**: 1.0  
**Date**: November 23, 2025  
**Status**: Active  
**Review Cycle**: Quarterly

---

**📌 [← Back to Main Documentation](private-aks-cluster-guide.md)**

---

## Related Documentation

This policy guide is part of the private AKS deployment documentation set:
- **[Private AKS Guide](private-aks-cluster-guide.md)** - Documentation overview and quick start
- **[Provision Private AKS Cluster](SOP-provision-private-aks-cluster.md)** - Complete deployment guide
- **[Workload Migration](SOP-workload-migration.md)** - Migrate from public to private AKS

> **⚠️ Important**: Deploy these policies **before** creating AKS clusters to ensure compliance from day one.
>
> **Deployment Order**: 
> 1. Deploy policies (this guide) → 2. [Provision private cluster](SOP-provision-private-aks-cluster.md) → 3. Deploy applications

---

## Overview

This document provides comprehensive Azure Policy definitions and assignments to **enforce private AKS clusters** and **secure Azure services** across your environment. Policies ensure consistent security posture and prevent misconfiguration.

**Policy Strategy**: This guide follows Microsoft's recommended baseline policies from the [AKS Baseline Architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) and includes validation procedures from the official [Use Azure Policy with AKS](https://learn.microsoft.com/en-us/azure/aks/use-azure-policy) documentation.

---

## 🎯 Policy Deployment Recommendation

### Management Group Level (HIGHLY RECOMMENDED)

**Why Management Groups?**
- ✅ **Scale**: Apply policies across multiple subscriptions at once
- ✅ **Consistency**: Enforce standards organization-wide
- ✅ **Inheritance**: Child subscriptions automatically inherit policies
- ✅ **Governance**: Central control for compliance and security
- ✅ **Efficiency**: Define once, apply everywhere

**Recommended Hierarchy**:
```
Root Management Group
└── Production Management Group
    ├── Landing Zones (Spoke Subscriptions)
    │   ├── Subscription 1 (inherits policies)
    │   ├── Subscription 2 (inherits policies)
    │   └── Subscription 3 (inherits policies)
    └── Platform (Hub Subscription)
        └── Hub Infrastructure
```

**Assign policies at**: `Production Management Group` or `Landing Zones` management group level

**Alternative**: Assign at subscription level if management groups are not available

---

## 🎯 Policy Deployment Phases

### Phase 1: Prerequisites (DEPLOY FIRST)
1. Install Azure Policy Add-on on existing clusters
2. Deploy in **Audit mode** to assess current state
3. Review compliance reports for 2-4 weeks

### Phase 2: Enforcement (After Assessment)
1. Switch critical policies to **Deny mode**
2. Deploy Policy Initiative (bundle all policies)
3. Monitor compliance dashboard

### Phase 3: Ongoing Governance
1. Review compliance monthly
2. Update policies as Azure best practices evolve
3. Handle exemptions through approval process

---

## 📋 Microsoft-Recommended Policies

### Policy Initiative: Pod Security Baseline (PRIORITY 1)

**What**: Microsoft's recommended baseline for pod security on Linux workloads  
**When to Deploy**: FIRST - Before deploying any workloads  
**Initiative Name**: `Kubernetes cluster pod security baseline standards for Linux-based workloads`  
**Initiative ID**: `/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d`

**Includes 13+ Policies**:
- ✅ No privileged containers
- ✅ No host namespace usage
- ✅ No host networking and ports
- ✅ Allowed volume types only
- ✅ Container capabilities restrictions
- ✅ AppArmor profile enforcement
- ✅ SELinux options restrictions
- ✅ And more...

**Assignment** (Start with Audit):
```bash
# Assign Pod Security Baseline Initiative at Management Group
az policy assignment create \
  --name "aks-pod-security-baseline" \
  --display-name "AKS Pod Security Baseline for Linux Workloads" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    },
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  }'

# After 2-4 weeks of monitoring, switch to Deny mode
az policy assignment update \
  --name "aks-pod-security-baseline" \
  --params '{
    "effect": {
      "value": "Deny"
    },
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  }'
```

**Validation**:
```bash
# Check if policy add-on is installed
kubectl get constrainttemplates

# Expected output: ~13 constraint templates from baseline initiative
# NAME                                     AGE
# k8sazureallowedcapabilities              23m
# k8sazureallowedusersgroups               23m
# k8sazureblockhostnamespace               23m
# k8sazurecontainerallowedimages           23m
# k8sazurecontainerallowedports            23m
# k8sazurecontainerlimits                  23m
# k8sazurecontainernoprivilege             23m
# k8sazurecontainernoprivilegeescalation   23m
# (and more...)
```

> **Note**: Policy assignments can take up to 20 minutes to sync into each cluster.

---

## 📋 Required Policies for Private AKS

### 1. Enforce Private AKS Clusters Only

**Policy Name**: `Require private AKS clusters`

**Purpose**: Prevent creation of AKS clusters with public API endpoints

**Built-in Policy**: `Azure Kubernetes Service Private Clusters should be enabled`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/040732e8-d947-40b8-95d6-854c95024bf8`

**Effect**: `Deny` - Blocks creation of non-private clusters

**Assignment**:
```bash
# Assign at management group level
az policy assignment create \
  --name "enforce-private-aks" \
  --display-name "Require Private AKS Clusters" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/040732e8-d947-40b8-95d6-854c95024bf8" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'

# Or assign at subscription level
az policy assignment create \
  --name "enforce-private-aks" \
  --display-name "Require Private AKS Clusters" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/040732e8-d947-40b8-95d6-854c95024bf8" \
  --scope "/subscriptions/<subscription-id>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

---

### 2. Restrict Public Endpoints for Azure Services

#### 2.1 Azure Container Registry (ACR)

**Policy Name**: `Container registries should disable public network access`

**Purpose**: Prevent ACR from having public endpoints

**Built-in Policy**: `Container registries should disable public network access`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/0fdf0491-d080-4575-b627-ad0e843cba0f`

**Effect**: `Deny` - Blocks public ACR creation

**Assignment**:
```bash
az policy assignment create \
  --name "deny-public-acr" \
  --display-name "Deny Public ACR Endpoints" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/0fdf0491-d080-4575-b627-ad0e843cba0f" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

#### 2.2 Azure Key Vault

**Policy Name**: `Azure Key Vault should disable public network access`

**Purpose**: Prevent Key Vault from having public endpoints

**Built-in Policy**: `Azure Key Vault should disable public network access`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/405c5871-3e91-4644-8a63-58e19d68ff5b`

**Effect**: `Deny` - Blocks public Key Vault creation

**Assignment**:
```bash
az policy assignment create \
  --name "deny-public-keyvault" \
  --display-name "Deny Public Key Vault Endpoints" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/405c5871-3e91-4644-8a63-58e19d68ff5b" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

---

## 🔐 Additional Recommended Policies

### 3. Enforce Private Endpoints for ACR

**Policy Name**: `Container registries should use private link`

**Purpose**: Require private endpoints for ACR connectivity

**Built-in Policy**: `Container registries should use private link`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/e8eef0a8-67cf-4729-a1b5-4f59c857f9c0`

**Effect**: `Audit` or `Deny`

**Assignment**:
```bash
az policy assignment create \
  --name "require-acr-private-link" \
  --display-name "Require ACR Private Link" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/e8eef0a8-67cf-4729-a1b5-4f59c857f9c0" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 4. Enforce Private Endpoints for Key Vault

**Policy Name**: `Azure Key Vault should use private link`

**Purpose**: Require private endpoints for Key Vault connectivity

**Built-in Policy**: `Azure Key Vault should use private link`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9`

**Effect**: `Audit` or `Deny`

**Assignment**:
```bash
az policy assignment create \
  --name "require-kv-private-link" \
  --display-name "Require Key Vault Private Link" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 5. Enforce AKS Azure CNI Networking

**Policy Name**: `Azure Kubernetes Service clusters should use Azure CNI`

**Purpose**: Ensure AKS clusters use Azure CNI (required for proper private networking)

**Custom Policy** (no built-in available):

**Policy Definition**:
```json
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
          "field": "Microsoft.ContainerService/managedClusters/networkProfile.networkPlugin",
          "notEquals": "azure"
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
      "metadata": {
        "displayName": "Effect",
        "description": "Enable or disable the execution of the policy"
      },
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Deny"
    }
  }
}
```

**Create and Assign**:
```bash
# Create custom policy definition
az policy definition create \
  --name "require-aks-azure-cni" \
  --display-name "Require AKS Azure CNI Networking" \
  --description "Ensures AKS clusters use Azure CNI networking" \
  --rules @policy-aks-azure-cni.json \
  --mode Indexed \
  --management-group <your-management-group-name>

# Assign policy
az policy assignment create \
  --name "enforce-aks-azure-cni" \
  --display-name "Enforce AKS Azure CNI" \
  --policy "require-aks-azure-cni" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

---

### 6. Enable Azure Policy Add-on for AKS (PREREQUISITE)

**Policy Name**: `Azure Policy Add-on for Kubernetes service (AKS) should be installed and enabled`

**Purpose**: Ensures policy enforcement capability exists on all clusters

**Built-in Policy**: `Azure Policy Add-on for Kubernetes service (AKS) should be installed and enabled on your clusters`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/0a15ec92-a229-4763-bb14-0ea34a568f8d`

**Effect**: `Audit` (cannot auto-install, must be done during cluster creation)

**Assignment**:
```bash
az policy assignment create \
  --name "require-aks-policy-addon" \
  --display-name "Require AKS Policy Add-on" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/0a15ec92-a229-4763-bb14-0ea34a568f8d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

**Enable on Existing Cluster**:
```bash
# Enable Azure Policy add-on
az aks enable-addons \
  --addons azure-policy \
  --name <cluster-name> \
  --resource-group <resource-group>

# Verify installation
kubectl get pods -n gatekeeper-system
kubectl get pods -n kube-system | grep azure-policy
```

---

### 7. HTTPS-Only Ingress

**Policy Name**: `Kubernetes clusters should be accessible only over HTTPS`

**Purpose**: Enforce TLS encryption for all ingress traffic

**Built-in Policy**: `Kubernetes clusters should be accessible only over HTTPS`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/1a5b4dca-0b6f-4cf5-907c-56316bc1bf3d`

**Effect**: `Deny`

**Assignment**:
```bash
az policy assignment create \
  --name "require-https-ingress" \
  --display-name "Require HTTPS for All Ingress" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/1a5b4dca-0b6f-4cf5-907c-56316bc1bf3d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    },
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  }'
```

---

### 8. Azure RBAC for Kubernetes Authorization

**Policy Name**: `Azure Kubernetes Service Clusters should enable Azure RBAC`

**Purpose**: Enforce Azure RBAC instead of Kubernetes RBAC for better governance

**Built-in Policy**: `Azure Kubernetes Service Clusters should have Azure RBAC enabled`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/2fd89692-3b6e-4b5f-84e9-c2e5d5c84f51`

**Effect**: `Audit` (cannot be changed after cluster creation)

**Assignment**:
```bash
az policy assignment create \
  --name "require-azure-rbac" \
  --display-name "Require Azure RBAC for AKS" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/2fd89692-3b6e-4b5f-84e9-c2e5d5c84f51" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 9. Disable Local Accounts

**Policy Name**: `Azure Kubernetes Service Clusters should have local authentication methods disabled`

**Purpose**: Force Microsoft Entra ID authentication only

**Built-in Policy**: `Azure Kubernetes Service Clusters should have local authentication methods disabled`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/993c2fcd-2b29-49d2-9eb0-df2c3a730c32`

**Effect**: `Deny`

**Assignment**:
```bash
az policy assignment create \
  --name "disable-local-accounts" \
  --display-name "Disable AKS Local Accounts" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/993c2fcd-2b29-49d2-9eb0-df2c3a730c32" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

---

### 10. Authorized IP Ranges for API Server

**Policy Name**: `Authorized IP ranges should be defined on Kubernetes Services`

**Purpose**: Restrict API server access to known IP ranges

**Built-in Policy**: `Authorized IP ranges should be defined on Kubernetes Services`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/0e246bcf-5f6f-4f87-bc6f-775d4712c7ea`

**Effect**: `Audit` (optional for private clusters, required for non-private)

**Assignment**:
```bash
az policy assignment create \
  --name "require-api-authorized-ips" \
  --display-name "Require Authorized IP Ranges for API Server" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/0e246bcf-5f6f-4f87-bc6f-775d4712c7ea" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 11. Microsoft Defender for Containers

**Policy Name**: `Azure Defender for Kubernetes should be enabled`

**Purpose**: Enable security threat detection and vulnerability scanning

**Built-in Policy**: `Microsoft Defender for Containers should be enabled`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/c9d007b8-75f5-4c5e-9a2b-9b3a6f1e6f3a`

**Effect**: `Audit`

**Assignment**:
```bash
az policy assignment create \
  --name "require-defender-containers" \
  --display-name "Require Microsoft Defender for Containers" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/c9d007b8-75f5-4c5e-9a2b-9b3a6f1e6f3a" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 12. Kubernetes Version Currency

**Policy Name**: `Kubernetes Services should be upgraded to a non-vulnerable Kubernetes version`

**Purpose**: Ensure clusters stay on supported Kubernetes versions (N-2 window)

**Built-in Policy**: `Kubernetes Services should be upgraded to a non-vulnerable Kubernetes version`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/fb893a29-21bb-418c-a157-e99480ec364c`

**Effect**: `Audit`

**Assignment**:
```bash
az policy assignment create \
  --name "audit-kubernetes-version" \
  --display-name "Audit Kubernetes Version Currency" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/fb893a29-21bb-418c-a157-e99480ec364c" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 13. Require AKS Managed Identity (No Service Principals)

**Policy Name**: `Kubernetes clusters should use managed identities`

**Purpose**: Enforce managed identities over service principals for better security

**Built-in Policy**: `Kubernetes clusters should use managed identities`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/da6e2401-19da-4532-9141-fb8fbde08431`

**Effect**: `Audit` or `Deny`

**Assignment**:
```bash
az policy assignment create \
  --name "require-aks-managed-identity" \
  --display-name "Require AKS Managed Identity" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/da6e2401-19da-4532-9141-fb8fbde08431" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Deny"
    }
  }'
```

---

### 14. Enforce AKS Standard Tier (Production SLA)

**Policy Name**: `Azure Kubernetes Service clusters should use Standard tier`

**Purpose**: Ensure production clusters use Standard tier for 99.95% SLA

**Custom Policy**:

**Policy Definition**:
```json
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
          "field": "Microsoft.ContainerService/managedClusters/sku.tier",
          "notEquals": "Standard"
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
      "metadata": {
        "displayName": "Effect",
        "description": "Enable or disable the execution of the policy"
      },
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Audit"
    }
  }
}
```

**Create and Assign**:
```bash
# Create custom policy definition
az policy definition create \
  --name "require-aks-standard-tier" \
  --display-name "Require AKS Standard Tier" \
  --description "Ensures AKS clusters use Standard tier for production SLA" \
  --rules @policy-aks-standard-tier.json \
  --mode Indexed \
  --management-group <your-management-group-name>

# Assign policy (Audit mode for flexibility)
az policy assignment create \
  --name "audit-aks-standard-tier" \
  --display-name "Audit AKS Standard Tier" \
  --policy "require-aks-standard-tier" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 15. Enforce AKS Availability Zones

**Policy Name**: `Azure Kubernetes Service clusters should use availability zones`

**Purpose**: Ensure high availability by requiring zone-redundant node pools

**Custom Policy**:

**Policy Definition**:
```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ContainerService/managedClusters/agentPools"
        },
        {
          "anyOf": [
            {
              "field": "Microsoft.ContainerService/managedClusters/agentPools/availabilityZones",
              "exists": false
            },
            {
              "field": "Microsoft.ContainerService/managedClusters/agentPools/availabilityZones[*]",
              "count": {
                "less": 2
              }
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
      "metadata": {
        "displayName": "Effect",
        "description": "Enable or disable the execution of the policy"
      },
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Audit"
    }
  }
}
```

---

### 16. Require Azure Monitor for AKS

**Policy Name**: `Azure Monitor addon should be enabled on Azure Kubernetes Service`

**Purpose**: Ensure monitoring is enabled for observability

**Built-in Policy**: `Azure Monitor addon should be enabled on Azure Kubernetes Service`  
**Policy ID**: `/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7`

**Effect**: `Audit` (don't deny, just alert)

**Assignment**:
```bash
az policy assignment create \
  --name "require-aks-monitoring" \
  --display-name "Require AKS Monitoring" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {
      "value": "Audit"
    }
  }'
```

---

### 17. Enforce ACR SKU (Premium for Private Endpoints)

**Policy Name**: `Container registries should use Premium SKU for private endpoints`

**Purpose**: Ensure ACR uses Premium SKU (required for private endpoints)

**Custom Policy**:

**Policy Definition**:
```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.ContainerRegistry/registries"
        },
        {
          "field": "Microsoft.ContainerRegistry/registries/sku.name",
          "notEquals": "Premium"
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
      "metadata": {
        "displayName": "Effect",
        "description": "Enable or disable the execution of the policy"
      },
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Deny"
    }
  }
}
```

---

## 📦 Policy Initiative (Recommended)

Create a **Policy Initiative** (also called Policy Set) to group all related policies together.

### AKS Private Cluster Initiative

**Create Initiative**:
```bash
# Create initiative definition JSON file
cat > aks-private-initiative.json <<EOF
{
  "properties": {
    "displayName": "Private AKS Baseline",
    "description": "Policy initiative to enforce private AKS clusters with secure networking",
    "policyDefinitions": [
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/040732e8-d947-40b8-95d6-854c95024bf8",
        "parameters": {
          "effect": {
            "value": "Deny"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/0fdf0491-d080-4575-b627-ad0e843cba0f",
        "parameters": {
          "effect": {
            "value": "Deny"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/405c5871-3e91-4644-8a63-58e19d68ff5b",
        "parameters": {
          "effect": {
            "value": "Deny"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/e8eef0a8-67cf-4729-a1b5-4f59c857f9c0",
        "parameters": {
          "effect": {
            "value": "Audit"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/a6abeaec-4d90-4a02-805f-6b26c4d3fbe9",
        "parameters": {
          "effect": {
            "value": "Audit"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/da6e2401-19da-4532-9141-fb8fbde08431",
        "parameters": {
          "effect": {
            "value": "Deny"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7",
        "parameters": {
          "effect": {
            "value": "Audit"
          }
        }
      }
    ]
  }
}
EOF

# Create initiative at management group level
az policy set-definition create \
  --name "private-aks-baseline" \
  --display-name "Private AKS Baseline" \
  --description "Enforces private AKS clusters with secure networking" \
  --definitions @aks-private-initiative.json \
  --management-group <your-management-group-name>

# Assign initiative
az policy assignment create \
  --name "enforce-private-aks-baseline" \
  --display-name "Enforce Private AKS Baseline" \
  --policy-set-definition "private-aks-baseline" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>"
```

---

## ✅ Comprehensive Policy Validation

### Step 1: Verify Policy Add-on Installation

```bash
# Check if Azure Policy add-on is installed and running
kubectl get pods -n gatekeeper-system

# Expected output: gatekeeper-controller and gatekeeper-audit pods running
# NAME                                             READY   STATUS    RESTARTS   AGE
# gatekeeper-audit-7b8c8c4f7d-xxxxx                1/1     Running   0          5m
# gatekeeper-controller-manager-0                  1/1     Running   0          5m
# gatekeeper-controller-manager-1                  1/1     Running   0          5m

# Check Azure Policy pods
kubectl get pods -n kube-system | grep azure-policy

# Expected output: azure-policy and azure-policy-webhook pods running
# azure-policy-xxxxx                               2/2     Running   0          5m
# azure-policy-webhook-xxxxx                       2/2     Running   0          5m
```

### Step 2: Verify Constraint Templates Synced

```bash
# List all constraint templates (policies synced to cluster)
kubectl get constrainttemplates

# Expected output: All policy constraint templates from assigned initiatives
# NAME                                             AGE
# k8sazureallowedcapabilities                      23m
# k8sazureallowedusersgroups                       23m
# k8sazureblockhostnamespace                       23m
# k8sazurecontainerallowedimages                   23m
# k8sazurecontainerallowedports                    23m
# k8sazurecontainerlimits                          23m
# k8sazurecontainernoprivilege                     23m
# k8sazurecontainernoprivilegeescalation           23m
# k8sazureenforceapparmor                          23m
# k8sazurehostfilesystem                           23m
# k8sazurehostnetworkingports                      23m
# k8sazurereadonlyrootfilesystem                   23m
# k8sazureserviceallowedports                      23m

# Check constraint instances (actual policy rules applied)
kubectl get constraints

# View specific constraint details
kubectl describe constraint <constraint-name>
```

> **⏱️ Important**: Policy assignments can take **up to 20 minutes** to sync into each cluster. If you don't see constraint templates immediately, wait and check again.

### Step 3: Test Pod Security Baseline Policies

#### Test 1: Reject Privileged Pod (Should DENY)

```bash
# Create test file
cat <<EOF > test-privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
  - name: nginx-privileged
    image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
    securityContext:
      privileged: true  # This violates policy
EOF

# Try to apply (should be DENIED)
kubectl apply -f test-privileged-pod.yaml

# Expected output:
# Error from server ([denied by azurepolicy-container-no-privilege-xxxxx] 
# Privileged container is not allowed: nginx-privileged, 
# securityContext: {"privileged": true})
```

#### Test 2: Accept Unprivileged Pod (Should ALLOW)

```bash
# Create compliant test file
cat <<EOF > test-unprivileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-unprivileged
spec:
  containers:
  - name: nginx-unprivileged
    image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
    # No privileged securityContext
EOF

# Apply compliant pod (should SUCCEED)
kubectl apply -f test-unprivileged-pod.yaml

# Expected output:
# pod/nginx-unprivileged created

# Verify pod is running
kubectl get pod nginx-unprivileged

# Expected output:
# NAME                 READY   STATUS    RESTARTS   AGE
# nginx-unprivileged   1/1     Running   0          10s

# Cleanup
kubectl delete pod nginx-unprivileged
```

### Step 4: Test HTTPS-Only Ingress (Policy 7)

```bash
# Test 1: HTTP ingress (should DENY if policy is in Deny mode)
cat <<EOF > test-http-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-http
  annotations:
    kubernetes.io/ingress.allow-http: "true"  # Violates HTTPS-only
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
EOF

kubectl apply -f test-http-ingress.yaml

# Expected output (if policy is in Deny mode):
# Error from server ([denied by azurepolicy-ingress-https-only-xxxxx] 
# Ingress must use HTTPS. Set kubernetes.io/ingress.allow-http to false)

# Test 2: HTTPS ingress with TLS (should ALLOW)
cat <<EOF > test-https-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-https
  annotations:
    kubernetes.io/ingress.allow-http: "false"
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 443
EOF

kubectl apply -f test-https-ingress.yaml

# Expected output:
# ingress.networking.k8s.io/test-https created

# Cleanup
kubectl delete ingress test-https --ignore-not-found
```

### Step 5: Check Azure Policy Compliance Dashboard

```bash
# Get compliance state for all policy assignments
az policy state list \
  --resource-group <resource-group> \
  --query "[].{Policy:policyDefinitionName, Compliance:complianceState, Resource:resourceId}" \
  --output table

# Get compliance summary at management group level
az policy state summarize \
  --management-group <your-management-group-name>

# Get compliance for specific AKS cluster
az policy state list \
  --resource "/subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.ContainerService/managedClusters/<cluster-name>" \
  --query "[?complianceState=='NonCompliant'].{Policy:policyDefinitionName, Reason:policyDefinitionAction}" \
  --output table

# Get total compliance percentage
az policy state summarize \
  --scope "/subscriptions/<subscription-id>" \
  --query "results.{Total:resourceDetails.nonCompliantCount, Compliant:resourceDetails.compliantCount}" \
  --output table
```

### Step 6: View Policy Logs in Cluster

```bash
# Check gatekeeper audit logs for policy violations
kubectl logs -n gatekeeper-system -l control-plane=audit-controller --tail=50

# Check Azure Policy pod logs
kubectl logs -n kube-system -l app=azure-policy --tail=50

# Check Azure Policy webhook logs
kubectl logs -n kube-system -l app=azure-policy-webhook --tail=50

# View policy violation events
kubectl get events --all-namespaces | grep -i "deny\|violat"
```

### Step 7: Test Azure Resource-Level Policies

```bash
# Test 1: Try to create non-private AKS cluster (should DENY)
az aks create \
  --resource-group test-rg \
  --name test-public-cluster \
  --enable-private-cluster false \
  --node-count 1 \
  --generate-ssh-keys

# Expected output:
# The resource 'test-public-cluster' is disallowed by policy. 
# (Policy: Require private AKS clusters; Assignment: enforce-private-aks)

# Test 2: Try to create public ACR (should DENY)
az acr create \
  --resource-group test-rg \
  --name testpublicacr123 \
  --sku Premium \
  --public-network-enabled true

# Expected output:
# The resource 'testpublicacr123' is disallowed by policy.
# (Policy: Container registries should disable public network access)

# Test 3: Create compliant private AKS cluster (should SUCCEED)
az aks create \
  --resource-group test-rg \
  --name test-private-cluster \
  --enable-private-cluster true \
  --node-count 1 \
  --generate-ssh-keys

# Expected output:
# Cluster creation succeeds (no policy violation)
```

### Step 8: Monitor Policy Compliance Over Time

```bash
# Create Azure Monitor alert for policy violations
az monitor metrics alert create \
  --name "AKS-Policy-Violations" \
  --resource-group <monitoring-rg> \
  --scopes "/subscriptions/<subscription-id>" \
  --condition "count > 0" \
  --description "Alert when AKS policies are violated"

# View compliance trends in Azure Portal
# Navigate to: Azure Portal > Policy > Compliance
# Filter by: Management Group = "<your-management-group-name>"
# View: Compliance over time chart
```

### Validation Checklist

| Test | Expected Result | Status |
|------|-----------------|--------|
| ✅ Policy add-on pods running | 4+ pods in Running state | ☐ |
| ✅ Constraint templates synced | 13+ templates from baseline | ☐ |
| ✅ Privileged pod denied | Error with policy violation message | ☐ |
| ✅ Unprivileged pod allowed | Pod created successfully | ☐ |
| ✅ Docker Hub image denied | Error with image restriction message | ☐ |
| ✅ ACR image allowed | Pod created successfully | ☐ |
| ✅ HTTP ingress denied | Error with HTTPS-only message | ☐ |
| ✅ HTTPS ingress allowed | Ingress created successfully | ☐ |
| ✅ Public cluster denied | AZ CLI error with policy violation | ☐ |
| ✅ Private cluster allowed | Cluster creation succeeds | ☐ |
| ✅ Compliance dashboard shows data | Non-zero resources evaluated | ☐ |

### Troubleshooting Policy Issues

```bash
# Issue: Policies not syncing to cluster
# Solution: Check policy add-on status
az aks show \
  --resource-group <rg-name> \
  --name <cluster-name> \
  --query "addonProfiles.azurepolicy"

# Issue: Constraint templates not appearing
# Solution: Check gatekeeper system health
kubectl get pods -n gatekeeper-system
kubectl logs -n gatekeeper-system -l control-plane=controller-manager

# Issue: Policies not blocking non-compliant resources
# Solution: Verify policy effect is set to "Deny" not "Audit"
az policy assignment show \
  --name "enforce-private-aks" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --query "parameters.effect.value"

# Issue: Too many false positives
# Solution: Add namespace exclusions
az policy assignment update \
  --name "aks-pod-security-baseline" \
  --params '{
    "excludedNamespaces": {
      "value": ["kube-system", "gatekeeper-system", "azure-arc", "monitoring"]
    }
  }'
```

---

## 📋 Policy Summary Table

| # | Policy Name | Type | Effect | Priority | Overlap with Baseline? |
|---|-------------|------|--------|----------|------------------------|
| 0 | **Pod Security Baseline Initiative** | Initiative | Audit→Deny | ⭐⭐⭐ Critical | N/A |
| 1 | Private AKS Clusters | Built-in | Deny | ⭐⭐⭐ Critical | No |
| 2.1 | Disable Public ACR | Built-in | Deny | ⭐⭐⭐ Critical | No |
| 2.2 | Disable Public Key Vault | Built-in | Deny | ⭐⭐⭐ Critical | No |
| 3 | ACR Private Link | Built-in | Audit | ⭐⭐ High | No |
| 4 | Key Vault Private Link | Built-in | Audit | ⭐⭐ High | No |
| 5 | AKS Azure CNI | Custom | Deny | ⭐⭐ High | No |
| 6 | Azure Policy Add-on | Built-in | Audit | ⭐⭐⭐ Critical | No |
| 7 | HTTPS-Only Ingress | Built-in | Deny | ⭐⭐ High | No |
| 8 | Azure RBAC for K8s | Built-in | Audit | ⭐⭐ High | No |
| 9 | Disable Local Accounts | Built-in | Deny | ⭐⭐⭐ Critical | No |
| 10 | API Authorized IPs | Built-in | Audit | ⭐ Medium | No |
| 11 | Defender for Containers | Built-in | Audit | ⭐⭐ High | No |
| 12 | Kubernetes Version Currency | Built-in | Audit | ⭐⭐ High | No |
| 13 | AKS Managed Identity | Built-in | Deny | ⭐⭐ High | No |
| 14 | AKS Standard Tier | Custom | Audit | ⭐ Medium | No |
| 15 | AKS Availability Zones | Custom | Audit | ⭐ Medium | No |
| 16 | AKS Monitoring | Built-in | Audit | ⭐ Medium | No |
| 17 | ACR Premium SKU | Custom | Deny | ⭐⭐ High | No |

**Initiative Categories**:
- **Resource**: Azure resource-level policies (Control Plane)
- **Workload**: Pod/container policies (Data Plane)
- **Security**: Security and threat detection
- **Compliance**: Version and compliance tracking
- **Observability**: Monitoring and logging

---

## 🚀 Deployment Workflow

### Step 1: Verify Management Group Structure
```bash
# List management groups
az account management-group list --query "[].{name:name, displayName:displayName}" -o table

# Verify your target management group exists
az account management-group show \
  --name <your-management-group-name> \
  --expand \
  --recurse
```

### Step 2: Deploy Pod Security Baseline Initiative (FIRST)
```bash
# Assign Microsoft's recommended baseline in AUDIT mode
az policy assignment create \
  --name "aks-pod-security-baseline" \
  --display-name "AKS Pod Security Baseline for Linux Workloads" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {"value": "Audit"},
    "excludedNamespaces": {"value": ["kube-system", "gatekeeper-system", "azure-arc"]}
  }'

# Wait 2-4 weeks, monitor compliance, then switch to Deny
```

### Step 3: Assign Critical Built-in Policies (Private Infrastructure)
```bash
# Policy 1: Enforce private AKS clusters
az policy assignment create \
  --name "enforce-private-aks" \
  --display-name "Require Private AKS Clusters" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/040732e8-d947-40b8-95d6-854c95024bf8" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Deny"}}'

# Policy 2.1: Deny public ACR
az policy assignment create \
  --name "deny-public-acr" \
  --display-name "Deny Public ACR Endpoints" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/0fdf0491-d080-4575-b627-ad0e843cba0f" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Deny"}}'

# Policy 2.2: Deny public Key Vault
az policy assignment create \
  --name "deny-public-keyvault" \
  --display-name "Deny Public Key Vault Endpoints" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/405c5871-3e91-4644-8a63-58e19d68ff5b" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Deny"}}'

# Policy 9: Disable local accounts
az policy assignment create \
  --name "disable-local-accounts" \
  --display-name "Disable AKS Local Accounts" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/993c2fcd-2b29-49d2-9eb0-df2c3a730c32" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Deny"}}'
```

### Step 4: Assign Workload Security Policies
```bash
# Policy 7: HTTPS-only ingress
az policy assignment create \
  --name "require-https-ingress" \
  --display-name "Require HTTPS for All Ingress" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/1a5b4dca-0b6f-4cf5-907c-56316bc1bf3d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{
    "effect": {"value": "Deny"},
    "excludedNamespaces": {"value": ["kube-system", "gatekeeper-system", "azure-arc"]}
  }'
```

### Step 5: Assign Audit-Mode Policies (Governance)
```bash
# Policy 6: Require Azure Policy add-on
az policy assignment create \
  --name "require-aks-policy-addon" \
  --display-name "Require AKS Policy Add-on" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/0a15ec92-a229-4763-bb14-0ea34a568f8d" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Audit"}}'

# Policy 12: Defender for Containers
az policy assignment create \
  --name "require-defender-containers" \
  --display-name "Require Microsoft Defender for Containers" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/c9d007b8-75f5-4c5e-9a2b-9b3a6f1e6f3a" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Audit"}}'

# Policy 13: Kubernetes version currency
az policy assignment create \
  --name "audit-kubernetes-version" \
  --display-name "Audit Kubernetes Version Currency" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/fb893a29-21bb-418c-a157-e99480ec364c" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Audit"}}'

# Policy 17: Azure Monitor
az policy assignment create \
  --name "require-aks-monitoring" \
  --display-name "Require AKS Monitoring" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7" \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --params '{"effect": {"value": "Audit"}}'
```

### Step 6: Create Custom Policies (Optional)
```bash
# Only if you need custom policies for AKS CNI, Standard tier, etc.
# See policy definitions in sections 5, 15, 16, 18 above
```

### Step 7: Validate All Policies
```bash
# Wait 20-30 minutes for policies to propagate

# Check compliance at management group level
az policy state summarize \
  --management-group <your-management-group-name>

# List all policy assignments
az policy assignment list \
  --scope "/providers/Microsoft.Management/managementGroups/<your-management-group-name>" \
  --query "[].{Name:name, DisplayName:displayName, EnforcementMode:enforcementMode}" \
  --output table

# Test on existing AKS cluster (if available)
# Follow validation steps in "Comprehensive Policy Validation" section above
```

### Step 8: Monitor Compliance Dashboard
```bash
# View in Azure Portal:
# https://portal.azure.com > Policy > Compliance
# Filter by Management Group: <your-management-group-name>

# Or use CLI:
az policy state list \
  --management-group <your-management-group-name> \
  --filter "complianceState eq 'NonCompliant'" \
  --query "[].{Resource:resourceId, Policy:policyDefinitionName}" \
  --output table
```

### Deployment Checklist

| Step | Action | Status | Notes |
|------|--------|--------|-------|
| 1 | Verify management group structure | ☐ | Confirm target scope |
| 2 | Deploy Pod Security Baseline (Audit) | ☐ | Microsoft initiative |
| 3 | Assign critical Deny policies | ☐ | Policies 1, 2.1, 2.2, 10 |
| 4 | Assign workload security (Deny) | ☐ | Policy 7 (formerly Policy 8) |
| 5 | Assign governance (Audit) | ☐ | Policies 6, 12, 13, 17 |
| 6 | Create custom policies (optional) | ☐ | Only if needed |
| 7 | Wait 30 min + validate | ☐ | Check compliance |
| 8 | Monitor for 2-4 weeks | ☐ | Review violations |
| 9 | Switch Baseline to Deny mode | ☐ | After validation |
| 10 | Document exemptions | ☐ | If any needed |

---

## 🔄 Policy Lifecycle Management

### Updating Policies

```bash
# Update policy assignment parameters
az policy assignment update \
  --name "enforce-private-aks" \
  --params '{
    "effect": {
      "value": "Audit"  # Change from Deny to Audit for testing
    }
  }'
```

### Exemptions (Use Sparingly)

```bash
# Create exemption for specific resource (e.g., dev/test)
az policy exemption create \
  --name "dev-cluster-exemption" \
  --display-name "Dev Cluster - Public Endpoint Exception" \
  --policy-assignment "/providers/Microsoft.Management/managementGroups/<your-management-group-name>/providers/Microsoft.Authorization/policyAssignments/enforce-private-aks" \
  --scope "/subscriptions/<sub-id>/resourceGroups/dev-rg/providers/Microsoft.ContainerService/managedClusters/dev-cluster" \
  --exemption-category "Waiver" \
  --expires-on "2026-01-31T23:59:59Z"
```

---

## 📞 Support

**Questions about policies?**
- Azure Policy Documentation: https://learn.microsoft.com/en-us/azure/governance/policy/
- Microsoft Support: https://azure.microsoft.com/support/

---

**Last Updated**: November 23, 2025  
**Next Review**: February 23, 2026
