# CI/CD: The Enabler of Cloud Agnostic Development

[← Kubernetes Platform](04-kubernetes-platform.md) | [Table of Contents](../OUTLINE.md) | [Next: Developer Experience →](06-developer-experience.md)

---

## Executive Summary

Traditional CI/CD tools like Jenkins tie build processes to specific infrastructure—plugins for AWS services, Azure integrations, GCP-specific deployments. This infrastructure coupling creates lock-in at the automation layer. When organizations switch clouds or adopt hybrid strategies, they must rewrite pipelines, reconfigure integrations, and retrain teams. Cloud-native CI/CD with Tekton solves this by running pipelines as Kubernetes resources—portable across any Kubernetes cluster, any cloud, any infrastructure.

True cloud independence requires the ability to build, test, and deploy applications consistently regardless of the underlying infrastructure. Cloud-native CI/CD with Tekton and GitOps practices enable organizations to create portable pipelines that run anywhere, produce verifiable artifacts, and integrate seamlessly with supply chain security. This is the operational foundation of digital sovereignty.

GitOps extends this portability to deployment: desired state lives in Git, controllers synchronize clusters to match, and deployments happen identically whether targeting AWS, Azure, or on-premises Kubernetes. The combination—cloud-native CI with Tekton, continuous deployment with ArgoCD—creates fully portable automation that enables true infrastructure independence.

---

## The Evolution of CI/CD

### From Jenkins to Cloud-Native Pipelines

Jenkins dominated CI/CD for over a decade, offering flexibility through plugins for every build tool, SCM system, and deployment target. This plugin ecosystem became Jenkins' constraint—plugins require maintenance, version compatibility management, and often tie to specific infrastructure. Scaling Jenkins means managing worker nodes, coordinating distributed builds, and maintaining stateful server infrastructure. Backing up Jenkins configuration requires preserving jobs, credentials, and plugin state—fragile and error-prone.

### Why Cloud-Native CI/CD Matters

Cloud-native CI/CD runs build workloads as Kubernetes pods—ephemeral, scalable, and infrastructure-independent. Pipeline definitions are Kubernetes CRDs (Custom Resource Definitions), managed like any application resource. Kubernetes handles scheduling, resource allocation, and failure recovery. State lives in etcd alongside cluster resources, backed up through cluster backup procedures. Scale matches workload demand automatically—more pipeline runs trigger more pods without manual capacity planning.

**Key Advantages:**
- Consistent execution environment
- Scalability and resource efficiency
- Portability across clusters
- Integration with Kubernetes ecosystem
- Declarative pipeline definitions

---

## Tekton: Cloud-Native CI/CD Building Blocks

### What is Tekton?

Tekton provides Kubernetes-native building blocks for CI/CD systems. Originally developed at Google, Tekton entered CNCF as an incubating project with contributions from Red Hat, IBM, VMware, and others. Tekton defines Tasks (reusable build steps), Pipelines (compositions of tasks), and Triggers (event-driven automation) as Kubernetes CRDs. Organizations compose Tasks into Pipelines, store pipeline definitions in Git, and execute pipelines through PipelineRuns—all using standard Kubernetes tooling.

**Core Concepts:**
- **Tasks**: Reusable build steps
- **Pipelines**: Composed sequences of tasks
- **PipelineRuns**: Execution instances
- **Triggers**: Event-driven automation
- **Results**: Passing data between tasks

### Technical Deep Dive: Tekton Architecture

```yaml
# Example: Tekton Task for building container images
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-image
spec:
  params:
    - name: IMAGE
      description: Reference of the image to build
  workspaces:
    - name: source
  steps:
    - name: build
      image: registry.access.redhat.com/ubi9/buildah:latest
      script: |
        buildah bud -t $(params.IMAGE) .
        buildah push $(params.IMAGE)
      securityContext:
        privileged: true
```

This Task defines a single build step using Build

ah (a daemonless container build tool). Tasks declare parameters (IMAGE), workspaces (shared storage for source code), and steps (containers that execute build logic). The security context enables privileged mode for Buildah's rootless container builds. Tasks are reusable—multiple pipelines can reference this build-image Task, parameterized differently for each use case.

### OpenShift Pipelines

OpenShift Pipelines packages Tekton with enterprise extensions: integrated web console for pipeline visualization, Tekton Chains for automatic artifact signing and provenance generation, and curated Task catalog with tested tasks for common operations. Red Hat provides long-term support, security patches, and validated integration with OpenShift services. Organizations get production-ready Tekton without cobbling together upstream components.

---

## Building Cloud-Agnostic Pipelines

### Decoupling from Cloud-Specific Services

