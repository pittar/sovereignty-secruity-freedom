# Air-Gapped Model Transfer: Secure Model Deployment in Disconnected Environments

## Overview

This example demonstrates transferring trained ML models to air-gapped environments (government, defense, financial trading) with cryptographic verification for integrity and authenticity.

## Model Export from Connected Environment

```bash
#!/bin/bash
# export-model-airgap.sh
# Export trained model with cryptographic verification

set -e  # Exit on error

MODEL_NAME="benefit-screening"
MODEL_VERSION="v3"
MODEL_REGISTRY="registry.example.com"
SIGNING_KEY="airgap-signing@example.gc.ca"

echo "=== Exporting Model for Air-Gap Transfer ==="
echo "Model: ${MODEL_NAME}:${MODEL_VERSION}"
echo ""

# Step 1: Export trained model as container image
echo "[1/6] Exporting model container image..."
podman save ${MODEL_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION} \
  -o ${MODEL_NAME}-${MODEL_VERSION}.tar

echo "Image size: $(du -h ${MODEL_NAME}-${MODEL_VERSION}.tar | cut -f1)"
echo ""

# Step 2: Generate cryptographic hash for integrity verification
echo "[2/6] Generating SHA256 hash..."
sha256sum ${MODEL_NAME}-${MODEL_VERSION}.tar > ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256

echo "SHA256: $(cat ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256 | cut -d' ' -f1)"
echo ""

# Step 3: Sign the hash with organizational GPG key
echo "[3/6] Signing with GPG key..."
gpg --local-user ${SIGNING_KEY} \
    --armor \
    --sign ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256

echo "Signature created: ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256.asc"
echo ""

# Step 4: Generate model card with metadata
echo "[4/6] Generating model card..."
cat > model-card-${MODEL_VERSION}.json <<EOF
{
  "model_name": "${MODEL_NAME}",
  "version": "${MODEL_VERSION}",
  "created_date": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "framework": "pytorch 2.1.0",
  "training_data": {
    "dataset": "applications-2024-anonymized",
    "rows": 450000,
    "date_range": "2024-01-01 to 2024-12-31",
    "pii_handling": "Confidential containers with attestation",
    "data_residency": "Canada (SSC data centers)"
  },
  "performance": {
    "accuracy": 0.94,
    "precision": 0.91,
    "recall": 0.89,
    "f1_score": 0.90,
    "auc_roc": 0.95
  },
  "bias_evaluation": {
    "demographic_parity": "PASS",
    "equalized_odds": "PASS",
    "fairness_report": "bias-evaluation-${MODEL_VERSION}.pdf",
    "tested_attributes": ["age", "gender", "province", "language"]
  },
  "compliance": {
    "dadm_assessment": "COMPLETED",
    "impact_level": "HIGH",
    "human_review_required": true,
    "audit_trail": "mlflow-tracking-${MODEL_VERSION}.log"
  },
  "trained_by": "ml-team@department.gc.ca",
  "approved_by": "ai-governance@department.gc.ca",
  "approval_date": "2025-02-10",
  "deployment_approval": "SECURITY_TEAM_APPROVED"
}
EOF

echo "Model card created: model-card-${MODEL_VERSION}.json"
echo ""

# Step 5: Create SBOM (Software Bill of Materials)
echo "[5/6] Generating SBOM..."
cat > model-sbom-${MODEL_VERSION}.json <<EOF
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "version": 1,
  "metadata": {
    "component": {
      "name": "${MODEL_NAME}",
      "version": "${MODEL_VERSION}",
      "type": "machine-learning-model"
    }
  },
  "components": [
    {
      "name": "pytorch",
      "version": "2.1.0",
      "type": "library"
    },
    {
      "name": "scikit-learn",
      "version": "1.3.2",
      "type": "library"
    },
    {
      "name": "pandas",
      "version": "2.1.0",
      "type": "library"
    },
    {
      "name": "numpy",
      "version": "1.24.0",
      "type": "library"
    }
  ],
  "externalReferences": [
    {
      "type": "vcs",
      "url": "https://github.example.gc.ca/ml-models/benefit-screening"
    },
    {
      "type": "build-system",
      "url": "https://kubeflow.ml.internal.gc.ca/runs/${MODEL_VERSION}"
    }
  ]
}
EOF

echo "SBOM created: model-sbom-${MODEL_VERSION}.json"
echo ""

# Step 6: Package everything for transfer
echo "[6/6] Creating transfer package..."
mkdir -p ${MODEL_NAME}-${MODEL_VERSION}-airgap-package
mv ${MODEL_NAME}-${MODEL_VERSION}.tar ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/
mv ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256.asc ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/
cp model-card-${MODEL_VERSION}.json ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/
cp model-sbom-${MODEL_VERSION}.json ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/

# Create verification checklist
cat > ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/VERIFICATION.md <<EOF
# Air-Gap Transfer Verification Checklist

## Package Contents
- ${MODEL_NAME}-${MODEL_VERSION}.tar (model container image)
- ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256.asc (signed hash)
- model-card-${MODEL_VERSION}.json (model metadata)
- model-sbom-${MODEL_VERSION}.json (dependencies)

## Verification Steps (On Air-Gapped Environment)

1. Verify GPG signature:
   \`\`\`bash
   gpg --verify ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256.asc
   \`\`\`
   Expected: "Good signature from '${SIGNING_KEY}'"

2. Verify file integrity:
   \`\`\`bash
   sha256sum -c ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256
   \`\`\`
   Expected: "${MODEL_NAME}-${MODEL_VERSION}.tar: OK"

3. Load into air-gapped registry:
   \`\`\`bash
   podman load -i ${MODEL_NAME}-${MODEL_VERSION}.tar
   podman tag localhost/${MODEL_NAME}:${MODEL_VERSION} \\
     airgap-registry.ssc.gc.ca/models/${MODEL_NAME}:${MODEL_VERSION}
   podman push airgap-registry.ssc.gc.ca/models/${MODEL_NAME}:${MODEL_VERSION}
   \`\`\`

## Approval Required
- Security team verification: __________
- AI governance review: __________
- Deployment authorization: __________
- Date approved for deployment: __________
EOF

echo ""
echo "=== Transfer Package Ready ==="
echo "Package directory: ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/"
echo ""
echo "Files to transfer:"
ls -lh ${MODEL_NAME}-${MODEL_VERSION}-airgap-package/
echo ""
echo "Next steps:"
echo "1. Copy package to approved secure removable media (encrypted USB, optical disc)"
echo "2. Physical transfer to air-gapped facility following security protocols"
echo "3. Run verification script on air-gapped environment"
echo "4. Load model into air-gapped registry after verification"
```

