# Tekton: Secure Build Pipeline with Image Signing and SBOM Generation

## Overview

This example demonstrates a complete Tekton CI/CD pipeline that builds container images, runs security scans, generates SBOMs, signs artifacts with Cosign, and generates provenance attestations. The pipeline runs identically on any Kubernetes cluster, enabling true cloud independence for CI/CD workflows.

## Pipeline Definition

```yaml
# secure-build-pipeline.yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: secure-build-pipeline
  namespace: ci-cd
spec:
  params:
    - name: git-url
      description: Git repository URL
    - name: git-revision
      description: Git branch/tag/commit
      default: main
    - name: image-name
      description: Fully qualified image name
    - name: dockerfile-path
      description: Path to Dockerfile in repo
      default: ./Dockerfile

  workspaces:
    - name: shared-workspace
      description: Workspace for source code and build artifacts

  tasks:
    # Task 1: Clone source repository
    - name: fetch-repository
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

    # Task 2: Run unit tests
    - name: run-tests
      runAfter: [fetch-repository]
      taskRef:
        name: pytest
        kind: ClusterTask
      params:
        - name: ARGS
          value: ["--cov", "--cov-report=xml"]
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 3: Run security linting (detect hardcoded secrets)
    - name: secret-scan
      runAfter: [fetch-repository]
      taskRef:
        name: gitleaks
        kind: ClusterTask
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 4: Build container image with Buildah
    - name: build-image
      runAfter: [run-tests, secret-scan]
      taskRef:
        name: buildah
        kind: ClusterTask
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: DOCKERFILE
          value: $(params.dockerfile-path)
        - name: TLSVERIFY
          value: "true"
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 5: Scan image for vulnerabilities
    - name: scan-image
      runAfter: [build-image]
      taskRef:
        name: trivy-scanner
        kind: Task
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: SEVERITY
          value: "CRITICAL,HIGH"
        - name: EXIT_CODE
          value: "1"  # Fail pipeline if critical vulnerabilities found

    # Task 6: Generate SBOM
    - name: generate-sbom
      runAfter: [scan-image]
      taskRef:
        name: syft-generate
        kind: Task
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: FORMAT
          value: "spdx-json"  # or cyclonedx-json

    # Task 7: Attach SBOM to image
    - name: attach-sbom
      runAfter: [generate-sbom]
      taskRef:
        name: cosign-attach
        kind: Task
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: SBOM_PATH
          value: "$(workspaces.source.path)/sbom.spdx.json"
      workspaces:
        - name: source
          workspace: shared-workspace

    # Task 8: Sign image with Cosign
    - name: sign-image
      runAfter: [attach-sbom]
      taskRef:
        name: cosign-sign
        kind: Task
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: COSIGN_EXPERIMENTAL
          value: "true"  # Use keyless signing with Fulcio/Rekor

    # Task 9: Tag image as latest (only for main branch)
    - name: tag-latest
      runAfter: [sign-image]
      when:
        - input: "$(params.git-revision)"
          operator: in
          values: ["main", "master"]
      taskRef:
        name: skopeo-copy
        kind: ClusterTask
      params:
        - name: srcImageURL
          value: "docker://$(params.image-name):$(tasks.fetch-repository.results.commit)"
        - name: destImageURL
          value: "docker://$(params.image-name):latest"

    # Task 10: Generate provenance attestation
    - name: create-attestation
      runAfter: [sign-image]
      taskRef:
        name: create-provenance
        kind: Task
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.fetch-repository.results.commit)
        - name: PIPELINE_RUN
          value: $(context.pipelineRun.name)
        - name: GIT_URL
          value: $(params.git-url)
        - name: GIT_COMMIT
          value: $(tasks.fetch-repository.results.commit)
```

## Supporting Task Definitions

### Trivy Vulnerability Scanner Task

```yaml
# trivy-scanner-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: trivy-scanner
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Image to scan
    - name: SEVERITY
      description: Vulnerability severities to check
      default: "CRITICAL,HIGH,MEDIUM"
    - name: EXIT_CODE
      description: Exit code when vulnerabilities are detected
      default: "0"
  steps:
    - name: scan
      image: aquasec/trivy:latest
      args:
        - image
        - --severity
        - $(params.SEVERITY)
        - --exit-code
        - $(params.EXIT_CODE)
        - --no-progress
        - --format
        - json
        - --output
        - /workspace/trivy-scan-results.json
        - $(params.IMAGE)
    - name: report
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: |
        #!/bin/bash
        echo "Vulnerability Scan Results:"
        cat /workspace/trivy-scan-results.json | python3 -m json.tool
```

