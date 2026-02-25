# Automate Everything: Ansible Automation Platform

[← AI/ML Platform](11-ai-platform.md) | [Table of Contents](../OUTLINE.md) | [Next: Post-Quantum Cryptography →](13-post-quantum-cryptography.md)

---

> **Platform Pillar:** Automation Engine (Ansible Automation Platform)

---

## Executive Summary

Digital sovereignty without automation is theory without practice. Organizations can architect perfectly portable infrastructure, select open source technologies, and design cloud-agnostic applications—yet fail to achieve sovereignty if operational complexity prevents them from actually migrating workloads, enforcing security policies, or managing infrastructure consistently across environments. **Ansible Automation Platform provides the automation engine that makes digital sovereignty operationally viable at enterprise scale.**

Ansible's agentless, human-readable automation works identically across heterogeneous infrastructure: on-premises servers, public clouds (AWS, Azure, GCP), network devices, security appliances, and edge systems. Organizations use Ansible to automate everything from initial RHEL deployment and security hardening, through OpenShift cluster provisioning, to application deployment and compliance validation. Because Ansible automation is defined as code (Infrastructure as Code), organizations gain reproducibility, version control, and audit trails—critical requirements for sovereignty and regulatory compliance.

The platform's upstream-first nature embodies digital sovereignty principles. Ansible is the world's most popular automation language, with a massive community contributing modules, roles, and best practices. Red Hat's Ansible Automation Platform provides enterprise hardening, support, and advanced capabilities (Event-Driven Ansible, automation mesh, analytics) while preserving compatibility with community Ansible. Organizations avoid vendor lock-in—Ansible automation is portable code that runs anywhere—while accessing enterprise features when needed. For Canadian government organizations, Ansible enables the operational consistency required to move workloads between clouds and on-premises infrastructure, enforce security policies at scale, and maintain sovereignty through transparent, auditable automation that isn't tied to any proprietary platform.

---

## Introduction: Automation as the Enabler of Sovereignty

### The Operational Reality Gap

Most digital sovereignty discussions focus on architectural decisions: which operating system, which container platform, which cloud provider. These choices matter, but they're insufficient. Organizations discover that **architectural sovereignty without operational automation leads to three failure modes:**

**1. Migration Paralysis**

Theoretical cloud portability means nothing if migrating a workload requires six months of manual effort, environment-specific configurations, and undocumented tribal knowledge. Teams abandon sovereignty strategies because "it's easier to stay where we are" than to operationalize the move.

**2. Configuration Drift**

Manual operations across heterogeneous environments create drift. AWS infrastructure configured one way, on-premises infrastructure another way, security policies implemented inconsistently. Drift undermines sovereignty—you can't confidently move workloads between environments that aren't actually equivalent.

**3. Compliance Blind Spots**

Regulatory requirements demand continuous compliance validation, not point-in-time assessments. Manual compliance checks don't scale, creating gaps where organizations believe they're compliant but can't prove it.

**Automation bridges this gap.** When infrastructure provisioning, security hardening, application deployment, and compliance validation are defined as code, sovereignty becomes operationally viable. Ansible provides the automation language that works consistently everywhere—making portability practical, not just possible.

### Why Ansible for Digital Sovereignty

Ansible's design principles align perfectly with sovereignty requirements:

**Agentless Architecture**

No agents to install, no daemons to manage, no vendor software running on target systems. Ansible connects over SSH (Linux/Unix) or WinRM (Windows) using native protocols. Organizations maintain full control—no third-party software embedded in infrastructure, no persistent connections to external automation services.

**Human-Readable Automation**

Ansible playbooks use YAML—readable by both humans and machines. Technical auditors can review automation code without specialized training. Compliance officers can understand what automation does. This transparency is critical for sovereignty—you can't claim control over infrastructure you can't understand.

**Declarative, Idempotent Operations**

Describe desired state, not procedures. Ansible ensures systems match that state, regardless of starting point. Re-running playbooks is safe—idempotent operations detect current state and make only necessary changes. This predictability enables confident automation in regulated environments.

