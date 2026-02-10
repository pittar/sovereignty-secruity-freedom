# Securing the Software Supply Chain

## Executive Summary

Software supply chain attacks have emerged as one of the most effective threat vectors for compromising systems at scale. SolarWinds demonstrated how a single compromised build system could affect 18,000 organizations, while Log4Shell showed how a vulnerability in a ubiquitous dependency could create global exposure overnight. Traditional security focused on perimeter defense; modern threats exploit the trust relationships in software dependencies and build pipelines.

Modern software supply chains are complex, with dependencies spanning hundreds of open source projects and commercial components. A single compromised artifact can affect thousands of applications. Securing this supply chain requires cryptographic verification, transparency, and industry-standard tooling. Red Hat's implementation of Sigstore and related technologies provides enterprise-grade supply chain security built on open standards.

Open source supply chain security tools—particularly Sigstore—provide transparency, cryptographic verification, and auditability without creating vendor lock-in. Unlike proprietary security platforms that become new dependencies, Sigstore operates as public infrastructure with open standards. Organizations can verify artifacts, generate provenance, and enforce policies using open tools that work across any cloud or infrastructure.

---

## The Software Supply Chain Security Crisis

### High-Profile Incidents

**SolarWinds (2020)**: Attackers compromised the build system for SolarWinds Orion, injecting malicious code into signed updates distributed to ~18,000 customers including US government agencies. The attack went undetected for months, demonstrating the catastrophic impact of build system compromise.

**Log4Shell (2021)**: A critical remote code execution vulnerability in Log4j, a logging library used by millions of applications, created instant global exposure. The challenge wasn't just patching—organizations first had to discover where Log4j existed in their dependency trees, often buried several layers deep.

**Codecov (2021)**: Attackers modified Codecov's Bash uploader script, harvesting credentials and tokens from thousands of CI/CD environments. The supply chain attack targeted developer tooling, exposing secrets across numerous organizations.

### The Scope of the Problem

Modern applications average 200+ dependencies when including transitive dependencies. Each dependency represents potential attack surface. A 2023 analysis found 88% of organizations experienced a software supply chain attack, with average remediation costs exceeding $4.5 million. Attack vectors include compromised dependencies, malicious package typosquatting, build system intrusion, stolen signing keys, and man-in-the-middle attacks during artifact distribution.

### Regulatory Response

**Executive Order 14028** (May 2021) mandates that software sold to US federal agencies must include Software Bills of Materials (SBOM) and follow secure software development practices. **EU Cyber Resilience Act** imposes similar requirements for products sold in European markets. **NIST Secure Software Development Framework (SSDF)** provides implementation guidance, while the **SLSA (Supply-chain Levels for Software Artifacts)** framework defines maturity levels for supply chain security practices.

---

## Open Source Standards for Supply Chain Security

### Why Open Standards Matter

Proprietary supply chain security platforms create new dependencies and lock-in. Vendor-specific signing infrastructure, proprietary attestation formats, and closed verification tools tie organizations to specific platforms. If the vendor changes pricing, discontinues features, or becomes unavailable, the entire security infrastructure becomes a liability. Open standards ensure that security investments remain portable—artifacts signed with open tools can be verified anywhere, by anyone, using freely available software.

### The Sigstore Project

Sigstore emerged from the Linux Foundation as public infrastructure for software signing and transparency. Launched in 2021 with contributions from Google, Red Hat, Purdue University, and others, Sigstore provides free signing and verification services designed to operate at internet scale. Unlike traditional PKI requiring organizations to manage long-lived signing keys, Sigstore enables keyless signing using existing identities (GitHub, Google, Microsoft accounts) with cryptographic proof recorded in an immutable transparency log.

**Sigstore Components:**

1. **Cosign**: Container and artifact signing
2. **Fulcio**: Keyless certificate authority
3. **Rekor**: Transparency log (immutable artifact ledger)
4. **Gitsign**: Git commit signing

---

## Technical Deep Dive: Sigstore Architecture

### Cryptographic Signing with Cosign

Cosign signs OCI artifacts (container images, Helm charts, WASMs) and stores signatures as separate OCI artifacts in the same registry. This approach keeps signatures alongside what they protect without requiring separate infrastructure. Cosign supports both key-based and keyless signing—key-based works with existing PKI infrastructure, while keyless integrates with OpenID Connect providers for ephemeral certificates issued by Fulcio.

