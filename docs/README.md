# Azure Kubernetes Service (AKS) - Enterprise Enablement Guides

**Project**: AKS Enterprise Enablement  
**Version**: 1.0  
**Date**: November 23, 2025  
**Status**: Active  
**Review Cycle**: Quarterly

---

## 📚 Overview

This documentation repository provides comprehensive guides for enterprise-grade Azure Kubernetes Service (AKS) operations. These artefacts are designed to enable your teams with production-ready procedures, governance frameworks, and operational best practices.

---

## 🎯 Deliverables

### 1. **[Private AKS Cluster Deployment Guide](private-aks-cluster-guide.md)** 🔐

Complete documentation set for deploying and managing production-grade private AKS clusters in hub-spoke architecture.

**Includes:**
- ✅ Step-by-step cluster provisioning (SOP)
- ✅ Azure Policy governance framework
- ✅ Workload migration playbook (public → private)
- ✅ Hub-spoke networking scenarios

**Use Cases:**
- Deploy new private AKS clusters with zero-trust networking
- Enforce governance policies at management group level
- Migrate existing workloads from public to private clusters

**📘 [→ Go to Private AKS Cluster Guide](private-aks-cluster-guide.md)**

---

### 2. **[AKS Upgrade Playbook](AKS-Upgrade-Playbook.md)** 🔄

Comprehensive guide for Kubernetes version upgrades, node pool upgrades, and OS SKU migrations with governance and rollback procedures.

**Includes:**
- ✅ Manual and automatic upgrade strategies
- ✅ Pre-upgrade validation checklist
- ✅ Rollback procedures
- ✅ Communication templates for application teams
- ✅ Policy-driven upgrade governance

**Use Cases:**
- Plan and execute Kubernetes version upgrades (e.g., 1.28 → 1.29)
- Upgrade node pool OS (Ubuntu 22.04 → 24.04)
- Coordinate upgrades across multiple application teams
- Implement automated upgrade policies with governance controls

**📘 [→ Go to AKS Upgrade Playbook](AKS-Upgrade-Playbook.md)**

---

## 🚀 Getting Started

### For New Private AKS Deployments
1. **Start here**: [Private AKS Cluster Guide](private-aks-cluster-guide.md)
2. **Deploy governance first**: [Azure Policy Guide](Azure-Policy-Guide.md)
3. **Provision cluster**: [SOP - Provision Private AKS](SOP-provision-private-aks-cluster.md)

### For Existing Cluster Upgrades
1. **Start here**: [AKS Upgrade Playbook](AKS-Upgrade-Playbook.md)
2. **Review pre-upgrade checklist**: Ensure compatibility
3. **Follow upgrade procedures**: Manual or automated path

### For Migrating Workloads to Private Clusters
1. **Review**: [Private AKS Cluster Guide](private-aks-cluster-guide.md)
2. **Follow migration steps**: [Workload Migration SOP](SOP-workload-migration.md)

---

## 📋 Documentation Structure

```
docs/
├── README.md (this file)                          # Main landing page
│
├── Private AKS Cluster Deployment
│   ├── private-aks-cluster-guide.md              # Private AKS overview
│   ├── SOP-provision-private-aks-cluster.md      # Step-by-step provisioning
│   ├── Azure-Policy-Guide.md                     # Governance policies
│   ├── SOP-workload-migration.md                 # Public → Private migration
│   ├── SOP-scenario-existing-hub.md              # Deploy with existing hub
│   └── SOP-scenario-standalone-hub.md            # Deploy standalone hub
│
└── AKS Upgrade Playbook
    └── AKS-Upgrade-Playbook.md                   # Comprehensive upgrade guide
```

---

## ⚠️ Important Notes

### Shell Environment
**IMPORTANT**: All SOPs use **bash shell syntax**. Ensure you are using:
- **Linux**: Native bash terminal
- **macOS**: Native Terminal (bash/zsh)
- **Windows**: Use **WSL2 (Windows Subsystem for Linux)** or **Git Bash**

**Do NOT use PowerShell** - command syntax is incompatible.

### Deployment Order
For new private AKS deployments:
1. **Deploy policies FIRST** → Prevent non-compliant resources
2. **Provision cluster** → Automated compliance enforcement
3. **Deploy applications** → Workloads inherit governance

---

## 📞 Support & Feedback

These artefacts are living documents and should be updated as Azure best practices evolve.

**Review Cycle**: Quarterly  
**Next Review**: February 2026

---

## 📄 Document History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-11-23 | Initial release with Private AKS and Upgrade Playbook | AKS Enablement Team |

---

**🎯 Ready to get started?**
- **New private clusters**: [Private AKS Cluster Guide →](private-aks-cluster-guide.md)
- **Upgrade existing clusters**: [AKS Upgrade Playbook →](AKS-Upgrade-Playbook.md)
