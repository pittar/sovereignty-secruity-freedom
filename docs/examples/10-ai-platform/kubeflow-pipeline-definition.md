# Kubeflow Pipeline: Automated ML Training Workflow

## Overview

This example demonstrates defining a Kubeflow Pipeline for reproducible, automated machine learning workflows with hyperparameter tuning and model registration.

## Pipeline Definition

```python
# kubeflow_fraud_detection_pipeline.py
# Define ML training pipeline as code using Kubeflow Pipelines SDK

from kfp import dsl
from kfp.dsl import Output, Model, Dataset, Metrics, Input

@dsl.component(
    base_image='registry.redhat.io/openshift-ai/pytorch-notebook:latest',
    packages_to_install=['scikit-learn==1.3.2', 'torch==2.1.0', 'pandas==2.1.0']
)
def load_and_preprocess_data(
    data_path: str,
    processed_data: Output[Dataset]
) -> None:
    """Load transaction data and perform feature engineering."""
    import pandas as pd
    from sklearn.preprocessing import StandardScaler
    import pickle

    # Load from on-prem storage
    print(f"Loading data from {data_path}")
    transactions = pd.read_parquet(f'{data_path}/*.parquet')

    # Feature engineering
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

    # Combine features and labels
    processed = pd.concat([X_encoded, y], axis=1)

    # Save processed data
    processed.to_parquet(processed_data.path, index=False)
    print(f"Processed {len(processed):,} samples")


@dsl.component(
    base_image='registry.redhat.io/openshift-ai/pytorch-notebook:latest',
    packages_to_install=['torch==2.1.0', 'scikit-learn==1.3.2', 'pandas==2.1.0']
)
def train_model(
    processed_data: Input[Dataset],
    learning_rate: float,
    batch_size: int,
    epochs: int,
    model_output: Output[Model],
    metrics_output: Output[Metrics]
) -> None:
    """Train fraud detection model with specified hyperparameters."""
    import torch
    import torch.nn as nn
    import pandas as pd
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import precision_recall_fscore_support, roc_auc_score
    import json

    # Load processed data
    data = pd.read_parquet(processed_data.path)
    X = data.drop('is_fraud', axis=1)
    y = data['is_fraud']

    X_train, X_val, y_train, y_val = train_test_split(
        X, y, test_size=0.2, stratify=y, random_state=42
    )

    # Model architecture
    class FraudDetectionModel(nn.Module):
        def __init__(self, input_dim):
            super().__init__()
            self.layers = nn.Sequential(
                nn.Linear(input_dim, 128),
                nn.ReLU(),
                nn.Dropout(0.3),
                nn.Linear(128, 64),
                nn.ReLU(),
                nn.Dropout(0.3),
                nn.Linear(64, 1),
                nn.Sigmoid()
            )

        def forward(self, x):
            return self.layers(x)

    # Training setup
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    print(f"Training on: {device}")

    model = FraudDetectionModel(X.shape[1]).to(device)
    optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
    criterion = nn.BCELoss()

    # Convert to tensors
    X_train_t = torch.tensor(X_train.values, dtype=torch.float32).to(device)
    y_train_t = torch.tensor(y_train.values, dtype=torch.float32).reshape(-1, 1).to(device)

    # Training loop
    model.train()
    for epoch in range(epochs):
        epoch_loss = 0

        for i in range(0, len(X_train_t), batch_size):
            batch_X = X_train_t[i:i+batch_size]
            batch_y = y_train_t[i:i+batch_size]

            optimizer.zero_grad()
            predictions = model(batch_X)
            loss = criterion(predictions, batch_y)
            loss.backward()
            optimizer.step()

            epoch_loss += loss.item()

        if (epoch + 1) % 10 == 0:
            avg_loss = epoch_loss / (len(X_train_t) / batch_size)
            print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")

    # Evaluation
    model.eval()
    with torch.no_grad():
        X_val_t = torch.tensor(X_val.values, dtype=torch.float32).to(device)
        predictions = model(X_val_t).cpu().numpy()

        # Calculate metrics
        auc_score = roc_auc_score(y_val, predictions)
        binary_preds = (predictions > 0.5).astype(int)
        precision, recall, f1, _ = precision_recall_fscore_support(
            y_val, binary_preds, average='binary'
        )

    # Log metrics for experiment tracking
    print(f"AUC-ROC: {auc_score:.4f}")
    print(f"Precision: {precision:.4f}, Recall: {recall:.4f}, F1: {f1:.4f}")

    metrics_output.log_metric('auc_roc', float(auc_score))
    metrics_output.log_metric('precision', float(precision))
    metrics_output.log_metric('recall', float(recall))
    metrics_output.log_metric('f1_score', float(f1))

    # Save model with metadata
    torch.save({
        'model_state_dict': model.state_dict(),
        'feature_columns': list(X.columns),
        'hyperparameters': {
            'learning_rate': learning_rate,
            'batch_size': batch_size,
            'epochs': epochs
        },
        'metrics': {
            'auc_roc': auc_score,
            'precision': precision,
            'recall': recall,
            'f1_score': f1
        }
    }, model_output.path)

    print(f"Model saved to {model_output.path}")


@dsl.component(
    base_image='registry.redhat.io/openshift-ai/pytorch-notebook:latest',
    packages_to_install=['mlflow==2.8.0', 'torch==2.1.0']
)
def register_model(
    model: Input[Model],
    metrics: Input[Metrics],
    min_f1_threshold: float,
    model_registry_uri: str
) -> str:
    """Register model in MLflow registry if it meets quality thresholds."""
    import mlflow
    import torch

    # Load model to check metrics
    checkpoint = torch.load(model.path)
    f1_score = checkpoint['metrics']['f1_score']

    print(f"Model F1 score: {f1_score:.4f}, Threshold: {min_f1_threshold:.4f}")

    # Check quality threshold
    if f1_score < min_f1_threshold:
        raise ValueError(
            f'Model F1 score {f1_score:.4f} below threshold {min_f1_threshold:.4f}. '
            'Model registration rejected.'
        )

    # Register with MLflow
    mlflow.set_tracking_uri(model_registry_uri)

    with mlflow.start_run(run_name="fraud-detection-training") as run:
        # Log hyperparameters
        mlflow.log_params(checkpoint['hyperparameters'])

        # Log metrics
        mlflow.log_metrics(checkpoint['metrics'])

        # Log model
        model_uri = mlflow.pytorch.log_model(
            pytorch_model=model.path,
            artifact_path="model",
            registered_model_name="fraud-detection-production"
        )

    print(f"Model registered in MLflow: {model_uri}")
    return model_uri


@dsl.pipeline(
    name='Fraud Detection Training Pipeline',
    description='End-to-end pipeline for training fraud detection model with hyperparameter tuning'
)
def fraud_detection_pipeline(
    data_path: str = '/data/transactions/2024',
    learning_rate: float = 0.001,
    batch_size: int = 256,
    epochs: int = 50,
    min_f1_threshold: float = 0.85,
    model_registry_uri: str = 'http://mlflow.mlops.svc.cluster.local:5000'
):
    """
    Training pipeline with configurable hyperparameters.

    Args:
        data_path: Path to training data (on-prem storage)
        learning_rate: Adam optimizer learning rate
        batch_size: Training batch size
        epochs: Number of training epochs
        min_f1_threshold: Minimum F1 score for model registration
        model_registry_uri: MLflow tracking server URI
    """

    # Step 1: Data preprocessing
    preprocess_task = load_and_preprocess_data(data_path=data_path)

    # Step 2: Model training (runs on GPU node)
    train_task = train_model(
        processed_data=preprocess_task.outputs['processed_data'],
        learning_rate=learning_rate,
        batch_size=batch_size,
        epochs=epochs
    )
    # Request GPU for training
    train_task.set_gpu_limit(1)
    train_task.set_memory_limit('32Gi')
    train_task.set_cpu_limit(8)

    # Step 3: Model registration (conditional on quality threshold)
    register_task = register_model(
        model=train_task.outputs['model_output'],
        metrics=train_task.outputs['metrics_output'],
        min_f1_threshold=min_f1_threshold,
        model_registry_uri=model_registry_uri
    )


# Compile pipeline to YAML
if __name__ == '__main__':
    from kfp import compiler

    compiler.Compiler().compile(
        pipeline_func=fraud_detection_pipeline,
        package_path='fraud_detection_pipeline.yaml'
    )

    print("Pipeline compiled to: fraud_detection_pipeline.yaml")
```

