# Digital Sovereignty Examples

This directory contains practical, production-ready examples demonstrating the technologies and patterns discussed in the Digital Sovereignty paper. Each example includes complete configuration files, detailed explanations, and guidance for real-world deployment.

## Directory Structure

```
examples/
├── 02-base-images/              # Container base image examples
├── 03-supply-chain/             # Supply chain security examples
├── 04-kubernetes-platform/      # Kubernetes and OpenShift platform examples
├── 05-cicd/                     # CI/CD pipeline examples
├── 06-developer-experience/     # Developer workspace examples
├── 08-confidential-containers/  # Confidential computing examples
└── 09-workload-identity/        # Zero trust workload identity examples
```

## Examples by Section

### Section 02: Base Images

**[dockerfile-ubi-python.md](02-base-images/dockerfile-ubi-python.md)**
- Building Python applications with UBI (Universal Base Image)
- Best practices for production container images
- Multi-stage builds and security considerations

### Section 03: Supply Chain Security

**[cosign-signing.md](03-supply-chain/cosign-signing.md)**
- Container image signing with Cosign
- Keyless signing using Fulcio and Rekor
- Image verification workflows

**[sbom-generation.md](03-supply-chain/sbom-generation.md)**
- Generating Software Bill of Materials (SBOMs)
- SPDX and CycloneDX formats
- Attaching SBOMs to container images

**[onprem-sigstore-deployment.md](03-supply-chain/onprem-sigstore-deployment.md)**
- Deploying Sigstore infrastructure on-premises
- Fulcio CA, Rekor transparency log, and Cosign
- Air-gapped supply chain security

**[airgap-signing-workflow.md](03-supply-chain/airgap-signing-workflow.md)**
- Signing and verifying images in air-gapped environments
- Offline transparency log and certificate authority
- Sovereign supply chain security

**[tekton-secure-pipeline.md](03-supply-chain/tekton-secure-pipeline.md)**
- Secure CI/CD pipeline with Tekton
- Automated image scanning, signing, and SBOM generation
- Integration with Sigstore

### Section 04: Kubernetes Platform

**[multi-cloud-deployment.md](04-kubernetes-platform/multi-cloud-deployment.md)**
- Deploying applications across AWS, Azure, and on-premises
- Cloud-agnostic Kubernetes configurations
- GitOps-based multi-cluster management

**[operator-custom-deployment.md](04-kubernetes-platform/operator-custom-deployment.md)**
- Using Operators to manage stateful applications
- PostgreSQL cluster with automated backups and HA
- Migrating from cloud managed services to operator-managed databases

### Section 05: CI/CD

**[tekton-secure-build-pipeline.md](05-cicd/tekton-secure-build-pipeline.md)**
- Complete Tekton pipeline with security scanning
- Image signing, SBOM generation, and provenance attestation
- Cloud-agnostic CI/CD

**[argocd-gitops-deployment.md](05-cicd/argocd-gitops-deployment.md)**
- GitOps continuous deployment with ArgoCD
- Multi-environment and multi-cluster deployments
- Progressive delivery with Argo Rollouts

### Section 06: Developer Experience

**[devfile-python-webapp.md](06-developer-experience/devfile-python-webapp.md)**
- Reproducible development environments with Devfiles
- Multi-container development (app, database, cache)
- Integration with OpenShift Dev Spaces

### Section 08: Confidential Containers

**[encrypted-workload-deployment.md](08-confidential-containers/encrypted-workload-deployment.md)**
- Deploying confidential containers with TEE protection
- Encrypted container images and attestation-based secrets
- Running Protected B workloads on public cloud

### Section 09: Workload Identity

**[spire-deployment.md](09-workload-identity/spire-deployment.md)**
- Zero trust workload identity with SPIRE
- Eliminating secrets with cryptographic identity
- Service-to-service mutual TLS authentication

## Using These Examples

### Prerequisites

Most examples assume you have access to:
- A Kubernetes or OpenShift cluster
- `kubectl` or `oc` CLI tools
- Container registry (e.g., Quay, Docker Hub)
- Basic familiarity with Kubernetes concepts

### Running Examples

1. **Read the Overview**: Each example starts with an overview explaining the use case and architecture

2. **Check Prerequisites**: Review any specific requirements or dependencies

3. **Follow Step-by-Step Instructions**: Examples are organized in logical steps with explanations