**Massive Ecosystem**

7,000+ Ansible modules automate everything: Linux systems, network devices (Cisco, Juniper, Arista), cloud platforms (AWS, Azure, GCP), security tools, databases, monitoring systems. The community has already solved most automation challenges—leverage global knowledge while maintaining local control.

---

## Technical Deep Dive: Ansible Automation Platform

### Core Capabilities

#### Infrastructure as Code (IaC)

Define entire infrastructure stacks as version-controlled Ansible playbooks and roles:

```yaml
# playbooks/rhel-baseline.yml
# Automate RHEL 9 deployment and security hardening
---
- name: RHEL 9 Security Baseline for Sovereignty
  hosts: rhel_servers
  become: true

  tasks:
    # Set timezone for Canadian government systems
    - name: Set timezone to Eastern
      timezone:
        name: America/Toronto

    # Enable FIPS mode for cryptographic compliance
    - name: Enable FIPS 140-2 mode
      command: fips-mode-setup --enable
      register: fips_result
      changed_when: fips_result.rc == 0
      when: ansible_facts['distribution_version'] is version('9.0', '>=')

    # Configure SELinux in enforcing mode
    - name: Ensure SELinux is enforcing
      selinux:
        policy: targeted
        state: enforcing

    # Apply DISA STIG security profile
    - name: Apply STIG security baseline
      include_role:
        name: redhat.rhel_system_roles.security_stig
      vars:
        security_level: high

    # Configure audit rules for Protected B compliance
    - name: Deploy audit rules for Protected B
      copy:
        src: files/audit-protected-b.rules
        dest: /etc/audit/rules.d/protected-b.rules
        owner: root
        group: root
        mode: '0640'
      notify: restart auditd

    # Subscribe to Red Hat Satellite for updates
    - name: Register with Red Hat Satellite
      redhat_subscription:
        state: present
        activationkey: "{{ satellite_activation_key }}"
        org_id: "{{ satellite_org_id }}"
        server_hostname: "{{ satellite_hostname }}"

    # Configure firewall for minimal attack surface
    - name: Configure firewall rules
      firewalld:
        permanent: true
        immediate: true
        service: "{{ item }}"
        state: enabled
      loop:
        - ssh
        - https

  handlers:
    - name: restart auditd
      service:
        name: auditd
        state: restarted
```

**Key Benefits:**
- Single playbook works identically on AWS, Azure, on-premises VMware, bare metal
- Version controlled in Git—full audit trail of infrastructure changes
- Tested in development, promoted through staging, applied in production
- Documented through code—what the automation does is how it's described

#### Multi-Cloud Orchestration

Automate infrastructure across clouds without cloud-specific tooling:

```yaml
# playbooks/hybrid-openshift-deployment.yml
# Deploy OpenShift clusters across AWS (dev), Azure (staging), on-premises (prod)
---
- name: Deploy OpenShift Across Hybrid Cloud
  hosts: localhost
  connection: local

  vars:
    base_domain: canada.gc.ca

  tasks:
    # AWS Development Cluster
    - name: Deploy OpenShift to AWS (Development)
      ansible.builtin.import_role:
        name: openshift_installer
      vars:
        cluster_name: ocp-dev
        cloud_provider: aws
        aws_region: ca-central-1
        compute_instance_type: m5.xlarge
        control_plane_instance_type: m5.2xlarge
        network_cidr: 10.10.0.0/16
        cluster_network_cidr: 10.128.0.0/14
        service_network_cidr: 172.30.0.0/16

    # Azure Staging Cluster
    - name: Deploy OpenShift to Azure Government (Staging)
      ansible.builtin.import_role:
        name: openshift_installer
      vars:
        cluster_name: ocp-staging
        cloud_provider: azure
        azure_region: canadacentral
        compute_instance_type: Standard_D4s_v3
        control_plane_instance_type: Standard_D8s_v3
        network_cidr: 10.20.0.0/16
        cluster_network_cidr: 10.128.0.0/14
        service_network_cidr: 172.30.0.0/16

    # On-Premises Production Cluster (SSC Infrastructure)
    - name: Deploy OpenShift On-Premises (Production)
      ansible.builtin.import_role:
        name: openshift_installer
      vars:
        cluster_name: ocp-prod
        cloud_provider: vsphere
        vcenter_server: "{{ vcenter_hostname }}"
        datacenter: SSC-Ottawa
        datastore: sovereign-datastore
        network: OpenShift-Network
        compute_cpu: 8
        compute_memory: 32768
        control_plane_cpu: 16
        control_plane_memory: 65536
        network_cidr: 10.30.0.0/16
        cluster_network_cidr: 10.128.0.0/14
        service_network_cidr: 172.30.0.0/16

    # Apply consistent security policies to all clusters
    - name: Apply security policies to all clusters
      ansible.builtin.include_role:
        name: openshift_security_policies
      loop:
        - ocp-dev
        - ocp-staging
        - ocp-prod
      loop_control:
        loop_var: cluster_name
```

