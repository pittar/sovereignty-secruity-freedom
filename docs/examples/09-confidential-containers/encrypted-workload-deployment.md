# Confidential Containers: Encrypted Workload Deployment

## Overview

This example demonstrates deploying a confidential container workload with hardware-based TEE (Trusted Execution Environment) protection, encrypted container images, and attestation-based secret management. Confidential containers protect data in use, ensuring even cloud administrators cannot access sensitive workload data or code.

## Prerequisites

```bash
# Verify confidential computing hardware support
lscpu | grep -i sev  # AMD SEV-SNP
lscpu | grep -i tdx  # Intel TDX
lscpu | grep -i sgx  # Intel SGX

# Check if Kata Containers runtime is available
oc get runtimeclass
# Should show: kata-cc (Kata Containers with Confidential Computing)

# Verify attestation service is deployed
oc get pods -n confidential-containers-system
# Should show: kbs-* (Key Broker Service)
```

## Architecture Components

```
┌──────────────────────────────────────────────┐
│  Application Code & Data                     │
│  (Plaintext only in TEE)                     │
├──────────────────────────────────────────────┤
│  Encrypted Container Image                   │
│  (Decrypted only in TEE)                     │
├──────────────────────────────────────────────┤
│  Kata Containers Runtime with CC             │
│  (Manages TEE lifecycle)                     │
├──────────────────────────────────────────────┤
│  TEE (AMD SEV-SNP / Intel TDX)              │
│  - Hardware-encrypted memory                 │
│  - Attestation agent                         │
│  - Secret injection                          │
├──────────────────────────────────────────────┤
│  Hypervisor (Cannot access TEE memory)       │
├──────────────────────────────────────────────┤
│  Cloud Infrastructure                        │
│  (AWS, Azure, GCP, On-premises)             │
└──────────────────────────────────────────────┘

External Components:
┌────────────────────────────┐
│  Key Broker Service (KBS)  │
│  - Verifies attestation    │
│  - Releases secrets        │
│  - Policy enforcement      │
└────────────────────────────┘
```

## Step 1: Encrypt Container Image

```bash
# Install skopeo if not available
dnf install -y skopeo

# Create encryption configuration
cat > ocicrypt-config.json <<EOF
{
  "key-providers": {
    "attestation-agent": {
      "grpc": "127.0.0.1:50000"
    }
  }
}
EOF

# Encrypt container image
# This encrypts layers with a key that KBS will provide after attestation
skopeo copy \
  --encryption-key provider:attestation-agent:kbs:///${KBS_URL}/default/image-key \
  --encryptconfig ocicrypt-config.json \
  docker://quay.io/example/sensitive-app:v1.0 \
  docker://quay.io/example/sensitive-app:v1.0-encrypted

# Verify image is encrypted
skopeo inspect docker://quay.io/example/sensitive-app:v1.0-encrypted | jq '.Layers'
# Should show encrypted media types: application/vnd.oci.image.layer.v1.tar+gzip+encrypted

# Push encrypted image to registry
podman login quay.io
podman push quay.io/example/sensitive-app:v1.0-encrypted
```

## Step 2: Configure Key Broker Service (KBS)

