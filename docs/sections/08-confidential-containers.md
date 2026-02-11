# Advanced Security: Confidential Containers

[← Platform Engineering](07-platform-engineering.md) | [Table of Contents](../OUTLINE.md) | [Next: Workload Identity →](09-workload-identity.md)

---

## Executive Summary

Security professionals have long secured data at rest through disk encryption and data in transit through TLS. Yet the most critical state—data in use during active processing—has remained exposed. Application memory contains plaintext data accessible to privileged users, hypervisors, and cloud administrators. For Canadian government workloads processing citizen data, healthcare records, financial transactions, and classified information, this exposure creates unacceptable risk and potential regulatory non-compliance.

**Confidential computing transforms existing public cloud infrastructure into sovereignty-compliant platforms.** Using hardware-based Trusted Execution Environments (TEEs)—isolated memory regions protected by CPU-level encryption—technologies like Intel TDX and AMD SEV-SNP provide cryptographic guarantees that cloud administrators cannot access workload memory even with full infrastructure privileges. Remote attestation enables cryptographic verification of the execution environment before releasing secrets. This technology is revolutionary for Canadian organizations already operating on AWS, Azure, or Google Cloud: you can immediately enhance security to meet Protected B requirements without migrating infrastructure, rewriting applications, or sacrificing cloud economics.

Confidential Containers brings this capability to Kubernetes through the CNCF Confidential Containers project. Organizations run sensitive workloads in TEE-backed pods with minimal workflow changes—same kubectl commands, same deployment patterns. For Canadian departments, this means processing sensitive citizen data on existing public cloud infrastructure with mathematical proofs of protection, meeting TBS security requirements through hardware-enforced isolation rather than trust in foreign cloud providers. Digital sovereignty extends beyond infrastructure location to cryptographic independence from infrastructure operators—you control your data even when using someone else's computers.

---

## The Data-In-Use Problem

### The Three States of Data

Data exists in three states, each requiring different protection mechanisms. Data at rest (stored on disks or object storage) faces threats from physical theft, unauthorized access, and backup exposure—mitigated through encryption at rest using AES-256 or similar algorithms. Data in transit (moving across networks) faces interception, man-in-the-middle attacks, and eavesdropping—protected through TLS, VPNs, and encrypted connections. Data in use (loaded in application memory during processing) faces the most exposure: operating systems, hypervisors, administrators, and memory-scanning attacks can access plaintext data. Traditional security architectures have no defense for this state—the assumption that "trusted" infrastructure protects memory has failed repeatedly through privilege escalation, insider threats, and cloud provider compromises.

**Current State:**
- **At Rest**: Encrypted ✓
- **In Transit**: Encrypted (TLS) ✓
- **In Use**: Plaintext ✗

### Threats to Data in Use

Privileged access is the primary threat to data in use. Cloud administrators with root access can dump process memory, attach debuggers, or install kernel modules that extract data from running applications. Hypervisors managing virtual machines have complete visibility into guest memory—a compromised hypervisor exposes all workload data. Operating system vulnerabilities enable privilege escalation allowing unprivileged users to read arbitrary memory. Nation-state actors with physical hardware access can perform cold-boot attacks extracting encryption keys from RAM. Even legitimate operations like crash dumps and debugging sessions create data exposure risks. Traditional access controls and audit logs provide detection, not prevention—data has already leaked once accessed.

**Attack Vectors:**
- Malicious cloud administrators
- Compromised hypervisors
- Memory dumping attacks
- Side-channel attacks
- Nation-state threats

### Regulatory and Compliance Drivers

GDPR Article 32 requires "appropriate technical and organizational measures" including encryption of personal data. Courts increasingly interpret this to include data-in-use protection when processing occurs in untrusted environments like public clouds. HIPAA Security Rule mandates safeguards for electronic protected health information (ePHI) during storage, transmission, and processing—creating liability when PHI exists in plaintext memory accessible to cloud providers. Financial services regulations (PCI-DSS, SOX, financial data privacy laws) increasingly demand proof that sensitive data remains protected during processing, not just storage. Confidential computing provides this proof through cryptographic attestation, transforming compliance from audit theater to mathematical verification.

---

## Confidential Computing Fundamentals

### What is Confidential Computing?