**Sovereignty Impact:**
- Same automation code deploys to any cloud or on-premises
- No lock-in to cloud-specific tools (CloudFormation, ARM templates, Terraform Cloud)
- Move workloads by re-running automation in different environment
- Consistent security policies enforced everywhere

#### Network Automation

Automate network infrastructure alongside compute:

```yaml
# playbooks/network-sovereignty.yml
# Configure network devices for sovereign infrastructure
---
- name: Configure Network for Hybrid Cloud Sovereignty
  hosts: network_devices
  gather_facts: false

  tasks:
    # Configure VLANs for environment segmentation
    - name: Configure VLANs
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 100
            name: RHEL-Servers
          - vlan_id: 200
            name: OpenShift-Nodes
          - vlan_id: 300
            name: Secure-DMZ
        state: merged
      when: ansible_network_os == 'ios'

    # Configure BGP for cloud interconnects
    - name: Configure BGP for AWS Direct Connect
      cisco.ios.ios_bgp:
        config:
          as_number: 65000
          router_id: 192.0.2.1
          neighbors:
            - neighbor: 169.254.1.1
              remote_as: 65001
              description: AWS Direct Connect
            - neighbor: 169.254.2.1
              remote_as: 65002
              description: Azure ExpressRoute
        state: merged

    # Configure access control lists
    - name: Deploy ACLs for sovereignty compliance
      cisco.ios.ios_acls:
        config:
          - afi: ipv4
            acls:
              - name: BLOCK-FOREIGN-IPS
                acl_type: extended
                aces:
                  - sequence: 10
                    grant: deny
                    protocol: ip
                    source:
                      address: 0.0.0.0
                      wildcard_bits: 255.255.255.255
                    destination:
                      address: 0.0.0.0
                      wildcard_bits: 255.255.255.255
        state: merged
```

**Network Sovereignty:**
- Infrastructure and network automated together
- Network configurations version controlled, auditable
- Consistent policies across sites
- No dependence on vendor-specific network management tools

#### Security and Compliance Automation

Continuous compliance validation and remediation:

```yaml
# playbooks/compliance-validation.yml
# Continuous compliance checking for Protected B systems
---
- name: Validate and Remediate Compliance
  hosts: protected_b_systems
  become: true

  tasks:
    # Run OpenSCAP compliance scan
    - name: Run STIG compliance scan
      ansible.builtin.command:
        cmd: >
          oscap xccdf eval
          --profile xccdf_org.ssgproject.content_profile_stig
          --results /var/tmp/stig-results.xml
          --report /var/tmp/stig-report.html
          /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
      register: oscap_result
      changed_when: false
      failed_when: false

    # Analyze results
    - name: Check compliance score
      ansible.builtin.command:
        cmd: >
          oscap xccdf eval
          --results-arf /var/tmp/stig-results.xml
      register: compliance_score
      changed_when: false
      failed_when: false

    # Remediate failures automatically
    - name: Generate remediation playbook
      ansible.builtin.command:
        cmd: >
          oscap xccdf generate fix
          --profile xccdf_org.ssgproject.content_profile_stig
          --fix-type ansible
          --output /var/tmp/remediation.yml
          /var/tmp/stig-results.xml
      when: oscap_result.rc != 0

    - name: Apply automated remediations
      ansible.builtin.include_tasks:
        file: /var/tmp/remediation.yml
      when: oscap_result.rc != 0

    # Verify post-quantum crypto configuration (RHEL 10.1+)
    - name: Verify PQC policy is active
      ansible.builtin.command:
        cmd: update-crypto-policies --show
      register: crypto_policy
      failed_when: "'PQ-' not in crypto_policy.stdout"
      changed_when: false

    # Log compliance results to central SIEM
    - name: Report compliance status
      ansible.builtin.uri:
        url: "https://siem.canada.gc.ca/api/compliance"
        method: POST
        body_format: json
        body:
          hostname: "{{ ansible_hostname }}"
          compliance_score: "{{ compliance_score.stdout }}"
          timestamp: "{{ ansible_date_time.iso8601 }}"
        headers:
          Authorization: "Bearer {{ siem_api_token }}"
      delegate_to: localhost
```

**Compliance Sovereignty:**
- Continuous validation, not point-in-time audits
- Automated remediation reduces human error
- Audit trails for every compliance check and fix
- No dependency on external compliance tools

#### Event-Driven Ansible

Respond to infrastructure events automatically:

```yaml
# rulebooks/security-response.yml
# Event-Driven Ansible rulebook for security automation
---
- name: Security Event Response Automation
  hosts: all
  sources:
    - ansible.eda.journald:
        match:
          - PRIORITY: "[0-3]"  # Emergency through Error

    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  rules:
    # Respond to SELinux violations
    - name: Remediate SELinux denials
      condition: event.MESSAGE is match(".*avc.*denied.*")
      action:
        run_job_template:
          name: "SELinux-Violation-Analysis"
          organization: "Security-Operations"

    # Respond to failed login attempts
    - name: Block brute force attacks
      condition: event.MESSAGE is match(".*Failed password.*")
      action:
        run_playbook:
          name: "playbooks/security-lockdown.yml"

    # Respond to vulnerability alerts
    - name: Patch critical CVEs immediately
      condition: event.severity == "critical"
      action:
        run_job_template:
          name: "Emergency-Patching"
          organization: "Security-Operations"
          extra_vars:
            cve_id: "{{ event.cve_id }}"
            affected_systems: "{{ event.systems }}"
```

**Event-Driven Sovereignty:**
- Infrastructure self-heals automatically
- Security incidents trigger immediate response
- Reduced mean time to remediation (MTTR)
- No reliance on external security orchestration platforms

### Integration with Red Hat Platform

#### Ansible + RHEL

Ansible is the natural automation layer for RHEL:

```yaml
# roles/rhel-lifecycle/tasks/main.yml
# Automate RHEL lifecycle management
---
- name: Perform in-place upgrade to RHEL 9
  ansible.builtin.include_role:
    name: infra.leapp.upgrade
  vars:
    leapp_upgrade_type: upgrade
    leapp_target_version: "9.4"

- name: Configure system for post-quantum cryptography
  ansible.builtin.command:
    cmd: update-crypto-policies --set PQ-DEFAULT
  when:
    - ansible_distribution == "RedHat"
    - ansible_distribution_major_version|int >= 10

- name: Subscribe to Red Hat Satellite for updates
  redhat_subscription:
    state: present
    activationkey: "{{ satellite_key }}"
    org_id: "{{ org_id }}"
```

#### Ansible + OpenShift

Automate OpenShift operations at scale:

```yaml
# playbooks/openshift-ops.yml
# Day 2 OpenShift operations automation
---
- name: OpenShift Platform Operations
  hosts: localhost
  connection: local

  tasks:
    # Deploy applications via GitOps
    - name: Deploy application with ArgoCD
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: argoproj.io/v1alpha1
          kind: Application
          metadata:
            name: citizen-portal
            namespace: argocd
          spec:
            project: default
            source:
              repoURL: https://git.canada.gc.ca/apps/citizen-portal
              targetRevision: main
              path: k8s
            destination:
              server: https://kubernetes.default.svc
              namespace: citizen-portal
            syncPolicy:
              automated:
                prune: true
                selfHeal: true

    # Backup etcd automatically
    - name: Schedule etcd backup
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: batch/v1
          kind: CronJob
          metadata:
            name: etcd-backup
            namespace: openshift-etcd
          spec:
            schedule: "0 2 * * *"
            jobTemplate:
              spec:
                template:
                  spec:
                    containers:
                      - name: etcd-backup
                        image: quay.io/openshift/origin-cli:latest
                        command:
                          - /bin/bash
                          - -c
                          - oc adm etcd backup --destination=/backup

    # Scale workloads based on schedule
    - name: Scale down non-production during off-hours
      kubernetes.core.k8s_scale:
        api_version: apps/v1
        kind: Deployment
        name: "{{ item }}"
        namespace: development
        replicas: 0
      loop: "{{ dev_workloads }}"
      when: ansible_date_time.hour|int >= 18  # After 6 PM
```

#### Ansible + Ansible Automation Platform

Self-service automation for developers and operators:

- **Automation Controller**: Web UI, RBAC, job scheduling, credentials management
- **Automation Hub**: Private automation content repository
- **Automation Mesh**: Distributed automation execution across geographies
- **Automation Analytics**: Insights into automation usage and efficiency

---

## Real-World Use Case: Canadian Federal Department Migration

### Scenario: Automated Cloud Repatriation

A Canadian federal department needs to migrate 200 applications from AWS back to on-premises Shared Services Canada (SSC) infrastructure to meet new data sovereignty requirements. Manual migration would take 18+ months. Ansible automation reduces this to 3 months.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│ Ansible Automation Platform (Automation Controller)         │
│ • Job Templates for each migration phase                    │
│ • RBAC: DevOps can run, Security approves                  │
│ • Audit logs for compliance reporting                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Executes automation
                            ▼
    ┌───────────────────────────────────────────────────┐
    │                                                   │
    ▼                                                   ▼
┌─────────────────────────┐               ┌───────────────────────────┐
│ AWS Environment         │               │ SSC On-Premises           │
│ • 200 applications      │               │ • Target infrastructure   │
│ • EC2, RDS, S3         │────Migrate───>│ • VMware + RHEL          │
│ • VPC networking        │   with        │ • PostgreSQL on RHEL     │
│                         │   Ansible     │ • Ceph storage           │
└─────────────────────────┘               └───────────────────────────┘
```

**Automation Phases:**

**Phase 1: Discovery and Assessment**
```yaml
- name: Discover AWS infrastructure
  amazon.aws.ec2_instance_info:
    region: ca-central-1
  register: aws_instances

- name: Analyze dependencies
  ansible.builtin.set_fact:
    migration_groups: "{{ aws_instances | group_by_dependencies }}"
```

**Phase 2: Provision Target Infrastructure**
```yaml
- name: Provision VMs on-premises
  community.vmware.vmware_guest:
    hostname: "{{ vcenter_hostname }}"
    datacenter: SSC-Ottawa
    template: rhel9-template
    name: "{{ item.name }}"
    hardware:
      memory_mb: "{{ item.memory }}"
      num_cpus: "{{ item.cpus }}"
  loop: "{{ migration_groups }}"
```

**Phase 3: Data Migration**
```yaml
- name: Migrate S3 data to Ceph
  ansible.builtin.command:
    cmd: >
      aws s3 sync
      s3://{{ source_bucket }}
      s3://{{ target_ceph_bucket }}
      --endpoint-url {{ ceph_s3_endpoint }}
```

**Phase 4: Application Deployment**
```yaml
- name: Deploy applications on-premises
  ansible.builtin.include_role:
    name: application_deployment
  vars:
    target_environment: ssc_onprem
    source_config: "{{ aws_config }}"