```yaml
# kbs-configmap.yaml
# Configuration for Key Broker Service
apiVersion: v1
kind: ConfigMap
metadata:
  name: kbs-config
  namespace: confidential-containers-system
data:
  kbs-config.toml: |
    # KBS server configuration
    [server]
    address = "0.0.0.0:8080"

    # Attestation verification settings
    [attestation]
    backend = "coco-as"  # Confidential Containers Attestation Service
    policy_engine = "opa"  # Open Policy Agent

    # Key repository
    [repository]
    type = "local"  # or "vault" for HashiCorp Vault integration
    path = "/opt/confidential-containers/kbs/repository"

    # Policies
    [policy]
    default_policy = "deny"  # Deny by default, explicit allow required
---
# kbs-policy.yaml
# OPA policy for secret release
apiVersion: v1
kind: ConfigMap
metadata:
  name: kbs-policy
  namespace: confidential-containers-system
data:
  policy.rego: |
    package policy

    import future.keywords

    # Default deny
    default allow = false

    # Allow if attestation is valid and measurements match
    allow {
      input.attestation.verified
      check_measurements
      check_image_policy
    }

    # Verify TEE measurements match expected values
    check_measurements {
      # Check kernel hash
      input.attestation.measurements.kernel == "expected_kernel_hash"

      # Check initrd hash
      input.attestation.measurements.initrd == "expected_initrd_hash"

      # Verify minimum TEE firmware version
      input.attestation.tee_platform.firmware_version >= "minimum_version"
    }

    # Verify container image is approved
    check_image_policy {
      # Only allow specific encrypted images
      allowed_images := {
        "quay.io/example/sensitive-app:v1.0-encrypted",
        "quay.io/example/secure-backend:v2.1-encrypted"
      }

      allowed_images[input.request.image]
    }
---
# kbs-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kbs
  namespace: confidential-containers-system
spec:
  replicas: 2  # HA deployment
  selector:
    matchLabels:
      app: kbs
  template:
    metadata:
      labels:
        app: kbs
    spec:
      containers:
      - name: kbs
        image: ghcr.io/confidential-containers/kbs:latest
        ports:
        - containerPort: 8080
          name: https
        volumeMounts:
        - name: config
          mountPath: /etc/kbs
        - name: repository
          mountPath: /opt/confidential-containers/kbs/repository
        - name: tls-certs
          mountPath: /etc/kbs/tls
        env:
        - name: KBS_CONFIG
          value: /etc/kbs/kbs-config.toml
      volumes:
      - name: config
        configMap:
          name: kbs-config
      - name: repository
        persistentVolumeClaim:
          claimName: kbs-repository
      - name: tls-certs
        secret:
          secretName: kbs-tls
---
apiVersion: v1
kind: Service
metadata:
  name: kbs
  namespace: confidential-containers-system
spec:
  selector:
    app: kbs
  ports:
  - port: 443
    targetPort: 8080
  type: LoadBalancer
```

## Step 3: Store Secrets in KBS

```bash
# Set KBS endpoint
export KBS_URL="https://kbs.confidential-containers-system.svc.cluster.local"

# Store image decryption key
kbs-client set-secret \
  --url $KBS_URL \
  --path default/image-key \
  --value "$(cat image-encryption-key.pem)"

# Store application database password
kbs-client set-secret \
  --url $KBS_URL \
  --path default/db-password \
  --value "super-secret-password"

# Store API credentials
kbs-client set-secret \
  --url $KBS_URL \
  --path default/api-credentials \
  --value "$(cat api-credentials.json)"

# Store TLS certificates
kbs-client set-secret \
  --url $KBS_URL \
  --path default/app-tls-cert \
  --value "$(cat app-tls.crt)"

kbs-client set-secret \
  --url $KBS_URL \
  --path default/app-tls-key \
  --value "$(cat app-tls.key)"

# List stored secrets
kbs-client list-secrets --url $KBS_URL
```

## Step 4: Deploy Confidential Container Workload

