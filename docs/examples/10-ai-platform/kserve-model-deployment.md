# KServe: Production Model Deployment

## Overview

This example demonstrates deploying a trained ML model to production using KServe for scalable, auto-scaling inference serving with GPU acceleration.

## InferenceService Configuration

```yaml
# fraud-detection-inferenceservice.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: fraud-detection
  namespace: ml-production
  annotations:
    # Require signed images for supply chain security
    policy.sigstore.dev/enforce: "true"
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      # Model from registry or storage
      storageUri: "pvc://model-storage/fraud-detection/v2"
      runtime: kserve-torchserve
      runtimeVersion: "0.8.2"

    # Scaling configuration
    minReplicas: 5  # Minimum replicas for availability
    maxReplicas: 20  # Auto-scale up to 20 based on load

    # Resource requirements per replica
    resources:
      requests:
        memory: 8Gi
        cpu: 2
        nvidia.com/gpu: 1  # Each replica gets 1 GPU
      limits:
        memory: 8Gi
        nvidia.com/gpu: 1

    # Autoscaling based on concurrent requests
    scaleTarget: 100  # Target 100 concurrent requests per replica
    scaleMetric: concurrency

  # Canary deployment for gradual rollout
  canaryTrafficPercent: 10  # Send 10% traffic to new version
---
# PersistentVolumeClaim for model storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: ml-production
spec:
  accessModes:
    - ReadWriteMany  # Multi-pod access
  resources:
    requests:
      storage: 100Gi
  storageClassName: ceph-rbd  # Distributed storage
---
# ServiceMonitor for Prometheus metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fraud-detection-metrics
  namespace: ml-production
spec:
  selector:
    matchLabels:
      serving.kserve.io/inferenceservice: fraud-detection
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
---
# NetworkPolicy: Restrict access to inference service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: fraud-detection-access
  namespace: ml-production
spec:
  podSelector:
    matchLabels:
      serving.kserve.io/inferenceservice: fraud-detection
  policyTypes:
    - Ingress
  ingress:
    # Only allow from application namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: payment-processing
      ports:
        - protocol: TCP
          port: 8080
```

## GitOps Deployment with ArgoCD

```yaml
# argocd-application-fraud-model.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fraud-detection-model
  namespace: argocd
spec:
  project: ml-production
  source:
    repoURL: https://github.com/example/ml-models-gitops
    targetRevision: main
    path: production/fraud-detection
  destination:
    server: https://kubernetes.default.svc
    namespace: ml-production
  syncPolicy:
    # Manual sync for production model updates (require approval)
    automated: null
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
  # Health checks for InferenceService
  ignoreDifferences:
    - group: serving.kserve.io
      kind: InferenceService
      jsonPointers:
        - /status
```

## Client Application: Inference with SPIFFE Authentication

```python
# fraud_detection_client.py
# Application calling model inference with mTLS authentication

import requests
from pyspiffe.workloadapi import WorkloadApiClient
from pyspiffe.svid.x509_svid import X509Svid
import json

def detect_fraud(transaction):
    """
    Call fraud detection model with SPIFFE-authenticated mTLS.

    Args:
        transaction: Dictionary with transaction features

    Returns:
        Dictionary with fraud probability and risk level
    """

    # Get SPIFFE identity for this application
    with WorkloadApiClient() as spiffe_client:
        svid: X509Svid = spiffe_client.fetch_x509_svid()

        # Create mTLS session with certificates
        session = requests.Session()
        session.cert = (svid.cert_chain_path, svid.private_key_path)
        session.verify = svid.trust_bundle_path

        # Prepare inference request
        inference_request = {
            "instances": [transaction]
        }

        # Call KServe inference endpoint
        try:
            response = session.post(
                'https://fraud-detection.ml-production.svc.cluster.local/v1/models/fraud-detection:predict',
                json=inference_request,
                timeout=0.100,  # 100ms timeout for real-time requirements
                headers={'Content-Type': 'application/json'}
            )

            response.raise_for_status()

            # Parse prediction
            result = response.json()
            fraud_probability = result['predictions'][0]['fraud_score']

            # Determine risk level
            if fraud_probability > 0.75:
                risk_level = "HIGH"
                action = "BLOCK"
            elif fraud_probability > 0.30:
                risk_level = "MEDIUM"
                action = "VERIFY"
            else:
                risk_level = "LOW"
                action = "APPROVE"

            return {
                'fraud_probability': fraud_probability,
                'risk_level': risk_level,
                'recommended_action': action
            }

        except requests.exceptions.Timeout:
            # Fail-open or fail-closed based on business requirements
            return {
                'error': 'Model timeout',
                'fraud_probability': None,
                'risk_level': 'UNKNOWN',
                'recommended_action': 'MANUAL_REVIEW'
            }

        except requests.exceptions.RequestException as e:
            # Log error and fail gracefully
            print(f"Model inference error: {e}")
            return {
                'error': str(e),
                'fraud_probability': None,
                'risk_level': 'UNKNOWN',
                'recommended_action': 'MANUAL_REVIEW'
            }


# Example usage
if __name__ == '__main__':
    # Transaction data
    transaction = {
        "transaction_amount": 1250.00,
        "merchant_category": "electronics",
        "time_since_last_transaction": 45,  # seconds
        "location_deviation": 850,  # km from usual location
        "device_fingerprint_match": False
    }

    # Get fraud prediction
    result = detect_fraud(transaction)

    print(f"Fraud Risk: {result['risk_level']}")
    print(f"Probability: {result['fraud_probability']:.2%}")
    print(f"Action: {result['recommended_action']}")
```

