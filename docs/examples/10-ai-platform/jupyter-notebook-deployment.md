# Jupyter Notebook: AI Development Environment on OpenShift AI

## Overview

This example demonstrates deploying a GPU-accelerated Jupyter notebook environment in OpenShift AI for data science and model development workflows.

## Notebook Configuration

```yaml
# jupyter-fraud-detection-notebook.yaml
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: fraud-detection-dev
  namespace: ml-development
  labels:
    app: jupyter-notebook
    team: data-science
spec:
  template:
    spec:
      serviceAccountName: notebook-fraud-detection
      containers:
        - name: notebook
          # UBI-based data science image with PyTorch
          image: registry.redhat.io/openshift-ai/pytorch-notebook:latest
          resources:
            requests:
              memory: 16Gi
              cpu: 4
              nvidia.com/gpu: 1  # Single GPU for development
            limits:
              memory: 16Gi
              nvidia.com/gpu: 1
          volumeMounts:
            - name: workspace
              mountPath: /home/jovyan
            - name: training-data
              mountPath: /data
              readOnly: true  # Read-only access to production data
            - name: shared-datasets
              mountPath: /datasets
          env:
            - name: JUPYTER_ENABLE_LAB
              value: "yes"  # Use JupyterLab interface
            - name: JUPYTER_TOKEN
              valueFrom:
                secretKeyRef:
                  name: jupyter-auth
                  key: token
            - name: GRANT_SUDO
              value: "no"  # Disable sudo for security
          ports:
            - containerPort: 8888
              name: notebook
              protocol: TCP
      volumes:
        - name: workspace
          persistentVolumeClaim:
            claimName: data-scientist-workspace-pvc  # Personal workspace
        - name: training-data
          persistentVolumeClaim:
            claimName: fraud-training-data-pvc  # Shared sensitive data
        - name: shared-datasets
          persistentVolumeClaim:
            claimName: team-datasets-pvc  # Team shared datasets
---
# PersistentVolumeClaim for personal workspace
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-scientist-workspace-pvc
  namespace: ml-development
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd  # NVMe storage for development
---
# Service for notebook access
apiVersion: v1
kind: Service
metadata:
  name: fraud-detection-notebook
  namespace: ml-development
spec:
  selector:
    notebook-name: fraud-detection-dev
  ports:
    - port: 8888
      targetPort: 8888
  type: ClusterIP
---
# Route for external access (optional, with OAuth)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: fraud-detection-notebook
  namespace: ml-development
  annotations:
    haproxy.router.openshift.io/timeout: "1h"  # Long timeout for notebooks
spec:
  to:
    kind: Service
    name: fraud-detection-notebook
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

## Example Development Workflow

```python
# fraud_detection_development.ipynb
# Jupyter notebook for fraud detection model development

import pandas as pd
import torch
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report, roc_auc_score

# ===== Data Loading =====
# Load training data from on-prem storage
# Data never leaves organizational boundaries
print("Loading transaction data from on-premises storage...")
transactions = pd.read_parquet('/data/transactions/2024/*.parquet')

print(f"Loaded {len(transactions):,} transactions")
print(f"Fraud rate: {transactions['is_fraud'].mean():.2%}")

# ===== Feature Engineering =====
features = [
    'transaction_amount',
    'merchant_category',
    'time_since_last_transaction',
    'location_deviation',
    'device_fingerprint_match'
]

X = transactions[features]
y = transactions['is_fraud']

# Encode categorical features
X_encoded = pd.get_dummies(X, columns=['merchant_category'])

# Train/test split
X_train, X_test, y_train, y_test = train_test_split(
    X_encoded, y, test_size=0.2, stratify=y, random_state=42
)

print(f"Training set: {len(X_train):,} samples")
print(f"Test set: {len(X_test):,} samples")

# ===== Model Development =====
# PyTorch neural network for fraud detection
class FraudDetectionModel(torch.nn.Module):
    def __init__(self, input_dim):
        super().__init__()
        self.layers = torch.nn.Sequential(
            torch.nn.Linear(input_dim, 128),
            torch.nn.ReLU(),
            torch.nn.Dropout(0.3),
            torch.nn.Linear(128, 64),
            torch.nn.ReLU(),
            torch.nn.Dropout(0.3),
            torch.nn.Linear(64, 1),
            torch.nn.Sigmoid()
        )

    def forward(self, x):
        return self.layers(x)

# Train on GPU if available
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Training on: {device}")

model = FraudDetectionModel(X_encoded.shape[1]).to(device)

# Training configuration
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
criterion = torch.nn.BCELoss()

# Convert to tensors
X_train_tensor = torch.tensor(X_train.values, dtype=torch.float32).to(device)
y_train_tensor = torch.tensor(y_train.values, dtype=torch.float32).reshape(-1, 1).to(device)

# Training loop
epochs = 50
batch_size = 256