```bash
# Example: Signing a container image
cosign sign --key cosign.key registry.example.com/myapp:v1.0

# Verifying a signed image
cosign verify --key cosign.pub registry.example.com/myapp:v1.0
```

### Keyless Signing with Fulcio

Fulcio acts as a certificate authority that issues short-lived certificates (valid for minutes) based on OIDC identity tokens. When a developer signs an artifact, they authenticate with their identity provider (GitHub, Google, etc.), Fulcio issues a certificate bound to that identity, the artifact gets signed, and the certificate is recorded in Rekor. Because certificates expire quickly, stolen certificates have limited utility. The signing event remains permanently verifiable through Rekor's transparency log.

**Benefits of Keyless Signing:**
- No key management overhead
- Integration with existing identity providers
- Short-lived certificates tied to identities
- Audit trail in transparency log

### The Rekor Transparency Log

Rekor provides an immutable, append-only ledger of artifact signatures and attestations. Built on the same Merkle tree technology used by Certificate Transparency, Rekor ensures that signatures cannot be backdated, removed, or modified after creation. Anyone can query Rekor to verify when an artifact was signed, by whom, and under what conditions. This transparency prevents attackers from signing malicious artifacts after compromising signing infrastructure—the temporal record in Rekor proves authenticity.

**How Rekor Works:**
- Merkle tree-based transparency log
- Public auditability without compromising privacy
- Integration with signing workflows
- Proof of inclusion and timestamp

### Software Bill of Materials (SBOM)

An SBOM catalogs all components in a software artifact—libraries, dependencies, versions, licenses, and origins. SPDX (Software Package Data Exchange) and CycloneDX are the dominant standards, with SPDX favored for compliance and CycloneDX optimized for security use cases. SBOMs enable vulnerability management (identify affected systems when CVEs are disclosed), license compliance (verify license compatibility), and supply chain transparency (understand dependency provenance).

**SBOM in Practice:**
```bash
# Generate SBOM for a container image
syft registry.example.com/myapp:v1.0 -o spdx-json > sbom.json

# Attach SBOM to image
cosign attach sbom --sbom sbom.json registry.example.com/myapp:v1.0
```

---

## Vulnerability Management

### Scanning and Detection

Vulnerability scanners like Clair, Trivy, Snyk, and Anchore analyze container images against CVE databases, matching packages to known vulnerabilities. These tools integrate with Sigstore—scan results can be attached as attestations to images, signed, and recorded in Rekor. Kubernetes admission controllers can query these attestations before allowing pod creation, blocking images with critical vulnerabilities or missing scan attestations.

### Remediation Workflows

When a CVE is disclosed, SBOMs identify affected images. Remediation involves rebuilding images with patched dependencies, scanning the new image to verify CVE resolution, signing the image with Cosign, and deploying through CI/CD pipelines. The entire remediation chain—patch, rebuild, scan, sign, deploy—should be automated and produce verifiable attestations at each stage.

### Policy Enforcement

Kubernetes admission controllers like Sigstore Policy Controller, Kyverno, or OPA Gatekeeper intercept pod creation requests and enforce policies: require signatures from specific identities, mandate SBOM presence, block images with high-severity CVEs, or verify SLSA provenance levels. Policies as code in Git enable audit trails and version control for security requirements.

---

## Red Hat's Integration and Implementation

### Red Hat's Sigstore Deployment

Red Hat signs all official container images, operators, and Helm charts using Sigstore, with signatures publicly verifiable through Rekor. Red Hat's signing infrastructure uses Fulcio for certificate issuance and stores signatures in `registry.redhat.io`. Every image in Red Hat's registries includes attached signatures and SBOMs, enabling customers to verify provenance and audit supply chain integrity.

### Integration with OpenShift

OpenShift includes built-in signature verification through machine config operators that configure `crio` and `podman` to enforce signature policies. Administrators define policies requiring signatures from specific issuers—OpenShift will refuse to run unsigned images or images from untrusted sources. The Sigstore Policy Controller integrates directly with OpenShift, enforcing keyless signature verification using Fulcio certificates.

