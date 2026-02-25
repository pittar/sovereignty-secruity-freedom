# RHEL AI: Running LLMs with InstructLab

## Overview

This example demonstrates how to run large language models directly on Red Hat Enterprise Linux using RHEL AI and InstructLab for model customization and serving.

## Installation and Setup

```bash
# Install RHEL AI packages
sudo dnf install rhel-ai instructlab

# Initialize InstructLab for model customization
ilab init --model granite-7b-lab

# Serve the model for inference
ilab serve --model granite-7b-lab --port 8000

# Query the model via API
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-7b-lab",
    "prompt": "Analyze the following transaction for fraud indicators:",
    "max_tokens": 256
  }'
```

## Key Elements

**RHEL AI Package Installation:**
- `rhel-ai` provides optimized AI runtime for RHEL systems
- `instructlab` enables model customization and alignment tuning
- Requires RHEL 9.x with appropriate GPU drivers installed

**InstructLab Model Initialization:**
- Downloads and configures the specified model (Granite 7B in this example)
- Supports various open source models including Llama, Mistral, and IBM Granite
- Model weights stored locally in `~/.local/share/instructlab/models/`

**Model Serving:**
- Exposes OpenAI-compatible API on specified port
- Supports both completion and chat endpoints
- Automatically optimizes for available hardware (CPU or GPU)

**API Interface:**
- OpenAI-compatible `/v1/completions` endpoint
- Accepts standard parameters: `model`, `prompt`, `max_tokens`, `temperature`
- Response includes generated text and token usage statistics

## Production Considerations

1. **GPU Acceleration:**
   ```bash
   # Verify GPU availability
   nvidia-smi

   # Serve with explicit GPU configuration
   ilab serve --model granite-7b-lab --gpu-layers 32
   ```

2. **Model Customization:**
   ```bash
   # Fine-tune model with organization-specific data
   ilab train --model granite-7b-lab --data-path ./custom-data

   # Serve customized model
   ilab serve --model granite-7b-lab-custom
   ```

3. **Production Deployment:**
   - Run as systemd service for automatic restart
   - Configure firewall rules for API port
   - Set up reverse proxy (nginx) for TLS termination
   - Implement rate limiting and authentication

4. **Air-Gap Deployment:**
   - Pre-download model weights on connected system
   - Transfer model directory via approved media
   - Install RHEL AI packages from local repository mirror

## When to Use This Pattern

- **Edge AI Deployment**: Running AI inference on RHEL systems at remote locations without Kubernetes
- **Single-Server Applications**: Simpler deployment when orchestration overhead isn't justified
- **Development Workstations**: Data scientists prototyping on local RHEL systems
- **Legacy Integration**: Adding AI capabilities to existing RHEL-based applications
- **Air-Gap Environments**: Classified or disconnected systems requiring AI capabilities

## Related Examples

- For Kubernetes-based deployment, see [AI Inference Server on OpenShift](ai-inference-server-deployment.md)
- For complete MLOps lifecycle, see [Jupyter Development Environment](jupyter-notebook-deployment.md) and [Kubeflow Pipeline](kubeflow-pipeline-definition.md)