```yaml
# confidential-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: confidential-app
  namespace: secure-workloads
spec:
  replicas: 3
  selector:
    matchLabels:
      app: confidential-app
  template:
    metadata:
      labels:
        app: confidential-app
      annotations:
        # Enable confidential computing for this pod
        io.katacontainers.config.runtime.cc: "true"
        # Attestation configuration
        io.katacontainers.config.attestation.enabled: "true"
        io.katacontainers.config.attestation.kbs_url: "https://kbs.confidential-containers-system.svc.cluster.local"
    spec:
      # Use Kata Containers with Confidential Computing runtime
      runtimeClassName: kata-cc

      containers:
      - name: secure-app
        # Encrypted container image
        image: quay.io/example/sensitive-app:v1.0-encrypted

        ports:
        - containerPort: 8443
          name: https
          protocol: TCP

        env:
        # KBS URL for attestation
        - name: ATTESTATION_URL
          value: "https://kbs.confidential-containers-system.svc.cluster.local"

        # Application will retrieve these secrets from KBS after attestation
        - name: DB_PASSWORD_PATH
          value: "kbs:///default/db-password"
        - name: API_CREDENTIALS_PATH
          value: "kbs:///default/api-credentials"

        # Secrets are injected into TEE after attestation
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true
        - name: app-data
          mountPath: /data

        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
          seccompProfile:
            type: RuntimeDefault

        # Resource limits
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"

        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /ready
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 10
          periodSeconds: 5

      volumes:
      # TLS certificates retrieved from KBS after attestation
      - name: tls-certs
        csi:
          driver: attestation.csi.k8s.io
          volumeAttributes:
            kbs-url: "https://kbs.confidential-containers-system.svc.cluster.local"
            secret-path: "default/app-tls-cert,default/app-tls-key"

      # Encrypted persistent storage
      - name: app-data
        persistentVolumeClaim:
          claimName: confidential-app-data

      # Affinity rules for hardware with TEE support
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              # Require nodes with AMD SEV-SNP or Intel TDX
              - key: feature.node.kubernetes.io/cpu-security.sev.snp
                operator: Exists
              # Or Intel TDX
              - key: feature.node.kubernetes.io/cpu-security.tdx
                operator: Exists

        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: confidential-app
              topologyKey: kubernetes.io/hostname
---
# confidential-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: confidential-app
  namespace: secure-workloads
spec:
  selector:
    app: confidential-app
  ports:
  - port: 443
    targetPort: 8443
    protocol: TCP
  type: LoadBalancer
---
# encrypted-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: confidential-app-data
  namespace: secure-workloads
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: encrypted-storage  # Storage class with encryption at rest
```

## Step 5: Verify Attestation and Deployment

```bash
# Deploy the application
oc apply -f confidential-deployment.yaml

# Watch pod startup
oc get pods -n secure-workloads -w

# Check pod is using Kata CC runtime
oc describe pod -n secure-workloads -l app=confidential-app | grep -A 5 "Runtime Class"

# Verify attestation occurred
oc logs -n secure-workloads -l app=confidential-app | grep -i attestation

# Check KBS logs for attestation verification
oc logs -n confidential-containers-system -l app=kbs | grep -i "attestation verified"

# Verify secrets were released
oc logs -n confidential-containers-system -l app=kbs | grep -i "secret released"

# Test application is running
oc get svc -n secure-workloads confidential-app
curl -k https://<EXTERNAL-IP>/health

# View attestation report (if application exposes it)
curl -k https://<EXTERNAL-IP>/attestation-report
```

## Application Code Example

```python
# app.py - Example application using KBS secrets
import os
import requests
import json

class KBSClient:
    """Client for retrieving secrets from Key Broker Service"""

    def __init__(self, kbs_url):
        self.kbs_url = kbs_url
        self.attestation_agent = "http://127.0.0.1:8006"  # Local AA service

    def get_secret(self, secret_path):
        """
        Retrieve secret from KBS after attestation
        secret_path format: kbs:///default/secret-name
        """
        # Parse secret path
        path = secret_path.replace("kbs:///", "")

        # Get attestation token from local attestation agent
        attest_resp = requests.get(f"{self.attestation_agent}/cdh/token")
        attestation_token = attest_resp.json()["token"]

        # Request secret from KBS with attestation token
        headers = {
            "Authorization": f"Bearer {attestation_token}",
            "Content-Type": "application/json"
        }

        resp = requests.get(
            f"{self.kbs_url}/kbs/v0/resource/{path}",
            headers=headers,
            verify=True  # Verify KBS TLS certificate
        )

        if resp.status_code == 200:
            return resp.text
        else:
            raise Exception(f"Failed to retrieve secret: {resp.status_code}")

# Initialize KBS client
kbs_url = os.getenv("ATTESTATION_URL")
kbs = KBSClient(kbs_url)

# Retrieve database password
db_password_path = os.getenv("DB_PASSWORD_PATH")
db_password = kbs.get_secret(db_password_path)

# Retrieve API credentials
api_creds_path = os.getenv("API_CREDENTIALS_PATH")
api_creds_json = kbs.get_secret(api_creds_path)
api_creds = json.loads(api_creds_json)

# Connect to database using retrieved secret
import psycopg2
conn = psycopg2.connect(
    host="postgres.secure-workloads.svc.cluster.local",
    database="app_db",
    user="app_user",
    password=db_password  # Retrieved from KBS after attestation
)

print("Successfully connected to database with KBS-provided credentials")
print("All sensitive data processing occurs in TEE-protected memory")
```