The Confidential Computing Consortium (CCC), a Linux Foundation project founded by Intel, Microsoft, Google, Red Hat, and others, defines confidential computing as protecting data in use through hardware-based Trusted Execution Environments. Unlike software-based isolation (containers, VMs) which trust the operating system and hypervisor, confidential computing uses CPU hardware features to create cryptographically isolated memory regions. The CPU encrypts memory contents with keys inaccessible to software, ensuring that even privileged code cannot read protected data. Remote attestation provides cryptographic proof of the execution environment's integrity before workloads receive sensitive data. This shifts security from "trust the cloud provider" to "verify through cryptography."

**Definition:**
Protection of data in use by performing computation in a hardware-based, attested Trusted Execution Environment.

### Trusted Execution Environments (TEEs)

TEEs are hardware-enforced isolated execution environments where code and data remain protected from the host operating system, hypervisor, and other software. Modern CPUs include dedicated silicon for memory encryption, integrity verification, and attestation. Each TEE technology offers different trade-offs between isolation granularity, memory size limits, and performance overhead. Intel SGX isolates individual processes in small enclaves with strong guarantees but limited memory. Intel TDX and AMD SEV isolate entire virtual machines, supporting larger workloads at slightly higher overhead. The CPU generates and manages encryption keys internally, preventing software access. This hardware root of trust provides security properties impossible through software alone.

**Major TEE Technologies:**

1. **Intel SGX** (Software Guard Extensions)
   - Enclave-based protection
   - Small trusted computing base
   - Limited memory size

2. **Intel TDX** (Trust Domain Extensions)
   - VM-level protection
   - Full VM encryption
   - Larger workloads

3. **AMD SEV** (Secure Encrypted Virtualization)
   - SEV: Memory encryption
   - SEV-ES: Register state encryption
   - SEV-SNP: Enhanced attestation

4. **ARM CCA** (Confidential Compute Architecture)
   - Realms for isolation
   - Growing adoption in edge/IoT

### Remote Attestation

Attestation enables cryptographic verification of a TEE's integrity before releasing secrets. When a workload starts in a TEE, the hardware measures (cryptographically hashes) the code loaded into the protected environment—kernel, drivers, application binaries. The TEE generates an attestation report containing these measurements, hardware identity, and security configuration, signed with a hardware-based key that cannot be forged by software. Remote verifiers check the signature using the CPU manufacturer's public keys, compare measurements against expected values, and verify security settings meet requirements. Only after successful attestation do verifiers release encryption keys and sensitive data to the TEE. This process ensures you're communicating with genuine hardware running known, unmodified code.

**Attestation Flow:**
1. TEE generates attestation report
2. Report includes measurements (hashes) of loaded code
3. Report signed by hardware key
4. Verifier checks signature and measurements
5. Encrypted secrets released only if valid

---

## Confidential Containers Project

### What are Confidential Containers?

The CNCF Confidential Containers project makes confidential computing accessible to Kubernetes users without requiring TEE expertise or workflow changes. Rather than forcing developers to learn new APIs or restructure applications, Confidential Containers extends standard Kubernetes with TEE-backed runtimes activated through runtime class selection. Developers deploy pods normally, Kubernetes schedules them to confidential computing-capable nodes, and the runtime automatically provisions TEE-protected VMs for workload execution. This abstraction enables gradual adoption—organizations can run confidential and non-confidential workloads side-by-side in the same cluster, moving workloads to TEE protection based on sensitivity rather than refactoring everything simultaneously.

**Goals:**
- Make confidential computing accessible to Kubernetes users
- Minimal changes to existing workflows
- Support multiple TEE technologies
- Open source and vendor-neutral

### Architecture Overview

Confidential Containers builds on Kata Containers, a CNCF project providing VM-based container runtime with strong isolation. Traditional container runtimes share the host kernel—adequate for process isolation but insufficient for confidential computing. Kata runs each pod in a lightweight VM with its own kernel, creating a hardware-enforced boundary. When combined with TEE technologies, this VM becomes a confidential guest protected by CPU encryption. The attestation agent inside the TEE performs remote attestation with the Key Broker Service, proving execution environment integrity. Upon successful attestation, the KBS releases decryption keys enabling the pod to access encrypted container images and secrets. This architecture ensures secrets never exist in plaintext outside TEEs—encrypted at rest, encrypted in transit, encrypted in use except within TEE-protected memory.

**Key Components:**

1. **Kata Containers Runtime**
   - Lightweight VM-based container runtime
   - Each pod runs in its own VM
   - Hardware-enforced isolation