## Model Import in Air-Gapped Environment

```bash
#!/bin/bash
# import-model-airgap.sh
# Import and verify model in air-gapped environment

set -e

MODEL_NAME="benefit-screening"
MODEL_VERSION="v3"
AIRGAP_REGISTRY="airgap-registry.ssc.gc.ca"
EXPECTED_SIGNER="airgap-signing@example.gc.ca"

echo "=== Air-Gap Model Import and Verification ==="
echo "Model: ${MODEL_NAME}:${MODEL_VERSION}"
echo ""

# Step 1: Verify GPG signature
echo "[1/5] Verifying GPG signature..."
gpg --verify ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256.asc 2>&1 | tee gpg-verification.log

# Check if signature is valid
if grep -q "Good signature from" gpg-verification.log; then
    echo "✓ GPG signature valid"

    # Verify it's from expected signer
    if grep -q "${EXPECTED_SIGNER}" gpg-verification.log; then
        echo "✓ Signed by authorized key: ${EXPECTED_SIGNER}"
    else
        echo "✗ ERROR: Signature from unauthorized key"
        exit 1
    fi
else
    echo "✗ ERROR: Invalid GPG signature"
    exit 1
fi
echo ""

# Step 2: Verify file integrity
echo "[2/5] Verifying file integrity (SHA256)..."
if sha256sum -c ${MODEL_NAME}-${MODEL_VERSION}.tar.sha256; then
    echo "✓ File integrity verified"
else
    echo "✗ ERROR: File integrity check failed"
    exit 1
fi
echo ""

# Step 3: Load model into local container storage
echo "[3/5] Loading model into container storage..."
podman load -i ${MODEL_NAME}-${MODEL_VERSION}.tar

# Verify image loaded successfully
if podman images | grep -q "${MODEL_NAME}.*${MODEL_VERSION}"; then
    echo "✓ Model image loaded successfully"
else
    echo "✗ ERROR: Model image not found after load"
    exit 1
fi
echo ""

# Step 4: Tag for air-gapped registry
echo "[4/5] Tagging for air-gapped registry..."
podman tag localhost/${MODEL_NAME}:${MODEL_VERSION} \
  ${AIRGAP_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION}

echo "✓ Tagged: ${AIRGAP_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION}"
echo ""

# Step 5: Push to air-gapped registry
echo "[5/5] Pushing to air-gapped registry..."
podman push ${AIRGAP_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION}

echo "✓ Model pushed to air-gapped registry"
echo ""

# Verify model is in registry
echo "Verifying model in registry..."
podman pull ${AIRGAP_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION}

if [ $? -eq 0 ]; then
    echo "✓ Model verified in air-gapped registry"
else
    echo "✗ ERROR: Could not pull model from registry"
    exit 1
fi

echo ""
echo "=== Import Complete ==="
echo "Model ready for deployment: ${AIRGAP_REGISTRY}/models/${MODEL_NAME}:${MODEL_VERSION}"
echo ""
echo "Model card: model-card-${MODEL_VERSION}.json"
echo "SBOM: model-sbom-${MODEL_VERSION}.json"
echo ""
echo "Deploy with:"
echo "  kubectl apply -f inferenceservice-${MODEL_NAME}-${MODEL_VERSION}.yaml"
```

## InferenceService for Air-Gapped Deployment

