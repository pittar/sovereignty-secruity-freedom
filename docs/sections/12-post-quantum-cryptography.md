# Post-Quantum Cryptography: Future-Proofing Digital Sovereignty

[← AI Platform](10-ai-platform.md) | [Table of Contents](../OUTLINE.md) | [Next: Conclusion →](13-conclusion.md)

---

## Executive Summary

Today's encryption—the foundation of digital security, privacy, and sovereignty—faces an existential threat. RSA, elliptic curve cryptography, and other public-key algorithms securing internet communications, VPNs, digital signatures, and encrypted data will become obsolete when large-scale quantum computers arrive. Cryptographers estimate this quantum threat will materialize within 10-20 years, but adversaries are already acting. The "harvest now, decrypt later" attack collects encrypted data today for future decryption when quantum computers become available—meaning sensitive government data, intellectual property, and citizen information encrypted with current algorithms face retroactive compromise years from now.

For Canadian government organizations, post-quantum cryptography (PQC) isn't optional—it's existential for long-term data sovereignty. Protected B data, classified information, citizen records, diplomatic communications, and long-lived secrets (infrastructure credentials, encryption keys, identity documents) must remain secure for decades. Current encryption algorithms provide zero protection against future quantum decryption. The Treasury Board Secretariat's security categorization requires cryptographic controls matching data sensitivity and lifespan; algorithms vulnerable to known future threats fail this requirement. Federal systems must transition to quantum-resistant cryptography now, before adversaries complete their harvest operations and before quantum computers arrive.

NIST finalized three post-quantum cryptography standards in August 2024 (FIPS 203, 204, 205) after an eight-year global evaluation process. Red Hat responded rapidly: **RHEL 10.1, released in early 2026, became the first major Linux distribution to enable post-quantum cryptography by default in its standard cryptographic policy**. TLS and SSH connections automatically use quantum-resistant key exchange (ML-KEM) where both endpoints support it. Red Hat signed RHEL packages with hybrid post-quantum signatures (ML-DSA + Ed448), protecting software supply chain integrity against future quantum attacks. Because RHEL underpins Red Hat's entire product portfolio—OpenShift, Ansible, Red Hat Enterprise Linux AI, Developer Hub—this PQC foundation enables quantum-safe operation across the full technology stack. Organizations deploying on RHEL-based platforms today inherit quantum resistance without application rewrites, protecting digital sovereignty through the quantum transition and beyond.

---

## The Quantum Computing Threat

### What Makes Quantum Computers Different

Classical computers process information as bits (0 or 1), performing operations sequentially or through massive parallelization. Quantum computers leverage quantum mechanical phenomena—superposition (existing in multiple states simultaneously) and entanglement (correlated states across quantum bits)—to explore solution spaces exponentially faster than classical computers for specific problem types. For factoring large numbers and solving discrete logarithm problems (the mathematical foundations of RSA and elliptic curve cryptography), quantum computers provide exponential speedup. A problem requiring billions of years on classical supercomputers could solve in hours on a sufficiently large quantum computer.

### Shor's Algorithm and Cryptographic Vulnerability

In 1994, mathematician Peter Shor proved that quantum computers can factor large numbers and solve discrete logarithms efficiently. Shor's algorithm breaks RSA encryption (factoring the product of two large primes) and elliptic curve cryptography (solving discrete logarithms on elliptic curves) in polynomial time—exponentially faster than the best-known classical algorithms. This means the encryption protecting:

- **TLS/HTTPS connections** encrypting web traffic, API calls, and cloud service communication
- **SSH sessions** securing remote administration of servers and infrastructure
- **VPN tunnels** protecting data in transit across networks
- **Digital signatures** verifying software authenticity, document integrity, and identity
- **Encrypted storage** protecting data at rest in databases, file systems, and backups

...will become retroactively decryptable when large-scale quantum computers emerge.

### The Timeline: When Will Quantum Computers Break Encryption?