2. **Attestation Agent**
   - Runs inside TEE
   - Performs attestation
   - Retrieves secrets

3. **Key Broker Service (KBS)**
   - Stores encrypted secrets
   - Verifies attestation
   - Releases secrets to valid TEEs

4. **Image Security**
   - Encrypted container images
   - Images decrypted only inside TEE
   - Prevents unauthorized inspection

```
┌─────────────────────────────────────────┐
│         Kubernetes Cluster              │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │  Confidential Pod                 │ │
│  │  ┌─────────────────────────────┐  │ │
│  │  │  Trusted Execution Env      │  │ │
│  │  │  (Intel TDX / AMD SEV-SNP)  │  │ │
│  │  │                             │  │ │
│  │  │  ┌─────────────────────┐    │  │ │
│  │  │  │  Container          │    │  │ │
│  │  │  │  (Encrypted image)  │    │  │ │
│  │  │  │                     │    │  │ │
│  │  │  │  Processing         │    │  │ │
│  │  │  │  Encrypted Data     │    │  │ │
│  │  │  └─────────────────────┘    │  │ │
│  │  │                             │  │ │
│  │  │  Attestation Agent          │  │ │
│  │  └─────────────────────────────┘  │ │
│  │                                   │ │
│  └───────────────────────────────────┘ │
│                                         │
│  Key Broker Service (KBS)               │
│  - Verifies attestation                 │
│  - Releases secrets                     │
└─────────────────────────────────────────┘
```

---

## Technical Deep Dive: Running Confidential Workloads

### Pod Configuration for Confidential Containers

```yaml
# Example: Confidential Container pod
apiVersion: v1
kind: Pod
metadata:
  name: confidential-workload
  annotations:
    io.katacontainers.config.runtime.cc: "true"
spec:
  runtimeClassName: kata-cc
  containers:
    - name: secure-app
      image: registry.example.com/encrypted-app:v1
      env:
        - name: ATTESTATION_URL
          value: "https://kbs.example.com"
      volumeMounts:
        - name: encrypted-data
          mountPath: /data
  volumes:
    - name: encrypted-data
      secret:
        secretName: confidential-data
```

The `runtimeClassName: kata-cc` field directs Kubernetes to use the Kata Containers runtime with confidential computing extensions. The annotation `io.katacontainers.config.runtime.cc: "true"` explicitly enables TEE protection for this pod. The attestation URL points to the Key Broker Service which will verify the TEE's attestation before releasing secrets. From the developer perspective, this is the only difference from standard pod definitions—TEE protection activates through configuration, not code changes.

### Image Encryption

Container image encryption prevents unauthorized inspection of application code and data. Encrypted images remain ciphertext in registries and during distribution—only TEEs with valid attestation can decrypt them. The encryption key is wrapped with a key encryption key (KEK) managed by the KBS. When a confidential pod starts, the attestation agent proves TEE integrity to the KBS, receives the KEK, unwraps the image encryption key, and decrypts the container image in protected memory. This ensures proprietary algorithms, embedded credentials, and sensitive data in images never exist in plaintext outside TEEs.

```bash
# Encrypt a container image
skopeo copy \
  --encryption-key provider:attestation-agent:kbs:///default/image-key \
  docker://registry.example.com/app:v1 \
  docker://registry.example.com/encrypted-app:v1
```

### Secret Management with Attestation

The Key Broker Service (KBS) implements policy-based secret release tied to attestation verification. Secrets are encrypted and stored in the KBS. When a TEE requests a secret, it presents an attestation report. The KBS verifies the report's cryptographic signature, checks that measurements match expected values (correct kernel, unmodified binaries, required security features enabled), and evaluates policies (only specific images can access certain secrets, minimum TEE version requirements, geographic restrictions). If all checks pass, the KBS encrypts the secret with the TEE's public key and returns it—only that specific TEE can decrypt it. This zero-trust model eliminates "privileged access" to secrets—even KBS administrators cannot retrieve plaintext secrets or bypass attestation requirements.

**→ See the complete example:** [Confidential Containers: Encrypted Workload Deployment](../examples/08-confidential-containers/encrypted-workload-deployment.md) for complete deployment with TEE protection, encrypted images, KBS configuration, and attestation-based secret management.

---

## Use Cases for Confidential Containers

### 1. Canadian Government Workloads on Public Cloud

