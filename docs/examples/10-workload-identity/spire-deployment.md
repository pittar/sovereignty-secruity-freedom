# SPIRE Deployment: Zero Trust Workload Identity

## Overview

This example demonstrates deploying SPIRE (SPIFFE Runtime Environment) on Kubernetes to provide zero trust workload identity. SPIRE eliminates the need for secrets by issuing short-lived cryptographic credentials based on verifiable workload properties. Services authenticate using mutual TLS with SPIFFE identities instead of API keys or passwords.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Workload A                                     │
│  ┌──────────────┐      ┌──────────────┐        │
│  │ Application  │◄────►│ SPIRE Agent  │        │
│  │              │ SVID │ (sidecar)    │        │
│  └──────────────┘      └──────┬───────┘        │
└─────────────────────────────────┼──────────────┘
                                  │
                       Workload API
                                  │
                    ┌─────────────▼────────────┐
                    │   SPIRE Server Cluster   │
                    │   - Attestation          │
                    │   - SVID Issuance        │
                    │   - Policy Enforcement   │
                    └─────────────┬────────────┘
                                  │
                                  │ mTLS
                                  │
┌─────────────────────────────────┼──────────────┐
│  Workload B                     │              │
│  ┌──────────────┐      ┌────────▼───────┐     │
│  │ Application  │◄────►│ SPIRE Agent    │     │
│  │              │ SVID │ (sidecar)      │     │
│  └──────────────┘      └────────────────┘     │
└──────────────────────────────────────────────  │
```

## Step 1: Deploy SPIRE Server

```yaml
# spire-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: spire
---
# spire-server-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-server
  namespace: spire
