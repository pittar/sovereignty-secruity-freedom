# Tekton Pipeline with Chains for Automated Signing

## Overview

This example demonstrates a complete Tekton CI/CD pipeline with Tekton Chains for automatic cryptographic signing, SBOM generation, and SLSA provenance attestation. Tekton Chains observes pipeline executions and automatically signs TaskRuns and PipelineRuns, generating verifiable supply chain metadata without manual intervention.

This pattern implements SLSA Level 2+ compliance with minimal developer friction—every build automatically produces signed artifacts with full provenance.

## Architecture Components

**Tekton Components:**
1. **Tekton Pipelines**: CI/CD execution engine
2. **Tekton Chains**: Automatic signing and attestation
3. **Tekton Tasks**: Reusable build, test, scan, and publish steps

**Supply Chain Security:**
- Automatic signing of TaskRuns and PipelineRuns
- SLSA provenance generation (in-toto attestation format)
- SBOM generation and attachment
- Signature storage in OCI registry
- Transparency log entries in Rekor

## Complete Secure Build Pipeline

```yaml
# Tekton Pipeline: Secure Container Build with Automated Signing
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: secure-container-build
  namespace: ci-cd
  annotations:
    # Tekton Chains configuration
    chains.tekton.dev/signed: "true"
spec:
  params:
    - name: git-url
      type: string
      description: Git repository URL
    - name: git-revision
      type: string
      description: Git revision (commit SHA, branch, or tag)
      default: main
    - name: image-name
      type: string
      description: Container image name (without tag)
    - name: image-tag
      type: string
      description: Container image tag
      default: latest
  
  workspaces:
    - name: shared-workspace
      description: Workspace for source code and build artifacts
  
  tasks:
    # Task 1: Clone Git Repository
    - name: git-clone
      taskRef:
        name: git-clone
        kind: ClusterTask
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
        - name: deleteExisting
          value: "true"
      workspaces:
        - name: output
          workspace: shared-workspace

    # Task 2: Run Unit Tests
    - name: unit-tests
      taskRef:
        name: pytest
      runAfter:
        - git-clone
      params:
        - name: requirements-file
          value: requirements.txt
        - name: args
          value: ["tests/unit/", "--cov", "--cov-report=xml"]
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 3: Static Security Scan (SAST)
    - name: security-scan
      taskRef:
        name: bandit
      runAfter:
        - git-clone
      params:
        - name: args
          value: ["-r", "src/", "-f", "json", "-o", "bandit-report.json"]
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 4: Build Container Image
    - name: build-image
      taskRef:
        name: buildah
        kind: ClusterTask
      runAfter:
        - unit-tests
        - security-scan
      params:
        - name: IMAGE
          value: registry.example.com/$(params.image-name):$(params.image-tag)
        - name: DOCKERFILE
          value: ./Dockerfile
        - name: CONTEXT
          value: .
        - name: FORMAT
          value: oci
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 5: Scan Container Image for Vulnerabilities
    - name: vulnerability-scan
      taskRef:
        name: trivy-scanner
      runAfter:
        - build-image
      params:
        - name: IMAGE
          value: registry.example.com/$(params.image-name):$(params.image-tag)
        - name: SEVERITY
          value: CRITICAL,HIGH
        - name: EXIT_CODE
          value: "1"  # Fail pipeline on critical vulnerabilities

    # Task 6: Generate SBOM
    - name: generate-sbom
      taskRef:
        name: syft-generate
      runAfter:
        - build-image
      params:
        - name: IMAGE
          value: registry.example.com/$(params.image-name):$(params.image-tag)
        - name: FORMAT
          value: spdx-json

    # Task 7: Push Image to Registry
    - name: push-image
      taskRef:
        name: buildah
        kind: ClusterTask
      runAfter:
        - vulnerability-scan
        - generate-sbom
      params:
        - name: IMAGE
          value: registry.example.com/$(params.image-name):$(params.image-tag)
        - name: PUSH_EXTRA_ARGS
          value: "--digestfile=/workspace/source/image-digest"
      workspaces:
        - name: source
          workspace: shared-workspace

  # Tekton Chains automatically:
  # 1. Observes PipelineRun completion
  # 2. Generates SLSA provenance attestation
  # 3. Signs provenance with configured key/keyless method
  # 4. Uploads signature to Rekor transparency log
  # 5. Attaches signed attestation to image in registry

---
# Custom Task: Syft SBOM Generation
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: syft-generate
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Container image to scan
    - name: FORMAT
      description: SBOM output format (spdx-json, cyclonedx-json, etc.)
      default: spdx-json
  
  results:
    - name: sbom-path
      description: Path to generated SBOM file
  
  steps:
    - name: generate-sbom
      image: registry.example.com/anchore/syft:latest
      script: |
        #!/bin/bash
        set -e
        
        echo "Generating SBOM for $(params.IMAGE)"
        
        syft $(params.IMAGE) \
          -o $(params.FORMAT) \
          --file /workspace/sbom.json
        
        echo "SBOM generated: /workspace/sbom.json"
        
        # Output SBOM path for downstream tasks
        echo -n "/workspace/sbom.json" | tee $(results.sbom-path.path)
        
        # Display SBOM summary
        echo "SBOM Package Summary:"
        jq '.packages | length' /workspace/sbom.json
      volumeMounts:
        - name: sbom-storage
          mountPath: /workspace
  
  volumes:
    - name: sbom-storage
      emptyDir: {}

---
# Custom Task: Trivy Vulnerability Scanner
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: trivy-scanner
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Container image to scan
    - name: SEVERITY
      description: Severities to scan for (CRITICAL,HIGH,MEDIUM,LOW)
      default: CRITICAL,HIGH
    - name: EXIT_CODE
      description: Exit code when vulnerabilities are found
      default: "0"
  
  results:
    - name: vulnerability-count
      description: Number of vulnerabilities found
  
  steps:
    - name: scan-image
      image: registry.example.com/aquasecurity/trivy:latest
      script: |
        #!/bin/bash
        set -e
        
        echo "Scanning $(params.IMAGE) for vulnerabilities"
        
        trivy image \
          --severity $(params.SEVERITY) \
          --format json \
          --output /workspace/trivy-report.json \
          --exit-code $(params.EXIT_CODE) \
          $(params.IMAGE)
        
        # Count vulnerabilities
        VULN_COUNT=$(jq '[.Results[].Vulnerabilities // [] | .[] ] | length' /workspace/trivy-report.json)
        echo "Found $VULN_COUNT vulnerabilities"
        echo -n "$VULN_COUNT" | tee $(results.vulnerability-count.path)
        
        # Display critical vulnerabilities
        echo "Critical Vulnerabilities:"
        jq -r '.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL") | "\(.PkgName): \(.VulnerabilityID) - \(.Title)"' \
          /workspace/trivy-report.json || echo "None"
      volumeMounts:
        - name: scan-results
          mountPath: /workspace
  
  volumes:
    - name: scan-results
      emptyDir: {}

---
# Tekton Chains Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: chains-config
  namespace: tekton-chains
data:
  # Artifact storage: store signatures as OCI artifacts
  artifacts.taskrun.format: "in-toto"
  artifacts.taskrun.storage: "oci"
  artifacts.oci.storage: "oci"
  
  # Transparency log: upload to Rekor
  transparency.enabled: "true"
  transparency.url: "https://rekor.sigstore.dev"  # Or internal Rekor URL
  
  # Signing configuration: keyless signing with Fulcio
  signers.kms.auth.sigstore.enabled: "true"
  signers.kms.auth.sigstore.fulcio-url: "https://fulcio.sigstore.dev"
  signers.kms.auth.sigstore.rekor-url: "https://rekor.sigstore.dev"
  signers.kms.auth.sigstore.issuer-url: "https://oauth2.sigstore.dev/auth"
  
  # Alternative: Key-based signing with external KMS
  # signers.kms.auth.address: "hashivault://vault.example.com:8200"
  # signers.kms.auth.token: "..."
  
  # SLSA provenance builder ID
  builddefinition.buildType: "https://tekton.dev/chains/v2"

---
# PipelineRun: Execute Secure Build Pipeline
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: myapp-build-v1.0
  namespace: ci-cd
  annotations:
    # Chains will sign this PipelineRun
    chains.tekton.dev/signed: "true"
spec:
  pipelineRef:
    name: secure-container-build
  
  params:
    - name: git-url
      value: https://github.com/example/myapp.git
    - name: git-revision
      value: v1.0.0
    - name: image-name
      value: myapp
    - name: image-tag
      value: v1.0.0
  
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
  
  # Pod template for security context
  podTemplate:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 1000
```