Canadian federal departments and provincial agencies often operate on public cloud infrastructure (AWS, Azure, Google Cloud) for specific workloads, yet face sovereignty concerns about foreign cloud administrator access to citizen data. Confidential containers enable sovereignty-compliant cloud operations: citizen PII, healthcare records, and Protected B data remain encrypted until loaded into TEE-protected containers; cloud provider administrators cannot access sensitive data even with root and hypervisor privileges; attestation provides auditable cryptographic proof meeting TBS security categorization requirements. This transforms cloud deployment from "trust the provider" to "verify through hardware cryptography."

**Example: Service Canada Benefit Processing**
- Citizen benefit applications processed on public cloud for elasticity
- Personal information encrypted at rest, in transit, and in use
- TEE protection ensures AWS/Azure administrators cannot access citizen data
- Hardware attestation provides evidence for security authorization
- Meet Protected B requirements on foreign cloud infrastructure

### 2. Regulated Workloads (Healthcare, Finance)

Healthcare organizations must protect patient data under PIPEDA and provincial health privacy laws, yet need to leverage cloud infrastructure for scalability and advanced analytics. Confidential containers enable compliant cloud processing: patient records remain encrypted until loaded into TEE-protected containers, cloud provider administrators cannot access PHI even with root privileges, attestation provides auditable proof of protection meeting regulatory requirements. Financial institutions similarly process sensitive transactions, customer data, and trading algorithms in confidential containers—achieving cloud economics without exposing regulated data to cloud infrastructure operators.

**Example: Provincial Health Authority Analytics**
- Patient data encrypted at rest and in transit
- Data processed in confidential containers on public cloud
- Even cloud provider cannot access patient data
- Cryptographic proof of compliance with provincial privacy laws

### 2. Multi-Party Computation

Multiple organizations often need to collaborate on analysis without sharing raw data—banks detecting fraud patterns, healthcare providers researching treatments, advertisers measuring campaign effectiveness without exposing customer data. Confidential containers enable secure multi-party computation: each party encrypts their data with keys released only to attested TEEs, computation occurs in confidential containers processing all parties' data, results are extracted while raw data never leaves TEE protection, no single party (including the infrastructure provider) can access others' data. This eliminates the need for mutual trust—cryptography replaces contracts.

**Example: Financial Fraud Detection**
- Multiple banks share fraud patterns
- Data never leaves TEEs unencrypted
- No single party can access raw data
- Collaborative analysis without trust

### 3. Confidential AI/ML Workloads

AI model training requires large datasets often containing sensitive information—customer behavior, medical images, financial transactions. Model providers want to train on this data without accessing it directly. Data owners want insights without exposing raw data or allowing model extraction. Confidential containers solve this conflict: training data remains encrypted except within TEE protection, model training code runs in confidential containers, data owners verify through attestation that only approved training code accesses their data, trained model weights extract without exposing training data, inference workloads process queries in TEEs protecting both model and input data.

**Example: Federated Learning**
- Model training on encrypted customer data
- Gradients computed in TEEs
- Model provider cannot access training data
- Data owner cannot access model weights

### 4. Edge Computing with Sensitive Data

Edge deployments process sensitive data in physically insecure locations—retail stores, manufacturing facilities, remote field sites. Traditional security assumes physical security and trusted operators, but edge environments often have neither. Confidential containers extend security to untrusted edge: surveillance footage processes in TEE-protected containers at edge locations, even local IT staff cannot access camera feeds or processed results, sensor data from industrial equipment remains encrypted during edge processing, attestation provides remote verification of edge security even when physical access is compromised. This enables edge computing deployment in hostile environments while maintaining cryptographic security guarantees.

---

## Integration with OpenShift

### Confidential Containers on OpenShift

OpenShift integrates Confidential Containers through operators that automate deployment and lifecycle management of TEE-enabled runtimes. The Confidential Containers operator installs and configures Kata Containers with TEE support, provisions Key Broker Service infrastructure, and manages attestation service configurations. OpenShift's node feature discovery automatically identifies hardware with TEE capabilities, enabling scheduler-based workload placement on confidential computing-capable nodes. Integration with OpenShift's security and compliance operators enables policy-driven confidential workload deployment—security teams define which workloads require TEE protection, policies automatically enforce runtime class selection, and audit logs track confidential workload compliance.

### Hardware Requirements