### Enterprise Sigstore Services

Organizations can deploy private Sigstore infrastructure using the Sigstore Scaffolding project, running Fulcio, Rekor, and certificate transparency infrastructure in their own clusters. This provides air-gapped deployments, custom root of trust, and integration with enterprise identity providers. Red Hat provides enterprise support for Sigstore infrastructure as part of OpenShift's trusted software supply chain features.

---

## Compliance and Attestation

### SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts, pronounced "salsa") defines four maturity levels for supply chain security: SLSA 1 (documentation of build process), SLSA 2 (signed provenance from builds), SLSA 3 (hardened build platforms preventing tampering), and SLSA 4 (two-party review of changes). Red Hat's build systems target SLSA 3+, with signed build provenance, hermetic builds, and tamper-resistant infrastructure.

### Build Provenance

Build provenance attestations document how artifacts were built—source repository, commit hash, build timestamp, builder identity, and dependencies. Tekton Chains automatically generates and signs provenance for pipeline builds, attaching in-toto attestations to resulting images. Consumers verify provenance to ensure artifacts weren't built from tampered source or compromised build systems.

### Compliance Mappings

Sigstore addresses multiple NIST SSDF requirements: cryptographic verification (PS.1), provenance tracking (PS.3), and automated vulnerability detection (PW.8). FedRAMP continuous monitoring requirements benefit from Rekor's auditability, while SBOM mandates in Executive Order 14028 are satisfied through Syft integration. The transparency and verifiability of Sigstore align with zero-trust principles required in modern security frameworks.

---

## End-to-End Example: Secure Supply Chain Pipeline

```yaml
# Tekton Pipeline example with signing and verification
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: secure-build-pipeline
spec:
  tasks:
    - name: build
      taskRef:
        name: buildah
    - name: scan
      taskRef:
        name: trivy-scan
      runAfter: [build]
    - name: generate-sbom
      taskRef:
        name: syft
      runAfter: [build]
    - name: sign
      taskRef:
        name: cosign-sign
      runAfter: [scan, generate-sbom]
    - name: publish
      taskRef:
        name: push-image
      runAfter: [sign]
```

This pipeline implements defense-in-depth: **build** creates the image, **scan** checks for vulnerabilities, **generate-sbom** catalogs components, **sign** creates cryptographic proof of integrity, and **publish** makes the verified artifact available. Each stage produces attestations attached to the image. Admission controllers later verify these attestations before allowing deployment, ensuring only scanned, signed artifacts with known provenance run in production.

---

## The Upstream Connection

### Red Hat's Contributions to Sigstore

Red Hat engineers serve as maintainers across Sigstore projects, including Cosign, Rekor, and Fulcio. Red Hat contributed the Sigstore Policy Controller for Kubernetes policy enforcement, drives integration with Tekton for automated signing in CI/CD, and operates production Sigstore infrastructure used across the industry. Engineering investment includes ~20+ dedicated engineers contributing to Sigstore core projects and ecosystem integrations.

### Community Growth and Adoption

Sigstore has achieved widespread adoption beyond its original contributors. GitHub signs npm packages with Sigstore, Google Cloud signs GKE images, Chainguard uses Sigstore for distroless image signing, and CNCF graduated projects increasingly adopt Sigstore for release signing. The public Sigstore infrastructure handles millions of signing operations monthly, demonstrating the scalability of open supply chain security.

---

## Key Benefits Summary

**For Technical Teams:**
- Automated signing and verification
- Integration with existing CI/CD
- Open source tools and standards

**For Organizations:**
- Reduced supply chain risk
- Compliance and audit readiness
- Vendor-neutral approach

**For Digital Sovereignty:**
- Transparent, auditable supply chain
- No proprietary lock-in
- Freedom to run infrastructure anywhere

---

## References and Further Reading

- [Sigstore Project](https://www.sigstore.dev/)
- [SLSA Framework](https://slsa.dev/)
- [NIST Secure Software Development Framework (SSDF)](https://csrc.nist.gov/Projects/ssdf)
- [Executive Order 14028 on Cybersecurity](https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/)
- [Red Hat Trusted Software Supply Chain](https://www.redhat.com/en/solutions/trusted-software-supply-chain)
