# AI Inference Server: vLLM Deployment on OpenShift

## Overview

This example demonstrates deploying Red Hat AI Inference Server (based on vLLM) on OpenShift for high-performance, scalable model serving with GPU acceleration.

## Deployment Configuration

```yaml
# vllm-inference-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-inference-server
  namespace: ai-inference
spec:
  replicas: 3  # Scale based on load
  selector:
    matchLabels:
      app: vllm-inference-server
  template:
    metadata:
      labels:
        app: vllm-inference-server
    spec:
      containers:
        - name: vllm
          image: registry.redhat.io/rhai/vllm-inference-server:latest
          args:
            - --model
            - /models/granite-13b-chat  # Red Hat Granite model
            - --served-model-name
            - granite-chat
            - --tensor-parallel-size
            - "2"  # Use 2 GPUs per replica for model parallelism
          env:
            - name: VLLM_ATTENTION_BACKEND
              value: "FLASHINFER"  # Optimized attention implementation
          resources:
            requests:
              memory: 48Gi
              cpu: 8
              nvidia.com/gpu: "2"  # Request 2 GPUs
            limits:
              memory: 48Gi
              nvidia.com/gpu: "2"
          volumeMounts:
            - name: model-storage
              mountPath: /models
              readOnly: true
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 5
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: model-registry-pvc
---
# Service exposing the inference API
apiVersion: v1
kind: Service
metadata:
  name: ai-inference-api
  namespace: ai-inference
spec:
  selector:
    app: vllm-inference-server
  ports:
    - port: 8000
      targetPort: 8000
      name: http
  type: LoadBalancer
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-inference-hpa
  namespace: ai-inference
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-inference-server
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: inference_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

## Key Elements

**vLLM Configuration:**
- `--tensor-parallel-size`: Distribute model across multiple GPUs for larger models
- `--served-model-name`: Alias for OpenAI-compatible API (clients call this name)
- `VLLM_ATTENTION_BACKEND=FLASHINFER`: Uses optimized attention kernel for faster inference

**GPU Resource Management:**
- Requests exactly 2 GPUs per pod via `nvidia.com/gpu: "2"`
- GPU Operator must be installed on OpenShift for GPU scheduling
- Pods scheduled only on nodes with available GPU resources

**Storage:**
- Model weights loaded from PersistentVolumeClaim (shared across pods)
- Read-only mount prevents accidental model modification
- Supports S3-compatible object storage (MinIO, Ceph RGW) via PVC

**Health Checks:**
- Liveness probe ensures container restart if server hangs
- Readiness probe prevents traffic routing to pods still loading models
- Model loading can take 30-60 seconds for large models

**Auto-Scaling:**
- HPA scales replicas based on CPU utilization and custom metrics
- Custom metric `inference_requests_per_second` requires Prometheus monitoring
- Scaling limited to 3-10 replicas (constrained by GPU availability)

## Production Considerations

1. **Model Storage Options:**
   ```yaml
   # Option 1: S3-compatible object storage
   volumeMounts:
     - name: model-storage
       mountPath: /models
   volumes:
     - name: model-storage
       csi:
         driver: s3.csi.aws.com
         volumeAttributes:
           bucketName: ai-models
           prefix: granite-13b-chat/

   # Option 2: NFS for multi-read access
   volumes:
     - name: model-storage
       nfs:
         server: nfs.example.com
         path: /models
   ```

2. **Multi-Model Serving:**
   ```yaml
   args:
     - --model
     - /models/granite-13b-chat
     - --model
     - /models/llama-2-70b
     # Serve multiple models from single deployment
   ```

3. **Resource Tuning:**
   - Increase `--tensor-parallel-size` for models >13B parameters
   - Adjust memory requests based on model size (rule of thumb: 2x model size in GB)
   - Monitor GPU memory utilization with `dcgm-exporter`

4. **Network Policies:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: vllm-inference-access
   spec:
     podSelector:
       matchLabels:
         app: vllm-inference-server
     ingress:
       - from:
           - namespaceSelector:
               matchLabels:
                 name: applications  # Only app namespace can access
         ports:
           - port: 8000
   ```

## When to Use This Pattern

- **Production LLM Serving**: Scalable, high-throughput inference for large language models
- **Multi-Cloud Portability**: Identical deployment across AWS, Azure, GCP, or on-premises
- **GPU Optimization**: Automatic batching and PagedAttention for efficient GPU utilization
- **OpenAI API Compatibility**: Drop-in replacement for OpenAI API with self-hosted models

## Performance Characteristics

**Throughput Optimization:**
- Continuous batching: Process multiple requests simultaneously
- PagedAttention: Efficient KV-cache management reduces memory overhead
- Typical throughput: 50-100 tokens/sec/GPU for 13B models

**Latency:**
- Time-to-first-token: 50-200ms (depends on prompt length)
- Inter-token latency: 10-20ms (generation speed)
- Cold start: 30-60 seconds (model loading time)

## Related Examples

- For single-server deployment, see [RHEL AI with InstructLab](rhel-ai-instructlab.md)
- For complete MLOps with model training, see [Kubeflow Pipeline](kubeflow-pipeline-definition.md)
- For production deployment with GitOps, see [KServe Model Deployment](kserve-model-deployment.md)
