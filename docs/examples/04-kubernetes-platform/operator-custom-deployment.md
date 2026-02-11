# Custom Operator: Deploying and Managing Complex Applications

## Overview

This example demonstrates using a Kubernetes Operator to automate the deployment and lifecycle management of a PostgreSQL database cluster with high availability, automated backups, and failover. Operators codify operational expertise as software, enabling declarative management of complex stateful applications.

## Installing an Operator from OperatorHub

```bash
# List available PostgreSQL operators
oc get packagemanifests -n openshift-marketplace | grep postgres

# Create a namespace for the database
oc new-project production-databases

# Install CloudNativePG Operator via CLI
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cloudnativepg
  namespace: production-databases
spec:
  channel: stable
  name: cloudnative-pg
  source: certified-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Verify operator installation
oc get csv -n production-databases
oc get pods -n production-databases
```

## Deploying a PostgreSQL Cluster Using the Operator

```yaml
# postgresql-cluster.yaml
# This custom resource is reconciled by the CloudNativePG Operator
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: financial-db
  namespace: production-databases
spec:
  # Cluster configuration
  instances: 3  # High availability with 3 replicas

  # Storage configuration
  storage:
    size: 500Gi
    storageClass: fast-ssd

  # PostgreSQL configuration
  postgresql:
    parameters:
      max_connections: "500"
      shared_buffers: "4GB"
      effective_cache_size: "12GB"
      work_mem: "32MB"
      maintenance_work_mem: "512MB"

  # Automated backup configuration
  backup:
    barmanObjectStore:
      destinationPath: s3://backups/financial-db
      s3Credentials:
        accessKeyId:
          name: aws-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
        maxParallel: 4
    retentionPolicy: "30d"

  # Monitoring integration
  monitoring:
    enablePodMonitor: true

  # High availability settings
  replicationSlots:
    highAvailability:
      enabled: true
    updateInterval: 30

  # Affinity rules to spread replicas across nodes
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              postgresql: financial-db
          topologyKey: kubernetes.io/hostname
```

## Creating a Scheduled Backup

```yaml
# scheduled-backup.yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: financial-db-daily
  namespace: production-databases
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  backupOwnerReference: self
  cluster:
    name: financial-db
  immediate: true  # Take first backup immediately
```

## Application Connection Configuration

```yaml
# application-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trading-app
  namespace: production-databases
spec:
  replicas: 3
  selector:
    matchLabels:
      app: trading-app
  template:
    metadata:
      labels:
        app: trading-app
    spec:
      containers:
      - name: app
        image: quay.io/example/trading-app:v1.0
        env:
        # Operator automatically creates connection secrets
        - name: DATABASE_HOST
          value: financial-db-rw.production-databases.svc.cluster.local
        - name: DATABASE_PORT
          value: "5432"
        - name: DATABASE_NAME
          value: trading
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: financial-db-app
              key: username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: financial-db-app
              key: password
```

## Monitoring Cluster Health

```bash
# Check cluster status
oc get cluster financial-db -n production-databases

# View detailed cluster information
oc describe cluster financial-db -n production-databases

# Check replication status
oc get pods -n production-databases -l postgresql=financial-db

# View operator logs
oc logs -n production-databases deployment/cloudnativepg-controller-manager

# Access PostgreSQL metrics
oc get podmonitor -n production-databases

# Check backup status
oc get backup -n production-databases
oc describe backup financial-db-daily-20240115020000
```

## Performing Database Operations

### Scaling the Cluster

```bash
# Scale to 5 instances
oc patch cluster financial-db -n production-databases --type='json' \
  -p='[{"op": "replace", "path": "/spec/instances", "value": 5}]'

# Operator automatically:
# - Provisions new PostgreSQL instances
# - Configures replication from primary
# - Updates load balancing
# - No downtime for applications
```

### Triggering Manual Backup

```yaml
# manual-backup.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: financial-db-before-upgrade
  namespace: production-databases
spec:
  cluster:
    name: financial-db
```

```bash
# Create backup
oc apply -f manual-backup.yaml

# Monitor backup progress
oc get backup financial-db-before-upgrade -n production-databases -w
```

### Restoring from Backup

```yaml
# restore-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: financial-db-restored
  namespace: production-databases
spec:
  instances: 3
  storage:
    size: 500Gi
    storageClass: fast-ssd

  # Bootstrap from backup
  bootstrap:
    recovery:
      backup:
        name: financial-db-before-upgrade
      recoveryTarget:
        targetTime: "2024-01-15 14:30:00"  # Point-in-time recovery
```

