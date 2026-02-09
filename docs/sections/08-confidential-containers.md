# Advanced Security: Confidential Containers

## Executive Summary

[2-3 paragraphs on the evolution from protecting data at rest and in transit to protecting data in use, the role of hardware-based security with TEEs, and how confidential containers enable secure multi-party computation and regulated workloads.]

Traditional security focuses on protecting data at rest (encryption on disk) and in transit (TLS). But data in use—actively being processed by applications—has remained vulnerable to privileged access and infrastructure compromise. Confidential computing, leveraging hardware-based Trusted Execution Environments (TEEs), extends encryption to data in use. Confidential Containers bring this technology to Kubernetes, enabling organizations to run sensitive workloads with cryptographic guarantees that even cloud providers and infrastructure administrators cannot access the data.

---

## The Data-In-Use Problem

### The Three States of Data

[Explanation of data at rest, in transit, and in use]

**Current State:**
- **At Rest**: Encrypted ✓
- **In Transit**: Encrypted (TLS) ✓
- **In Use**: Plaintext ✗

### Threats to Data in Use

[Who can access data during processing]

**Attack Vectors:**
- Malicious cloud administrators
- Compromised hypervisors
- Memory dumping attacks
- Side-channel attacks
- Nation-state threats

### Regulatory and Compliance Drivers

[GDPR, HIPAA, financial services regulations requiring data-in-use protection]

---

## Confidential Computing Fundamentals

### What is Confidential Computing?

[Introduction to Confidential Computing Consortium and the technology]

**Definition:**
Protection of data in use by performing computation in a hardware-based, attested Trusted Execution Environment.

### Trusted Execution Environments (TEEs)

[Technical explanation of TEE technologies]

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

[How attestation provides cryptographic proof of execution environment]

**Attestation Flow:**
1. TEE generates attestation report
2. Report includes measurements (hashes) of loaded code
3. Report signed by hardware key
4. Verifier checks signature and measurements
5. Encrypted secrets released only if valid

---

## Confidential Containers Project

### What are Confidential Containers?

[Introduction to the CNCF Confidential Containers project]

**Goals:**
- Make confidential computing accessible to Kubernetes users
- Minimal changes to existing workflows
- Support multiple TEE technologies
- Open source and vendor-neutral

### Architecture Overview

[Technical architecture of Confidential Containers]

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

[Explanation of runtime class and annotations]

### Image Encryption

[How to encrypt container images]

```bash
# Encrypt a container image
skopeo copy \
  --encryption-key provider:attestation-agent:kbs:///default/image-key \
  docker://registry.example.com/app:v1 \
  docker://registry.example.com/encrypted-app:v1
```

### Secret Management with Attestation

[How secrets are released only to attested TEEs]

---

## Use Cases for Confidential Containers

### 1. Regulated Workloads (Healthcare, Finance)

[Processing sensitive data with regulatory requirements]

**Example: Healthcare Data Processing**
- Patient data encrypted at rest and in transit
- Data processed in confidential containers
- Even cloud provider cannot access patient data
- Cryptographic proof of compliance

### 2. Multi-Party Computation

[Scenarios where multiple organizations collaborate on sensitive data]

**Example: Financial Fraud Detection**
- Multiple banks share fraud patterns
- Data never leaves TEEs unencrypted
- No single party can access raw data
- Collaborative analysis without trust

### 3. Confidential AI/ML Workloads

[Training and inference on sensitive data]

**Example: Federated Learning**
- Model training on encrypted customer data
- Gradients computed in TEEs
- Model provider cannot access training data
- Data owner cannot access model weights

### 4. Edge Computing with Sensitive Data

[Processing at the edge with strong security guarantees]

---

## Integration with OpenShift

### Confidential Containers on OpenShift

[How Red Hat integrates confidential containers with OpenShift]

### Hardware Requirements

[What infrastructure is needed for confidential computing]

**Supported Platforms:**
- Bare metal with Intel TDX or AMD SEV-SNP
- Public cloud instances with confidential computing support
  - Azure Confidential Computing VMs
  - AWS EC2 instances with AMD SEV-SNP
  - Google Cloud Confidential VMs

---

## Performance Considerations

### Overhead and Trade-offs

[Performance impact of running in TEEs]

**Typical Overhead:**
- Memory encryption: 3-10% performance impact
- Attestation: One-time startup cost
- Encrypted images: Minimal runtime impact

### When to Use Confidential Containers

[Decision criteria for adopting confidential computing]

---

## The Upstream Connection

### Red Hat's Confidential Containers Contributions

[Specific contributions to CNCF Confidential Containers project]

### Collaboration with Hardware Vendors

[Working with Intel, AMD, ARM on TEE support]

---

## Security Model and Threat Analysis

### What Confidential Containers Protect Against

[Detailed threat model]

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

[How to maximize security when using confidential containers]

---

## Real-World Example: Secure Multi-Cloud AI

[Scenario: Financial institution running fraud detection across clouds]

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

[What's coming next in confidential computing]

### Standardization Efforts

[Industry standards for confidential computing]

---

## Key Benefits Summary

**For Technical Teams:**
- Hardware-based security guarantees
- Integration with existing Kubernetes workflows
- Open source and vendor-neutral

**For Organizations:**
- Meet stringent compliance requirements
- Enable multi-party collaboration
- Reduce insider threat risk

**For Digital Sovereignty:**
- Run sensitive workloads on any infrastructure
- Cryptographic independence from cloud providers
- No trust required in infrastructure operators

---

## References and Further Reading

- [Confidential Computing Consortium](https://confidentialcomputing.io/)
- [CNCF Confidential Containers Project](https://github.com/confidential-containers)
- [Intel TDX Documentation]
- [AMD SEV Documentation]
- [Kata Containers](https://katacontainers.io/)