Quantum computing progress accelerates but faces substantial engineering challenges. Current quantum computers demonstrate dozens to hundreds of qubits with high error rates, requiring error correction that consumes most computational capacity. Breaking 2048-bit RSA encryption (current standard) requires roughly 20 million noisy qubits or 4,000 logical qubits with error correction—far beyond today's capabilities.

**Expert estimates vary widely:**
- **Optimistic projections**: 10-15 years until cryptographically relevant quantum computers
- **Conservative estimates**: 20-30 years, citing fundamental engineering obstacles
- **Government planning**: NSA, NIST, and intelligence agencies worldwide assume 10-20 year timeline for transition planning

Regardless of precise timeline, two critical points drive urgency:

1. **"Harvest Now, Decrypt Later" is happening today**: Adversaries with long-term intelligence objectives are capturing encrypted traffic, storing databases, and collecting encrypted data for future decryption. State-sponsored actors have both resources and motivation to warehouse decades of encrypted communications for retroactive intelligence gathering once quantum computers mature.

2. **Cryptographic transitions take decades**: The migration from MD5 to SHA-256, from SSLv3 to TLS 1.3, and from 1024-bit RSA to 2048-bit keys each took 10-15 years to achieve majority deployment. Organizations must begin PQC adoption now to complete transition before quantum computers arrive.

### Canadian Government Implications

The Canadian Security Intelligence Service (CSIS), Communications Security Establishment (CSE), and Canadian Centre for Cyber Security (CCCS) face quantum threats to classified communications, intelligence operations, and long-term secrets. Beyond national security agencies, federal departments handling citizen data must consider data lifespan:

- **Healthcare records**: Protected for 75+ years, containing lifetime medical history
- **Pension and benefits data**: Decades of financial and personal information
- **Immigration and citizenship records**: Permanent identity documents and background information
- **Tax records**: Multi-decade financial data subject to strict privacy requirements
- **Infrastructure credentials**: Root certificates, signing keys, and encryption keys with 10-30 year lifespans

Data encrypted today with RSA or elliptic curve algorithms and harvested by adversaries becomes readable in 10-20 years—well within the protection period required for most government data. CCCS guidance and TBS security categorization increasingly recognize quantum threats as imminent rather than theoretical, requiring PQC adoption as standard practice for new deployments and accelerated migration for existing systems handling long-lived sensitive data.

---

## Post-Quantum Cryptography Standards

### NIST's PQC Standardization Process

In 2016, NIST launched a global competition to identify quantum-resistant cryptographic algorithms, receiving 82 initial submissions from cryptographers worldwide. Over eight years, NIST evaluated algorithms through multiple rounds of cryptanalysis, performance testing, and public review. In August 2024, NIST finalized three post-quantum cryptography standards:

**FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM)**
- Based on the CRYSTALS-Kyber algorithm
- **Purpose**: Key establishment for encryption (replacing RSA key exchange and ECDH)
- **Use cases**: TLS connections, VPN key exchange, encrypted communications
- **Security levels**: ML-KEM-512, ML-KEM-768, ML-KEM-1024 (equivalent to AES-128, AES-192, AES-256)
- **Performance**: Fast key generation and encapsulation, reasonable key sizes

**FIPS 204: Module-Lattice-Based Digital Signature Algorithm (ML-DSA)**
- Based on the CRYSTALS-Dilithium algorithm
- **Purpose**: Digital signatures (replacing RSA signatures and ECDSA)
- **Use cases**: Software signing, document authentication, TLS certificates, identity verification
- **Security levels**: ML-DSA-44, ML-DSA-65, ML-DSA-87 (increasing security)
- **Trade-offs**: Larger signature sizes than classical algorithms but fast signing/verification

**FIPS 205: Stateless Hash-Based Digital Signature Algorithm (SLH-DSA)**
- Based on SPHINCS+ algorithm
- **Purpose**: Alternative digital signature scheme with different security assumptions
- **Use cases**: High-security signing where performance is less critical
- **Security basis**: Hash function security (different mathematical foundation than ML-DSA)
- **Trade-offs**: Larger signatures, slower performance, but conservative security assumptions

### Mathematical Foundations: Why These Algorithms Resist Quantum Attacks