---
# spire-server-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: spire-server-cluster-role
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: ["authentication.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]
---
# spire-server-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spire-server-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: spire-server-cluster-role
subjects:
- kind: ServiceAccount
  name: spire-server
  namespace: spire
---
# spire-server-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spire-server
  namespace: spire
data:
  server.conf: |
    server {
      bind_address = "0.0.0.0"
      bind_port = "8081"
      trust_domain = "example.com"
      data_dir = "/run/spire/data"
      log_level = "INFO"
      default_x509_svid_ttl = "1h"
      ca_subject = {
        country = ["CA"],
        organization = ["Example Org"],
        common_name = "example.com",
      }
    }

    plugins {
      DataStore "sql" {
        plugin_data {
          database_type = "postgres"
          connection_string = "postgresql://spire:spire@postgres:5432/spire?sslmode=disable"
        }
      }

      KeyManager "disk" {
        plugin_data {
          keys_path = "/run/spire/data/keys.json"
        }
      }

      NodeAttestor "k8s_psat" {
        plugin_data {
          clusters = {
            "production-cluster" = {
              service_account_allow_list = ["spire:spire-agent"]
            }
          }
        }
      }

      Notifier "k8sbundle" {
        plugin_data {
          namespace = "spire"
          config_map = "spire-bundle"
        }
      }
    }

    health_checks {
      listener_enabled = true
      bind_address = "0.0.0.0"
      bind_port = "8080"
      live_path = "/live"
      ready_path = "/ready"
    }
---
# spire-server-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: spire-server
  namespace: spire
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spire-server
  serviceName: spire-server
  template:
    metadata:
      labels:
        app: spire-server
    spec:
      serviceAccountName: spire-server
      containers:
      - name: spire-server
        image: ghcr.io/spiffe/spire-server:1.8.0
        args:
          - -config
          - /run/spire/config/server.conf
        ports:
        - containerPort: 8081
          name: grpc
        - containerPort: 8080
          name: healthz
        volumeMounts:
        - name: spire-config
          mountPath: /run/spire/config
          readOnly: true
        - name: spire-data
          mountPath: /run/spire/data
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: spire-config
        configMap:
          name: spire-server
  volumeClaimTemplates:
  - metadata:
      name: spire-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
---
# spire-server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: spire-server
  namespace: spire
spec:
  type: ClusterIP
  ports:
  - name: grpc
    port: 8081
    targetPort: 8081
    protocol: TCP
  selector:
    app: spire-server
---
# spire-bundle-configmap.yaml
# Trust bundle for workload verification
apiVersion: v1
kind: ConfigMap
metadata:
  name: spire-bundle
  namespace: spire
```

## Step 2: Deploy SPIRE Agent

```yaml
# spire-agent-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spire-agent
  namespace: spire
---
# spire-agent-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: spire-agent-cluster-role
rules:
- apiGroups: [""]
  resources: ["pods","nodes"]
  verbs: ["get"]
---
# spire-agent-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spire-agent-cluster-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: spire-agent-cluster-role
subjects:
- kind: ServiceAccount
  name: spire-agent
  namespace: spire
---
# spire-agent-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spire-agent
  namespace: spire
data:
  agent.conf: |
    agent {
      data_dir = "/run/spire"
      log_level = "INFO"
      server_address = "spire-server.spire.svc.cluster.local"
      server_port = "8081"
      trust_domain = "example.com"
    }

    plugins {
      NodeAttestor "k8s_psat" {
        plugin_data {
          cluster = "production-cluster"
        }
      }

      KeyManager "memory" {
        plugin_data {}
      }

      WorkloadAttestor "k8s" {
        plugin_data {
          skip_kubelet_verification = true
        }
      }
    }

    health_checks {
      listener_enabled = true
      bind_address = "0.0.0.0"
      bind_port = "8080"
      live_path = "/live"
      ready_path = "/ready"
    }
---
# spire-agent-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: spire-agent
  namespace: spire
spec:
  selector:
    matchLabels:
      app: spire-agent
  template:
    metadata:
      labels:
        app: spire-agent
    spec:
      hostPID: true
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      serviceAccountName: spire-agent
      initContainers:
      - name: init
        image: ghcr.io/spiffe/wait-for-it:latest
        args:
        - -t
        - "30"
        - spire-server.spire.svc.cluster.local:8081
      containers:
      - name: spire-agent
        image: ghcr.io/spiffe/spire-agent:1.8.0
        args:
          - -config
          - /run/spire/config/agent.conf
        volumeMounts:
        - name: spire-config
          mountPath: /run/spire/config
          readOnly: true
        - name: spire-bundle
          mountPath: /run/spire/bundle
          readOnly: true
        - name: spire-agent-socket
          mountPath: /run/spire/sockets
        - name: spire-token
          mountPath: /var/run/secrets/tokens
        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 60
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
      volumes:
      - name: spire-config
        configMap:
          name: spire-agent
      - name: spire-bundle
        configMap:
          name: spire-bundle
      - name: spire-agent-socket
        hostPath:
          path: /run/spire/sockets
          type: DirectoryOrCreate
      - name: spire-token
        projected:
          sources:
          - serviceAccountToken:
              path: spire-agent
              expirationSeconds: 7200
              audience: spire-server
```

## Step 3: Register Workloads

```yaml
# spiffe-id-registration.yaml
# Using SPIRE Controller Manager CRDs for declarative registration
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: payment-service
spec:
  # SPIFFE ID template using pod metadata
  spiffeIDTemplate: "spiffe://example.com/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"

  # Select pods for this identity
  podSelector:
    matchLabels:
      app: payment-service

  # Workload selector templates
  workloadSelectorTemplates:
    - "k8s:ns:{{ .PodMeta.Namespace }}"
    - "k8s:sa:{{ .PodSpec.ServiceAccountName }}"
    - "k8s:pod-label:app:payment-service"

  # DNS names to include in X.509 SVID
  dnsNameTemplates:
    - "payment-service.{{ .PodMeta.Namespace }}.svc.cluster.local"

  # TTL for issued SVIDs
  ttl: "1h"
---
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: order-service
spec:
  spiffeIDTemplate: "spiffe://example.com/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      app: order-service
  workloadSelectorTemplates:
    - "k8s:ns:{{ .PodMeta.Namespace }}"
    - "k8s:sa:{{ .PodSpec.ServiceAccountName }}"
  dnsNameTemplates:
    - "order-service.{{ .PodMeta.Namespace }}.svc.cluster.local"
  ttl: "1h"
```

## Step 4: Deploy Workloads with SPIFFE Identity

```yaml
# payment-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      serviceAccountName: payment-service-sa
      containers:
      - name: app
        image: quay.io/example/payment-service:v1.0
        ports:
        - containerPort: 8443
          name: https
        env:
        - name: SPIFFE_ENDPOINT_SOCKET
          value: "unix:///run/spire/sockets/agent.sock"
        volumeMounts:
        # Mount SPIRE agent socket
        - name: spire-agent-socket
          mountPath: /run/spire/sockets
          readOnly: true
      volumes:
      - name: spire-agent-socket
        hostPath:
          path: /run/spire/sockets
          type: Directory
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: production
---
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      serviceAccountName: order-service-sa
      containers:
      - name: app
        image: quay.io/example/order-service:v1.0
        ports:
        - containerPort: 8443
          name: https
        env:
        - name: SPIFFE_ENDPOINT_SOCKET
          value: "unix:///run/spire/sockets/agent.sock"
        - name: PAYMENT_SERVICE_URL
          value: "https://payment-service.production.svc.cluster.local:8443"
        volumeMounts:
        - name: spire-agent-socket
          mountPath: /run/spire/sockets
          readOnly: true
      volumes:
      - name: spire-agent-socket
        hostPath:
          path: /run/spire/sockets
          type: Directory
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: production
```

## Step 5: Application Code Using SPIFFE

### Go Application Example

```go
// payment-service/main.go
package main

import (
	"context"
	"crypto/tls"
	"log"
	"net/http"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
	"github.com/spiffe/go-spiffe/v2/workloadapi"
)

func main() {
	ctx := context.Background()

	// Create X.509 source from SPIRE Workload API
	source, err := workloadapi.NewX509Source(ctx)
	if err != nil {
		log.Fatalf("Unable to create X509Source: %v", err)
	}
	defer source.Close()

	// Get our SPIFFE ID
	svid, err := source.GetX509SVID()
	if err != nil {
		log.Fatalf("Unable to get X509SVID: %v", err)
	}
	log.Printf("My SPIFFE ID: %s", svid.ID)

	// Configure TLS to use SPIFFE SVIDs
	tlsConfig := tlsconfig.MTLSServerConfig(source, source, tlsconfig.AuthorizeAny())

	// Create HTTPS server with mutual TLS
	server := &http.Server{
		Addr:      ":8443",
		TLSConfig: tlsConfig,
	}

	http.HandleFunc("/process-payment", processPaymentHandler)
	http.HandleFunc("/health", healthHandler)

	log.Println("Payment service starting on :8443")
	log.Fatal(server.ListenAndServeTLS("", ""))
}

func processPaymentHandler(w http.ResponseWriter, r *http.Request) {
	// Extract client SPIFFE ID from verified certificate
	if r.TLS == nil || len(r.TLS.VerifiedChains) == 0 {
		http.Error(w, "No client certificate", http.StatusUnauthorized)
		return
	}

	clientSPIFFEID, err := spiffeid.FromString(
		r.TLS.VerifiedChains[0][0].URIs[0].String(),
	)
	if err != nil {
		http.Error(w, "Invalid SPIFFE ID", http.StatusUnauthorized)
		return
	}

	log.Printf("Request from SPIFFE ID: %s", clientSPIFFEID)

	// Authorize based on SPIFFE ID
	// Only allow order-service to process payments
	expectedID := spiffeid.RequireFromString("spiffe://example.com/ns/production/sa/order-service-sa")
	if !clientSPIFFEID.Equal(expectedID) {
		http.Error(w, "Unauthorized", http.StatusForbidden)
		return
	}

	// Process payment...
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status": "success"}`))
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("OK"))
}
```

### Client Application (Order Service)

```go
// order-service/main.go
package main

