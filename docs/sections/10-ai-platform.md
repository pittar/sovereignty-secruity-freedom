# The AI/ML Platform: Enterprise AI with Digital Sovereignty

[← Workload Identity](09-workload-identity.md) | [Table of Contents](../OUTLINE.md) | [Next: Conclusion →](11-conclusion.md)

---

## Executive Summary

Artificial intelligence and machine learning transform business capabilities—from fraud detection and personalized recommendations to predictive maintenance and intelligent automation. However, traditional AI deployment creates severe digital sovereignty risks: training data uploaded to cloud AI services exposes proprietary information and customer PII, model inference through external APIs leaks sensitive queries and intellectual property, and vendor-specific AI platforms create lock-in while preventing workload portability. Organizations deploying AI face a fundamental tension—innovation demands powerful AI capabilities, while sovereignty requires retaining complete control over data, models, and infrastructure.

Regulatory frameworks increasingly mandate AI sovereignty. The European Union's AI Act classifies AI systems by risk level and imposes strict requirements for high-risk applications. Canada's Directive on Automated Decision-Making (DADM), the world's first AI-specific legally binding instrument, requires government departments to ensure transparency, accountability, and data sovereignty for automated decision systems. Data residency requirements (GDPR, Canadian privacy law, financial services regulations) prohibit sending training data and inference requests to external jurisdictions. Beyond compliance, competitive dynamics demand protecting proprietary AI models—organizations invest millions training custom models that represent strategic intellectual property.

Red Hat AI delivers enterprise-grade AI/ML infrastructure built entirely on open source foundations, enabling the complete AI lifecycle from development through production deployment across any environment. The platform spans three integrated offerings: **Red Hat Enterprise Linux AI (RHEL AI)** for running AI models directly on RHEL systems with optimized performance, **Red Hat AI Inference Server** based on vLLM providing portable inference across any cloud, any accelerator, and any model format, and **Red Hat OpenShift AI** offering comprehensive MLOps platform capabilities for building, training, serving, and monitoring both predictive and generative AI models. This architecture enables true digital sovereignty—organizations run AI workloads on-premises, in air-gapped environments, or across multiple clouds, moving workloads freely without vendor lock-in while maximizing hardware investments through unified tooling.

---

## The AI/ML Infrastructure Challenge

### Why AI Requires Specialized Platform Infrastructure

Traditional application development workflows don't translate directly to AI/ML workloads. Software development produces deterministic code—given identical inputs, the same code produces identical outputs. AI model development introduces fundamental differences: models are statistical artifacts learned from data rather than explicitly programmed logic, model quality depends critically on training data characteristics and volume, and model performance degrades over time as real-world data distributions shift (requiring continuous retraining). Infrastructure requirements differ dramatically—training deep learning models demands GPU acceleration (100-1000x faster than CPU-only training), model serving requires specialized inference servers optimized for batching and caching, and experiment tracking needs versioning not just for code but for data, hyperparameters, and model artifacts.

The AI/ML lifecycle encompasses distinct phases each with unique infrastructure needs:

**1. Development and Experimentation**
- Interactive environments (Jupyter notebooks) for data exploration
- GPU-accelerated compute for rapid iteration
- Access to training datasets and feature stores
- Version control for notebooks, code, and experiments

**2. Data Preparation and Engineering**
- Scalable data processing for large datasets (ETL/ELT pipelines)
- Feature engineering and transformation
- Data validation and quality checking
- Privacy-preserving data handling (anonymization, encryption)

**3. Model Training**
- Distributed training across multiple GPUs and nodes
- Hyperparameter tuning and AutoML
- Experiment tracking (metrics, parameters, artifacts)
- Checkpoint management for long-running training jobs

**4. Model Evaluation and Validation**
- Model testing against holdout datasets
- Bias and fairness evaluation
- Performance benchmarking (accuracy, latency, throughput)
- Model explainability and interpretability

**5. Model Deployment and Serving**
- Containerized model serving with auto-scaling
- A/B testing and canary deployments
- Multi-model serving and routing
- GPU sharing and optimization for inference

**6. Monitoring and Operations**
- Model performance tracking (accuracy, drift)
- Infrastructure metrics (latency, throughput, resource utilization)
- Retraining triggers and automation
- Audit logging and compliance reporting