Post-quantum algorithms rely on mathematical problems believed to be hard even for quantum computers:

**Lattice Problems (ML-KEM, ML-DSA)**: Finding short vectors in high-dimensional lattices is computationally hard for both classical and quantum computers. No quantum algorithm provides significant speedup for lattice problems, making them ideal for post-quantum cryptography. Lattice-based schemes offer good performance and reasonable key/signature sizes.

**Hash-Based Signatures (SLH-DSA)**: Security depends only on the properties of cryptographic hash functions. If SHA-256 or SHA-3 remain secure against quantum computers (current belief), then hash-based signatures remain secure. This provides conservative fallback if lattice-based cryptography faces unexpected attacks.

### Hybrid Approaches: Bridging Classical and Quantum-Resistant Cryptography

During transition, hybrid cryptographic schemes combine classical and post-quantum algorithms, providing security if either remains unbroken:

**Hybrid Key Exchange**: Combine ML-KEM with ECDH (elliptic curve Diffie-Hellman). The resulting key derives from both algorithms—protected against quantum computers by ML-KEM, protected against unexpected ML-KEM breaks by ECDH. TLS 1.3 extensions support hybrid key exchange, enabling gradual PQC adoption without abandoning proven classical cryptography.

**Hybrid Digital Signatures**: Combine ML-DSA with Ed25519 or RSA. Signatures verify only if both signatures are valid. Red Hat's RPM package signing uses this approach: packages are signed with both ML-DSA-87 and Ed448, ensuring verification succeeds on quantum-vulnerable systems (Ed448 only) and quantum-resistant systems (both algorithms).

Hybrid approaches provide migration path from classical to post-quantum cryptography, enabling organizations to adopt PQC gradually while maintaining compatibility with systems not yet updated.

---

## Red Hat's Leadership in Post-Quantum Cryptography

### RHEL 10.1: First Major Linux Distribution with PQC Enabled by Default

Red Hat Enterprise Linux 10.1, released in early 2026, represents a watershed moment for quantum-safe computing. RHEL 10.1 became **the first and currently only major Linux distribution** to enable post-quantum cryptography by default in its standard cryptographic policy. This isn't experimental or opt-in—it's production-ready, fully supported, and active for all RHEL 10.1 systems unless explicitly disabled.

**What "Enabled by Default" Means:**

When RHEL 10.1 systems establish TLS connections (HTTPS, secure APIs) or SSH sessions, they automatically negotiate post-quantum key exchange using ML-KEM if the remote endpoint supports it. Applications using OpenSSL, GnuTLS, or NSS—the standard cryptographic libraries in RHEL—inherit PQC support without code changes. Network connections become quantum-resistant transparently as organizations deploy RHEL 10.1.

**Cryptographic Library Support:**

- **OpenSSL**: Full support for ML-KEM key exchange and ML-DSA signatures in TLS 1.3
- **GnuTLS**: ML-KEM and ML-DSA support with hybrid mode compatibility
- **NSS (Network Security Services)**: Mozilla's crypto library with PQC support
- **Go crypto libraries**: Native ML-KEM and ML-DSA implementations

All libraries are generally available (GA) and fully supported—not technology preview. Organizations can deploy PQC in production with Red Hat support and SLAs.

### Quantum-Resistant Package Signing

Red Hat's RPM package signing infrastructure underwent comprehensive upgrade to support post-quantum cryptography. All RHEL 10.1 packages are signed with **hybrid signatures combining ML-DSA-87 (quantum-resistant) and Ed448 (classical elliptic curve)**. This hybrid approach ensures:

- **Quantum resistance**: ML-DSA-87 protects against future quantum attacks
- **Backward compatibility**: Ed448 verifies on older systems not yet supporting PQC
- **Defense in depth**: Both algorithms must be broken to forge signatures

This milestone protects software supply chain integrity against "harvest now, forge later" attacks where adversaries capture signed packages today, waiting for quantum computers to forge malicious signatures in the future.

### RHEL as Foundation for Red Hat Portfolio