AWS CodeBuild, Azure Pipelines, and GCP Cloud Build integrate tightly with their respective clouds—authentication, artifact storage, deployment targets all assume the cloud provider. Migrating pipelines between clouds requires rewriting build configurations, reconfiguring credentials, and adapting to different pipeline DSLs. Tekton eliminates this coupling: pipelines run on Kubernetes regardless of infrastructure, reference container images from any OCI registry, and deploy to any Kubernetes cluster.

### Portable Build Definitions

Tekton pipelines defined as YAML manifests in Git move seamlessly across clusters. The same pipeline definition running on AWS EKS works identically on Azure AKS, GCP GKE, or on-premises OpenShift. Authentication uses Kubernetes ServiceAccounts and Secrets—consistent across infrastructure. Artifact storage uses any OCI-compliant registry. Deployment targets any Kubernetes cluster. This portability enables genuine multi-cloud CI/CD.

**This portability is critical for hybrid cloud strategies common in government and regulated industries.** Canadian federal departments often develop and test applications on public cloud infrastructure for rapid iteration (leveraging cloud elasticity and managed services during development), then deploy production workloads to Shared Services Canada data centers or departmental on-premises infrastructure to meet Protected B data residency requirements. Tekton enables this pattern: the same pipeline validates code in AWS development cluster, builds container images using identical base images (UBI), runs security scans, and produces signed artifacts—these artifacts deploy identically to on-premises production through GitOps, without pipeline modifications. Development velocity benefits from cloud resources while production sovereignty is maintained through on-premises deployment, all using one set of pipeline definitions stored in Git. When policy or cost considerations change, workloads shift between cloud and on-premises without CI/CD rewrites.

```yaml
# Example: Complete pipeline with multiple stages
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: cloud-agnostic-pipeline
spec:
  params:
    - name: git-url
    - name: image-name
  workspaces:
    - name: shared-workspace
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
      workspaces:
        - name: output
          workspace: shared-workspace

    - name: run-tests
      taskRef:
        name: pytest
      runAfter: [fetch-repository]
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: build-image
      taskRef:
        name: buildah
      runAfter: [run-tests]
      params:
        - name: IMAGE
          value: $(params.image-name)
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: scan-image
      taskRef:
        name: trivy-scan
      runAfter: [build-image]
      params:
        - name: IMAGE
          value: $(params.image-name)

    - name: sign-image
      taskRef:
        name: cosign-sign
      runAfter: [scan-image]
      params:
        - name: IMAGE
          value: $(params.image-name)
```

**→ See the complete example:** [Tekton: Secure Build Pipeline with Image Signing and SBOM Generation](../examples/05-cicd/tekton-secure-build-pipeline.md) for complete pipeline with security scanning, image signing, provenance attestation, and event triggers.

### Multi-Cloud Deployment Strategies

Pipelines can deploy artifacts to multiple clouds simultaneously or conditionally based on environment. Use when conditions in Tekton to target specific clusters—deploy to AWS for US traffic, Azure for European traffic, on-premises for sensitive workloads. Credentials for each cluster are stored as Kubernetes Secrets, pipelines reference them without hardcoded cloud-specific configurations. This pattern enables blue-green deployments across clouds, geographic distribution, and disaster recovery strategies.

**Hybrid deployment strategies** extend this multi-cloud model to include on-premises and edge infrastructure as first-class deployment targets. Organizations implement progressive deployment patterns: development environments in public cloud (AWS, Azure) for cost and elasticity, staging environments in on-premises infrastructure mirroring production, and production workloads distributed based on data sensitivity and regulatory requirements. A typical Canadian government deployment pattern uses Tekton pipelines to: build and test in cloud development cluster, promote validated artifacts to on-premises staging in SSC data centers for integration testing, deploy to on-premises production for citizen-facing services handling Protected B data, and optionally deploy less-sensitive components to cloud for geographic distribution and elastic scaling. The pipeline logic remains consistent—only target cluster credentials change. GitOps with ArgoCD manages promotion across environments, with Git commits triggering deployments to cloud, on-premises, or hybrid configurations from a single source of truth.

---

## GitOps: Declarative Continuous Deployment

### What is GitOps?

GitOps treats Git as the single source of truth for infrastructure and application state. Desired state—Kubernetes manifests, Helm charts, Kustomize overlays—lives in Git repositories. Controllers monitor Git, detect changes, and synchronize clusters to match desired state. This declarative, Git-centric approach brings version control, audit trails, and rollback capabilities to deployments. Changes happen through pull requests with code review, not manual kubectl commands or scripts.