### Syft SBOM Generation Task

```yaml
# syft-generate-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: syft-generate
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Image to generate SBOM for
    - name: FORMAT
      description: SBOM format (spdx-json, cyclonedx-json)
      default: "spdx-json"
  workspaces:
    - name: source
      description: Workspace to write SBOM
  steps:
    - name: generate
      image: anchore/syft:latest
      args:
        - packages
        - $(params.IMAGE)
        - --output
        - $(params.FORMAT)
        - --file
        - $(workspaces.source.path)/sbom.$(params.FORMAT)
    - name: display
      image: registry.access.redhat.com/ubi9/ubi-minimal:latest
      script: |
        #!/bin/bash
        echo "Generated SBOM:"
        ls -lh $(workspaces.source.path)/sbom.*
        echo "Package count:"
        grep -c '"name":' $(workspaces.source.path)/sbom.* || echo "N/A"
```

### Cosign Attach Task

```yaml
# cosign-attach-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: cosign-attach
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Image to attach SBOM to
    - name: SBOM_PATH
      description: Path to SBOM file
  workspaces:
    - name: source
      description: Workspace containing SBOM
  steps:
    - name: attach
      image: gcr.io/projectsigstore/cosign:latest
      script: |
        #!/bin/sh
        cosign attach sbom \
          --sbom $(workspaces.source.path)/sbom.* \
          --type spdx \
          $(params.IMAGE)
        echo "SBOM attached to $(params.IMAGE)"
```

### Cosign Sign Task

```yaml
# cosign-sign-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: cosign-sign
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
      description: Image to sign
    - name: COSIGN_EXPERIMENTAL
      description: Enable keyless signing
      default: "true"
  steps:
    - name: sign
      image: gcr.io/projectsigstore/cosign:latest
      env:
        - name: COSIGN_EXPERIMENTAL
          value: $(params.COSIGN_EXPERIMENTAL)
      script: |
        #!/bin/sh
        # Keyless signing using Fulcio and Rekor
        cosign sign $(params.IMAGE) --yes
        echo "Image signed: $(params.IMAGE)"

        # Verify the signature immediately
        cosign verify $(params.IMAGE) \
          --certificate-identity-regexp=".*" \
          --certificate-oidc-issuer-regexp=".*"
```

### Provenance Attestation Task

```yaml
# create-provenance-task.yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: create-provenance
  namespace: ci-cd
spec:
  params:
    - name: IMAGE
    - name: PIPELINE_RUN
    - name: GIT_URL
    - name: GIT_COMMIT
  steps:
    - name: create-attestation
      image: gcr.io/projectsigstore/cosign:latest
      script: |
        #!/bin/sh

        # Create in-toto attestation
        cat > /tmp/predicate.json <<EOF
        {
          "buildType": "https://tekton.dev/attestations/chains@v2",
          "builder": {
            "id": "https://tekton.dev/chains/v2"
          },
          "invocation": {
            "configSource": {
              "uri": "$(params.GIT_URL)",
              "digest": {
                "sha1": "$(params.GIT_COMMIT)"
              }
            }
          },
          "metadata": {
            "buildStartedOn": "$(date -Iseconds)",
            "buildFinishedOn": "$(date -Iseconds)",
            "completeness": {
              "parameters": true,
              "environment": true,
              "materials": true
            }
          },
          "materials": [
            {
              "uri": "$(params.GIT_URL)",
              "digest": {
                "sha1": "$(params.GIT_COMMIT)"
              }
            }
          ]
        }
        EOF

        # Attach attestation to image
        cosign attest $(params.IMAGE) \
          --predicate /tmp/predicate.json \
          --type slsaprovenance \
          --yes

        echo "Provenance attestation created for $(params.IMAGE)"
```

## PipelineRun (Execution Instance)

```yaml
# pipelinerun.yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: secure-build-run-12345
  namespace: ci-cd
spec:
  pipelineRef:
    name: secure-build-pipeline
  params:
    - name: git-url
      value: https://github.com/example/myapp
    - name: git-revision
      value: main
    - name: image-name
      value: quay.io/example/myapp
    - name: dockerfile-path
      value: ./Dockerfile
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
          storageClassName: fast-ssd
```

## Triggering Pipeline from Git Events