```

**Phase 5: Validation and Cutover**
```yaml
- name: Run smoke tests
  ansible.builtin.uri:
    url: "https://{{ item.hostname }}/health"
    status_code: 200
  loop: "{{ migrated_apps }}"

- name: Update DNS for cutover
  community.dns.dns_record:
    zone: canada.gc.ca
    record: "{{ item.name }}"
    type: A
    value: "{{ item.new_ip }}"
  loop: "{{ migrated_apps }}"
```

### Benefits Achieved

**Migration Velocity:**
- 200 applications migrated in 3 months vs. 18 months manual
- Parallel migrations: 10+ applications simultaneously
- Weekend cutovers: automated testing and rollback

**Consistency and Quality:**
- Zero configuration drift: all apps use identical automation
- Reduced errors: automation eliminates manual mistakes
- Repeatable process: same playbooks for all migrations

**Sovereignty Compliance:**
- Data never transits foreign networks (encrypted point-to-point)
- Audit logs prove compliance: every action recorded
- Rollback capability: automated rollback if issues detected

**Cost Savings:**
- $2M saved on cloud egress fees (automated data transfer optimization)
- 60% reduction in migration labor costs
- Reusable automation for future migrations

---

## The Upstream Connection: Ansible Community

### Red Hat's Ansible Contributions

Red Hat created Ansible (originally) and continues massive investment:

**Core Ansible Development:**
- ~200 Red Hat engineers work on Ansible full-time
- Ansible automation language development (Python core)
- Module ecosystem: network, cloud, security, infrastructure
- Performance optimization and scalability improvements

**Community Leadership:**
- Red Hat employees maintain major Ansible collections
- Conference organization: AnsibleFest, regional meetups
- Documentation and learning content
- Integration with CNCF and Linux Foundation projects

**Open Source Commitment:**
- Ansible remains Apache 2.0 licensed (permissive open source)
- All core development happens in public GitHub repositories
- Community feedback drives roadmap
- No "enterprise-only" Ansible features in closed source

### Ansible Collections Ecosystem

**7,000+ Modules** across collections:

**Infrastructure:**
- `ansible.posix` - Linux/Unix system management
- `community.vmware` - VMware automation
- `openstack.cloud` - OpenStack orchestration

**Cloud Platforms:**
- `amazon.aws` - AWS automation (170+ modules)
- `azure.azcollection` - Azure automation
- `google.cloud` - GCP automation

**Network:**
- `cisco.ios` - Cisco IOS automation
- `junipernetworks.junos` - Juniper automation
- `arista.eos` - Arista EOS automation

**Security:**
- `ansible.security` - Security automation framework
- `community.crypto` - Certificate and encryption management
- `fortinet.fortios` - Fortinet firewall automation

**Containers and Kubernetes:**
- `kubernetes.core` - Kubernetes resource management
- `containers.podman` - Podman container automation
- `community.docker` - Docker automation

---

## Key Benefits Summary

### For Technical Teams

**Operational Efficiency:**
- Reduce deployment time from hours to minutes
- Eliminate configuration drift across environments
- Self-service automation reduces wait times
- Event-driven automation enables self-healing infrastructure

**Quality and Reliability:**
- Idempotent operations prevent mistakes
- Version-controlled automation = reproducible results
- Testing automation before production reduces risk
- Rollback capability for failed changes

**Skills and Productivity:**
- Human-readable YAML (not programming language)
- Reusable roles and collections reduce duplication
- Massive community provides solutions to common problems
- Transferable skills across organizations

### For Organizations

**Digital Sovereignty:**
- Portable automation: works on any infrastructure
- No vendor lock-in: Ansible is open source
- Transparent operations: automation is documented through code
- Compliance automation proves sovereignty

**Cost Optimization:**
- Reduce manual operations costs
- Optimize resource utilization through automation
- Accelerate migrations (cloud to on-premises)
- Automation ROI: typical 3-6 month payback

**Risk Mitigation:**
- Consistent operations reduce security vulnerabilities
- Audit trails for all infrastructure changes
- Disaster recovery: infrastructure recreated from code
- Reduced dependence on specific individuals (tribal knowledge in code)

### For Digital Sovereignty

**Operational Independence:**
- Automate infrastructure without cloud-specific tools
- Move workloads between environments confidently
- Enforce security policies consistently everywhere
- Prove compliance through automation logs

**Vendor Independence:**
- Open source automation language (no licensing fees)
- Works with any infrastructure (cloud, on-premises, edge)
- Community-driven: global knowledge, local control
- Commercial support optional, not required

**Regulatory Compliance:**
- Automated compliance validation and remediation
- Audit trails for every change
- Infrastructure as Code enables change review
- Continuous compliance, not point-in-time

---

## References and Further Reading

### Ansible Automation Platform

**Product Documentation:**
- [Ansible Automation Platform Documentation](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/)
- [Ansible Automation Platform Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.4/html/red_hat_ansible_automation_platform_installation_guide/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)

**Red Hat Ansible Resources:**
- [Red Hat Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible)
- [Ansible Automation Hub](https://console.redhat.com/ansible/automation-hub)
- [Red Hat Learning: Ansible Courses](https://www.redhat.com/en/services/training/all-courses-exams?f%5B0%5D=taxonomy_product_line%3AAutomation)

### Upstream Ansible Project

**Community Resources:**
- [Ansible Community](https://www.ansible.com/community)
- [Ansible GitHub Repository](https://github.com/ansible/ansible)
- [Ansible Galaxy (Community Roles)](https://galaxy.ansible.com/)
- [Ansible Collections Index](https://docs.ansible.com/ansible/latest/collections/index.html)

**Learning and Documentation:**
- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible User Guide](https://docs.ansible.com/ansible/latest/user_guide/index.html)
- [Ansible Module Index](https://docs.ansible.com/ansible/latest/collections/index_module.html)

### Event-Driven Ansible

**Event-Driven Automation:**
- [Event-Driven Ansible Documentation](https://ansible.readthedocs.io/projects/rulebook/)
- [Event-Driven Ansible GitHub](https://github.com/ansible/event-driven-ansible)
- [Introduction to Event-Driven Ansible](https://www.redhat.com/en/blog/introducing-event-driven-ansible)

### Infrastructure as Code

**IaC Best Practices:**
- [Infrastructure as Code with Ansible](https://www.redhat.com/en/topics/automation/what-is-infrastructure-as-code-iac)
- [GitOps and Ansible](https://www.redhat.com/en/topics/devops/what-is-gitops)
- [Ansible and Terraform Integration](https://developer.hashicorp.com/terraform/tutorials/provision/ansible)

### Network Automation

**Network DevOps:**
- [Ansible Network Automation](https://docs.ansible.com/ansible/latest/network/index.html)
- [Network Automation with Ansible (Red Hat Training)](https://www.redhat.com/en/services/training/do457-ansible-network-automation)
- [Cisco and Ansible Integration](https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/ansible-cisco-nexus-automation.html)

### Security Automation

**Security and Compliance:**
- [Ansible Security Automation](https://www.ansible.com/use-cases/security-automation)
- [OpenSCAP and Ansible Integration](https://docs.openshift.com/container-platform/latest/security/security_profiles_operator/spo-advanced.html)
- [Automating Security with Ansible](https://www.redhat.com/en/blog/automating-security-ansible-automation-platform)

### Canadian Government Context

**Government Automation:**
- [GC Enterprise Architecture Framework](https://www.canada.ca/en/government/system/digital-government/policies-standards.html)
- [Directive on Service and Digital](https://www.tbs-sct.canada.ca/pol/doc-eng.aspx?id=32601)
- [Protected B Automation Requirements](https://www.cyber.gc.ca/en/guidance/itsp50103-guidance-defence-depth-cloud-based-services)

---

**Previous:** [11. The AI/ML Platform: Enterprise AI with Digital Sovereignty ←](11-ai-platform.md)

**Next:** [13. Post-Quantum Cryptography: Future-Proofing Digital Sovereignty →](13-post-quantum-cryptography.md)