## Key Elements

**Automatic Signing with Tekton Chains:**
- Chains controller watches for completed TaskRuns and PipelineRuns
- Generates SLSA provenance (in-toto attestation) with build metadata
- Signs attestation with configured method (keyless or key-based)
- Uploads signature to Rekor for transparency
- Attaches signed attestation to resulting container image

**SLSA Provenance Content:**
- **Subject**: Container image digest
- **Predicate**: Build metadata (materials, builder, recipe, metadata)
- **Materials**: Source repository, commit SHA, dependencies
- **Builder**: Tekton version, cluster identity
- **Recipe**: Pipeline definition, parameters
- **Metadata**: Timestamps, reproducibility guarantees

**Defense-in-Depth Security:**
1. **Static Analysis**: Security scan (Bandit) before build
2. **Unit Tests**: Functional correctness validation
3. **Vulnerability Scan**: CVE detection (Trivy) post-build
4. **SBOM Generation**: Component transparency (Syft)
5. **Cryptographic Signing**: Integrity and authenticity (Chains)
6. **Transparency Log**: Immutable audit trail (Rekor)

**Pipeline Fail-Fast:**
- Unit tests fail → stop pipeline before building image
- Security scan finds issues → stop before build
- Vulnerability scan finds critical CVEs → stop before push
- Prevents shipping vulnerable or broken artifacts