## Key Elements

**KServe InferenceService:**
- Declarative model deployment via Kubernetes CRD
- Automatic containerization of model with serving runtime
- Built-in autoscaling and load balancing

**Model Storage:**
- `storageUri` supports multiple backends: PVC, S3, GCS, Azure Blob
- PVC with ReadWriteMany for shared access across replicas
- Models loaded on pod startup (cold start: 10-30 seconds)

**Auto-Scaling:**
- `scaleMetric: concurrency` scales based on concurrent requests
- `scaleTarget: 100` aims for 100 concurrent requests per replica
- HPA controls replicas between minReplicas and maxReplicas

**Canary Deployment:**
- `canaryTrafficPercent: 10` sends 10% traffic to new model version
- Gradually increase percentage after validation (20% → 50% → 100%)
- Instant rollback if new model performance degrades

**Security:**
- SPIFFE/SPIRE mTLS authentication between client and model
- NetworkPolicy restricts access to specific namespaces
- Signed image verification via Sigstore policy

## Production Considerations

1. **Multi-Model Serving:**
   ```yaml
   # Serve multiple models from single InferenceService
   spec:
     predictor:
       model:
         storageUri: "pvc://model-storage/"
         # ModelMesh automatically routes to models by name
         runtime: kserve-mlserver
   ```

2. **A/B Testing:**
   ```yaml
   # Split traffic between model versions
   spec:
     predictor:
       # Version A
     canary:
       # Version B
     canaryTrafficPercent: 50  # Equal split for A/B test
   ```

3. **Custom Preprocessing:**
   ```yaml
   # Add transformer for custom preprocessing
   spec:
     transformer:
       containers:
         - name: preprocessor
           image: registry.example.com/fraud-preprocessor:latest
           env:
             - name: FEATURE_SCALING
               value: "standard"
     predictor:
       # ... model serving config
   ```

4. **Model Monitoring:**
   ```yaml
   # Custom metrics for model performance
   apiVersion: v1
   kind: Service
   metadata:
     name: fraud-detection-metrics
     annotations:
       prometheus.io/scrape: "true"
       prometheus.io/port: "8080"
       prometheus.io/path: "/metrics"
   ```

## Monitoring and Observability

```python
# Prometheus metrics for model performance
from prometheus_client import Counter, Histogram, Gauge
import time

# Metrics
inference_requests = Counter(
    'fraud_model_requests_total',
    'Total inference requests',
    ['model_version', 'result']
)

inference_latency = Histogram(
    'fraud_model_latency_seconds',
    'Inference latency in seconds',
    ['model_version']
)

model_predictions = Counter(
    'fraud_model_predictions_total',
    'Predictions by risk level',
    ['risk_level']
)

# Instrumented inference function
def detect_fraud_monitored(transaction, model_version='v2'):
    start_time = time.time()

    try:
        result = detect_fraud(transaction)

        # Record metrics
        latency = time.time() - start_time
        inference_latency.labels(model_version=model_version).observe(latency)
        inference_requests.labels(
            model_version=model_version,
            result='success'
        ).inc()
        model_predictions.labels(
            risk_level=result['risk_level']
        ).inc()

        return result

    except Exception as e:
        inference_requests.labels(
            model_version=model_version,
            result='error'
        ).inc()
        raise
```

## Performance Characteristics

**Latency:**
- Cold start: 10-30 seconds (model loading)
- Warm inference: 20-100ms p99 (GPU-accelerated)
- Network overhead: 5-10ms (in-cluster mTLS)

**Throughput:**
- Single GPU replica: 100-500 requests/second (depends on model size)
- Batching: 2-5x throughput improvement for high concurrency
- Horizontal scaling: Near-linear with replicas (up to network limits)

**Resource Utilization:**
- Memory: ~2x model size (PyTorch overhead)
- GPU memory: 40-80% utilization with optimal batching
- CPU: Minimal (preprocessing only)

## When to Use This Pattern

- **Production ML Serving**: Scalable, reliable model deployment with SLAs
- **Real-Time Inference**: Low-latency predictions (<100ms)
- **Auto-Scaling Workloads**: Traffic varies (fraud detection, recommendations)
- **Multi-Model Deployment**: Serve multiple models with unified interface
- **GitOps Workflows**: Declarative model deployment with version control

## Related Examples

- For model training, see [Kubeflow Pipeline Definition](kubeflow-pipeline-definition.md)
- For simpler inference, see [AI Inference Server Deployment](ai-inference-server-deployment.md)
- For development, see [Jupyter Notebook Environment](jupyter-notebook-deployment.md)
- For air-gapped deployment, see [Air-Gapped Model Transfer](airgap-model-transfer.md)