for epoch in range(epochs):
    model.train()
    epoch_loss = 0

    for i in range(0, len(X_train_tensor), batch_size):
        batch_X = X_train_tensor[i:i+batch_size]
        batch_y = y_train_tensor[i:i+batch_size]

        optimizer.zero_grad()
        predictions = model(batch_X)
        loss = criterion(predictions, batch_y)
        loss.backward()
        optimizer.step()

        epoch_loss += loss.item()

    if (epoch + 1) % 10 == 0:
        print(f"Epoch {epoch+1}/{epochs}, Loss: {epoch_loss/len(X_train):.4f}")

# ===== Model Evaluation =====
model.eval()
with torch.no_grad():
    X_test_tensor = torch.tensor(X_test.values, dtype=torch.float32).to(device)
    predictions = model(X_test_tensor).cpu().numpy()

    # Calculate metrics
    auc_score = roc_auc_score(y_test, predictions)
    binary_predictions = (predictions > 0.5).astype(int)

    print(f"\nModel Performance:")
    print(f"AUC-ROC: {auc_score:.4f}")
    print("\nClassification Report:")
    print(classification_report(y_test, binary_predictions))

# ===== Save Model =====
# Save for pipeline deployment
model_save_path = '/home/jovyan/models/fraud_model_v1.pt'
torch.save({
    'model_state_dict': model.state_dict(),
    'feature_columns': list(X_encoded.columns),
    'auc_score': auc_score
}, model_save_path)

print(f"Model saved to: {model_save_path}")
```

## Key Elements

**GPU-Accelerated Development:**
- Single GPU allocated for rapid experimentation
- PyTorch automatically uses CUDA when available
- Faster iteration cycles compared to CPU-only development

**Data Access Patterns:**
- `/home/jovyan`: Personal workspace for notebooks and code (RWO PVC)
- `/data`: Read-only access to production training data (security)
- `/datasets`: Team-shared datasets for collaboration (RWX PVC)

**Security Considerations:**
- `GRANT_SUDO=no`: Prevents privilege escalation
- Read-only mount for sensitive training data
- Token-based authentication via Kubernetes secret
- Service account with limited RBAC permissions

**JupyterLab Interface:**
- Modern web-based IDE with file browser, terminal, and extensions
- Git integration for version control
- Extension ecosystem for specialized tools

## Production Considerations

1. **Resource Management:**
   ```yaml
   # Resource quotas for namespace
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: ml-development-quota
   spec:
     hard:
       requests.nvidia.com/gpu: "4"  # Max 4 GPUs for all notebooks
       requests.memory: "64Gi"
   ```

2. **Idle Notebook Culling:**
   ```yaml
   # Notebook Controller configuration (OpenShift AI)
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: notebook-controller-config
   data:
     culling_enabled: "true"
     idle_time: "3600"  # Cull notebooks idle for 1 hour
   ```

3. **Pre-configured Environments:**
   ```dockerfile
   # Custom notebook image with pre-installed packages
   FROM registry.redhat.io/openshift-ai/pytorch-notebook:latest

   USER root
   RUN pip install --no-cache-dir \
       scikit-learn==1.3.2 \
       xgboost==2.0.0 \
       shap==0.42.0 \
       mlflow==2.8.0

   USER 1001
   ```

4. **Shared Conda Environments:**
   ```yaml
   # ConfigMap with shared conda environment
   volumeMounts:
     - name: conda-envs
       mountPath: /opt/conda/envs
       readOnly: true
   volumes:
     - name: conda-envs
       persistentVolumeClaim:
         claimName: shared-conda-envs-pvc
   ```

## When to Use This Pattern

- **Interactive Model Development**: Data scientists experimenting with algorithms and features
- **Data Exploration**: Analyzing datasets before defining automated pipelines
- **Prototyping**: Rapid iteration before productionizing in Kubeflow Pipelines
- **Educational Environments**: Training data scientists on ML workflows
- **Collaborative Research**: Teams sharing notebooks and datasets

## Performance Tips

1. **Data Loading Optimization:**
   - Use Parquet format for columnar data (10-100x faster than CSV)
   - Leverage Dask for datasets larger than memory
   - Cache preprocessed features to avoid repeated computation

2. **GPU Utilization:**
   - Monitor with `nvidia-smi` or `gpustat` in notebook terminal
   - Use mixed precision (FP16) training for 2-3x speedup
   - Batch size tuning for optimal GPU memory usage

3. **Notebook Performance:**
   - Clear cell outputs for large visualizations
   - Use `%%time` and `%%timeit` magic commands for profiling
   - Restart kernel periodically to free memory

## Related Examples

- For production pipeline deployment, see [Kubeflow Pipeline Definition](kubeflow-pipeline-definition.md)
- For model serving, see [KServe Model Deployment](kserve-model-deployment.md)
- For simpler single-server development, see [RHEL AI with InstructLab](rhel-ai-instructlab.md)