### Upgrading PostgreSQL Version

```bash
# Operator handles rolling upgrade automatically
oc patch cluster financial-db -n production-databases --type='json' \
  -p='[{"op": "replace", "path": "/spec/imageName", "value": "ghcr.io/cloudnative-pg/postgresql:16"}]'

# Operator performs:
# 1. Upgrade secondary replicas first
# 2. Verify replica health
# 3. Perform controlled failover
# 4. Upgrade former primary
# 5. Minimize downtime (typically < 1 minute)
```

## Key Elements

**Declarative Management:**
- Define desired database state (3 instances, 500GB storage, daily backups)
- Operator continuously reconciles actual state to match desired state
- No manual configuration of replication, failover, or backup scripts

**Automated Operations:**
- **High Availability:** Operator configures streaming replication, monitors health, performs automatic failover
- **Backups:** Continuous WAL archiving + scheduled full backups to S3-compatible storage
- **Scaling:** Add/remove instances declaratively, operator handles replication configuration
- **Upgrades:** Rolling upgrades with minimal downtime, automated health checks

**Production Ready:**
- Automated monitoring integration with Prometheus
- Point-in-time recovery capability
- Affinity rules for spreading replicas across failure domains
- Configurable connection pooling and load balancing

**Operational Benefits:**
- Codifies database expertise: replication setup, failover procedures, backup strategies
- Consistent deployments across development, staging, production environments
- Self-healing: operator detects failures and automatically recovers
- Simplified operations: `oc get cluster` shows health, no need for PostgreSQL-specific tools

## How Operators Enable Sovereignty

**Portability of Operational Knowledge:**
- Operator runs identically on AWS, Azure, on-premises vSphere
- Same PostgreSQL cluster manifest works across infrastructure
- Operational automation migrates with applications
- No dependence on cloud provider managed databases (RDS, Azure Database, Cloud SQL)

**Infrastructure Independence:**
- Replace AWS RDS with operator-managed PostgreSQL on any infrastructure
- No vendor lock-in to proprietary managed database services
- Full control over database configuration, version, and operations
- Can migrate from cloud managed services to on-premises without application changes

**Audit and Compliance:**
- All operations recorded in Kubernetes events and operator logs
- Custom resources stored in Git enable audit trail of changes
- No "black box" managed services—full visibility into database configuration
- Regulatory compliance through on-premises deployment when required

## Production Considerations

**Resource Planning:**
```yaml
# PostgreSQL resource requirements
resources:
  requests:
    memory: "8Gi"
    cpu: "4"
  limits:
    memory: "16Gi"
    cpu: "8"

# Consider:
# - PostgreSQL memory needs (shared_buffers, work_mem, effective_cache_size)
# - CPU for query processing and replication
# - Network bandwidth for replication and backup
# - Storage IOPS for database workload
```

**Backup Strategy:**
- Test restore procedures regularly (monthly minimum)
- Store backups in geographically separate location from primary
- Validate point-in-time recovery capability
- Monitor backup success and alert on failures
- Consider backup retention policy (30 days typical, adjust for compliance)

**Security:**
```yaml
# Enable TLS for client connections
postgresql:
  parameters:
    ssl: "on"

# Use cert-manager for certificate management
certificates:
  serverTLSSecret: postgresql-server-cert
  clientCASecret: postgresql-ca-cert

# Encrypt backups
backup:
  barmanObjectStore:
    encryption: AES256
```

**Monitoring and Alerting:**
- Configure Prometheus alerts for replication lag
- Monitor backup success/failure
- Alert on failover events
- Track query performance metrics
- Set up PagerDuty/OpsGenie integration for critical alerts

## When to Use This Pattern

- **Migrating from cloud managed databases** (AWS RDS, Azure Database) to maintain sovereignty
- **Complex stateful applications** requiring high availability and automated operations
- **Multi-environment consistency** (dev/staging/prod need identical database configuration)
- **Compliance requirements** mandating on-premises database deployment
- **Cost optimization** replacing expensive managed services with self-operated databases

## Related Examples

- See `multi-cloud-deployment.md` for running operator-managed databases across clouds
- See `disaster-recovery-testing.md` for backup and restore procedures
- See Section 05 for CI/CD pipelines deploying operator-managed applications
