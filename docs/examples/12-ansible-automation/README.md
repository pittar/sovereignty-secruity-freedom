# Ansible Automation Examples

This directory contains practical Ansible automation examples demonstrating how to achieve digital sovereignty through Infrastructure as Code across hybrid cloud environments.

## Purpose

Section 12 establishes **Ansible Automation Platform as Pillar 3: Automation Engine** — the operational layer that makes digital sovereignty practical at enterprise scale. These examples demonstrate real-world automation patterns for sovereign infrastructure.

## Topics Covered

### 1. RHEL Lifecycle Automation

Examples automating Red Hat Enterprise Linux deployment, configuration, and lifecycle management:

**Planned Examples:**
- **RHEL Baseline Configuration:** Complete security hardening playbooks (SELinux, FIPS, STIGs, audit rules)
- **RHEL Lifecycle Management:** In-place upgrades with Leapp, patching strategies, satellite integration
- **Multi-Architecture Deployment:** Provisioning RHEL across x86_64, ARM64, s390x architectures
- **Post-Quantum Crypto Configuration:** Enabling PQC on RHEL 10.1+ systems
- **Compliance Automation:** OpenSCAP scanning and automated remediation

### 2. Hybrid Cloud Infrastructure

Automation for cloud-agnostic infrastructure provisioning:

**Planned Examples:**
- **Multi-Cloud OpenShift Deployment:** Deploy OpenShift to AWS, Azure, on-premises with consistent automation
- **Cloud Migration Automation:** Automated workload migration from AWS/Azure to on-premises
- **Infrastructure as Code Patterns:** Reusable roles and playbooks for hybrid infrastructure
- **Cloud Cost Optimization:** Automated resource scheduling and rightsizing

### 3. Network Automation

Infrastructure and network configuration as code:

**Planned Examples:**
- **BGP Configuration for Cloud Interconnects:** AWS Direct Connect, Azure ExpressRoute automation
- **VLAN and Network Segmentation:** Automated network isolation for security zones
- **Firewall Rule Management:** Consistent security policies across environments
- **SD-WAN Configuration:** Hybrid cloud connectivity automation

### 4. Security and Compliance Automation

Continuous security validation and remediation:

**Planned Examples:**
- **STIG Compliance Validation:** Automated DISA STIG scanning and remediation
- **Certificate Management:** Automated certificate lifecycle (generation, renewal, deployment)
- **Security Event Response:** Event-Driven Ansible for security incident automation
- **Vulnerability Patching:** Automated CVE remediation workflows

### 5. OpenShift Day 2 Operations

Kubernetes/OpenShift operational automation:

**Planned Examples:**
- **GitOps Application Deployment:** ArgoCD and Ansible integration
- **Cluster Backup and Recovery:** Automated etcd backup, disaster recovery procedures
- **Multi-Cluster Management:** Consistent policies across OpenShift clusters
- **Resource Quota and Limit Automation:** Automated capacity management

### 6. Event-Driven Ansible

Reactive automation responding to infrastructure events:

**Planned Examples:**
- **Self-Healing Infrastructure:** Automatic remediation of common failures
- **Security Incident Response:** Automated response to security events
- **Performance-Driven Scaling:** Event-triggered resource scaling
- **Compliance Drift Detection:** Automated detection and correction of configuration drift

### 7. Application Lifecycle Automation

End-to-end application deployment automation:

**Planned Examples:**
- **CI/CD Integration:** Ansible in Jenkins, Tekton, GitLab pipelines
- **Blue-Green Deployments:** Zero-downtime deployment automation
- **Database Migration Automation:** Automated schema updates and data migration
- **Application Health Monitoring:** Automated health checks and rollback

### 8. Canadian Government Use Cases

Sovereignty-focused automation patterns:

**Planned Examples:**
- **Protected B Compliance Automation:** Automated compliance for sensitive government data
- **Bilingual Application Deployment:** French/English deployment automation
- **Air-Gapped Environment Automation:** Automation in disconnected government networks
- **Federal Cloud Migration:** Complete migration playbooks for cloud repatriation

## Example Structure

Each example follows this structure:

