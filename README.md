# AKS Enablement: Private Cluster Migration Guide

This repository contains comprehensive documentation for migrating Azure Kubernetes Service (AKS) clusters from public to private configurations, including governance policies, detailed migration procedures, and operational best practices.

## 📚 Documentation Overview

This enablement package provides enterprise-ready guidance for deploying and managing private AKS clusters with hub-spoke network architecture.

### Quick Navigation

- **[Start Here: Documentation Hub](docs/README.md)** - Main documentation landing page
- **[Azure Policy Guide](docs/Azure-Policy-Guide.md)** - Governance and compliance policies
- **[Migration SOP](docs/SOP-workload-migration.md)** - Migrate from public to private AKS
- **[Private AKS Deployment](docs/SOP-provision-private-aks-cluster.md)** - Deploy new private clusters
- **[Upgrade Playbook](docs/AKS-Upgrade-Playbook.md)** - Post-migration operations

---

## 🎯 Key Deliverables

### 1. Azure Policy Guide
**File**: [`docs/Azure-Policy-Guide.md`](docs/Azure-Policy-Guide.md)

Comprehensive Azure Policy definitions to enforce private AKS clusters and secure Azure services across your environment.

**Includes**:
- 17 custom policies for AKS security and governance
- Microsoft's Pod Security Baseline Initiative (13+ policies)
- Deployment instructions for management groups
- Validation and testing procedures

### 2. Migration Standard Operating Procedures
**Files**: 
- [`docs/SOP-workload-migration.md`](docs/SOP-workload-migration.md)
- [`docs/SOP-provision-private-aks-cluster.md`](docs/SOP-provision-private-aks-cluster.md)

Detailed step-by-step procedures for migrating production workloads from public to private AKS clusters.

**Includes**:
- Pre-migration assessment and inventory
- Blue-green deployment strategy
- Stateful and stateless workload migration
- Rollback procedures
- Validation and health checks

### 3. AKS Upgrade Playbook
**File**: [`docs/AKS-Upgrade-Playbook.md`](docs/AKS-Upgrade-Playbook.md)

Post-migration operational guide for Kubernetes version upgrades and cluster maintenance.

**Includes**:
- Manual and automatic upgrade strategies
- Pre-upgrade validation scripts
- Communication templates
- Rollback procedures
- Policy-driven upgrade governance

---

## 🚀 Getting Started

### For Policy Deployment
Start with the **[Azure Policy Guide](docs/Azure-Policy-Guide.md)** to establish governance before deploying clusters.

### For New Private AKS Deployment
Follow the **[Private AKS Deployment SOP](docs/SOP-provision-private-aks-cluster.md)** for hub-spoke architecture setup.

### For Migration from Public AKS
Use the **[Migration SOP](docs/SOP-workload-migration.md)** to migrate existing workloads to private clusters.

### For Post-Migration Operations
Reference the **[Upgrade Playbook](docs/AKS-Upgrade-Playbook.md)** for ongoing cluster maintenance.

---

## 📋 Prerequisites

Before starting, ensure you have:
- Azure subscription with Contributor and User Access Administrator roles
- Azure CLI (version 2.50.0 or later)
- kubectl (version 1.27 or later)
- Existing Azure landing zone architecture (hub-spoke recommended)

---

## 🏗️ Architecture Overview

This guidance implements a secure hub-spoke network architecture:

- **Hub VNet**: Azure Firewall, Azure Bastion, DNS resolution
- **Spoke VNet**: Private AKS cluster, private endpoints for ACR/Key Vault
- **Security**: Network Security Groups, User-Defined Routes, Azure Policy
- **Connectivity**: VNet peering, private DNS zones, no public endpoints

---

## 📖 Document Structure

```
docs/
├── README.md                              # Documentation hub
├── Azure-Policy-Guide.md                  # Governance policies
├── SOP-workload-migration.md              # Migration procedures
├── SOP-provision-private-aks-cluster.md   # Private AKS deployment
└── AKS-Upgrade-Playbook.md                # Upgrade and maintenance
```

---

## 🔐 Security Highlights

- ✅ Private AKS clusters (no public API endpoints)
- ✅ Private endpoints for ACR and Key Vault
- ✅ Workload identity for Azure service authentication
- ✅ Azure Policy enforcement for compliance
- ✅ Network isolation with hub-spoke architecture
- ✅ Azure Firewall for egress traffic control

---

## 🎓 Best Practices Included

All documentation follows Microsoft's official AKS best practices:
- [AKS Baseline Architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks)
- [Private AKS Clusters](https://learn.microsoft.com/en-us/azure/aks/private-clusters)
- [AKS Security Best Practices](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security)

---

## 📝 License

This documentation is provided as-is for Azure enablement purposes.

---

## 🤝 Support

For questions or issues, please refer to the official Microsoft documentation links provided throughout the guides.

---

**Last Updated**: November 23, 2025  
**Version**: 1.0