## Triggering Pipeline Execution

```bash
# Deploy compiled pipeline to OpenShift AI
kubectl apply -f fraud_detection_pipeline.yaml -n ml-pipelines

# Trigger pipeline run with custom hyperparameters
kfp run create \
  --pipeline-name "Fraud Detection Training Pipeline" \
  --experiment-name fraud-detection-experiments \
  --run-name fraud-model-v2 \
  --param learning_rate=0.0005 \
  --param epochs=100 \
  --param min_f1_threshold=0.90

# Monitor pipeline execution
kfp run list --experiment-name fraud-detection-experiments

# Get run details
kfp run get <RUN_ID>
```

## Key Elements

**Component-Based Architecture:**
- Each step is a reusable `@dsl.component` with explicit inputs/outputs
- Components versioned with base image and package dependencies
- Data passed between components via typed artifacts (Dataset, Model, Metrics)

**GPU Scheduling:**
- `train_task.set_gpu_limit(1)` requests GPU for training component
- Kubernetes scheduler places pod on GPU-enabled node
- Other components (preprocessing, registration) run on CPU nodes

**Quality Gates:**
- Model registration conditional on F1 score threshold
- Pipeline fails if model quality insufficient
- Prevents deploying low-quality models to production

**Experiment Tracking:**
- MLflow integration for centralized experiment management
- Hyperparameters, metrics, and models logged automatically
- Compare runs to identify best configurations