## Key Elements

**Hardware-Based TEE Protection:**
- AMD SEV-SNP or Intel TDX encrypts container memory at hardware level
- Cloud administrators cannot access TEE memory even with root/hypervisor access
- Cryptographic measurements verify TEE integrity

**Encrypted Container Images:**
- Images remain encrypted in registry and during distribution
- Only TEEs with valid attestation can decrypt images
- Protects proprietary code and embedded secrets

**Attestation-Based Secret Management:**
- Zero-trust secret release based on cryptographic attestation
- Secrets never exist in plaintext outside TEEs
- KBS policy enforces which TEEs can access which secrets

**Cloud Independence:**
- Same confidential container workloads run on AWS, Azure, GCP, or on-premises
- TEE protection independent of cloud provider
- Enables sovereign workloads on foreign cloud infrastructure

## Production Considerations

**Monitoring and Logging:**
```yaml
# Prometheus ServiceMonitor for KBS
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kbs-metrics
  namespace: confidential-containers-system
spec:
  selector:
    matchLabels:
      app: kbs
  endpoints:
  - port: metrics
    interval: 30s
```

**Backup and Disaster Recovery:**
- Backup KBS repository (secrets) to encrypted storage
- Replicate KBS across availability zones
- Test secret recovery procedures

**Performance Tuning:**
- TEE initialization adds ~1-2 seconds to pod startup
- Encrypted image decryption adds overhead proportional to image size
- Consider smaller base images to reduce startup time

**Cost Considerations:**
- Confidential computing instances cost 10-30% more than standard instances
- AWS: M6a/C6a with SEV-SNP
- Azure: DCasv5/DCadsv5 with SEV-SNP or DCsv3/DCdsv3 with SGX
- GCP: N2D with AMD SEV

## When to Use This Pattern

- **Protected B+ government data** on public cloud infrastructure
- **Healthcare/financial regulated workloads** requiring protection in use
- **Multi-party computation** where parties don't trust each other
- **Proprietary algorithms** processing sensitive data on untrusted infrastructure
- **Third-party data processing** where data owner requires cryptographic proof of protection

## Benefits for Canadian Digital Sovereignty

**Sovereign Workloads on Foreign Infrastructure:**
- Process Canadian citizen data on AWS/Azure with cryptographic protection
- Cloud administrators cannot access data even with physical server access
- Meet Protected B requirements on foreign public cloud

**Regulatory Compliance:**
- Cryptographic proof of data protection for auditors
- Hardware attestation provides evidence for security authorization
- Enables cloud adoption while maintaining compliance

**Strategic Flexibility:**
- Use public cloud for elasticity and global scale
- Maintain cryptographic sovereignty over data
- Credible repatriation option: workloads run identically on-premises

## Related Examples

- See `../03-supply-chain/cosign-signing.md` for signing encrypted images
- See `../09-workload-identity/spiffe-attestation.md` for workload identity integration
- See Section 08 for detailed confidential computing architecture
