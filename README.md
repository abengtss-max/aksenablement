# AKS Enablement

This repository contains comprehensive documentation for migrating Azure Kubernetes Service (AKS) clusters from public to private configurations, including governance policies, detailed migration procedures, and operational best practices.

## 📚 Documentation Overview

This enablement package provides enterprise-ready guidance for deploying and managing private AKS clusters with hub-spoke network architecture.

### Quick Navigation

- **[Start Here: Documentation Hub](docs/README.md)** - Main documentation landing page
- **[Azure Policy Guide](docs/Azure-Policy-Guide.md)** - Governance and compliance policies
- **[Private AKS Deployment](docs/SOP-provision-private-aks-cluster.md)** - Deploy new private clusters
- **[Migration SOP](docs/SOP-workload-migration.md)** - Migrate workloads to private AKS
- **[Upgrade Playbook](docs/AKS-Upgrade-Playbook.md)** - Post-migration operations

---

## 🎯 Key Deliverables

### 1. Azure Policy Guide
**File**: [`docs/Azure-Policy-Guide.md`](docs/Azure-Policy-Guide.md)

Comprehensive Azure Policy definitions to enforce private AKS clusters and secure Azure services across your environment.

**Includes**:
- 16 custom policies for AKS security and governance
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

- **Hub VNet** (10.0.0.0/22): Azure Firewall, Azure Bastion, DNS resolution
- **Spoke VNet** (10.100.0.0/20): Private AKS cluster, private endpoints for ACR/Key Vault
- **Security**: Network Security Groups, User-Defined Routes, Azure Policy
- **Connectivity**: VNet peering, private DNS zones, private endpoints only

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

- ✅ Private AKS clusters with private API endpoints only
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

## ⚠️ Disclaimer

This documentation is provided for educational and informational purposes only. While every effort has been made to ensure accuracy and adherence to Microsoft Azure best practices, the author makes no warranties or representations regarding the completeness, accuracy, or suitability of this content for any particular purpose.

**Important Notice:**
- This guidance is provided "as-is" without warranty of any kind, either express or implied
- Users are solely responsible for testing, validating, and implementing these procedures in their own environments
- Always conduct thorough testing in non-production environments before implementing changes in production systems
- Verify all configurations against your organization's specific security policies and compliance requirements

By using this documentation, you acknowledge and agree that you do so at your own risk and that the author shall not be held liable for any consequences arising from its use.

---

## 📝 License

This documentation is provided as-is for Azure enablement purposes.

---

## 🤝 Support

For questions or issues, please refer to the official Microsoft documentation links provided throughout the guides.

---

**Last Updated**: November 23, 2025  
**Version**: 1.0