```yaml
# inferenceservice-benefit-screening-airgap.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: benefit-screening-airgap
  namespace: ml-serving
  annotations:
    # Document air-gap deployment approval
    deployment.gc.ca/approval-id: "AIRGAP-2025-02-010"
    deployment.gc.ca/approved-by: "security-team@department.gc.ca"
    deployment.gc.ca/dadm-assessment: "COMPLETED-HIGH-IMPACT"
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      # Model served from air-gapped internal registry
      storageUri: "pvc://airgap-model-storage/benefit-screening/v3"
      runtime: kserve-torchserve
      runtimeVersion: "0.8.2"  # Qualified version in air-gapped environment

    # Fixed scaling in air-gapped (no external autoscaling metrics)
    minReplicas: 3
    maxReplicas: 3  # Static scaling for predictable resource allocation

    resources:
      requests:
        nvidia.com/gpu: 1
        memory: 16Gi
        cpu: 4
      limits:
        nvidia.com/gpu: 1
        memory: 16Gi
        cpu: 4
---
# PersistentVolumeClaim for air-gapped model storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airgap-model-storage
  namespace: ml-serving
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: airgap-ceph-rbd  # Internal Ceph storage
```

## Key Elements

**Cryptographic Verification Chain:**
1. **SHA256 Hash**: Ensures file integrity (detects corruption)
2. **GPG Signature**: Proves authenticity and prevents tampering
3. **Expected Signer Check**: Validates signature from authorized key
4. **Registry Verification**: Confirms successful upload

**Model Metadata:**
- **Model Card**: Performance metrics, bias evaluation, compliance documentation
- **SBOM**: Complete dependency list for security review
- **Verification Checklist**: Step-by-step import procedure

**Air-Gap Security:**
- No network connectivity during transfer (physical media only)
- Multi-level verification before deployment
- Approval workflow with documented authorization
- Audit trail for compliance (DADM, security policies)

## Production Considerations

1. **Encrypted Transfer Media:**
   ```bash
   # Encrypt package for physical transfer
   tar czf - ${MODEL_NAME}-${MODEL_VERSION}-airgap-package | \
     gpg --symmetric --cipher-algo AES256 \
     -o ${MODEL_NAME}-${MODEL_VERSION}-transfer.tar.gz.gpg

   # Decrypt on air-gapped environment
   gpg --decrypt ${MODEL_NAME}-${MODEL_VERSION}-transfer.tar.gz.gpg | \
     tar xzf -
   ```

2. **Internal PyPI Mirror:**
   ```bash
   # Transfer ML framework dependencies
   pip download torch==2.1.0 scikit-learn==1.3.2 pandas==2.1.0 \
     -d ml-dependencies/

   # On air-gapped: Install from local directory
   pip install --no-index --find-links=ml-dependencies/ torch scikit-learn pandas
   ```

3. **Container Image Mirror:**
   ```bash
   # Mirror base images for air-gap
   skopeo copy \
     docker://registry.access.redhat.com/ubi9/ubi-minimal:9.3 \
     oci-archive:ubi9-minimal-9.3.tar

   # Import to air-gapped registry
   skopeo copy \
     oci-archive:ubi9-minimal-9.3.tar \
     docker://airgap-registry.ssc.gc.ca/ubi9/ubi-minimal:9.3
   ```

4. **Model Update Cadence:**
   - **Monthly**: Routine retraining with new data
   - **Quarterly**: Major model architecture updates
   - **Emergency**: Critical security patches or bias issues
   - Each update follows full verification workflow

## Security Best Practices

1. **Key Management:**
   - GPG signing keys stored in HSM (Hardware Security Module)
   - Separate signing keys for development, staging, production
   - Annual key rotation with documented procedures

2. **Transfer Protocol:**
   - Use encrypted, tamper-evident physical media
   - Two-person integrity for model transfer
   - Documented chain of custody

3. **Verification Logs:**
   - Save all verification outputs for audit trail
   - Include timestamps and verifier identity
   - Store in immutable audit log system

4. **Deployment Approval:**
   - Security team verification required
   - AI governance review for DADM compliance
   - Documented approval before production deployment

## When to Use This Pattern

- **Classified Government Systems**: Defense, intelligence, law enforcement
- **Air-Gapped Financial Systems**: High-frequency trading, clearing houses
- **Critical Infrastructure**: Power grid, telecommunications, transportation
- **Research Labs**: Proprietary IP protection, pre-publication research
- **Compliance Requirements**: Data sovereignty, regulated industries

## Performance Impact

**Transfer Time:**
- Model export: 5-15 minutes (depends on model size)
- Physical transfer: Hours to days (security protocols, logistics)
- Import and verification: 10-30 minutes
- Total: Plan 1-3 business days for complete workflow

**Operational Overhead:**
- Initial setup: Establish signing keys, procedures, approvals
- Per-transfer: Verification, documentation, approval workflow
- Trade-off: Security and compliance worth the overhead

## Related Examples

- For development, see [Jupyter Notebook Environment](jupyter-notebook-deployment.md)
- For training pipelines, see [Kubeflow Pipeline Definition](kubeflow-pipeline-definition.md)
- For production deployment, see [KServe Model Deployment](kserve-model-deployment.md)
- For connected environments, see [AI Inference Server Deployment](ai-inference-server-deployment.md)
