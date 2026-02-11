# Cosign: Basic Container Image Signing and Verification

## Overview

This example demonstrates the fundamental workflow for signing and verifying container images using Cosign, the industry-standard tool for container signing. Cosign integrates seamlessly with OCI registries, storing signatures alongside images without requiring separate infrastructure.

## Basic Signing Workflow

```bash
# Sign a container image with a private key
cosign sign --key cosign.key registry.example.com/myapp:v1.0

# Enter private key password when prompted
# Cosign will:
# 1. Generate a signature for the image manifest
# 2. Create a signature artifact in OCI format
# 3. Push the signature to the same registry as the image
# 4. Optionally record the signature in Rekor transparency log

# Verify the signed image
cosign verify --key cosign.pub registry.example.com/myapp:v1.0

# Output will show:
# - Signature validation status
# - Certificate details (for keyless signing)
# - Rekor transparency log entry (if uploaded)
```

## Key Elements

**Key-Based Signing:**
- Uses traditional public/private key cryptography
- Private key (`cosign.key`) signs the image manifest
- Public key (`cosign.pub`) verifies signatures
- Keys can be generated with `cosign generate-key-pair`
- Suitable for organizations with existing PKI infrastructure

**Registry Integration:**
- Signatures stored as OCI artifacts in the same registry
- No separate signature storage infrastructure needed
- Signature tags follow pattern: `sha256-[digest].sig`
- Works with any OCI-compliant registry (Docker Hub, Harbor, Quay, etc.)

**Transparency Log (Rekor):**
- By default, Cosign uploads signatures to public Rekor instance
- Provides immutable audit trail of signing events
- Enables verification of when an artifact was signed
- Can be disabled with `--tlog-upload=false` for private deployments

## Advanced Usage

### Keyless Signing with OIDC

For environments without key management infrastructure, Cosign supports keyless signing using OpenID Connect identities:

```bash
# Keyless signing (uses OIDC authentication)
cosign sign registry.example.com/myapp:v1.0

# Cosign will:
# 1. Open browser for OIDC authentication (GitHub, Google, Microsoft, etc.)
# 2. Request short-lived certificate from Fulcio CA
# 3. Sign image with ephemeral key
# 4. Record certificate and signature in Rekor
# 5. Discard ephemeral key (no key management needed!)

# Verify keyless signature
cosign verify \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://github.com/login/oauth \
  registry.example.com/myapp:v1.0
```

### Signing Multiple Artifacts

Sign images with additional metadata and attestations:

```bash
# Sign with custom annotations
cosign sign --key cosign.key \
  -a release=production \
  -a team=platform-engineering \
  -a approved-by=john.doe@example.com \
  registry.example.com/myapp:v1.0

# Verify and display annotations
cosign verify --key cosign.pub \
  registry.example.com/myapp:v1.0 | jq '.[] | .optional'
```

### Air-Gapped Signing

For disconnected environments, disable Rekor upload:

```bash
# Sign without uploading to transparency log
cosign sign --key cosign.key \
  --tlog-upload=false \
  registry.airgap.corp/myapp:v1.0

# Export signature for transfer
cosign save --signature-only registry.airgap.corp/myapp:v1.0 > myapp-v1.0.sig

# In air-gapped environment, attach signature
cosign attach signature --signature myapp-v1.0.sig registry.airgap.corp/myapp:v1.0

# Verify in air-gapped environment
cosign verify --key cosign.pub registry.airgap.corp/myapp:v1.0
```

## When to Use This Pattern

**Key-Based Signing:**
- Organizations with existing PKI infrastructure
- Air-gapped or disconnected environments
- Requirements for long-lived signing keys
- Integration with hardware security modules (HSM)

**Keyless Signing:**
- Development teams without key management experience
- Cloud-native environments with OIDC identity providers
- Temporary or ephemeral build environments
- Compliance requirements for transparency logs

**General Use Cases:**
- CI/CD pipelines signing build artifacts
- Release engineering signing official distributions
- Security teams verifying third-party images before deployment
- Compliance requirements for software provenance

## Security Considerations

1. **Key Protection**: Store private keys in secure locations (HSM, key management systems, encrypted storage)
2. **Key Rotation**: Regularly rotate signing keys and re-sign critical images
3. **Verification Enforcement**: Use Kubernetes admission controllers to enforce signature verification
4. **Transparency**: Enable Rekor uploads for auditability unless air-gap prevents it
5. **Identity Verification**: For keyless signing, carefully validate certificate identity and issuer

## Related Examples

- For SBOM generation and signing, see [sbom-generation.md](sbom-generation.md)
- For on-premises Sigstore deployment, see [onprem-sigstore-deployment.md](onprem-sigstore-deployment.md)
- For air-gapped workflows, see [airgap-signing-workflow.md](airgap-signing-workflow.md)
- For CI/CD integration, see [tekton-secure-pipeline.md](tekton-secure-pipeline.md)
