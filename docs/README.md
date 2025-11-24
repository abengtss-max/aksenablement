# Azure Kubernetes Service (AKS) - Enterprise Enablement Guides

Comprehensive documentation for deploying and managing private AKS clusters with enterprise-grade security and governance.

---

## 📚 Documentation Structure

Follow this logical sequence for successful private AKS implementation:

### 1️⃣ **[Azure Policy Guide](Azure-Policy-Guide.md)** - Governance Framework
**Deploy this FIRST** to enforce compliance before infrastructure deployment.

- 17 custom Azure policies for AKS security
- Pod Security Baseline Initiative (13+ Microsoft policies)
- Management group deployment procedures
- Phased enforcement (Audit → Deny mode)

**Why first?** Policies prevent misconfiguration during deployment.

---

### 2️⃣ **[Private AKS Deployment SOP](SOP-provision-private-aks-cluster.md)** - Infrastructure Setup
**Deploy hub-spoke architecture** with Azure Firewall, Bastion, ACR, and Key Vault.

- Hub-spoke network architecture (10.0.0.0/16 hub, 10.1.0.0/16 spoke)
- Azure Firewall with FQDN egress rules
- Azure Bastion for secure management
- Private endpoints for ACR and Key Vault
- Complete validation checklist

**Outcome:** Production-ready private AKS cluster with zero internet exposure.

---

### 3️⃣ **[Workload Migration SOP](SOP-workload-migration.md)** - Application Migration
**Migrate existing workloads** from public to private AKS clusters.

- Blue-green deployment strategy
- Stateless and stateful workload migration (Velero)
- DNS cutover procedures (60-second TTL)
- Comprehensive rollback procedures (3 scenarios)
- Pre-migration assessment scripts

**Outcome:** Zero-downtime migration with validated rollback options.

---

### 4️⃣ **[AKS Upgrade Playbook](AKS-Upgrade-Playbook.md)** - Post-Migration Operations
**Maintain cluster security** with automated and manual upgrade strategies.

- Auto-upgrade channels (`patch` recommended)
- Pre-upgrade validation (12-check bash script)
- Manual minor version upgrades for stateful workloads
- Rollback procedures
- Communication templates (7-day and 30-day notices)

**Outcome:** Proactive maintenance with minimal disruption.

---

## 🚀 Quick Start Paths

### Path A: New Private AKS Deployment
```
1. Azure Policy Guide → Deploy governance policies
2. Private AKS Deployment SOP → Provision infrastructure
3. Deploy applications → Test in private environment
4. AKS Upgrade Playbook → Configure maintenance strategy
```

### Path B: Migrate Existing Public AKS
```
1. Azure Policy Guide → Audit existing environment
2. Private AKS Deployment SOP → Build new private cluster
3. Workload Migration SOP → Migrate applications
4. AKS Upgrade Playbook → Configure maintenance strategy
```

### Path C: Upgrade Existing Private AKS
```
1. AKS Upgrade Playbook → Follow upgrade procedures
2. Azure Policy Guide (optional) → Review policy compliance
```---

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