## Verification Workflow

After the pipeline completes, verify the signed artifact:

```bash
# Verify SLSA provenance attestation
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity=https://kubernetes.io/namespaces/ci-cd/serviceaccounts/tekton-chains \
  --certificate-oidc-issuer=https://kubernetes.default.svc.cluster.local \
  registry.example.com/myapp:v1.0.0

# Expected output:
# - Verified: Signature validation passed
# - Attestation payload: SLSA provenance JSON

# Extract and inspect provenance
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity=https://kubernetes.io/namespaces/ci-cd/serviceaccounts/tekton-chains \
  --certificate-oidc-issuer=https://kubernetes.default.svc.cluster.local \
  registry.example.com/myapp:v1.0.0 \
  | jq -r '.payload | @base64d | fromjson | .predicate'

# Provenance includes:
# - buildType: "https://tekton.dev/chains/v2"
# - materials: [{"uri": "git+https://github.com/example/myapp.git", "digest": {"sha1": "..."}}]
# - recipe: Pipeline definition and parameters
# - metadata: Build start/finish times, reproducibility

# Verify SBOM attestation (if Chains configured to sign SBOMs)
cosign verify-attestation \
  --type spdxjson \
  --certificate-identity=... \
  registry.example.com/myapp:v1.0.0
```

## Policy Enforcement

Use Kubernetes admission controllers to enforce signed provenance:

```yaml
# Sigstore Policy Controller: Require SLSA provenance
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-slsa-provenance
spec:
  images:
    - glob: "registry.example.com/**"
  
  authorities:
    - name: tekton-chains
      keyless:
        url: https://fulcio.sigstore.dev
        identities:
          # Require signatures from Tekton Chains service account
          - issuer: https://kubernetes.default.svc.cluster.local
            subject: https://kubernetes.io/namespaces/ci-cd/serviceaccounts/tekton-chains
      attestations:
        - name: require-slsa
          predicateType: https://slsa.dev/provenance/v0.2
          policy:
            type: cue
            data: |
              // Require SLSA buildType
              predicate: buildType: "https://tekton.dev/chains/v2"
              
              // Require specific Git repository
              predicate: materials: [...{
                uri: =~"^git\\+https://github\\.com/example/.*"
              }]
              
              // Require recent build (within 24 hours)
              import "time"
              _now: time.Now()
              _buildTime: time.Parse(time.RFC3339, predicate.metadata.buildFinishedOn)
              _age: time.Since(_buildTime)
              _age < time.ParseDuration("24h")
```