4. **Customize for Your Environment**: Replace placeholder values (registry URLs, domains, etc.) with your own

5. **Review Production Considerations**: Each example includes guidance for production deployments

### Example Structure

Each example typically includes:

```markdown
# Example Title

## Overview
- What the example demonstrates
- Use case and business value

## Prerequisites
- Required tools and access
- Assumed knowledge

## Architecture
- Visual representation (when applicable)
- Component relationships

## Step-by-Step Implementation
- Detailed configuration files
- Explanatory comments
- Command examples

## Key Elements
- Important concepts explained
- How components work together

## Production Considerations
- Scaling, HA, monitoring
- Security hardening
- Performance tuning

## When to Use This Pattern
- Appropriate scenarios
- Trade-offs and alternatives

## Related Examples
- Links to complementary examples
```

## Digital Sovereignty Themes

These examples demonstrate key sovereignty principles:

### 1. Open Standards and Open Source
- All examples use open-source projects (Kubernetes, Tekton, SPIRE, etc.)
- No proprietary vendor lock-in
- Community-driven standards (SPIFFE, SLSA, in-toto)

### 2. Infrastructure Portability
- Same configurations work on AWS, Azure, GCP, and on-premises
- Cloud-agnostic architectures
- Freedom to migrate between environments

### 3. Supply Chain Security
- Cryptographic verification of software artifacts
- Transparency through SBOMs and provenance attestation
- Control over the software supply chain

### 4. Zero Trust Security
- Eliminate secrets and API keys
- Cryptographic workload identity
- Hardware-based confidential computing

### 5. Regulatory Compliance
- Patterns for handling Protected B data
- Audit trails and attestation
- Data sovereignty on foreign infrastructure

## Contributing

When adding new examples:

1. Follow the established structure (see above)
2. Include complete, working configurations
3. Explain the "why" not just the "how"
4. Connect to digital sovereignty themes
5. Provide production considerations
6. Link to related examples

## Getting Help

- **Paper Sections**: Reference the main paper sections for conceptual background
- **Upstream Documentation**: Each example links to official project documentation
- **Community Support**: Join the communities for the technologies used

## Examples by Use Case

### Migrating from Cloud-Specific Services

- [operator-custom-deployment.md](04-kubernetes-platform/operator-custom-deployment.md) - Replace AWS RDS with operator-managed PostgreSQL
- [tekton-secure-build-pipeline.md](05-cicd/tekton-secure-build-pipeline.md) - Replace CodeBuild/Azure Pipelines with Tekton
- [argocd-gitops-deployment.md](05-cicd/argocd-gitops-deployment.md) - Cloud-agnostic deployment automation

### Achieving Regulatory Compliance

- [encrypted-workload-deployment.md](08-confidential-containers/encrypted-workload-deployment.md) - Protected B workloads on public cloud
- [airgap-signing-workflow.md](03-supply-chain/airgap-signing-workflow.md) - Air-gapped supply chain security
- [onprem-sigstore-deployment.md](03-supply-chain/onprem-sigstore-deployment.md) - Sovereign artifact signing infrastructure

### Eliminating Secrets

- [spire-deployment.md](09-workload-identity/spire-deployment.md) - Zero trust workload identity
- [encrypted-workload-deployment.md](08-confidential-containers/encrypted-workload-deployment.md) - Attestation-based secret management

### Multi-Cloud Deployment

- [multi-cloud-deployment.md](04-kubernetes-platform/multi-cloud-deployment.md) - Applications across AWS, Azure, on-prem
- [argocd-gitops-deployment.md](05-cicd/argocd-gitops-deployment.md) - Multi-cluster GitOps

### Developer Productivity

- [devfile-python-webapp.md](06-developer-experience/devfile-python-webapp.md) - Reproducible development environments
- [tekton-secure-build-pipeline.md](05-cicd/tekton-secure-build-pipeline.md) - Automated build and security scanning

## License

These examples are provided as educational material to accompany the Digital Sovereignty paper. They demonstrate real-world implementations of the concepts discussed and are intended to help organizations achieve digital sovereignty through open-source technologies.

## Feedback

Examples are living documents. If you find issues, have suggestions for improvements, or want to contribute additional examples, please reach out through the project repository.

---

[← Back to Main Documentation](../README.md) | [Table of Contents](../OUTLINE.md)