import (
	"context"
	"crypto/tls"
	"io"
	"log"
	"net/http"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
	"github.com/spiffe/go-spiffe/v2/workloadapi"
)

func main() {
	ctx := context.Background()

	// Create X.509 source from SPIRE Workload API
	source, err := workloadapi.NewX509Source(ctx)
	if err != nil {
		log.Fatalf("Unable to create X509Source: %v", err)
	}
	defer source.Close()

	// Call payment service with SPIFFE mTLS
	err = callPaymentService(source)
	if err != nil {
		log.Fatalf("Failed to call payment service: %v", err)
	}
}

func callPaymentService(source *workloadapi.X509Source) error {
	// Expected SPIFFE ID of payment service
	paymentServiceID := spiffeid.RequireFromString(
		"spiffe://example.com/ns/production/sa/payment-service-sa",
	)

	// Create HTTP client with mTLS using SPIFFE SVIDs
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: tlsconfig.MTLSClientConfig(
				source,
				source,
				tlsconfig.AuthorizeID(paymentServiceID), // Verify server identity
			),
		},
	}

	// Make request
	resp, err := client.Post(
		"https://payment-service.production.svc.cluster.local:8443/process-payment",
		"application/json",
		nil,
	)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	log.Printf("Response: %s", body)
	return nil
}
```

## Verification and Monitoring

```bash
# Check SPIRE Server health
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server healthcheck

