# ArgoCD: GitOps Continuous Deployment

## Overview

This example demonstrates using ArgoCD for GitOps-based continuous deployment across multiple environments and clusters. ArgoCD monitors Git repositories and automatically synchronizes cluster state to match desired state defined in Git, enabling declarative, auditable deployments with automatic drift detection and self-healing.

## ArgoCD Installation

```bash
# Install ArgoCD operator on OpenShift
oc create namespace argocd
oc apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD components to be ready
oc wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Get initial admin password
oc -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access ArgoCD UI
oc port-forward svc/argocd-server -n argocd 8080:443
# Navigate to https://localhost:8080
```

## Git Repository Structure

```
app-manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── replica-count.yaml
│   │   └── ingress.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── replica-count.yaml
│   │   └── ingress.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── replica-count.yaml
│       ├── ingress.yaml
│       └── resource-limits.yaml
└── README.md
```

## Base Application Manifests

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: apps
spec:
  replicas: 1  # Overridden by overlays
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: quay.io/example/web-app:latest  # Overridden by Kustomize
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: apps
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: apps
data:
  environment: "base"
  log_level: "info"
---
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

## Environment Overlays

### Development Environment

```yaml
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: development
namePrefix: dev-
commonLabels:
  environment: development
resources:
  - ../../base
patchesStrategicMerge:
  - replica-count.yaml
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - environment=development
      - log_level=debug
images:
  - name: quay.io/example/web-app
    newTag: dev-abc123  # Updated by CI pipeline
---
# overlays/development/replica-count.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1  # Single replica for dev
```

### Production Environment

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
namePrefix: prod-
commonLabels:
  environment: production
resources:
  - ../../base
patchesStrategicMerge:
  - replica-count.yaml
  - resource-limits.yaml
configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - environment=production
      - log_level=info
images:
  - name: quay.io/example/web-app
    newTag: v1.2.3  # Promoted from staging after testing
---
# overlays/production/replica-count.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # High availability
---
# overlays/production/resource-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
```

## ArgoCD Application Definitions

### Development Application

```yaml
# argocd-apps/development.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-dev
  namespace: argocd
spec:
  project: default

  # Source configuration
  source:
    repoURL: https://github.com/example/app-manifests
    targetRevision: main
    path: overlays/development

  # Destination cluster and namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: development

  # Sync policy
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Force cluster state to match Git
      allowEmpty: false  # Prevent accidental deletion of all resources
    syncOptions:
      - CreateNamespace=true  # Auto-create namespace if missing
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Ignore differences in certain fields
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignore if HPA modifies replicas
```

### Staging Application

```yaml
# argocd-apps/staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/app-manifests
    targetRevision: main
    path: overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Production Application (Manual Sync)

```yaml
# argocd-apps/production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-prod
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/example/app-manifests
    targetRevision: production-release  # Dedicated branch for prod
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Manual sync for production (require approval)
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m

  # Notifications for prod changes
  notifications:
    - name: production-sync
      triggers:
        - on-deployed
        - on-health-degraded
```

## Multi-Cluster Deployment with ApplicationSet

```yaml
# applicationset-multi-cluster.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-app-multi-cluster
  namespace: argocd
spec:
  generators:
    # Cluster generator: deploy to all clusters with label
    - clusters:
        selector:
          matchLabels:
            environment: production

  template:
    metadata:
      name: 'web-app-{{name}}'
    spec:
      project: production
      source:
        repoURL: https://github.com/example/app-manifests
        targetRevision: production-release
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Register Multiple Clusters

```bash
# Add production cluster in AWS
argocd cluster add aws-prod-cluster \
  --name aws-prod \
  --label environment=production \
  --label region=us-east-1

# Add production cluster in Azure
argocd cluster add azure-prod-cluster \
  --name azure-prod \
  --label environment=production \
  --label region=canadacentral

# Add on-premises cluster
argocd cluster add onprem-prod-cluster \
  --name onprem-prod \
  --label environment=production \
  --label region=canada-ottawa

# List registered clusters
argocd cluster list
```

## Progressive Delivery with Argo Rollouts

```yaml
# rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app-rollout
  namespace: production
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: quay.io/example/web-app:v1.2.3
        ports:
        - containerPort: 8080

  # Canary deployment strategy
  strategy:
    canary:
      steps:
        - setWeight: 20   # Route 20% traffic to new version
        - pause:
            duration: 5m  # Wait 5 minutes
        - setWeight: 40
        - pause:
            duration: 5m
        - setWeight: 60
        - pause:
            duration: 5m
        - setWeight: 80
        - pause:
            duration: 5m

      # Automated rollback on high error rate
      analysis:
        templates:
          - templateName: error-rate-analysis
        startingStep: 1

      # Traffic routing
      trafficRouting:
        istio:
          virtualService:
            name: web-app-vsvc
            routes:
              - primary