Confidential computing requires CPUs with specific TEE technologies and BIOS/firmware configuration enabling these features. Public cloud providers offer confidential computing instances—Azure's DCsv3/DCasv5 series, AWS EC2 instances with AMD SEV-SNP, Google Confidential VMs—that abstract hardware complexity. On-premises deployments need recent server hardware (Intel 4th gen Xeon Scalable or later for TDX, AMD EPYC 3rd gen or later for SEV-SNP) with firmware updates enabling confidential computing. Network isolation and secure boot configurations enhance TEE security. Organizations can start with cloud instances for development and testing, then migrate to on-premises hardware for production when sovereignty or cost considerations dictate.

**Supported Platforms:**
- Bare metal with Intel TDX or AMD SEV-SNP
- Public cloud instances with confidential computing support
  - Azure Confidential Computing VMs
  - AWS EC2 instances with AMD SEV-SNP
  - Google Cloud Confidential VMs

---

## Performance Considerations

### Overhead and Trade-offs

TEE protection introduces performance overhead from memory encryption, attestation procedures, and VM-based isolation. Modern TEE implementations (TDX, SEV-SNP) incur 3-10% performance impact for most workloads through memory encryption overhead—acceptable for security-sensitive applications but noticeable for performance-critical workloads. Attestation happens at workload startup, creating one-time latency (500ms-2s) that impacts short-lived tasks more than long-running services. VM-based isolation (Kata Containers) adds memory overhead and slight I/O performance impact compared to shared-kernel containers. Network-intensive applications may see higher overhead from encrypted network tunnels between TEEs. Organizations should benchmark representative workloads to measure actual impact before committing production deployments.

**Typical Overhead:**
- Memory encryption: 3-10% performance impact
- Attestation: One-time startup cost
- Encrypted images: Minimal runtime impact

### When to Use Confidential Containers

Adopt confidential containers when threat model includes infrastructure compromise, regulatory requirements mandate data-in-use protection, multi-party computation requires cryptographic guarantees rather than contractual trust, or IP protection demands preventing cloud provider access to proprietary algorithms. Don't adopt when performance overhead is unacceptable, workloads don't process sensitive data, threat model focuses on external attackers rather than privileged access, or hardware availability constraints prevent deployment. Start with highest-sensitivity workloads—financial transactions, healthcare records, AI models on sensitive data—and expand based on results. Not all workloads require confidential computing; apply based on data sensitivity and regulatory requirements.

---

## The Upstream Connection

### Red Hat's Confidential Containers Contributions

Red Hat engineers serve as maintainers of the CNCF Confidential Containers project, contributing core functionality for Kata Containers TEE integration, Key Broker Service implementation, and attestation protocols. Red Hat contributed OpenShift-specific operators for confidential containers deployment, RHEL-optimized guest kernels for TEE VMs, and integration with Red Hat's security and compliance tooling. Engineering investment includes ~10 engineers dedicated to upstream confidential containers development, with additional teams working on RHEL and OpenShift integration. Red Hat participates in Confidential Computing Consortium standards development, ensuring interoperability across TEE technologies and cloud providers.

### Collaboration with Hardware Vendors

Red Hat works directly with Intel, AMD, and ARM on TEE technology development and validation. This collaboration includes early access to hardware for testing and validation, feedback on TEE API design and capabilities, joint development of reference implementations and deployment guides, and certification programs ensuring RHEL and OpenShift work reliably on confidential computing hardware. Red Hat's position as major Linux contributor and enterprise platform provider gives unique influence on TEE evolution—ensuring enterprise requirements (reliability, manageability, performance) inform hardware design rather than being afterthoughts.

---

## Security Model and Threat Analysis

### What Confidential Containers Protect Against

Confidential containers protect against privileged software threats: malicious cloud administrators cannot access TEE memory even with root access and kernel modules; compromised hypervisors cannot read or modify workload memory protected by CPU encryption; operating system vulnerabilities enabling privilege escalation fail to breach TEE boundaries; memory-scraping attacks targeting sensitive data in RAM encounter only encrypted contents; insider threats from infrastructure operators meet cryptographic enforcement, not policy. The threat model assumes CPU hardware and firmware are trustworthy (validated through attestation), network communications are encrypted, and application code is free from vulnerabilities—TEEs protect data from infrastructure, not from application bugs.

**Protected:**
- Malicious cloud administrators
- Compromised hypervisor
- Memory attacks
- OS-level vulnerabilities in host