Red Hat's comprehensive product portfolio—OpenShift, Ansible Automation Platform, Red Hat OpenShift AI, Red Hat Developer Hub, Red Hat Enterprise Linux AI—builds on RHEL as the operating system and runtime foundation. RHEL 10.1's PQC support automatically propagates throughout the ecosystem:

**OpenShift**: Kubernetes platform inherits RHEL PQC for control plane communication, pod-to-pod encryption, and external API access
**Ansible**: Automation workflows use RHEL cryptography for SSH connections to managed systems
**OpenShift AI**: ML platform benefits from quantum-resistant data protection for training data and model serving
**Developer Hub**: Platform engineering tools secure with PQC for authentication and API access

Organizations deploying Red Hat products on RHEL 10.1 foundation gain quantum resistance across the entire stack without individual product updates or application modifications. This architectural leverage—shared operating system foundation across products—enables Red Hat to deliver PQC comprehensively and rapidly.

---

## Technical Deep Dive: PQC in Practice

### TLS with Post-Quantum Key Exchange

TLS 1.3 establishes encrypted connections using key exchange algorithms that derive shared secrets between client and server. Classical TLS uses ECDH (elliptic curve Diffie-Hellman)—vulnerable to quantum computers. TLS with ML-KEM replaces or augments ECDH with quantum-resistant key exchange.

**Hybrid X25519+ML-KEM-768 Key Exchange (RHEL 10.1 Default):**

```bash
# OpenSSL s_client connecting to PQC-enabled server
openssl s_client -connect example.com:443 -groups X25519MLKEM768

# Output shows hybrid key exchange:
# Peer signing digest: SHA256
# Peer signature type: ML-DSA
# Server Temp Key: X25519MLKEM768, 1216 bits
# TLS session cipher: TLS_AES_256_GCM_SHA384
```

The connection uses both X25519 (classical elliptic curve) and ML-KEM-768 (post-quantum) for key establishment. The resulting encryption key depends on both—remaining secure if either algorithm resists attack.

**Performance Impact:**

ML-KEM adds minimal overhead to TLS handshakes. Key exchange operations complete in microseconds, with slightly larger handshake messages (~1-2KB additional data). For most applications, performance impact is negligible—latency increases by less than 5%, well within acceptable bounds for production use.

### SSH with ML-KEM Key Exchange

SSH protects remote administration of Linux servers, Kubernetes nodes, and cloud instances. RHEL 10.1 OpenSSH supports ML-KEM for quantum-resistant SSH connections:

```bash
# SSH with hybrid X25519+ML-KEM-768 key exchange
ssh -oKexAlgorithms=sntrup761x25519-sha512@openssh.com user@server.example.com

# SSH client and server negotiate PQC if both support it
# Legacy systems fall back to classical key exchange
```

SSH connections between RHEL 10.1 systems automatically use hybrid key exchange, protecting against quantum eavesdropping on administrative sessions.

### Package Signature Verification

RPM package management verifies signatures before installation, ensuring packages haven't been tampered with and originate from trusted sources. RHEL 10.1 rpm tool supports hybrid signature verification:

```bash
# Verify package signature (automatic during yum/dnf install)
rpm --checksig kernel-6.14.0-1.el10_1.x86_64.rpm

# Output shows both classical and PQC signature verification:
# kernel-6.14.0-1.el10_1.x86_64.rpm: digests signatures OK
# - Ed448 signature: OK
# - ML-DSA-87 signature: OK
```

Older RHEL versions (RHEL 8, RHEL 9) verify only the Ed448 signature, maintaining backward compatibility. RHEL 10.1 verifies both signatures, rejecting packages where either fails validation.

---

## Canadian Government PQC Adoption Strategy

### Federal Requirements and Timeline

Canadian government organizations must align PQC adoption with Treasury Board Secretariat guidance, CCCS recommendations, and data classification requirements. While specific deadlines remain under development as of 2026, clear priorities emerge:

**Immediate Priority (2026-2027): Protect High-Value Targets from Harvest Attacks**
- Classified communications (Secret and Top Secret)
- Long-lived credentials (root certificates, signing keys)
- Strategic data with decades-long sensitivity (intelligence, diplomatic)
- Critical infrastructure control systems

