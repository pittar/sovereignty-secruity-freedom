# RHEL Foundation Examples

This directory contains practical examples, configuration templates, and deployment patterns demonstrating how Red Hat Enterprise Linux enables digital sovereignty as the foundational operating system layer.

## Purpose

Section 02 establishes RHEL as **Pillar 1: Ubiquitous Operating System** — the consistent, secure foundation that enables true digital sovereignty across all infrastructure. These examples demonstrate the practical implementation of concepts covered in [02-rhel-foundation.md](../../sections/02-rhel-foundation.md).

## Topics Covered

### 1. Cross-Platform Deployment Consistency

Examples demonstrating identical RHEL behavior across infrastructure platforms:

**Planned Examples:**
- **Multi-Cloud RHEL Provisioning:** Terraform/Ansible configurations deploying identical RHEL 9 across AWS, Azure, GCP, and on-premises VMware
- **Cloud-to-On-Premises Migration:** Step-by-step process migrating RHEL workload from AWS to bare metal with zero application changes
- **Edge Deployment with RHEL for Edge:** Image builder configurations for immutable, minimal-footprint edge systems using rpm-ostree
- **Air-Gapped Installation Guide:** Red Hat Satellite configuration for fully disconnected RHEL deployments in classified environments

### 2. Security Hardening and Compliance

Practical security configurations enabling sovereign, compliant infrastructure:

**Planned Examples:**
- **SELinux Policy Customization:** Creating custom SELinux policies for application confinement beyond default targeted policy
- **FIPS 140-2 Mode Configuration:** Step-by-step FIPS enablement, validation, and troubleshooting common issues
- **STIG Compliance Automation:** Ansible playbooks implementing DISA STIGs for government compliance
- **OpenSCAP Security Scanning:** Automated compliance scanning and remediation with SCAP Security Guide
- **Kernel Lockdown Mode:** Enabling and configuring lockdown mode for high-security environments
- **Post-Quantum Cryptography (RHEL 10.1):** Configuring PQC-enabled cryptography and hybrid mode

### 3. Lifecycle Management

Examples demonstrating RHEL's predictable 10+ year lifecycle management:

**Planned Examples:**
- **In-Place Upgrade (RHEL 8 to RHEL 9):** Using Leapp for major version upgrades with minimal disruption
- **Red Hat Satellite Deployment:** Setting up centralized patch management, content views, and lifecycle environments
- **Extended Update Support (EUS):** Configuring EUS for extended support on minor releases
- **Automated Patching Strategy:** Ansible playbooks for staged, automated security patching across environments

### 4. Multi-Architecture Support

Demonstrating hardware sovereignty through multi-architecture consistency:

**Planned Examples:**
- **x86_64 to ARM64 Migration:** Migrating workloads from x86 to AWS Graviton (ARM) instances
- **Cross-Architecture Container Builds:** Building multi-arch container images (x86/ARM) with buildah/podman
- **Architecture-Specific Optimizations:** Leveraging architecture-specific features while maintaining portability

### 5. Subscription Model and Cost Optimization

Practical examples of RHEL subscription management aligned with sovereignty principles:

**Planned Examples:**
- **Developer Subscription Setup:** Registering and configuring no-cost developer subscriptions for development environments
- **Simple Content Access Configuration:** Enabling SCA for flexible subscription management
- **Cost-Optimized Subscription Strategy:** Mapping production/non-production subscription tiers to business risk

### 6. Hybrid Cloud Architecture Patterns

Real-world deployment patterns enabling sovereignty through hybrid infrastructure:

**Planned Examples:**
- **Canadian Government Hybrid Pattern:** Complete architecture from section 02's use case (AWS dev, Azure staging, on-premises production)
- **Hybrid CI/CD Pipeline:** Jenkins/Tekton pipelines executing across cloud and on-premises RHEL infrastructure
- **Disaster Recovery Across Clouds:** RHEL-based DR strategy with cross-cloud replication
- **Burst-to-Cloud Pattern:** On-premises primary with cloud burst capacity using identical RHEL configurations

### 7. Automation with Ansible

Infrastructure-as-code examples demonstrating environment-agnostic automation:

**Planned Examples:**
- **RHEL Security Baseline Playbook:** Comprehensive Ansible playbook applying hardening across all RHEL systems
- **Multi-Environment Inventory:** Ansible inventory structure for AWS, Azure, VMware, bare metal
- **Idempotent System Configuration:** Ansible roles ensuring consistent RHEL configuration state
- **Integration with Red Hat Satellite:** Ansible playbooks using Satellite for content management

### 8. Container Runtime Configuration

RHEL as the host OS for enterprise container platforms:

**Planned Examples:**
- **Podman Rootless Configuration:** Secure rootless container deployment on RHEL
- **CRI-O for Kubernetes:** Configuring CRI-O container runtime for OpenShift/Kubernetes nodes
- **SELinux Container Confinement:** Leveraging SELinux for container isolation and security
- **FIPS Mode Container Workloads:** Running FIPS-compliant containers on FIPS-enabled RHEL