**Not Protected:**
- Vulnerabilities in application code
- Side-channel attacks (partial mitigation)
- Physical access to hardware

### Security Best Practices

Minimize TEE attack surface by running minimal container images (UBI Minimal or Hummingbird) reducing code subject to vulnerabilities. Verify attestation evidence carefully—don't accept any attestation, verify measurements match expected values and security configuration meets requirements. Encrypt all data before it reaches infrastructure—even with TEE protection, defense in depth demands end-to-end encryption. Regularly update TEE firmware and platform software as vendors patch vulnerabilities. Monitor attestation failures as potential indicators of compromise. Implement network segmentation isolating confidential workloads from untrusted networks. Audit key broker access patterns for anomalies. Remember TEEs protect data from infrastructure, not from application vulnerabilities—still require secure coding practices, input validation, and vulnerability management.

---

## Real-World Example: Secure Multi-Cloud AI

A global financial institution develops fraud detection models using transaction data from multiple regional subsidiaries, each operating in different regulatory jurisdictions with strict data residency requirements. Model training requires aggregating data across regions, but regulations prohibit moving raw transaction data outside jurisdictional boundaries. Cloud infrastructure offers needed compute power but regulators demand proof that cloud providers cannot access customer financial data. Traditional approaches (on-premises only or data aggregation through legal agreements) are inadequate—on-premises limits compute scalability, legal agreements don't provide technical guarantees.

**Requirements:**
- Customer transaction data must remain encrypted
- Model training across multiple data sources
- Compliance with financial regulations
- Multi-cloud deployment (AWS and on-prem)

**Solution:**
1. Deploy OpenShift with confidential containers support
2. Encrypt customer data and container images
3. Run ML training in confidential pods
4. Use attestation to verify execution environment
5. Model results extracted, raw data never exposed

**Benefits:**
- Cryptographic proof of data protection
- Cloud independence maintained
- Compliance requirements met
- Collaboration without data sharing

---

## Future Directions

### Emerging TEE Technologies

Next-generation TEE technologies focus on reducing overhead, increasing memory limits, and expanding capabilities. Intel TDX and AMD SEV-SNP represent current production technologies, while future CPU generations promise improved performance and larger encrypted memory regions. ARM's Confidential Compute Architecture targets edge and mobile devices, extending confidential computing beyond datacenter servers. RISC-V processors are incorporating TEE capabilities through open specifications, enabling custom confidential computing silicon. GPU confidential computing emerges for AI workloads requiring hardware acceleration with TEE protection. These advances will make confidential computing default rather than special-case security enhancement.

### Standardization Efforts

The Confidential Computing Consortium drives industry standards for attestation formats, key exchange protocols, and TEE capabilities discovery. IETF develops standards for remote attestation procedures and secure enclaves. NIST provides guidance on confidential computing for government use cases. These standardization efforts ensure interoperability—attestation evidence generated by Intel hardware verifiable by AMD-based validators, applications portable across TEE technologies, tooling working across vendors. Standardization prevents vendor lock-in at the confidential computing layer, preserving sovereignty even when leveraging hardware-based security features.

---

## Key Benefits Summary

**For Technical Teams:**
- Hardware-based security guarantees through CPU-level encryption
- Integration with existing Kubernetes workflows and OpenShift
- Open source and vendor-neutral (works on any TEE-capable hardware)

**For Organizations:**
- **Enhance existing cloud security immediately**—no migration required to improve protection
- Meet stringent compliance requirements (TBS Protected B, PIPEDA, provincial privacy laws)
- Enable multi-party collaboration without trust or data sharing
- Reduce insider threat and privileged access risk

**For Digital Sovereignty:**
- **Use existing cloud providers more securely** through cryptographic guarantees
- Cryptographic independence from cloud infrastructure operators
- No trust required in foreign cloud administrators or hypervisors
- Process Canadian citizen data on any infrastructure while maintaining sovereignty
- Maintain option to repatriate to on-premises when policy requires

---

## References and Further Reading

- [Confidential Computing Consortium](https://confidentialcomputing.io/)
- [CNCF Confidential Containers Project](https://github.com/confidential-containers)
- [Intel TDX Documentation](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html)
- [AMD SEV Documentation](https://www.amd.com/en/developer/sev.html)
- [Kata Containers](https://katacontainers.io/)

---

**Next:** [9. Zero Trust Workload Identity →](09-workload-identity.md)