**Near-Term (2027-2030): Protected B and Sensitive Government Data**
- Citizen records with long protection periods (healthcare, pensions, immigration)
- Federal financial systems and tax data
- Law enforcement and justice system data
- Departmental secrets and operational data

**Long-Term (2030-2035): Comprehensive Migration**
- All government systems transition to PQC by NIST's deprecation timeline
- Classical algorithms removed from approved cryptographic profiles
- Legacy systems remediated or decommissioned

### Deployment Patterns for Canadian Departments

**Pattern 1: New Systems on RHEL 10.1 Foundation**

Departments deploying new applications on RHEL 10.1 inherit PQC automatically. TLS-based services (web applications, microservices, APIs) use quantum-resistant key exchange without code changes. SSH administrative access gains PQC protection. This pattern requires no application-level PQC awareness—operating system foundation provides quantum resistance.

**Pattern 2: Hybrid Infrastructure with Progressive Migration**

Existing infrastructure (RHEL 8, RHEL 9) coexists with new RHEL 10.1 deployments during multi-year migration. Hybrid cryptography ensures interoperability: RHEL 10.1 systems negotiate PQC when connecting to other PQC-capable systems, falling back to classical cryptography for legacy endpoints. This enables gradual rollout without "flag day" cutover.

**Pattern 3: Air-Gapped Environments with Manual PQC Deployment**

Classified and air-gapped systems receive RHEL 10.1 through secure media transfer procedures. Package signatures verify using both classical and PQC algorithms, protecting against future quantum-enabled supply chain attacks. Air-gapped PQC deployment requires no internet connectivity—all cryptographic operations occur locally using pre-distributed trust roots.

### Compliance with TBS and CCCS Guidance

Treasury Board's security categorization requires cryptographic controls matching data sensitivity and lifespan. For Protected B data with decades-long sensitivity, quantum-vulnerable encryption fails to meet control requirements once quantum threats are recognized as feasible. CCCS guidance on cryptographic algorithms increasingly emphasizes PQC adoption timelines aligned with NIST recommendations.

**Audit and Compliance Benefits:**

- **Cryptographic Inventory**: RHEL 10.1 provides system-wide view of algorithms in use, enabling compliance reporting
- **Policy-Based Configuration**: Cryptographic policies enforce PQC usage consistently across infrastructure
- **Transition Tracking**: Organizations monitor PQC adoption progress through system policies and connection logs
- **Evidence of Due Diligence**: Early PQC adoption demonstrates proactive risk management for audit and oversight

---

## The Upstream Connection

### Red Hat's Contributions to Open Source PQC

Red Hat engineers contribute extensively to upstream projects implementing post-quantum cryptography:

**OpenSSL**: Red Hat maintains OpenSSL packages in RHEL and contributes to upstream ML-KEM and ML-DSA implementation, performance optimization, and TLS integration. Red Hat's OpenSSL expertise (decades of experience maintaining OpenSSL in RHEL) positions the company to ensure PQC implementations meet enterprise requirements for stability, performance, and security.

**GnuTLS**: Red Hat supports GnuTLS development, contributing PQC algorithm implementations and TLS protocol extensions. GnuTLS provides alternative to OpenSSL, offering diversity in cryptographic library ecosystem.

**NSS (Network Security Services)**: Red Hat collaborates with Mozilla Foundation on NSS, the cryptographic library underlying Firefox and used extensively in Red Hat products. PQC support in NSS enables quantum-resistant cryptography in web browsers, email clients, and identity systems.

**Linux Kernel**: Cryptographic APIs in the Linux kernel support PQC algorithms for kernel-mode cryptography (IPsec, dm-crypt, etc.). Red Hat kernel engineers ensure ML-KEM and ML-DSA integrate cleanly with kernel cryptographic framework.

**FIDO Alliance and WebAuthn**: Red Hat participates in standards bodies defining PQC integration for web authentication, ensuring passkeys and strong authentication mechanisms transition to quantum-resistant algorithms.

