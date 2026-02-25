# On-Premises Sigstore Infrastructure Deployment

## Overview

This example demonstrates how to deploy a complete Sigstore infrastructure (Fulcio, Rekor, Certificate Transparency) in an on-premises or air-gapped Kubernetes cluster. This enables organizations to maintain digital sovereignty over their software supply chain security infrastructure without dependency on external services.

On-premises Sigstore provides keyless signing with organizational identity providers, maintains internal transparency logs for compliance, and ensures supply chain security operates within organizational boundaries.

## Architecture Components

**Sigstore Components:**
1. **Fulcio**: Certificate Authority for issuing short-lived signing certificates
2. **Rekor**: Transparency log for immutable signature records
3. **Trillian**: Merkle tree backend for Rekor's cryptographic guarantees
4. **Certificate Transparency (CT) Log**: Monitors certificate issuance for audit

**Supporting Infrastructure:**
- **MySQL/PostgreSQL**: Database backend for Trillian
- **Redis**: Caching and rate limiting for Rekor
- **Keycloak/LDAP**: Organizational identity provider for OIDC authentication

## Deployment Manifest

```yaml
# On-Premises Sigstore Deployment
# Prerequisites: Kubernetes cluster, persistent storage, DNS configured

apiVersion: v1
kind: Namespace
metadata:
  name: sigstore
---
# MySQL Database for Trillian
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: sigstore
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: sigstore
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: registry.internal.corp/mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: root-password
            - name: MYSQL_DATABASE
              value: trillian
            - name: MYSQL_USER
              value: trillian
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: trillian-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
---
# Trillian Log Server (Merkle tree backend for Rekor)
apiVersion: v1
kind: Service
metadata:
  name: trillian-log
  namespace: sigstore
spec:
  ports:
    - port: 8090
      name: grpc
    - port: 8091
      name: http
  selector:
    app: trillian-log
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trillian-log
  namespace: sigstore
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trillian-log
  template:
    metadata:
      labels:
        app: trillian-log
    spec:
      containers:
        - name: trillian-log-server
          image: registry.internal.corp/sigstore/trillian-log-server:v1.5.0
          args:
            - --storage_system=mysql
            - --mysql_uri=trillian:$(MYSQL_PASSWORD)@tcp(mysql.sigstore.svc.cluster.local:3306)/trillian
            - --rpc_endpoint=0.0.0.0:8090
            - --http_endpoint=0.0.0.0:8091
            - --quota_system=mysql
          env:
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: trillian-password
          ports:
            - containerPort: 8090
              name: grpc
            - containerPort: 8091
              name: http
---
# Trillian Log Signer (processes Merkle tree updates)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trillian-log-signer
  namespace: sigstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trillian-log-signer
  template:
    metadata:
      labels:
        app: trillian-log-signer
    spec:
      containers:
        - name: trillian-log-signer
          image: registry.internal.corp/sigstore/trillian-log-signer:v1.5.0
          args:
            - --storage_system=mysql
            - --mysql_uri=trillian:$(MYSQL_PASSWORD)@tcp(mysql.sigstore.svc.cluster.local:3306)/trillian
            - --force_master=true
            - --rpc_endpoint=0.0.0.0:8090
            - --http_endpoint=0.0.0.0:8091
          env:
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: trillian-password
---
# Redis for Rekor caching
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: sigstore
spec:
  ports:
    - port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: sigstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: registry.internal.corp/redis:7-alpine
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
      volumes:
        - name: redis-data
          emptyDir: {}
---
# Rekor Server (Transparency Log)
apiVersion: v1
kind: Service
metadata:
  name: rekor
  namespace: sigstore
spec:
  type: LoadBalancer  # Or ClusterIP with Ingress
  ports:
    - port: 443
      targetPort: 3000
      name: https
  selector:
    app: rekor
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rekor
  namespace: sigstore
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rekor
  template:
    metadata:
      labels:
        app: rekor
    spec:
      containers:
        - name: rekor-server
          image: registry.internal.corp/sigstore/rekor-server:v1.3.0
          args:
            - serve
            - --trillian_log_server.address=trillian-log.sigstore.svc.cluster.local:8090
            - --redis_server.address=redis.sigstore.svc.cluster.local:6379
            - --rekor_server.address=0.0.0.0:3000
            - --enable_retrieve_api=true
            - --trillian_log_server.tlog_id=$(TREE_ID)
          env:
            - name: TREE_ID
              valueFrom:
                configMapKeyRef:
                  name: rekor-config
                  key: tree-id
          ports:
            - containerPort: 3000
              name: http
          livenessProbe:
            httpGet:
              path: /api/v1/log/publicKey
              port: 3000
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /api/v1/log/publicKey
              port: 3000
            initialDelaySeconds: 10
---
# Fulcio Certificate Authority
apiVersion: v1
kind: Service
metadata:
  name: fulcio
  namespace: sigstore
spec:
  type: LoadBalancer  # Or ClusterIP with Ingress
  ports:
    - port: 443
      targetPort: 8080
      name: https
    - port: 5554
      targetPort: 5554
      name: grpc
  selector:
    app: fulcio
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fulcio
  namespace: sigstore
spec:
  replicas: 3  # High availability for CA
  selector:
    matchLabels:
      app: fulcio
  template:
    metadata:
      labels:
        app: fulcio
    spec:
      containers:
        - name: fulcio
          image: registry.internal.corp/sigstore/fulcio:v1.4.0
          args:
            - serve
            - --port=8080
            - --grpc-port=5554
            - --ca=fileca
            - --fileca-key=/var/run/fulcio-secrets/key.pem
            - --fileca-cert=/var/run/fulcio-secrets/cert.pem
            - --fileca-key-passwd=$(KEY_PASSWORD)
            - --ct-log-url=http://ctlog.sigstore.svc.cluster.local:6962/test
            # OIDC configuration for enterprise identity provider
            - --oidc-issuer=https://keycloak.internal.corp/auth/realms/sigstore
            - --oidc-client-id=fulcio
            - --oidc-type=email
            # Configure accepted certificate identity types
            - --certificateidentity-uri-email-regex=.*@corp\.com$
          env:
            - name: KEY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: fulcio-secrets
                  key: key-password
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 5554
              name: grpc
          volumeMounts:
            - name: fulcio-ca
              mountPath: /var/run/fulcio-secrets
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
      volumes:
        - name: fulcio-ca
          secret:
            secretName: fulcio-ca-keys  # Contains organizational root CA
---
# Certificate Transparency Log
apiVersion: v1
kind: Service
metadata:
  name: ctlog
  namespace: sigstore
spec:
  ports:
    - port: 6962
      name: http
  selector:
    app: ctlog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ctlog
  namespace: sigstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ctlog
  template:
    metadata:
      labels:
        app: ctlog
    spec:
      containers:
        - name: ctlog
          image: registry.internal.corp/sigstore/certificate-transparency-go:v1.1.0
          args:
            - --logtostderr
            - --tree_id=$(CT_TREE_ID)
            - --tree_type=LOG
            - --trillian_address=trillian-log.sigstore.svc.cluster.local:8090
            - --http_endpoint=0.0.0.0:6962
          env:
            - name: CT_TREE_ID
              valueFrom:
                configMapKeyRef:
                  name: ctlog-config
                  key: tree-id
          ports:
            - containerPort: 6962
              name: http
```