Traditional cloud AI services (AWS SageMaker, Azure ML, Google Vertex AI) provide these capabilities but at the cost of sovereignty—training data uploaded to cloud providers, model inference through proprietary APIs, vendor-specific tooling creating lock-in, and no option for on-premises or air-gapped deployment.

### The Digital Sovereignty Imperative for AI

AI workloads amplify sovereignty concerns beyond traditional applications:

**Data Exposure Risks:**
- **Training Data**: Models learn from sensitive data (customer transactions, medical records, proprietary business information). Uploading training datasets to cloud AI services exposes this data to external infrastructure, third-party access, and potential jurisdictional legal claims.
- **Inference Queries**: Every prediction request reveals information—fraud detection queries expose transaction patterns, medical diagnosis requests contain patient data, recommendation system queries reveal customer behavior. External AI APIs log these requests, creating comprehensive data exposure.
- **Model Weights**: Trained models represent millions or billions of dollars in R&D investment. Model weights encode patterns from proprietary data and represent competitive intellectual property. Cloud-hosted models risk extraction, copying, or competitive intelligence gathering.

**Regulatory Compliance Requirements:**
- **GDPR (Europe)**: Requires data minimization, purpose limitation, and data subject rights. Processing PII through external AI services violates these principles without complex legal frameworks.
- **Canadian Privacy Law**: The Personal Information Protection and Electronic Documents Act (PIPEDA) and provincial privacy laws restrict cross-border data transfers. Government departments using AI must comply with the Directive on Automated Decision-Making requiring transparency and data sovereignty.
- **Financial Services**: Banking regulations (PCI-DSS, regional banking laws) prohibit sending transaction data to external processors without specific agreements. AI fraud detection using cloud services creates regulatory violations.
- **Healthcare**: HIPAA (US), PHIPA (Canada), and equivalent health privacy laws mandate strict controls on patient data. Medical AI models trained or served through cloud platforms risk compliance failures.

**Vendor Lock-in and Strategic Risk:**
- Proprietary AI platforms use vendor-specific APIs, formats, and tooling
- Models trained with platform-specific features can't easily migrate
- Pricing changes, service discontinuations, or geopolitical events create business continuity risk
- Lack of portability prevents multi-cloud strategies and negotiating leverage

### Government AI Policy: The Canadian Example

Canada leads globally in AI governance, providing a model for government AI sovereignty requirements. The **Directive on Automated Decision-Making (DADM)**, launched in 2019, established the world's first AI-specific legally binding instrument. The directive applies to all Canadian federal government departments and agencies using automated systems to make or assist in administrative decisions affecting external clients.

**DADM Key Requirements:**

1. **Algorithmic Impact Assessment (AIA)**: Departments must conduct risk assessments before deploying automated decision systems, evaluating impact on rights, health, well-being, economic interests, and environmental sustainability.

2. **Transparency and Explainability**: Meaningful explanation of automated decisions must be available to affected individuals. For high-impact systems, detailed documentation of training data, model logic, and decision factors is required.

3. **Human-in-the-Loop**: Recourse mechanisms allowing humans to review automated decisions and request human intervention for high-impact systems.

4. **Data Quality and Bias Testing**: Regular testing for bias and fairness issues, with documentation of mitigation strategies.

5. **Data Sovereignty**: While not explicitly stated, the directive's requirements for transparency, auditability, and recourse implicitly demand infrastructure control. External cloud AI services make compliance difficult—how can departments explain decisions made by opaque external APIs? How can they audit training data when it's processed in foreign jurisdictions?

The **AI Strategy for the Federal Public Service 2025-2027**, published by the Office of the Chief Information Officer, reinforces these themes. The strategy prioritizes:
- Responsible AI development and deployment
- Data governance and sovereignty
- Open source and interoperable AI systems
- Skills development for AI literacy

Government procurement considerations increasingly favor solutions meeting data sovereignty requirements. Departments evaluating Software-as-a-Service (SaaS) AI platforms must verify compliance with security, privacy, and **data sovereignty requirements**. Geopolitical concerns (particularly around U.S. access to Canadian data through mechanisms like CLOUD Act) intensify the need for on-premises or Canadian-hosted AI infrastructure.

**Digital Sovereignty Requirements for Government AI:**
- Training data must remain within Canadian jurisdiction
- Model inference cannot leak sensitive government or citizen data to foreign services
- AI systems must be auditable and explainable (incompatible with black-box cloud APIs)
- Continuity of operations requires infrastructure independence from foreign vendors
- Open source foundations enable transparency and avoid vendor lock-in