```
example-name/
├── README.md                    # Context, prerequisites, instructions
├── playbooks/                   # Ansible playbooks
│   ├── site.yml                # Main playbook
│   └── *.yml                   # Additional playbooks
├── roles/                       # Ansible roles (if applicable)
├── inventories/                 # Inventory files for different environments
│   ├── development/
│   ├── staging/
│   └── production/
├── group_vars/                  # Group variables
├── host_vars/                   # Host variables
└── tests/                       # Ansible Molecule tests (if applicable)
```

## Relationship to Other Sections

These examples demonstrate **operational sovereignty** by automating:
- **Section 02 (RHEL):** OS deployment, security hardening, lifecycle management
- **Section 05 (OpenShift):** Kubernetes cluster provisioning and operations
- **Section 04 (Supply Chain):** Automated signing, scanning, compliance validation
- **Section 13 (PQC):** Post-quantum cryptography configuration automation
- **Section 09 (Confidential Containers):** Secure workload deployment automation

Ansible is the operational layer that makes all other pillars practical at scale.

## Prerequisites

To work with these examples, you need:

- **Ansible Access:**
  - Ansible Core 2.15+ (community, free)
  - Ansible Automation Platform 2.4+ (enterprise features)

- **Infrastructure Access:**
  - RHEL systems (physical, virtual, cloud)
  - Network access to target systems (SSH, WinRM)
  - Cloud credentials (AWS, Azure, GCP) for cloud examples

- **Tools:**
  - Git for version control
  - Python 3.9+ (Ansible dependency)
  - SSH key-based authentication configured

- **Knowledge:**
  - Basic YAML syntax
  - Linux/Unix administration
  - Network and cloud concepts
  - Ansible fundamentals

## Running Examples

### 1. Clone Repository
```bash
git clone https://github.com/your-org/sovereignty-paper
cd sovereignty-paper/docs/examples/12-ansible-automation
```

### 2. Install Ansible
```bash
# Community Ansible (free)
python3 -m pip install ansible

# Or use system packages
sudo dnf install ansible-core  # RHEL/Fedora
```

### 3. Configure Inventory
```bash
# Edit inventory for your environment
vi inventories/development/hosts.yml
```

### 4. Run Playbooks
```bash
# Syntax check first
ansible-playbook playbooks/site.yml --syntax-check

# Dry run (check mode)
ansible-playbook playbooks/site.yml --check

# Execute
ansible-playbook playbooks/site.yml
```

## Best Practices

### Version Control
- All automation in Git
- Encrypted secrets with Ansible Vault
- Branch strategy for environments (dev/staging/prod)

### Idempotency
- Design playbooks to be re-runnable
- Use Ansible modules (not shell commands) when possible
- Test idempotency: run twice, should be unchanged second time

### Security
- Never commit secrets in plain text
- Use Ansible Vault for sensitive variables
- Rotate credentials regularly
- Follow principle of least privilege

### Testing
- Test in development first
- Use Ansible Molecule for role testing
- Validate syntax before execution
- Document expected outcomes

### Documentation
- README for each playbook/role
- Inline comments for complex logic
- Variable descriptions in defaults/main.yml
- Maintain runbook documentation

## Contributing Examples

Examples should:
- Demonstrate sovereignty-aligned automation patterns
- Work across hybrid cloud (AWS, Azure, on-premises)
- Follow Ansible best practices
- Include complete documentation
- Be tested on current Ansible versions
- Document Canadian government compliance aspects (where applicable)

## Additional Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Automation Platform Docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/)
- [Ansible Galaxy](https://galaxy.ansible.com/) - Community roles and collections
- [Red Hat Ansible Learning](https://www.redhat.com/en/services/training/all-courses-exams?f%5B0%5D=taxonomy_product_line%3AAutomation)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)

## Notes

This directory currently contains planned examples. Contributions are welcome to implement automation patterns that demonstrate operational sovereignty at scale.

For questions about specific examples or to propose new automation patterns, please reference the use cases and patterns described in [section 12](../../sections/12-ansible-automation.md).

---

**Related Sections:**
- [← Section 12: Ansible Automation](../../sections/12-ansible-automation.md)
- [→ Section 13: Post-Quantum Cryptography Examples](../13-post-quantum-cryptography/)
- [← Back to Index](../../INDEX.md)
