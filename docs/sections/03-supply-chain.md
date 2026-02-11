# Securing the Software Supply Chain

[← Base Images](02-base-images.md) | [Table of Contents](../OUTLINE.md) | [Next: Kubernetes Platform →](04-kubernetes-platform.md)

---

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

**Canadian government organizations** face similar supply chain security imperatives. The Canadian Centre for Cyber Security (CCCS) provides guidance on software supply chain risk management, emphasizing the need for verifiable software provenance and transparency. Treasury Board Secretariat (TBS) security categorization for Protected B and above workloads requires documented software composition and vulnerability management—requirements that SBOMs and cryptographic signing directly address. Federal procurement increasingly requires suppliers to demonstrate supply chain security practices. Open source supply chain security tools like Sigstore align with these requirements while avoiding vendor lock-in: Canadian departments can verify artifact signatures, generate compliance reports, and audit software provenance using freely available tools without dependence on proprietary security platforms that could introduce new sovereignty risks.

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

**→ See the complete example:** [Cosign: Basic Container Image Signing and Verification](../examples/03-supply-chain/cosign-signing.md) for key-based signing, keyless signing, and advanced workflows.

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

**→ See the complete example:** [SBOM Generation with Syft and Cosign](../examples/03-supply-chain/sbom-generation.md) for SPDX and CycloneDX formats, automated generation, and verification workflows.

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

## Air-Gapped and On-Premises Supply Chain Security

### Why Air-Gapped Supply Chain Security Matters

Air-gapped environments—systems completely isolated from external networks—are mandated for classified government systems, defense contractors, critical infrastructure (power grids, water treatment), financial trading floors, and research facilities with intellectual property protection. These environments cannot connect to public Sigstore infrastructure (Fulcio, Rekor), external container registries, or internet-based vulnerability databases. Yet supply chain security remains critical: compromised artifacts in air-gapped environments are often more damaging due to the sensitive nature of systems and difficulty of incident response in isolated networks.

On-premises supply chain infrastructure provides sovereignty even without air-gap requirements: organizations maintain complete control over signing keys, transparency logs, and artifact registries without dependency on external services. This aligns with digital sovereignty principles—security infrastructure operates within organizational boundaries, subject to internal governance rather than external service providers.

**Air-Gap and On-Prem Requirements:**
- **Artifact Signing**: Sign container images without external Sigstore access
- **Verification**: Validate signatures using internal trust roots
- **Transparency**: Maintain immutable audit logs for compliance
- **SBOM Management**: Generate and verify SBOMs in disconnected environments
- **Vulnerability Scanning**: Scan images against internal CVE databases
- **Registry Operations**: Mirror and manage container images internally
- **Key Management**: Secure storage and rotation of signing keys

### On-Premises Sigstore Infrastructure

Organizations deploy private Sigstore infrastructure using the Sigstore Scaffolding project, which provides Helm charts and deployment automation for running Fulcio, Rekor, and Certificate Transparency (CT) logs in disconnected environments. This architecture replicates public Sigstore functionality within organizational boundaries, enabling keyless signing with organizational identity providers (LDAP, Active Directory, Keycloak) and maintaining transparency logs for internal audit.

**On-Premises Sigstore Components:**

```yaml
# Example: On-Premises Sigstore Architecture
apiVersion: v1
kind: Namespace
metadata:
  name: sigstore
---
# Fulcio - Internal Certificate Authority
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fulcio
  namespace: sigstore
spec:
  replicas: 3  # HA deployment
  template:
    spec:
      containers:
        - name: fulcio
          image: registry.internal.corp/sigstore/fulcio:v1.4.0
          args:
            - serve
            - --port=8080
            - --grpc-port=5554
            - --ca=fileca
            - --fileca-key=/var/run/fulcio-secrets/key.pem
            - --fileca-cert=/var/run/fulcio-secrets/cert.pem
            - --ct-log-url=http://ctlog.sigstore.svc.cluster.local:6962
            # OIDC configuration for enterprise IDP
            - --oidc-issuer=https://keycloak.internal.corp/auth/realms/sigstore
            - --oidc-client-id=fulcio
          volumeMounts:
            - name: fulcio-secrets
              mountPath: /var/run/fulcio-secrets
              readOnly: true
      volumes:
        - name: fulcio-secrets
          secret:
            secretName: fulcio-ca-keys  # Organizational root CA keys
---
# Rekor - Transparency Log
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rekor-server
  namespace: sigstore
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: rekor
          image: registry.internal.corp/sigstore/rekor-server:v1.3.0
          args:
            - serve
            - --trillian_log_server.address=trillian-log.sigstore.svc.cluster.local:8090
            - --redis_server.address=redis.sigstore.svc.cluster.local:6379
            - --rekor_server.address=0.0.0.0:3000
          ports:
            - containerPort: 3000
              name: http
---
# Trillian - Merkle Tree Backend for Rekor
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trillian-log
  namespace: sigstore
spec:
  template:
    spec:
      containers:
        - name: trillian
          image: registry.internal.corp/sigstore/trillian-log-server:v1.5.0
          args:
            - --storage_system=mysql
            - --mysql_uri=trillian:password@tcp(mysql.sigstore.svc.cluster.local:3306)/trillian
```