These requirements apply beyond government. Financial services, healthcare, critical infrastructure, and any organization handling sensitive data faces similar sovereignty imperatives for AI deployment.

---

## Red Hat AI: The Sovereign AI/ML Platform

Red Hat AI delivers enterprise-grade AI/ML infrastructure built on open source foundations, enabling digital sovereignty through portable, vendor-independent tooling that runs anywhere—on-premises, in air-gapped environments, or across any cloud provider. The platform integrates three core offerings covering the full AI lifecycle:

### Red Hat Enterprise Linux AI (RHEL AI)

**RHEL AI** enables running AI models directly on Red Hat Enterprise Linux systems with optimized performance and enterprise support. Built on InstructLab, an open source project for large language model development and customization, RHEL AI provides tools for model serving, fine-tuning, and inference on RHEL infrastructure.

**Key Capabilities:**
- **Model Serving on RHEL**: Deploy AI models (especially large language models) directly on RHEL servers without requiring Kubernetes or container orchestration
- **InstructLab Integration**: Use InstructLab's alignment tuning and skill development to customize models with enterprise-specific knowledge
- **Hardware Optimization**: Support for GPU acceleration (NVIDIA, AMD) with optimized RHEL drivers and libraries
- **Enterprise Support**: Red Hat subscription support for AI workloads on RHEL with security updates and lifecycle management
- **Air-Gap Capable**: Fully functional in disconnected environments without internet access

**Use Cases:**
- Edge AI deployment on RHEL systems at remote locations
- Single-server AI applications without orchestration complexity
- Rapid prototyping and development on developer workstations
- Legacy system integration where containerization isn't feasible

**→ See the complete example:** [RHEL AI: Running LLMs with InstructLab](../examples/10-ai-platform/rhel-ai-instructlab.md) for installation, model customization, GPU acceleration, and production deployment considerations including air-gap scenarios.

### Red Hat AI Inference Server

The **Red Hat AI Inference Server**, based on the open source vLLM project, provides high-performance, portable model inference across any environment. vLLM's PagedAttention algorithm and continuous batching deliver industry-leading throughput for large language model serving, while OpenAI-compatible API support ensures application portability.

**"Any Cloud, Any Accelerator, Any Model" Philosophy:**

1. **Any Cloud**: Deploy identical inference infrastructure on AWS, Azure, Google Cloud, IBM Cloud, or on-premises OpenShift. Applications use the same API endpoints regardless of underlying infrastructure, enabling seamless workload migration.

2. **Any Accelerator**: Support for diverse GPU hardware including NVIDIA (A100, H100, L4, L40S), AMD Instinct, and Intel Data Center GPUs. Organizations choose hardware based on cost, availability, and performance needs without rewriting inference code.

3. **Any Model**: Serve models in multiple formats (Hugging Face, GGUF, safetensors) and frameworks (PyTorch, TensorFlow). Support for both open source models (Llama, Mistral, Granite) and proprietary custom models.

**Hardware Investment Optimization:**

Organizations with diverse GPU hardware across different data centers and cloud regions use a single inference stack. A model validated on NVIDIA L4 GPUs in AWS can deploy identically on AMD Instinct GPUs on-premises, maximizing hardware utilization without maintaining multiple inference frameworks.

**→ See the complete example:** [AI Inference Server: vLLM Deployment on OpenShift](../examples/10-ai-platform/ai-inference-server-deployment.md) for scalable LLM serving with GPU acceleration, auto-scaling configuration, multi-model serving, and performance optimization strategies.

### Red Hat OpenShift AI

**Red Hat OpenShift AI** provides a comprehensive MLOps platform for the complete AI/ML lifecycle, integrating development, training, deployment, and monitoring capabilities on the OpenShift Kubernetes platform. Built on the upstream Kubeflow project and extended with enterprise features, OpenShift AI enables data scientists and ML engineers to build and deploy models at scale while IT operations maintain security, governance, and infrastructure control.