### Collaboration with Standards Bodies

Red Hat engineers participate in IETF (Internet Engineering Task Force) working groups defining PQC protocol integration for TLS, SSH, IPsec, and other network protocols. These standards ensure interoperability across vendors and platforms—Red Hat's PQC implementations work with Microsoft, Google, Cisco, and other ecosystem participants.

Red Hat collaborates with NIST during PQC standardization, providing feedback on algorithm specifications, implementation guidance, and transition strategies. As major enterprise Linux vendor, Red Hat represents deployment reality—ensuring standards work in production environments serving millions of systems.

---

## Challenges and Considerations

### Increased Signature and Key Sizes

Post-quantum algorithms generate larger keys, ciphertext, and signatures compared to classical algorithms:

| Algorithm Type | Classical Size | PQC Size | Ratio |
|---|---|---|---|
| Public keys | 256 bits (ECDSA) | 1,568 bytes (ML-DSA-65) | ~50x larger |
| Signatures | 64 bytes (ECDSA) | 3,293 bytes (ML-DSA-65) | ~51x larger |
| TLS handshake | ~300 bytes | ~1,500 bytes | ~5x larger |

**Implications:**

- **Network bandwidth**: Slightly increased handshake sizes (usually negligible for broadband connections)
- **Storage**: Larger certificates and signature storage requirements
- **Embedded systems**: Constrained devices may require algorithm selection balancing security and resource constraints

For most enterprise systems, these increases are manageable. Modern servers, cloud instances, and network infrastructure handle larger cryptographic objects without performance degradation. Only highly constrained IoT devices or embedded systems face meaningful trade-offs.

### Algorithm Agility and Future Standards

Post-quantum cryptography remains younger than classical public-key cryptography. While NIST's algorithms underwent rigorous evaluation, cryptanalysis continues. Organizations should maintain **algorithm agility**—the ability to switch cryptographic algorithms if weaknesses emerge:

- **Avoid hard-coding algorithms**: Use cryptographic libraries and policies rather than embedding algorithm choices in application code
- **Plan for updates**: Design systems to tolerate cryptographic algorithm changes without major refactoring
- **Monitor cryptanalysis**: Stay informed about PQC research and emerging threats

Hybrid schemes provide insurance during PQC maturation—if unexpected ML-KEM weaknesses emerge, classical ECDH provides fallback protection until next-generation PQC algorithms deploy.

### Performance Considerations

ML-KEM and ML-DSA perform well on modern CPUs, with modest overhead compared to classical algorithms:

**ML-KEM Key Exchange:**
- Key generation: <1ms
- Encapsulation: <1ms
- Decapsulation: <1ms

**ML-DSA Signatures:**
- Signing: <1ms (ML-DSA-65)
- Verification: <1ms

For most applications, PQC performance is acceptable. CPU-intensive scenarios (high-throughput TLS termination, massive-scale signing operations) may require tuning or hardware acceleration. Red Hat's performance engineering ensures RHEL 10.1 PQC implementations optimize for real-world workloads.

---

## Real-World Use Case: Canadian Federal Department PQC Migration

**Organization**: Large federal department with 50,000 employees, 10,000 servers, and citizen-facing services processing Protected B data.

**Challenge**: Multi-year infrastructure with mixed RHEL versions (RHEL 7, RHEL 8, RHEL 9), hundreds of applications, and long-lived data requiring protection against quantum threats (healthcare records, benefits data, tax information). Department must begin PQC adoption without disrupting operations or requiring application rewrites.

**Solution: Progressive Migration Strategy**

**Phase 1 (2026): New Deployments on RHEL 10.1**
- All new application deployments use RHEL 10.1 foundation
- OpenShift clusters upgrade to RHEL 10.1 worker nodes for new namespaces
- External-facing APIs deploy on PQC-enabled infrastructure
- Hybrid cryptography ensures compatibility with existing systems