# List registered entries
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server entry show

# Check SPIRE Agent health on a node
kubectl exec -n spire spire-agent-xxxxx -- /opt/spire/bin/spire-agent healthcheck

# View issued SVIDs from agent
kubectl exec -n spire spire-agent-xxxxx -- /opt/spire/bin/spire-agent api fetch x509

# Monitor SVID issuance metrics (Prometheus)
# spire_server_svid_issued_total
# spire_agent_workload_api_connection_count
```

## Key Elements

**Zero Trust Authentication:**
- No API keys or passwords in application code or configuration
- Services authenticate with short-lived X.509 certificates (1-hour TTL)
- Automatic certificate rotation without application restart

**Workload Attestation:**
- SPIRE verifies workload identity using Kubernetes properties
- ServiceAccount, Namespace, Pod labels provide verifiable attributes
- No shared secrets for authentication

**Authorization by Identity:**
- Services authorize requests based on SPIFFE ID, not IP address
- Fine-grained access control: only specific services can call specific endpoints
- Network location becomes irrelevant

**Infrastructure Portability:**
- Same SPIRE deployment works on AWS EKS, Azure AKS, GCP GKE, on-premises
- No dependency on cloud provider IAM systems
- Workload identity follows applications across infrastructure

## Production Considerations

**High Availability:**
- Run multiple SPIRE Server replicas with shared database
- Use PostgreSQL or MySQL for server datastore
- Deploy agents as DaemonSet for coverage on all nodes

**Federation Across Clusters:**
- Establish trust between SPIRE servers in different clusters
- Enable cross-cluster service authentication
- Manage trust domains and bundle exchange

**Monitoring and Alerting:**
- Track SVID issuance rates
- Alert on attestation failures
- Monitor agent connectivity to server

## When to Use This Pattern

- **Eliminating secrets** from application configuration
- **Zero trust security** for service-to-service authentication
- **Regulatory compliance** requiring cryptographic identity
- **Multi-cluster** deployments needing consistent identity
- **Cloud migration** maintaining security architecture across environments

## Related Examples

- See `spiffe-service-mesh.md` for integration with Istio/Envoy
- See `../08-confidential-containers/encrypted-workload-deployment.md` for combining SPIFFE with TEEs
- See Section 09 for detailed SPIFFE/SPIRE architecture