**GitOps Principles:**
1. **Declarative**: System state described declaratively
2. **Versioned**: State stored in Git
3. **Automated**: Changes applied automatically
4. **Continuously Reconciled**: Actual state matches desired state

### ArgoCD and OpenShift GitOps

ArgoCD is the CNCF graduated project for GitOps continuous delivery. It monitors Git repositories for application manifests, compares desired state (in Git) with actual state (in Kubernetes), and synchronizes clusters automatically. ArgoCD supports multi-cluster deployments—managing applications across hundreds of clusters from a centralized control plane. OpenShift GitOps packages ArgoCD with Red Hat support, single sign-on integration, and validated OpenShift compatibility.

**ArgoCD Capabilities:**
- Automatic deployment from Git repositories
- Multi-cluster application delivery
- Sync policies and health monitoring
- Rollback and drift detection
- Web UI and CLI

### Technical Deep Dive: GitOps Workflow

```yaml
# Example: ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/manifests
    targetRevision: HEAD
    path: kubernetes/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This Application manifest tells ArgoCD to monitor the Git repository, sync manifests from kubernetes/production path to the production namespace, and automatically prune resources removed from Git while self-healing resources modified in-cluster. Changes committed to Git trigger automated deployments—no manual kubectl apply required. ArgoCD detects drift when cluster state diverges from Git and corrects it automatically.

**→ See the complete example:** [ArgoCD: GitOps Continuous Deployment](../examples/05-cicd/argocd-gitops-deployment.md) for multi-environment and multi-cluster deployments, progressive delivery with Argo Rollouts, and automated notifications.

---

## Integration with Supply Chain Security

### Signing Pipeline Artifacts

Tekton Chains automatically signs pipeline artifacts and generates provenance attestations. When pipelines complete, Chains signs the resulting container images with Cosign using keyless signing (Fulcio) or configured keys. Signatures attach to images in the registry, and signing events record in Rekor's transparency log. This automation embeds supply chain security into CI/CD without requiring developers to manage signing infrastructure.

### Generating and Attaching SBOMs

Pipeline tasks call Syft to generate SBOMs for container images, producing SPDX or CycloneDX formatted catalogs. Cosign attaches SBOMs to images as separate artifacts in the registry. Downstream consumers query registries for SBOMs, enabling vulnerability management and compliance checks without accessing source code or build systems. This pattern makes SBOMs first-class artifacts alongside container images.

### Build Attestation and Provenance

Tekton Chains generates in-toto attestations documenting build provenance: source repository, commit SHA, pipeline definition, build timestamp, and materials (dependencies). These attestations prove how artifacts were built, satisfying SLSA requirements for provenance. Admission controllers can verify attestations before allowing deployments, ensuring only properly built artifacts run in production.

```yaml
# Example: Attestation task
- name: create-attestation
  taskRef:
    name: create-build-attestation
  params:
    - name: IMAGE
      value: $(params.image-name)
    - name: PIPELINE_RUN
      value: $(context.pipelineRun.name)
    - name: GIT_COMMIT
      value: $(tasks.fetch-repository.results.commit)