```yaml
# github-eventlistener.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
  namespace: ci-cd
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: github
          params:
            - name: secretRef
              value:
                secretName: github-webhook-secret
                secretKey: token
            - name: eventTypes
              value: ["push"]
      bindings:
        - ref: github-push-binding
      template:
        ref: secure-build-trigger-template
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
  namespace: ci-cd
spec:
  params:
    - name: gitrepositoryurl
      value: $(body.repository.url)
    - name: gitrevision
      value: $(body.head_commit.id)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: secure-build-trigger-template
  namespace: ci-cd
spec:
  params:
    - name: gitrepositoryurl
    - name: gitrevision
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: secure-build-run-
      spec:
        pipelineRef:
          name: secure-build-pipeline
        params:
          - name: git-url
            value: $(tt.params.gitrepositoryurl)
          - name: git-revision
            value: $(tt.params.gitrevision)
          - name: image-name
            value: quay.io/example/myapp
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 5Gi
```

## Key Elements

**Cloud-Agnostic CI/CD:**
- Tekton runs on any Kubernetes cluster (AWS EKS, Azure AKS, GCP GKE, OpenShift)
- Pipeline definitions are portable YAML manifests stored in Git
- No dependency on cloud provider CI/CD services (CodeBuild, Azure Pipelines, Cloud Build)
- Same pipeline executes identically across infrastructure

**Supply Chain Security:**
- Vulnerability scanning with Trivy fails pipeline if critical CVEs detected
- SBOM generation with Syft produces SPDX/CycloneDX artifacts
- Image signing with Cosign provides cryptographic verification
- Provenance attestations document build process for SLSA compliance
- All security artifacts stored in registry alongside images

**Automation and Triggers:**
- Git webhooks automatically trigger pipelines on push/PR events
- Event-driven architecture eliminates manual pipeline execution
- Interceptors validate webhook signatures and filter events
- TriggerTemplates parameterize PipelineRuns from webhook payloads

**Security Best Practices:**
- Tasks run with minimal privileges (no root except where required)
- Secrets managed through Kubernetes Secrets with RBAC
- Workspaces use encrypted persistent volumes
- Network policies isolate pipeline pods
- Admission controllers enforce security policies

## Monitoring Pipeline Execution

```bash
# List pipeline runs
oc get pipelinerun -n ci-cd

# Watch pipeline run progress
oc get pipelinerun secure-build-run-12345 -n ci-cd -w

# View detailed pipeline run status
oc describe pipelinerun secure-build-run-12345 -n ci-cd

# Get logs from specific task
oc logs -n ci-cd -l tekton.dev/pipelineRun=secure-build-run-12345 \
  -l tekton.dev/pipelineTask=build-image

# View all task logs
tkn pipelinerun logs secure-build-run-12345 -f -n ci-cd
```

## Verifying Signed Images

```bash
# Verify image signature
cosign verify quay.io/example/myapp:abc123 \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*"

# Retrieve SBOM
cosign download sbom quay.io/example/myapp:abc123 > sbom.json

# Verify provenance attestation
cosign verify-attestation quay.io/example/myapp:abc123 \
  --type slsaprovenance \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*"

# Display attestation predicate
cosign verify-attestation quay.io/example/myapp:abc123 \
  --type slsaprovenance \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*" \
  | jq -r '.payload' | base64 -d | jq '.predicate'
```

## Production Considerations

**Resource Limits:**
```yaml
# Add resource limits to tasks
spec:
  steps:
    - name: build
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
```

**Caching for Performance:**
- Use Tekton workspace caching to speed up builds
- Configure BuildKit cache or Buildah volume caching
- Reuse dependency layers across pipeline runs

**Multi-Architecture Builds:**
```yaml
# Build for multiple architectures
- name: buildah-multiarch
  params:
    - name: PLATFORMS
      value: "linux/amd64,linux/arm64"
```

**Notifications:**
- Integrate with Slack/Teams for pipeline failure alerts
- Send email notifications on security scan failures
- Create GitHub commit statuses from pipeline results

## When to Use This Pattern

- **Migrating from cloud-specific CI/CD** (CodeBuild, Azure Pipelines) to portable solution
- **Multi-cloud deployments** requiring consistent pipelines across infrastructure
- **Supply chain security requirements** mandating image signing and SBOMs
- **Regulatory compliance** needing audit trails and provenance
- **Digital sovereignty** avoiding lock-in to cloud provider CI/CD services

## Related Examples

- See `airgap-signing-workflow.md` for signing in air-gapped environments
- See `argocd-gitops-deployment.md` for deploying signed images with GitOps
- See `cosign-signing.md` for detailed Cosign signing workflows
