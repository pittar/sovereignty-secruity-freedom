# SBOM Generation with Syft

## Overview

This example demonstrates how to generate Software Bills of Materials (SBOMs) for container images using Syft, attach SBOMs to images as signed attestations, and query SBOMs for vulnerability management and compliance.

An SBOM catalogs all software components in an artifact—libraries, dependencies, versions, licenses, and origins. SBOMs enable vulnerability management (identify affected systems when CVEs are disclosed), license compliance (verify license compatibility), and supply chain transparency (understand dependency provenance).

## Basic SBOM Generation

```bash
# Generate SBOM for a container image in SPDX JSON format
syft registry.example.com/myapp:v1.0 -o spdx-json > sbom.json

# Syft will:
# 1. Pull the container image
# 2. Analyze all layers for package managers (npm, pip, maven, go.mod, etc.)
# 3. Extract package names, versions, and metadata
# 4. Generate SBOM in SPDX 2.3 format

# Alternative: Generate in CycloneDX format (optimized for security use cases)
syft registry.example.com/myapp:v1.0 -o cyclonedx-json > sbom-cyclonedx.json

# Generate SBOM in table format for human reading
syft registry.example.com/myapp:v1.0 -o table
```

## Key Elements

**SBOM Standards:**
- **SPDX (Software Package Data Exchange)**: ISO standard, favored for compliance and legal use cases
- **CycloneDX**: OWASP standard, optimized for security vulnerability management
- Both formats supported by Syft, choice depends on organizational requirements

**Package Detection:**
- Syft analyzes container image layers for package manager artifacts
- Supports 20+ ecosystems: npm, pip, gem, maven, go.mod, cargo, apk, deb, rpm, etc.
- Detects both application dependencies and OS packages
- No container execution required (static analysis)

**Output Formats:**
- JSON formats for machine processing and attestation
- Table/text formats for human review
- SPDX tag-value format for maximum compatibility
- Output can be signed and attached to images

## Attaching SBOMs to Images

```bash
# Generate SBOM
syft registry.example.com/myapp:v1.0 -o spdx-json > sbom.json

# Attach SBOM to the image as an artifact
cosign attach sbom --sbom sbom.json registry.example.com/myapp:v1.0

# The SBOM is now stored in the registry alongside the image
# Retrieval: cosign download sbom registry.example.com/myapp:v1.0

# Sign the attached SBOM
cosign sign --key cosign.key registry.example.com/myapp:v1.0

# This signs both the image AND the attached SBOM attestation
```

## Advanced Usage

### SBOM Attestation with In-Toto Format

Generate signed SBOM attestations in in-toto format for SLSA compliance:

```bash
# Generate SBOM attestation (combines SBOM with signature in one artifact)
cosign attest --key cosign.key \
  --predicate sbom.json \
  --type spdxjson \
  registry.example.com/myapp:v1.0

# This creates an in-toto attestation with:
# - Subject: the container image digest
# - Predicate: the SBOM content
# - Signature: cryptographic proof of integrity

# Verify and retrieve SBOM attestation
cosign verify-attestation --key cosign.pub \
  --type spdxjson \
  registry.example.com/myapp:v1.0 | jq '.payload | @base64d | fromjson'
```

### Scanning SBOMs for Vulnerabilities

Use Grype (companion tool to Syft) to scan SBOMs for known vulnerabilities:

```bash
# Generate SBOM
syft registry.example.com/myapp:v1.0 -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json

# Output shows:
# - Package name and version
# - CVE ID and severity
# - Fixed version (if available)
# - CVSS score

# Example output:
# NAME     INSTALLED  VULNERABILITY   SEVERITY
# libssl   1.1.1d     CVE-2021-3711   High
# python   3.9.0      CVE-2021-23336  Medium
```

### SBOM Comparison for Dependency Drift Detection

Compare SBOMs between image versions to detect dependency changes:

```bash
# Generate SBOMs for two versions
syft registry.example.com/myapp:v1.0 -o spdx-json > sbom-v1.0.json
syft registry.example.com/myapp:v1.1 -o spdx-json > sbom-v1.1.json

# Use jq to extract and compare package lists
diff \
  <(jq -r '.packages[] | "\(.name):\(.versionInfo)"' sbom-v1.0.json | sort) \
  <(jq -r '.packages[] | "\(.name):\(.versionInfo)"' sbom-v1.1.json | sort)

# Shows added, removed, and version-changed dependencies
```

### CI/CD Integration: Automated SBOM Generation

```bash
# In a CI/CD pipeline (e.g., Tekton, GitHub Actions)

# 1. Build image
podman build -t registry.example.com/myapp:${VERSION} .

# 2. Push image
podman push registry.example.com/myapp:${VERSION}

# 3. Generate SBOM
syft registry.example.com/myapp:${VERSION} -o spdx-json > sbom.json

# 4. Scan SBOM for critical vulnerabilities
grype sbom:sbom.json --fail-on critical

# 5. Attach and sign SBOM
cosign attest --key ${SIGNING_KEY} \
  --predicate sbom.json \
  --type spdxjson \
  registry.example.com/myapp:${VERSION}

# 6. Verify before deployment
cosign verify-attestation --key ${VERIFY_KEY} \
  --type spdxjson \
  registry.example.com/myapp:${VERSION}
```

## Querying SBOMs for Compliance

Extract specific information from SBOMs using jq:

```bash
# List all packages and versions
jq -r '.packages[] | "\(.name) \(.versionInfo)"' sbom.json

# Find packages with specific licenses (e.g., GPL)
jq -r '.packages[] | select(.licenseDeclared | contains("GPL")) | .name' sbom.json

# Count total dependencies
jq '.packages | length' sbom.json

# Extract packages from specific ecosystem (e.g., Python packages)
jq -r '.packages[] | select(.name | startswith("python:")) | .name' sbom.json

# Generate license summary
jq -r '.packages[].licenseDeclared' sbom.json | sort | uniq -c | sort -rn
```

## When to Use This Pattern

**SBOM Generation:**
- All production container images (compliance requirement for federal software)
- Third-party images before deployment (supply chain due diligence)
- Release artifacts for distribution to customers
- Vulnerability management and incident response preparation

**SPDX Format:**
- Legal and compliance use cases (license verification, export control)
- Integration with compliance management platforms
- Long-term archival (ISO standard stability)

**CycloneDX Format:**
- Security-focused workflows (vulnerability scanning, risk analysis)
- Integration with security tools (Dependency-Track, OWASP tools)
- Continuous vulnerability monitoring

**SBOM Attestation:**
- Zero-trust environments requiring cryptographic proof
- SLSA Level 2+ compliance (signed provenance)
- Policy enforcement with admission controllers

## Compliance and Standards

1. **Executive Order 14028**: Requires SBOMs for software sold to US federal government
2. **NIST SSDF**: Software supply chain security framework references SBOM requirements
3. **EU Cyber Resilience Act**: Mandates transparency for software components
4. **SLSA Framework**: SBOM attestations contribute to SLSA Level 2+ compliance
5. **ISO/IEC 5962**: SPDX is an ISO standard for software component metadata

## Security Considerations

1. **SBOM Integrity**: Always sign SBOMs to prevent tampering
2. **Confidentiality**: SBOMs reveal software composition—restrict access appropriately
3. **Timeliness**: Regenerate SBOMs after any dependency updates
4. **Verification**: Validate SBOM signatures before trusting vulnerability scan results
5. **Completeness**: Ensure Syft detects all package managers used in images

## Related Examples

- For signing SBOMs, see [cosign-signing.md](cosign-signing.md)
- For automated pipeline integration, see [tekton-secure-pipeline.md](tekton-secure-pipeline.md)
- For air-gapped SBOM workflows, see [airgap-signing-workflow.md](airgap-signing-workflow.md)
- For on-premises infrastructure, see [onprem-sigstore-deployment.md](onprem-sigstore-deployment.md)