## When to Use This Pattern

**Ideal For:**
- Organizations requiring SLSA Level 2+ compliance
- Teams without security expertise (automates signing)
- Cloud-native applications with Kubernetes CI/CD
- Regulated industries requiring build provenance (finance, healthcare, government)

**Benefits:**
- Zero developer friction (signing is automatic)
- Consistent provenance across all builds
- Integration with existing Tekton pipelines (minimal changes)
- Verifiable supply chain without manual steps

**Considerations:**
- Requires Tekton as CI/CD platform (or adapt pattern to other systems)
- Chains configuration complexity for advanced scenarios
- Storage requirements for OCI-stored attestations

## Advanced Configuration

### Key-Based Signing with Vault

For organizations preferring key-based signing:

```yaml
# Chains configuration for HashiCorp Vault
apiVersion: v1
kind: ConfigMap
metadata:
  name: chains-config
  namespace: tekton-chains
data:
  signers.kms.auth.address: "hashivault://vault.example.com:8200"
  signers.kms.auth.token: "s.XXXXXXXXXX"  # Vault token (use Secret in production)
  signers.kms.kmsref: "vault://chains-signing-key"
  
  # Alternative: Google Cloud KMS
  # signers.kms.kmsref: "gcpkms://projects/PROJECT/locations/LOCATION/keyRings/RING/cryptoKeys/KEY"
  
  # Alternative: AWS KMS
  # signers.kms.kmsref: "awskms:///arn:aws:kms:REGION:ACCOUNT:key/KEY-ID"
```

### Custom Attestation Types

Generate custom attestations for organization-specific requirements:

```yaml
# Custom Task: Security Approval Attestation
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: security-approval
spec:
  params:
    - name: IMAGE
    - name: APPROVER
  steps:
    - name: generate-approval
      image: registry.example.com/security-tools:latest
      script: |
        cat > approval-attestation.json <<EOF
        {
          "approved": true,
          "approver": "$(params.APPROVER)",
          "timestamp": "$(date -Iseconds)",
          "security-scan": "passed",
          "compliance-check": "passed"
        }
        EOF
        
        # Sign custom attestation
        cosign attest --key /keys/signing-key.pem \
          --predicate approval-attestation.json \
          --type https://example.com/security-approval/v1 \
          $(params.IMAGE)
```

## Troubleshooting

**Chains not signing artifacts:**
```bash
# Check Chains controller logs
kubectl logs -n tekton-chains deployment/tekton-chains-controller

# Verify Chains configuration
kubectl get configmap chains-config -n tekton-chains -o yaml

# Check PipelineRun/TaskRun has signing annotation
kubectl get pipelinerun myapp-build-v1.0 -o yaml | grep chains.tekton.dev/signed
```

**Verification failures:**
```bash
# Check certificate identity and issuer match Chains configuration
cosign verify-attestation --certificate-identity=... --certificate-oidc-issuer=...

# Verify Rekor transparency log entry
cosign verify-attestation --rekor-url=https://rekor.sigstore.dev ...

# Check Fulcio root certificate trust
cosign verify-attestation --certificate-chain=... ...
```

## Security Considerations

1. **Service Account Identity**: Chains signs using Kubernetes service account identity; restrict RBAC
2. **Key Protection**: If using key-based signing, protect KMS access credentials
3. **Transparency Logs**: Monitor Rekor for unexpected signing activity
4. **Provenance Policies**: Enforce provenance verification in admission controllers
5. **Pipeline Security**: Harden Tekton pipelines (non-root, resource limits, network policies)

## Related Examples

- For manual signing workflows, see [cosign-signing.md](cosign-signing.md)
- For SBOM generation details, see [sbom-generation.md](sbom-generation.md)
- For on-premises Sigstore, see [onprem-sigstore-deployment.md](onprem-sigstore-deployment.md)
- For air-gapped environments, see [airgap-signing-workflow.md](airgap-signing-workflow.md)