## Key Elements

**High Availability:**
- Multiple replicas for Fulcio (3+) and Rekor (2+)
- Persistent storage for MySQL/PostgreSQL
- Load balancing across service instances
- Health checks and readiness probes

**Security Configuration:**
- Fulcio integrates with organizational OIDC provider (Keycloak, Active Directory, Okta)
- Private keys stored in Kubernetes secrets (production: use external KMS or HSM)
- Certificate identity restrictions (e.g., only `@corp.com` email addresses)
- TLS encryption for all service communication (not shown, use cert-manager)

**Integration with Enterprise PKI:**
- Fulcio CA can chain to organizational root CA
- Certificates issued by Fulcio validate against enterprise trust stores
- HSM integration for FIPS 140-2 compliance (configure with `--hsm-caroot-id`)

**Scalability:**
- Trillian database requires capacity planning (growth rate: ~1KB per signature)
- Rekor API rate limiting via Redis
- MySQL/PostgreSQL tuning for write-heavy workloads

## Initialization Steps

After deploying the manifests, initialize the Trillian trees:

```bash
# Create Trillian tree for Rekor
kubectl exec -n sigstore deployment/trillian-log-server -- \
  /usr/bin/createtree --admin_server=localhost:8090

# Output: Created tree <TREE_ID>
# Save this TREE_ID to rekor-config ConfigMap

# Create Trillian tree for Certificate Transparency log
kubectl exec -n sigstore deployment/trillian-log-server -- \
  /usr/bin/createtree --admin_server=localhost:8090

# Save CT TREE_ID to ctlog-config ConfigMap

# Create ConfigMaps with tree IDs
kubectl create configmap rekor-config \
  --from-literal=tree-id=<REKOR_TREE_ID> \
  -n sigstore

kubectl create configmap ctlog-config \
  --from-literal=tree-id=<CT_TREE_ID> \
  -n sigstore

# Restart Rekor and CT log deployments to use new tree IDs
kubectl rollout restart deployment/rekor -n sigstore
kubectl rollout restart deployment/ctlog -n sigstore
```

