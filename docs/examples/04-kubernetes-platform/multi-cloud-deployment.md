# Multi-Cloud OpenShift Deployment

## Overview

This example demonstrates deploying identical applications across multiple cloud providers (AWS, Azure, on-premises) using OpenShift, showcasing true cloud independence and workload portability. The same application manifests work unchanged across all infrastructure targets.

## Cluster Configuration Examples

### AWS Cluster Configuration

```yaml
# install-config-aws.yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: openshift-aws-prod
platform:
  aws:
    region: us-east-1
    # Availability zones for HA
    zones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
compute:
  - name: worker
    platform:
      aws:
        type: m5.2xlarge
    replicas: 3
controlPlane:
  name: master
  platform:
    aws:
      type: m5.xlarge
  replicas: 3
pullSecret: '{"auths": {...}}'
sshKey: |
  ssh-rsa AAAAB3NzaC1yc2E...
```

### Azure Cluster Configuration

```yaml
# install-config-azure.yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: openshift-azure-prod
platform:
  azure:
    region: canadacentral  # Canadian data sovereignty
    baseDomainResourceGroupName: openshift-rg
    # Availability zones for HA
    zones:
      - "1"
      - "2"
      - "3"
compute:
  - name: worker
    platform:
      azure:
        type: Standard_D8s_v3
    replicas: 3
controlPlane:
  name: master
  platform:
    azure:
      type: Standard_D4s_v3
  replicas: 3
pullSecret: '{"auths": {...}}'
sshKey: |
  ssh-rsa AAAAB3NzaC1yc2E...
```

### On-Premises (vSphere) Cluster Configuration

```yaml
# install-config-vsphere.yaml
apiVersion: v1
baseDomain: internal.example.com
metadata:
  name: openshift-onprem-prod
platform:
  vsphere:
    vcenter: vcenter.internal.example.com
    username: administrator@vsphere.local
    password: ${VSPHERE_PASSWORD}
    datacenter: Datacenter1
    defaultDatastore: datastore1
    cluster: Cluster1
    network: VM Network
    apiVIP: 10.0.0.5
    ingressVIP: 10.0.0.6
compute:
  - name: worker
    replicas: 3
controlPlane:
  name: master
  replicas: 3
pullSecret: '{"auths": {...}}'
sshKey: |
  ssh-rsa AAAAB3NzaC1yc2E...
```

## Application Deployment (Identical Across All Clusters)

```yaml
# deployment.yaml - Works identically on AWS, Azure, on-prem
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trading-platform
  namespace: financial-services
spec:
  replicas: 3
  selector:
    matchLabels:
      app: trading-platform
  template:
    metadata:
      labels:
        app: trading-platform
    spec:
      containers:
      - name: app
        image: quay.io/example/trading-platform:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_HOST
          value: postgresql.financial-services.svc.cluster.local
        - name: REGION
          valueFrom:
            configMapKeyRef:
              name: cluster-config
              key: region
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        # Storage abstraction - works with AWS EBS, Azure Disk, or vSphere VMDK
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: trading-data
---
apiVersion: v1
kind: Service
metadata:
  name: trading-platform
  namespace: financial-services
spec:
  selector:
    app: trading-platform
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer  # Automatically provisions AWS ELB, Azure LB, or on-prem HAProxy
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trading-data
  namespace: financial-services
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd  # Maps to different backends per infrastructure
```

## StorageClass Definitions (Infrastructure-Specific)

These are the only configuration changes needed per environment:

```yaml
# AWS - uses EBS volumes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
---
# Azure - uses Azure Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed
allowVolumeExpansion: true
---
# On-Premises - uses vSphere volumes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: thin
  datastore: datastore1
allowVolumeExpansion: true
```

## GitOps Deployment with ArgoCD