**Reproducibility:**
- Pipeline definition in code (version control)
- Exact package versions specified in components
- Data versioning via path parameter

## Production Considerations

1. **Hyperparameter Tuning:**
   ```python
   # Parallel runs with different hyperparameters
   from kfp import client

   kfp_client = client.Client()

   # Grid search over learning rates
   for lr in [0.001, 0.0005, 0.0001]:
       kfp_client.create_run_from_pipeline_func(
           fraud_detection_pipeline,
           arguments={
               'learning_rate': lr,
               'epochs': 100
           },
           run_name=f'fraud-model-lr-{lr}'
       )
   ```

2. **Scheduled Retraining:**
   ```yaml
   # CronJob to trigger monthly retraining
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: fraud-model-retraining
   spec:
     schedule: "0 2 1 * *"  # 2 AM on 1st of each month
     jobTemplate:
       spec:
         template:
           spec:
             containers:
               - name: trigger-pipeline
                 image: registry.redhat.io/kfp-client:latest
                 command:
                   - kfp
                   - run
                   - create
                   - --pipeline-name
                   - "Fraud Detection Training Pipeline"
   ```

3. **Data Drift Detection:**
   ```python
   # Add data validation component
   @dsl.component
   def validate_data_distribution(
       data_path: str,
       reference_stats_path: str
   ) -> bool:
       """Check if data distribution has drifted from reference."""
       from scipy.stats import ks_2samp
       import pandas as pd

       current = pd.read_parquet(data_path)
       reference = pd.read_parquet(reference_stats_path)

       # Kolmogorov-Smirnov test
       statistic, p_value = ks_2samp(
           current['transaction_amount'],
           reference['transaction_amount']
       )

       if p_value < 0.05:
           print(f"WARNING: Data drift detected (p={p_value:.4f})")
           return False

       return True
   ```

4. **Pipeline Caching:**
   ```python
   # Enable caching for expensive preprocessing
   preprocess_task = load_and_preprocess_data(data_path=data_path)
   preprocess_task.execution_options.caching_strategy.max_cache_staleness = "P7D"
   # Reuse cached results if data unchanged for 7 days
   ```

## When to Use This Pattern

- **Production ML Workflows**: Automated, reproducible training pipelines
- **Experiment Management**: Tracking hyperparameter tuning and model variants
- **Scheduled Retraining**: Monthly/quarterly model updates with new data
- **Team Collaboration**: Shared pipeline definitions in Git
- **Compliance**: Audit trail of model training (DADM requirements)

## Performance Optimization

1. **Parallel Data Processing:**
   - Use Dask or Spark for large dataset preprocessing
   - Partition data across multiple preprocessing components

2. **Distributed Training:**
   - Increase `set_gpu_limit()` for multi-GPU training
   - Use PyTorch DistributedDataParallel for model parallelism

3. **Pipeline Efficiency:**
   - Enable caching for expensive, deterministic steps
   - Use smaller validation sets for faster iteration

## Related Examples

- For development iteration, see [Jupyter Notebook Environment](jupyter-notebook-deployment.md)
- For production deployment, see [KServe Model Deployment](kserve-model-deployment.md)
- For inference serving, see [AI Inference Server Deployment](ai-inference-server-deployment.md)