**Phase 2 (2027-2028): In-Place Upgrades and Replatforming**
- RHEL 8 Extended Lifecycle Support (ELS) systems upgrade to RHEL 9, then RHEL 10.1
- Critical systems (authentication, PKI, encryption key management) prioritized for PQC migration
- SSH administrative access transitions to quantum-resistant key exchange
- TLS internal service mesh upgrades to support PQC

**Phase 3 (2029-2030): Legacy System Remediation**
- Remaining RHEL 7 systems (end of life) decommissioned or containerized on RHEL 10.1
- Air-gapped and classified systems receive RHEL 10.1 through secure media
- Comprehensive PQC coverage achieved for Protected B and above data

**Results:**
- Zero application code changes required (PQC transparent to applications)
- Citizen-facing services quantum-protected within 18 months
- Complete infrastructure PQC coverage within 4 years
- Compliance with TBS and CCCS cryptographic requirements
- Protection against "harvest now, decrypt later" attacks for long-lived data

---

## Key Benefits Summary

**For Technical Teams:**
- Transparent PQC adoption through operating system foundation
- No application code changes required for TLS and SSH
- Hybrid schemes provide transition path and compatibility
- Fully supported in RHEL 10.1 with Red Hat SLAs

**For Organizations:**
- Protect against "harvest now, decrypt later" attacks
- Future-proof cryptographic investments for 20+ year timeframe
- Meet evolving TBS and CCCS cryptographic requirements
- Enable gradual migration without disruption

**For Digital Sovereignty:**
- **Data Protection Beyond Quantum Transition**: Canadian government data remains secure through quantum computing emergence
- **Standards-Based Approach**: Open NIST standards avoid vendor lock-in in PQC layer
- **Upstream Community Leadership**: Red Hat contributions ensure PQC works on sovereign infrastructure
- **Long-Term Independence**: Quantum-resistant cryptography prevents future decryption dependency on external actors

---

## Conclusion: The Urgency of PQC Adoption

The quantum computing threat is not hypothetical—it's a matter of timing. Adversaries are harvesting encrypted data today. Cryptographic transitions require decades. Organizations must act now to protect long-lived sensitive data from future quantum decryption.

Red Hat's RHEL 10.1 with PQC enabled by default provides the foundation for quantum-safe digital sovereignty. Canadian government organizations deploying on RHEL 10.1 inherit quantum resistance across their infrastructure, protecting citizen data, classified information, and strategic assets through the quantum transition and beyond.

The transition to post-quantum cryptography represents one of the most significant cryptographic migrations in computing history—comparable in scope to the transition from DES to AES or from MD5 to SHA-256, but with far greater urgency due to the "harvest now, decrypt later" threat. Organizations delaying PQC adoption risk irrecoverable compromise of sensitive data already captured by adversaries.

Digital sovereignty in the quantum era demands quantum-resistant cryptography. Red Hat provides the open source foundation to achieve it.

---

## References and Further Reading

- [NIST Post-Quantum Cryptography Standards (FIPS 203, 204, 205)](https://www.nist.gov/news-events/news/2024/08/nist-releases-first-3-finalized-post-quantum-encryption-standards)
- [What's new in post-quantum cryptography in RHEL 10.1](https://www.redhat.com/en/blog/whats-new-post-quantum-cryptography-rhel-101)
- [Post-quantum cryptography in Red Hat Enterprise Linux 10](https://www.redhat.com/en/blog/post-quantum-cryptography-red-hat-enterprise-linux-10)
- [From if to how: A year of post-quantum reality](https://www.redhat.com/en/blog/if-how-year-post-quantum-reality)
- [How Red Hat is integrating post-quantum cryptography into our products](https://www.redhat.com/en/blog/how-red-hat-integrating-post-quantum-cryptography-our-products)
- [NIST Post-Quantum Cryptography Project](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [NIST IR 8547: Transition to Post-Quantum Cryptography Standards](https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf)
- [The road to quantum-safe cryptography in Red Hat OpenShift](https://www.redhat.com/en/blog/road-to-quantum-safe-cryptography-red-hat-openshift)

---

**Next:** [13. Conclusion: The Path to Digital Sovereignty →](13-conclusion.md)
