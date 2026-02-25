# Air-Gapped Artifact Signing and Verification Workflow

## Overview

This example demonstrates a complete workflow for signing, transferring, and verifying container images in air-gapped (completely network-isolated) environments. Air-gapped systems cannot connect to external Sigstore infrastructure, container registries, or vulnerability databases, requiring specialized processes for supply chain security.

This workflow ensures cryptographic integrity across the air-gap boundary, maintains audit trails, and enables verification in disconnected environments.

## Architecture Overview

**Environments:**
1. **Connected Environment** (Development/CI): Internet access, external registries, public Sigstore
2. **Transfer Mechanism**: Secure removable media with cryptographic verification
3. **Air-Gapped Environment** (Production): Network-isolated, internal registries, on-prem Sigstore

**Security Principles:**
- Cryptographic verification at every transfer point
- Multiple signature layers (artifact signatures, transfer manifests)
- Immutable audit trail in both environments
- No trusted automated transfers (manual verification required)

## Complete Air-Gap Workflow

```bash
# ============================================================================
# CONNECTED ENVIRONMENT (Development/CI with Internet Access)
# ============================================================================

# 1. Build container image
podman build -t myapp:v1.0 .

# 2. Tag for connected registry
podman tag myapp:v1.0 registry.connected.corp/myapp:v1.0

# 3. Push to connected registry
podman push registry.connected.corp/myapp:v1.0

# 4. Sign image with organizational key (disable Rekor upload for air-gap prep)
cosign sign --key /secure/signing-key.pem \
  --tlog-upload=false \
  registry.connected.corp/myapp:v1.0

# Note: --tlog-upload=false prevents uploading to public Rekor
# Signature will be recorded in on-prem Rekor after air-gap transfer

# 5. Generate Software Bill of Materials (SBOM)
syft registry.connected.corp/myapp:v1.0 -o spdx-json > myapp-v1.0-sbom.json

# 6. Scan for vulnerabilities BEFORE air-gap transfer
grype registry.connected.corp/myapp:v1.0 --fail-on critical

# If vulnerabilities found, STOP and remediate before transfer
# Critical vulnerabilities should never cross the air-gap

# 7. Attach SBOM to image
cosign attach sbom --sbom myapp-v1.0-sbom.json registry.connected.corp/myapp:v1.0

# 8. Export container image and signature
# Save image as tarball
podman save registry.connected.corp/myapp:v1.0 -o myapp-v1.0.tar

# Export signature bundle
cosign save --signature-only registry.connected.corp/myapp:v1.0 > myapp-v1.0.sig

# SBOM already saved in step 5

# 9. Create cryptographic manifest for transfer integrity verification
# This ensures the transfer media hasn't been tampered with
sha256sum myapp-v1.0.tar myapp-v1.0.sig myapp-v1.0-sbom.json > manifest.sha256

# 10. Sign the manifest with organizational GPG key
# This provides non-repudiation: only authorized users can create valid transfer packages
gpg --armor --sign manifest.sha256

# Output: manifest.sha256.asc (GPG-signed checksums)

# 11. Create transfer documentation
cat > transfer-metadata.json <<EOF
{
  "image": "myapp:v1.0",
  "digest": "sha256:$(podman inspect registry.connected.corp/myapp:v1.0 --format '{{.Digest}}')",
  "built_at": "$(date -Iseconds)",
  "signed_by": "build-pipeline@corp.com",
  "sbom_packages": $(jq '.packages | length' myapp-v1.0-sbom.json),
  "vulnerability_scan": "passed",
  "approved_by": "security-team@corp.com"
}