## Client Configuration

Configure Cosign to use on-premises Sigstore infrastructure:

```bash
# Set environment variables for on-prem Sigstore
export SIGSTORE_REKOR_URL=https://rekor.internal.corp
export SIGSTORE_FULCIO_URL=https://fulcio.internal.corp
export SIGSTORE_CT_LOG_PUBLIC_KEY_FILE=/path/to/ctlog-public-key.pem

# Sign using on-prem infrastructure
cosign sign registry.internal.corp/myapp:v1.0

# Cosign will:
# 1. Authenticate with organizational OIDC (Keycloak)
# 2. Request certificate from internal Fulcio instance
# 3. Sign image with ephemeral key
# 4. Upload signature to internal Rekor instance

# Verify using on-prem infrastructure
cosign verify \
  --rekor-url=https://rekor.internal.corp \
  --certificate-identity=user@corp.com \
  --certificate-oidc-issuer=https://keycloak.internal.corp/auth/realms/sigstore \
  registry.internal.corp/myapp:v1.0
```

## When to Use This Pattern

**On-Premises Deployment:**
- Organizations requiring data sovereignty over supply chain security
- Compliance requirements prohibiting external service dependencies
- Integration with enterprise identity and PKI infrastructure
- Custom policy enforcement for certificate issuance

**Air-Gapped Deployment:**
- Classified government systems
- Defense contractors with network isolation requirements
- Critical infrastructure (power grids, financial systems)
- Research facilities with intellectual property protection

**Hybrid Deployment:**
- Public Sigstore for open-source releases
- Private Sigstore for internal/proprietary software
- Multi-cloud environments with regional data residency requirements

## Operational Considerations

1. **Database Maintenance**: Trillian database grows continuously; plan for capacity and backup
2. **Key Rotation**: Rotate Fulcio CA keys periodically; old certificates remain verifiable via Rekor
3. **Monitoring**: Track Rekor entry rate, Fulcio certificate issuance, database performance
4. **Disaster Recovery**: Backup Trillian database and Rekor state; immutability prevents log reconstruction
5. **Upgrades**: Test Sigstore component upgrades in staging; coordinate Fulcio/Rekor version compatibility

## Security Considerations

1. **CA Key Protection**: Store Fulcio CA private keys in HSM for production deployments
2. **Access Control**: Restrict access to Trillian database and Fulcio key material
3. **Identity Verification**: Configure strict OIDC identity policies in Fulcio
4. **Audit Logging**: Enable audit logs for certificate issuance and Rekor entries
5. **Network Isolation**: Deploy Sigstore components in dedicated network zones

## Related Examples

- For client-side signing with on-prem Sigstore, see [cosign-signing.md](cosign-signing.md)
- For air-gapped workflows, see [airgap-signing-workflow.md](airgap-signing-workflow.md)
- For CI/CD integration, see [tekton-secure-pipeline.md](tekton-secure-pipeline.md)
- For SBOM integration, see [sbom-generation.md](sbom-generation.md)