**Platform Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Red Hat OpenShift AI                         │
│                   (Kubernetes-Native MLOps)                     │
├─────────────────────────────────────────────────────────────────┤
│  Model Development       │  Model Training   │  Model Serving   │
│  - Jupyter Notebooks     │  - Distributed    │  - KServe        │
│  - VS Code Integration   │  - GPU Scheduling │  - ModelMesh     │
│  - Python Environments   │  - Pipelines      │  - Auto-scaling  │
├──────────────────────────┴───────────────────┴──────────────────┤
│                   Model Operations & Governance                  │
│  - Model Registry  │  Model Monitoring  │  Experiment Tracking │
│  - Version Control │  Drift Detection   │  Metadata Management │
├─────────────────────────────────────────────────────────────────┤
│              Red Hat OpenShift (Kubernetes Platform)            │
│  - Multi-cloud / On-prem / Air-gapped deployment               │
│  - GPU Operator (NVIDIA, AMD, Intel)                           │
│  - Storage (OpenShift Data Foundation / Ceph)                  │
│  - Security (SPIFFE/SPIRE, RBAC, Network Policies)            │
└─────────────────────────────────────────────────────────────────┘
```

**Core Capabilities:**

1. **Interactive Development Environments**
   - Jupyter notebooks with GPU acceleration
   - Pre-configured data science environments (Python, R, Julia)
   - Integration with VS Code for IDE-based workflows
   - Shared workspaces for team collaboration

2. **Pipeline Orchestration (Kubeflow Pipelines)**
   - Define ML workflows as code (Python SDK)
   - Automated training pipelines with hyperparameter tuning
   - Reproducible experiments with versioned artifacts
   - Integration with Git for pipeline version control

3. **Distributed Training**
   - Multi-GPU training with frameworks (PyTorch, TensorFlow, Ray)
   - Distributed data processing (Spark integration)
   - Automatic GPU scheduling and allocation
   - Checkpoint management for fault tolerance

4. **Model Serving (KServe)**
   - Deploy models as scalable REST/gRPC endpoints
   - Multi-framework support (TensorFlow, PyTorch, ONNX, scikit-learn)
   - Canary deployments and A/B testing
   - GPU inference optimization

5. **Model Monitoring and Observability**
   - Real-time performance metrics (latency, throughput, accuracy)
   - Data drift detection
   - Model bias and fairness monitoring
   - Integration with Prometheus and Grafana

**Deployment Flexibility: The Sovereignty Advantage**

OpenShift AI runs identically across environments, enabling true workload portability:

- **Public Cloud**: Deploy on AWS, Azure, Google Cloud with managed OpenShift
- **On-Premises**: Run in corporate data centers on bare metal or virtualized infrastructure
- **Hybrid**: Develop in cloud, train on-prem on sensitive data, serve models in edge locations
- **Air-Gapped**: Complete MLOps in disconnected environments (government, defense, financial trading)
- **Edge**: Lightweight deployments at remote sites with central management

This flexibility addresses digital sovereignty requirements directly:
- Develop models in a public cloud sandbox using synthetic data
- Move the validated pipeline to on-premises infrastructure for training on real customer data
- Deploy the trained model back to multi-cloud for global inference serving
- Retain the ability to migrate workloads based on regulatory changes, cost optimization, or vendor negotiations

**Integration with Platform Stack:**

OpenShift AI leverages the complete Red Hat platform ecosystem described in previous sections:

- **UBI Base Images**: All OpenShift AI containers built on UBI for consistency and security
- **Supply Chain Security**: Model artifacts signed with Sigstore, SBOM generation for model dependencies
- **GitOps**: ArgoCD deployment of models and pipelines with version control
- **Confidential Containers**: Train models on sensitive data with cryptographic guarantees (see Use Case below)
- **SPIFFE/SPIRE**: mTLS authentication for model serving endpoints
- **Developer Hub**: Catalog of models, datasets, and ML pipelines with self-service access

---

## Technical Deep Dive: End-to-End AI/ML Lifecycle

This section demonstrates a complete AI workflow using Red Hat AI, from initial development through production deployment, emphasizing sovereignty through infrastructure control.

**Scenario**: A financial services firm building a fraud detection model for credit card transactions. Requirements include:
- Training data contains PII and must remain on-premises (GDPR, PCI-DSS compliance)
- Model must be explainable for regulatory audit (Canadian DADM, EU AI Act)
- Production inference must support 10,000 TPS with <100ms latency
- Capability to move workloads between on-prem and cloud based on capacity needs

### Phase 1: Development Environment Setup

Data scientists use Jupyter notebooks in OpenShift AI, with GPU access for experimentation.

**→ See the complete example:** [Jupyter Notebook: AI Development Environment on OpenShift AI](../examples/10-ai-platform/jupyter-notebook-deployment.md) for GPU-accelerated development environment, data access patterns, complete fraud detection model development workflow, and production considerations including resource quotas and idle notebook culling.

### Phase 2: Automated Training Pipeline

Once the model architecture is validated, data scientists define a Kubeflow Pipeline for reproducible training with hyperparameter tuning.

**→ See the complete example:** [Kubeflow Pipeline: Automated ML Training Workflow](../examples/10-ai-platform/kubeflow-pipeline-definition.md) for complete pipeline definition with data preprocessing, GPU-accelerated training, quality gates, MLflow integration, hyperparameter tuning, scheduled retraining, and data drift detection.

### Phase 3: Model Deployment with KServe

Deploy the trained model to production using KServe for scalable inference serving.

**→ See the complete example:** [KServe: Production Model Deployment](../examples/10-ai-platform/kserve-model-deployment.md) for complete InferenceService configuration, GitOps deployment with ArgoCD, auto-scaling setup, canary deployments, multi-model serving, monitoring with Prometheus, and performance optimization strategies.

### Phase 4: Production Inference with SPIFFE Authentication

Applications call the fraud detection model via authenticated mTLS using SPIFFE workload identities for zero-trust security.

**→ See complete client implementation in:** [KServe: Production Model Deployment](../examples/10-ai-platform/kserve-model-deployment.md#client-application-inference-with-spiffe-authentication) for SPIFFE-authenticated inference client with error handling, timeout management, and risk classification logic.

---

## Real-World Use Case: Canadian Government Department AI Deployment

**Organization**: Canadian federal department processing benefit applications (hypothetical example illustrating DADM compliance)

**Challenge**: The department receives 500,000+ benefit applications annually. Manual review by caseworkers takes 4-6 weeks per application, creating backlogs and delayed services for citizens. An automated decision system could triage applications—flagging high-risk cases for human review while fast-tracking straightforward approvals. However, this falls squarely under the Directive on Automated Decision-Making (DADM) as a high-impact automated system affecting citizens' economic interests.

**Sovereignty Requirements**:

1. **Data Residency**: Application data contains Canadian citizen PII (social insurance numbers, financial information, health details). Data cannot leave Canadian jurisdiction under PIPEDA and Treasury Board policies.

2. **Transparency and Explainability**: DADM requires meaningful explanations for automated decisions. The AI system must provide human-readable rationale for why an application was flagged or approved.

3. **Auditability**: Office of the Auditor General must be able to inspect training data, model logic, and decision records. External cloud AI services make this impractical.

4. **Bias Testing**: Regular evaluation for demographic bias required. The department needs access to model internals and training data to conduct fairness audits.

**Solution Architecture: OpenShift AI in Shared Services Canada Data Centers**

The department deploys OpenShift AI entirely within Shared Services Canada (SSC) government data centers located in Canadian territory.

**Infrastructure**:
- **Primary Site**: SSC Data Center (Ottawa region)
  - 16x NVIDIA L40S GPUs for model training
  - 32x NVIDIA L4 GPUs for inference serving
  - 200TB Ceph storage for training data and model registry
  - OpenShift 4.15 on RHEL 9.4, air-gapped configuration

**Implementation**:

1. **Development**: Data scientists develop models in Jupyter environments with GPU acceleration
2. **Training**: Kubeflow Pipelines with confidential containers protecting PII during processing
3. **Explainability**: SHAP integration provides feature importance for each decision
4. **Serving**: KServe deployment with SPIFFE authentication
5. **Bias Monitoring**: Automated fairness evaluation across protected characteristics

**DADM Compliance Achieved**:

✅ **Algorithmic Impact Assessment**: Completed using OpenShift AI model cards
✅ **Transparency**: SHAP explainability provides decision rationale
✅ **Human-in-the-Loop**: Model provides recommendations; caseworkers make final decisions
✅ **Data Sovereignty**: 100% Canadian infrastructure, zero foreign data exposure

**Benefits Realized**:
- Processing time reduced from 4-6 weeks to 1-2 weeks for fast-track cases
- 40% reduction in caseworker hours for routine applications
- Full DADM compliance with auditable trail
- Zero dependency on foreign AI services

---

## On-Premises and Air-Gapped AI Deployment

### Why On-Premises AI Matters for Sovereignty

On-premises deployment provides complete data control, meeting regulatory requirements that cloud AI services cannot satisfy. OpenShift AI deploys identically on-premises as in cloud environments, providing full MLOps capabilities within organizational data centers.

**Air-Gapped Deployment Patterns:**

**→ See the complete example:** [Air-Gapped Model Transfer: Secure Model Deployment in Disconnected Environments](../examples/10-ai-platform/airgap-model-transfer.md) for complete export/import workflows with cryptographic verification, model cards, SBOM generation, encrypted transfer procedures, and InferenceService deployment in air-gapped government environments.

---

## The Upstream Connection

### Red Hat's AI/ML Open Source Leadership

Red Hat invests significantly in open source AI/ML infrastructure projects, ensuring enterprise requirements shape community directions.

**Key Contributions:**

- **Kubeflow**: Red Hat engineers maintain Kubeflow Pipelines components, focusing on multi-tenancy, RBAC integration, and air-gapped deployment support
- **KServe**: Contributions to standardized model serving, multi-framework support, and GPU optimization
- **InstructLab**: Red Hat launched InstructLab as open source project for LLM alignment and skill development
- **Kubernetes AI/ML SIG**: Participation in standards for GPU sharing, scheduling, and resource management

**Engineering Investment**:
- ~20 Red Hat engineers dedicated to AI/ML open source projects full-time
- Focus on enterprise features: security, multi-tenancy, air-gap support
- Upstream-first development philosophy

---

## Key Benefits Summary

**For Data Scientists and ML Engineers**:
- Familiar tools (Jupyter, Python, PyTorch/TensorFlow) with enterprise support
- GPU acceleration without infrastructure management complexity
- Reproducible experiments through pipeline orchestration
- Self-service environments via Developer Hub integration

**For IT Operations and Platform Teams**:
- Unified platform for AI/ML alongside traditional applications
- Consistent security, networking, and storage across workloads
- Multi-tenancy with namespace isolation and RBAC
- Monitoring and observability with standard tools

**For Organizations**:
- Avoid vendor lock-in through open source foundations
- Maximize hardware investment across cloud, on-prem, and hybrid
- Meet regulatory requirements (GDPR, DADM, industry regulations)
- Protect intellectual property (models and training data)
- Cost optimization through flexible deployment

**For Digital Sovereignty**:
- **Data Sovereignty**: Training data and inference queries remain within organizational boundaries
- **Infrastructure Independence**: Deploy on-premises, air-gapped, or in sovereign clouds
- **Model IP Protection**: Proprietary models never exposed to external services
- **Regulatory Compliance**: Meet data residency and transparency requirements
- **Strategic Autonomy**: No dependency on foreign AI platforms
- **Transparency**: Open source foundations enable audit and verification

---

## References and Further Reading

### Canadian AI Policy
- [Directive on Automated Decision-Making](https://www.tbs-sct.canada.ca/pol/doc-eng.aspx?id=32592) - Treasury Board of Canada Secretariat
- [Responsible use of artificial intelligence in government](https://www.canada.ca/en/government/system/digital-government/digital-government-innovations/responsible-use-ai.html) - Government of Canada
- [Canada's 2026 privacy priorities: data sovereignty, open banking and AI](https://www.osler.com/en/insights/reports/2025-legal-outlook/canadas-2026-privacy-priorities-data-sovereignty-open-banking-and-ai/) - Osler Legal Analysis

### Red Hat AI Platform
- [Red Hat AI](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux/ai) - Red Hat Enterprise Linux AI
- [Red Hat OpenShift AI](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai) - Enterprise MLOps Platform
- [InstructLab](https://instructlab.ai/) - Open source LLM alignment

### Upstream AI/ML Projects
- [Kubeflow](https://www.kubeflow.org/) - ML toolkit for Kubernetes
- [KServe](https://kserve.github.io/) - Standardized model serving
- [vLLM](https://github.com/vllm-project/vllm) - High-performance LLM inference
- [MLflow](https://mlflow.org/) - ML lifecycle management

### AI Governance
- [EU AI Act](https://artificialintelligenceact.eu/) - European Union AI regulation
- [OECD AI Principles](https://oecd.ai/en/ai-principles) - International AI governance
- [Confidential Computing Consortium](https://confidentialcomputing.io/) - Hardware-based AI privacy

---

**Next:** [11. Conclusion: The Path to Digital Sovereignty →](11-conclusion.md)