```

---

## Artifact Promotion and Environment Parity

### Immutable Artifacts

Build container images once, tag with immutable identifiers (commit SHA, build number), and promote the same image through environments. Never rebuild from source for each environment—this introduces variability and invalidates testing. Pipelines produce artifacts in CI, test in staging, and deploy identical artifacts to production. GitOps manifests reference specific image tags, ensuring consistency across environments.

### Environment Configuration Management

Kustomize or Helm manage environment-specific configuration overlays. Base manifests define common application structure, overlays provide environment-specific values—replica counts, resource limits, ingress hostnames. Git repositories organize environments as directories (base/, staging/, production/), ArgoCD syncs appropriate overlays to clusters. Configuration changes flow through Git commits, not runtime modifications.

### Progressive Delivery

ArgoCD Rollouts enables advanced deployment strategies: canary releases (gradual traffic shift to new versions), blue-green deployments (instant cutover between versions), and automated rollback based on metrics. Rollouts integrate with Prometheus for metric-based promotion gates—automatically proceed if error rates stay low, rollback if metrics degrade. This automation reduces deployment risk without manual oversight.

---

## The Upstream Connection

### Red Hat's Tekton Contributions

Red Hat engineers serve as Tekton maintainers and steering committee members. Red Hat contributed Tekton Triggers (event-driven pipelines), Tekton Chains (automated signing), and numerous enhancements to core pipeline functionality. Red Hat employs ~15 engineers dedicated to upstream Tekton development, ensuring the project evolves to meet enterprise requirements while remaining vendor-neutral.

### Leadership in CD Foundation

Red Hat holds governing board seats at the Continuous Delivery Foundation (CDF), which hosts Tekton, Jenkins X, Spinnaker, and related projects. Red Hat engineers participate in CDF technical committees, drive interoperability initiatives across CD projects, and advocate for open standards in CI/CD tooling. This investment ensures the CD ecosystem remains open and portable.

---

## Security Best Practices

### Pipeline Security

Pipeline workspaces should use encrypted persistent volumes for sensitive data. Secrets injection uses Kubernetes Secrets with RBAC controlling access—pipeline ServiceAccounts receive minimum required permissions. Network policies isolate pipeline pods from unrelated workloads. Admission controllers enforce security policies: prohibit privileged containers in pipelines (except specific tasks requiring it), require resource limits, mandate network policy presence.

### Least Privilege and Isolation

Tasks run with dedicated ServiceAccounts scoped to specific namespaces and operations. Build tasks access source repositories and registries, not production databases. Deployment tasks access target clusters but not source code. This separation limits blast radius if pipeline components are compromised. Use Kubernetes Pod Security Standards to enforce baseline security requirements for pipeline pods.

### Audit and Compliance

Pipeline runs generate audit logs capturing: who triggered the run, what code was built, which tests passed/failed, what artifacts were produced, where deployments occurred. These logs flow to centralized logging systems (OpenShift Logging, external SIEM) for compliance reporting. Tekton Results API stores pipeline execution history, enabling compliance queries and historical analysis.

---

## Real-World Example: End-to-End Pipeline

This scenario demonstrates complete CI/CD with supply chain security integration. Developer commits trigger automated pipelines that build, test, scan for vulnerabilities, generate SBOMs, sign artifacts with cryptographic proof, and deploy through GitOps. The entire process produces verifiable audit trail—from commit to production deployment—satisfying compliance requirements while maintaining cloud independence.

**Scenario:**
1. Developer commits code to Git
2. Tekton Trigger fires pipeline
3. Pipeline checks out code, runs tests
4. Builds container image with Buildah
5. Scans image with Trivy
6. Generates SBOM with Syft
7. Signs image and SBOM with Cosign
8. Pushes artifacts to registry
9. ArgoCD detects new image
10. Automated deployment to staging
11. Manual approval gate for production
12. Deployment to production via GitOps

---

## Multi-Cluster and Multi-Cloud Deployment

### Deploying to Multiple Clusters

ArgoCD ApplicationSets provide declarative multi-cluster deployment. Define cluster list once (AWS, Azure, on-premises), ApplicationSet generates Application resources for each cluster automatically. Changes to cluster list propagate applications to new clusters without manual duplication. This pattern scales to hundreds of clusters across cloud providers and geographic regions.

### ApplicationSets in ArgoCD

ApplicationSets use generators (list, cluster, Git, matrix) to template applications across targets. List generators explicitly define clusters, cluster generators discover clusters dynamically through labels, Git generators create applications from repository structure. Combined with Kustomize overlays or Helm values, ApplicationSets deploy applications across diverse infrastructure while maintaining environment-specific configurations.

```yaml
# Example: ApplicationSet for multi-cluster deployment
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cloud-app
spec:
  generators:
    - list:
        elements:
          - cluster: aws-production
            url: https://aws.k8s.example.com
          - cluster: azure-production
            url: https://azure.k8s.example.com
          - cluster: onprem-production
            url: https://onprem.k8s.example.com
  template:
    metadata:
      name: '{{cluster}}-myapp'
    spec:
      project: default
      source:
        repoURL: https://github.com/example/manifests
        path: 'overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: production
```

---

## Key Benefits Summary

**For Technical Teams:**
- Portable, reusable pipeline definitions
- Integration with modern security practices
- Version-controlled deployment state

**For Organizations:**
- Consistent build and deployment processes
- Reduced cloud vendor dependencies
- Audit trail and compliance

**For Digital Sovereignty:**
- Freedom to run CI/CD anywhere
- No proprietary pipeline lock-in
- Transparent, open-source tooling

---

## References and Further Reading

- [Tekton Project](https://tekton.dev/)
- [ArgoCD](https://argoproj.github.io/cd/)
- [OpenShift Pipelines Documentation](https://docs.openshift.com/pipelines/latest/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/gitops/latest/)
- [GitOps Principles](https://opengitops.dev/)
- [Continuous Delivery Foundation](https://cd.foundation/)
- [Tekton Chains for Supply Chain Security](https://tekton.dev/docs/chains/)

---

**Next:** [6. Standardizing the Developer Experience →](06-developer-experience.md)