**→ See the complete example:** [On-Premises Sigstore Deployment](../examples/03-supply-chain/onprem-sigstore-deployment.md) for complete deployment with Fulcio, Rekor, and Trillian, including HA configuration and enterprise integration.

**Integration with Organizational PKI:**

On-premises Fulcio can chain to organizational root certificate authorities, enabling verification by existing enterprise systems without distributing new trust roots. This integration allows:

- **Certificate Validation**: Fulcio-issued certificates validated by existing enterprise PKI infrastructure
- **Hardware Security Modules (HSM)**: Fulcio private keys stored in HSMs (PKCS#11) for FIPS 140-2 compliance
- **Policy Integration**: Tie certificate issuance to organizational identity policies (multi-factor authentication, role-based access)
- **Audit Integration**: Integrate Rekor logs with enterprise SIEM systems

### Air-Gapped Artifact Signing Workflows

In completely disconnected environments, organizations use key-based signing with locally-managed keys rather than keyless signing (which requires OIDC connectivity). This approach trades key management overhead for air-gap compatibility.

**Air-Gapped Signing Pattern:**

```bash
# On connected environment (development/CI):
# Build and sign artifacts, export signatures

# 1. Build container image
podman build -t myapp:v1.0 .

# 2. Sign image with organizational key
cosign sign --key /secure/signing-key.pem \
  --tlog-upload=false \  # No external Rekor upload in air-gap prep
  registry.connected.corp/myapp:v1.0

# 3. Export signature bundle
cosign save --signature-only registry.connected.corp/myapp:v1.0 > myapp-v1.0.sig

# 4. Export image
podman save registry.connected.corp/myapp:v1.0 -o myapp-v1.0.tar

# 5. Generate SBOM
syft registry.connected.corp/myapp:v1.0 -o spdx-json > myapp-v1.0-sbom.json

# 6. Create cryptographic manifest for integrity verification
sha256sum myapp-v1.0.tar myapp-v1.0.sig myapp-v1.0-sbom.json > manifest.sha256

# 7. Sign manifest with organizational GPG key
gpg --armor --sign manifest.sha256

# Transfer to air-gapped environment:
# - myapp-v1.0.tar (container image)
# - myapp-v1.0.sig (cosign signature)
# - myapp-v1.0-sbom.json (SBOM)
# - manifest.sha256.asc (signed integrity manifest)

# On air-gapped environment:
# 1. Verify transfer integrity
gpg --verify manifest.sha256.asc
sha256sum -c manifest.sha256

# 2. Load image into air-gapped registry
podman load -i myapp-v1.0.tar
podman tag localhost/myapp:v1.0 registry.airgap.corp/myapp:v1.0
podman push registry.airgap.corp/myapp:v1.0

# 3. Upload signature to air-gapped registry
cosign attach signature --signature myapp-v1.0.sig registry.airgap.corp/myapp:v1.0

# 4. Attach SBOM
cosign attach sbom --sbom myapp-v1.0-sbom.json registry.airgap.corp/myapp:v1.0

# 5. Record in on-prem Rekor instance
cosign upload blob \
  --rekor-url https://rekor.airgap.corp \
  --f myapp-v1.0.sig

# 6. Verify signature in air-gapped environment
cosign verify --key /secure/verification-key.pub \
  --rekor-url https://rekor.airgap.corp \
  registry.airgap.corp/myapp:v1.0
```

**→ See the complete example:** [Air-Gapped Signing Workflow](../examples/03-supply-chain/airgap-signing-workflow.md) for complete workflows including image transfer, signature verification, and policy enforcement in disconnected environments.

### Image Mirroring for Air-Gapped Environments

Organizations mirror external container images (UBI, OpenShift, third-party dependencies) into air-gapped registries using controlled transfer processes. This ensures software supply chain security extends to externally-sourced artifacts.

**Mirroring Workflow with Verification:**

```bash
# On connected environment with internet access:

# 1. Mirror Red Hat container images with signature verification
oc adm catalog mirror \
  registry.redhat.io/redhat/redhat-operator-index:v4.15 \
  file://offline-catalog \
  --manifests-only

# 2. Download mirrored images
oc image mirror \
  -f offline-catalog/mapping.txt \
  file://offline-images

# 3. Verify all signatures before transfer
for image in $(cat offline-catalog/mapping.txt | awk '{print $1}'); do
  cosign verify --key /secure/redhat-public-key.pub "$image" || exit 1
done

# 4. Generate transfer manifest with checksums
find offline-images -type f -exec sha256sum {} \; > transfer-manifest.sha256

# 5. Sign transfer manifest
gpg --armor --sign transfer-manifest.sha256

# Transfer via secure removable media:
# - offline-images/ (container image blobs)
# - offline-catalog/ (metadata)
# - transfer-manifest.sha256.asc (signed checksums)

# On air-gapped environment:

# 1. Verify transfer integrity
gpg --verify transfer-manifest.sha256.asc
sha256sum -c transfer-manifest.sha256

# 2. Mirror to air-gapped registry
oc image mirror \
  --from-dir=offline-images \
  file://offline-images \
  registry.airgap.corp/mirror

# 3. Verify signatures in air-gapped registry
for image in $(cat offline-catalog/mapping.txt | awk '{print $2}'); do
  cosign verify --key /secure/redhat-public-key.pub \
    "registry.airgap.corp/mirror/$image"
done
```

### Vulnerability Database Management in Air-Gap

Air-gapped vulnerability scanning requires maintaining internal CVE databases synchronized from external sources during periodic security updates.

**Air-Gap Vulnerability Management:**

```bash
# On connected environment (periodic sync - e.g., weekly):

# 1. Download latest CVE databases
trivy image --download-db-only --db-repository trivy-db.airgap.corp

# 2. Export Trivy database
trivy image --cache-dir /tmp/trivy-cache \
  --download-db-only

# 3. Package database for transfer
tar czf trivy-db-$(date +%Y%m%d).tar.gz -C /tmp/trivy-cache .

# 4. Generate checksum and sign
sha256sum trivy-db-$(date +%Y%m%d).tar.gz > trivy-db-$(date +%Y%m%d).tar.gz.sha256
gpg --armor --sign trivy-db-$(date +%Y%m%d).tar.gz.sha256

# Transfer to air-gapped environment

# On air-gapped environment:

# 1. Verify database integrity
gpg --verify trivy-db-$(date +%Y%m%d).tar.gz.sha256.asc
sha256sum -c trivy-db-$(date +%Y%m%d).tar.gz.sha256

# 2. Extract to internal Trivy cache
tar xzf trivy-db-$(date +%Y%m%d).tar.gz -C /var/lib/trivy/cache

# 3. Configure Trivy to use local database
trivy image --skip-db-update \
  --offline-scan \
  registry.airgap.corp/myapp:v1.0
```

### Policy Enforcement in On-Prem and Air-Gap

Kubernetes admission controllers enforce supply chain policies using internal Sigstore infrastructure and verification keys.

**Air-Gap Policy Configuration:**

```yaml
# Sigstore Policy Controller configuration for air-gapped cluster
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-policy-controller
  namespace: cosign-system
data:
  # Use on-prem Fulcio and Rekor
  fulcio-url: "https://fulcio.airgap.corp"
  rekor-url: "https://rekor.airgap.corp"
  ctlog-url: "https://ctlog.airgap.corp"
---
# ClusterImagePolicy requiring internal signatures
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-internal-signatures
spec:
  images:
    - glob: "registry.airgap.corp/**"
  authorities:
    - name: internal-sigstore
      keyless:
        url: https://fulcio.airgap.corp
        identities:
          # Require signatures from organizational identities
          - issuer: "https://keycloak.airgap.corp/auth/realms/sigstore"
            subject: ".*@corp\\.com$"  # Organizational email domain
    - name: redhat-signatures
      key:
        # Verify Red Hat signatures using public key
        data: |
          -----BEGIN PUBLIC KEY-----
          [Red Hat public key]
          -----END PUBLIC KEY-----
  policy:
    type: cue
    data: |
      // Require both signature and SBOM attestation
      predicateType: "https://spdx.dev/Document"
```

### On-Premises Supply Chain Operations

Daily operations in on-premises/air-gapped supply chain infrastructure require automation for sustainability.

**Operational Patterns:**

1. **Automated Image Builds**: Tekton pipelines in on-prem OpenShift with Chains for automatic signing
2. **Scheduled Mirroring**: CronJobs synchronizing external images during security update windows
3. **Compliance Reporting**: Automated SBOM aggregation and vulnerability reporting for audit
4. **Key Rotation**: Periodic rotation of signing keys with HSM integration
5. **Capacity Planning**: Rekor log growth management, registry storage monitoring
6. **Disaster Recovery**: Backup procedures for Rekor transparency logs, registry data

**Example: Automated On-Prem Build with Signing**

```yaml
# Tekton Pipeline with Chains for automatic signing in on-prem cluster
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: onprem-secure-build
  namespace: ci-cd
  annotations:
    # Chains configuration for automatic signing
    chains.tekton.dev/signed: "true"
spec:
  params:
    - name: git-url
    - name: image-name
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)

    - name: build
      taskRef:
        name: buildah
      runAfter: [clone]
      params:
        - name: IMAGE
          value: registry.airgap.corp/$(params.image-name)

    - name: scan
      taskRef:
        name: trivy-scan
      runAfter: [build]
      params:
        - name: IMAGE
          value: registry.airgap.corp/$(params.image-name)
        # Use air-gapped Trivy database
        - name: TRIVY_OFFLINE_SCAN
          value: "true"

    - name: generate-sbom
      taskRef:
        name: syft
      runAfter: [build]
      params:
        - name: IMAGE
          value: registry.airgap.corp/$(params.image-name)

  # Tekton Chains automatically:
  # 1. Signs the TaskRun with organizational key
  # 2. Generates SLSA provenance
  # 3. Uploads to on-prem Rekor instance
  # 4. Attaches signature to image in registry
---
# Chains configuration for on-prem Sigstore
apiVersion: v1
kind: ConfigMap
metadata:
  name: chains-config
  namespace: tekton-chains
data:
  artifacts.taskrun.format: "in-toto"
  artifacts.taskrun.storage: "oci"
  artifacts.oci.storage: "oci"
  transparency.enabled: "true"
  transparency.url: "https://rekor.airgap.corp"
  # Use organizational signing key
  signers.kms.auth.address: "hashivault://vault.airgap.corp:8200"
```

---

## Compliance and Attestation

### SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts, pronounced "salsa") defines four maturity levels for supply chain security: SLSA 1 (documentation of build process), SLSA 2 (signed provenance from builds), SLSA 3 (hardened build platforms preventing tampering), and SLSA 4 (two-party review of changes). Red Hat's build systems target SLSA 3+, with signed build provenance, hermetic builds, and tamper-resistant infrastructure.

### Build Provenance

Build provenance attestations document how artifacts were built—source repository, commit hash, build timestamp, builder identity, and dependencies. Tekton Chains automatically generates and signs provenance for pipeline builds, attaching in-toto attestations to resulting images. Consumers verify provenance to ensure artifacts weren't built from tampered source or compromised build systems.

### Compliance Mappings

Sigstore addresses multiple NIST SSDF requirements: cryptographic verification (PS.1), provenance tracking (PS.3), and automated vulnerability detection (PW.8). FedRAMP continuous monitoring requirements benefit from Rekor's auditability, while SBOM mandates in Executive Order 14028 are satisfied through Syft integration. The transparency and verifiability of Sigstore align with zero-trust principles required in modern security frameworks.

**For Canadian organizations**, Sigstore's transparency log model supports TBS audit requirements and Office of the Auditor General reviews. When departments deploy applications, auditors can query Rekor to verify when artifacts were signed, by whom, and under what attestation conditions—creating cryptographic audit trails that satisfy government accountability standards. CCCS guidance on supply chain security emphasizes "trust but verify" principles; Sigstore's open transparency logs enable verification without requiring trust in vendor-controlled infrastructure. Because Sigstore is open source and can be deployed on-premises (as demonstrated in the air-gapped deployment sections), Canadian departments maintain complete control over their supply chain security infrastructure while leveraging industry-standard tooling. This approach balances security rigor with sovereignty—meeting compliance requirements without creating dependencies on foreign security services.

**→ See the complete example:** [Tekton Secure Pipeline](../examples/03-supply-chain/tekton-secure-pipeline.md) for end-to-end pipeline with automated signing, SBOM generation, and vulnerability scanning.

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

---

**Next:** [4. Cloud Independence: Enterprise Kubernetes →](04-kubernetes-platform.md)