### 9. Monitoring and Observability

Operational visibility for RHEL infrastructure:

**Planned Examples:**
- **Red Hat Insights Integration:** Proactive system health monitoring and recommendations
- **Prometheus Node Exporter:** Standardized metrics collection from RHEL hosts
- **Centralized Logging with rsyslog:** Log aggregation from distributed RHEL infrastructure
- **Performance Co-Pilot (PCP):** Advanced performance monitoring and analysis

### 10. Migration from Proprietary OS

Sovereignty-driven migration patterns:

**Planned Examples:**
- **Windows Server to RHEL Migration:** Application replatforming patterns and compatibility considerations
- **Converting Existing Linux Distributions:** Migration paths from Ubuntu/CentOS to RHEL
- **Workload Assessment Tools:** Using Red Hat Discovery and Assessment tools for migration planning

## Example Structure

Each example follows this structure:

```
example-name/
├── README.md              # Context, prerequisites, and step-by-step instructions
├── configs/               # Configuration files (Kickstart, Ansible, etc.)
├── scripts/               # Automation scripts
├── docs/                  # Additional documentation, diagrams, decision records
└── tests/                 # Validation tests (Ansible, serverspec, etc.)
```

## Relationship to Other Sections

These examples demonstrate the **foundational OS layer** that enables:
- **Section 03 (Base Images):** RHEL hosts run container runtimes executing UBI-based containers
- **Section 05 (Kubernetes/OpenShift):** Every OpenShift node runs RHEL CoreOS (RHCOS), derived from RHEL
- **Section 06 (CI/CD):** CI/CD pipelines execute on RHEL infrastructure (agents, runners, builders)
- **Section 09 (Confidential Containers):** Hardware-based security (TDX, SEV) requires RHEL kernel support
- **Section 13 (Post-Quantum Cryptography):** PQC implementation begins at RHEL's crypto libraries

RHEL is the infrastructure layer; all other capabilities build upon it.

## Key Differentiation: RHEL vs. Base Images

| Aspect | RHEL (This Section) | UBI/Hummingbird (Section 03) |
|--------|---------------------|------------------------------|
| **What** | Full operating system (kernel, systemd, services) | Minimal container base images |
| **Where** | Host infrastructure (VMs, bare metal, edge) | Inside containers |
| **Lifecycle** | 10-year system lifecycle | Rebuild regularly for security |
| **Security** | SELinux, FIPS, kernel hardening | Package selection, CVE scanning |
| **Use Case** | Infrastructure foundation | Application packaging |

## How to Use These Examples

1. **Start with your environment:** Identify which deployment pattern matches your infrastructure (cloud, on-premises, hybrid, edge, air-gapped)

2. **Follow security baselines:** Begin with security hardening examples (SELinux, FIPS, STIGs) before deploying applications

3. **Automate from the start:** Use Ansible examples for consistent, repeatable configurations

4. **Validate compliance:** Apply OpenSCAP scanning to verify security posture

5. **Plan lifecycle management:** Configure Satellite or subscription management before scaling

## Prerequisites

To work with these examples, you need:

- **Access to RHEL:**
  - Red Hat Developer Subscription (free): https://developers.redhat.com/register
  - Organization's enterprise subscription

- **Infrastructure Access:**
  - Cloud account (AWS/Azure/GCP) for public cloud examples
  - On-premises hypervisor (VMware/KVM) or bare metal for private cloud examples

- **Tools:**
  - Ansible 2.15+
  - Terraform 1.5+ (for IaC examples)
  - Git for version control

- **Knowledge:**
  - Basic Linux administration
  - Understanding of enterprise security requirements
  - Familiarity with your target deployment environment

## Contributing Examples

Examples should:
- Demonstrate real-world scenarios aligned with section 02's sovereignty themes
- Include complete configuration files and step-by-step instructions
- Be tested on current RHEL versions (RHEL 9.x minimum)
- Follow Red Hat best practices and security guidelines
- Include validation steps to verify successful implementation
- Document both "what" (implementation) and "why" (sovereignty benefit)

## Additional Resources

- [Red Hat Enterprise Linux Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9)
- [RHEL Security Hardening Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/)
- [Red Hat Customer Portal](https://access.redhat.com/)
- [Red Hat Developer Program](https://developers.redhat.com/)
- [Red Hat Learning Resources](https://www.redhat.com/en/services/training-and-certification)

## Notes

This directory currently contains planned examples. Contributions are welcome to implement these examples or suggest additional scenarios that demonstrate RHEL's role as the foundational operating system for digital sovereignty.

For questions about specific examples or to propose new examples, please reference the use cases and patterns described in [section 02](../../sections/02-rhel-foundation.md).

---

**Related Sections:**
- [← Section 02: RHEL Foundation](../../sections/02-rhel-foundation.md)
- [→ Section 03: Base Images Examples](../03-base-images/)
- [← Back to Index](../../INDEX.md)