```yaml
# argocd-application.yaml
# Deploy the same app to all clusters
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trading-platform-aws
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/trading-platform
    targetRevision: main
    path: k8s/overlays/aws
  destination:
    server: https://api.openshift-aws-prod.example.com:6443
    namespace: financial-services
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trading-platform-azure
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/trading-platform
    targetRevision: main
    path: k8s/overlays/azure
  destination:
    server: https://api.openshift-azure-prod.example.com:6443
    namespace: financial-services
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trading-platform-onprem
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/trading-platform
    targetRevision: main
    path: k8s/overlays/onprem
  destination:
    server: https://api.openshift-onprem-prod.internal.example.com:6443
    namespace: financial-services
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Key Elements

**Infrastructure Abstraction:**
- Identical application manifests work across AWS, Azure, and on-premises vSphere
- Kubernetes APIs (Deployment, Service, PVC) abstract infrastructure differences
- Only infrastructure-specific components (StorageClass) change per environment
- Load balancers automatically provision appropriate backend (ELB, Azure LB, HAProxy)

**High Availability:**
- All cluster configurations use 3 availability zones for control plane resilience
- Worker nodes distributed across zones for application fault tolerance
- Identical HA architecture regardless of cloud provider

**Cloud Independence:**
- Applications use standard Kubernetes resources, not cloud-specific services
- No AWS Lambda, Azure Functions, or proprietary managed services
- Storage abstraction through PVC enables backend swapping without code changes
- Network abstraction through Services works identically across providers

**GitOps Consistency:**
- ArgoCD deploys from the same Git repository to all clusters
- Kustomize overlays handle environment-specific configuration
- Automated sync ensures cluster state matches Git state
- Single source of truth for all environments

**Migration Path:**
- Start on public cloud (AWS/Azure) for rapid deployment
- Develop and validate on elastic cloud infrastructure
- When ready, deploy identical workloads to on-premises Canadian infrastructure
- Zero application code changes required for migration

## Production Considerations

**Multi-Cluster Management:**
```yaml
# Use Advanced Cluster Management for centralized governance
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: openshift-aws-prod
spec:
  hubAcceptsClient: true
---
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: openshift-azure-prod
spec:
  hubAcceptsClient: true
---
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: openshift-onprem-prod
spec:
  hubAcceptsClient: true
```

**Global Policy Enforcement:**
- Use ACM Policy to enforce security requirements across all clusters
- Require network policies, resource limits, image signatures globally
- Automate compliance reporting and remediation

**Service Mesh Federation:**
- Connect services across clusters securely using service mesh
- Enable gradual traffic migration between clouds
- Implement disaster recovery failover between regions

**Disaster Recovery:**
- Use OADP (Velero) to backup cluster state to S3-compatible storage
- Maintain backup copies in multiple geographic locations
- Test restore procedures regularly across different infrastructure

## When to Use This Pattern

- **Regulatory compliance** requiring data residency in specific regions/countries
- **Risk mitigation** avoiding single cloud vendor dependency
- **Cost optimization** leveraging competitive pricing across providers
- **Merger/acquisition** integrating infrastructure from different organizations
- **Geographic distribution** serving users from local infrastructure
- **Sovereignty requirements** maintaining option to repatriate to Canadian data centers

## Benefits for Canadian Digital Sovereignty

**Strategic Flexibility:**
- Start on public cloud for rapid deployment and elastic scale
- Repatriate to Canadian on-premises infrastructure when sovereignty requires it
- Move workloads between environments based on policy, not technical constraints

**Vendor Independence:**
- Not locked into AWS, Azure, or any single cloud provider
- Maintain competitive leverage through credible exit options
- Avoid vendor-specific services that create dependencies

**Regulatory Compliance:**
- Deploy Protected B workloads on Canadian on-premises infrastructure
- Use public cloud for less sensitive workloads when appropriate
- Adapt to evolving Canadian government cloud policies without redesign

**Operational Consistency:**
- Same skills, tools, and processes across all infrastructure
- Unified security model and compliance framework
- Simplified operations through standardization

## Related Examples

- See `operator-custom-deployment.md` for automating application lifecycle across clusters
- See `gitops-multi-environment.md` for managing configuration across environments
- See `disaster-recovery-testing.md` for backup and restore procedures