---
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
  namespace: production
spec:
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      successCondition: result < 0.05  # Less than 5% error rate
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{status=~"5.."}[2m]))
            /
            sum(rate(http_requests_total[2m]))
```

## Monitoring and Operations

### Check Application Health

```bash
# List all applications
argocd app list

# Get application details
argocd app get web-app-prod

# Show application diff
argocd app diff web-app-prod

# View application history
argocd app history web-app-prod

# View sync status
argocd app sync-status web-app-prod
```

### Manual Operations

```bash
# Sync application manually
argocd app sync web-app-prod

# Sync specific resource
argocd app sync web-app-prod --resource Deployment:production/prod-web-app

# Hard refresh (re-fetch from Git)
argocd app get web-app-prod --refresh

# Rollback to previous version
argocd app rollback web-app-prod 3  # Rollback to revision 3

# Terminate sync if stuck
argocd app terminate-op web-app-prod
```

### Configure Notifications

```yaml
# argocd-notifications-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} has been deployed to {{.app.spec.destination.namespace}}.
      Sync status: {{.app.status.sync.status}}
      Health status: {{.app.status.health.status}}
      Repository: {{.app.spec.source.repoURL}}
      Revision: {{.app.status.sync.revision}}
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "color": "good",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }, {
            "title": "Health Status",
            "value": "{{.app.status.health.status}}",
            "short": true
          }]
        }]
```

## Key Elements

**Declarative Configuration:**
- All application and infrastructure configuration in Git
- Changes through pull requests with code review
- Git history provides complete audit trail
- Rollback by reverting Git commits

**Automated Deployment:**
- ArgoCD automatically syncs cluster state to match Git
- Self-healing corrects manual changes or drift
- Prune policy removes resources deleted from Git
- Retry logic handles transient failures

**Multi-Environment Management:**
- Kustomize overlays provide environment-specific configuration
- Shared base manifests reduce duplication
- Environment promotion through Git branch/tag strategy
- Manual approval for production deployments

**Multi-Cluster Deployment:**
- ApplicationSets deploy to multiple clusters from single definition
- Cluster generators automatically discover target clusters
- Consistent application state across geographies
- Enables cloud independence and disaster recovery

**Progressive Delivery:**
- Argo Rollouts enables canary and blue-green deployments
- Automated metric-based rollback
- Gradual traffic shifting
- Reduces deployment risk

## Production Considerations

**Access Control:**
```yaml
# AppProject for production with restrictions
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications
  sourceRepos:
    - https://github.com/example/app-manifests  # Allowed repos
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
  roles:
    - name: prod-deployer
      policies:
        - p, proj:production:prod-deployer, applications, sync, production/*, allow
```

**Webhook Configuration for Instant Sync:**
```yaml
# Configure Git webhook for immediate sync on push
# Add webhook in GitHub/GitLab settings:
# URL: https://argocd.example.com/api/webhook
# Secret: configured in ArgoCD
```

**High Availability:**
- Run multiple ArgoCD replicas
- Use HA Redis for caching
- Configure multiple repo servers for scaling
- Set up monitoring and alerting

## When to Use This Pattern

- **Managing multiple environments** (dev/staging/prod) with consistent process
- **Multi-cluster deployments** across AWS, Azure, on-premises
- **Compliance and audit requirements** needing change tracking
- **Drift detection and prevention** ensuring cluster matches desired state
- **Cloud independence** deploying identical applications across infrastructure

## Benefits for Digital Sovereignty

**Infrastructure Portability:**
- Same GitOps workflow deploys to any Kubernetes cluster
- No dependency on cloud provider CD services
- Applications deploy identically across AWS, Azure, on-premises

**Audit and Compliance:**
- Complete audit trail in Git history
- All changes approved through pull request process
- Drift detection ensures production matches approved state

**Operational Consistency:**
- Single deployment process across all infrastructure
- Reduces operational complexity
- Enables migration between clouds without process changes

## Related Examples

- See `tekton-secure-build-pipeline.md` for building artifacts deployed by ArgoCD
- See `multi-cloud-deployment.md` for cluster configuration across providers
- See Section 03 for image verification in ArgoCD sync hooks
