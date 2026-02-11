# Devfile: Python Web Application Development Environment

## Overview

This example demonstrates a comprehensive Devfile specification for a Python Flask web application with PostgreSQL database, Redis cache, and integrated development tools. Devfiles enable reproducible, containerized development environments that work identically across local machines, cloud workspaces, and CI/CD pipelines.

## Complete Devfile

```yaml
# devfile.yaml
schemaVersion: 2.2.0
metadata:
  name: python-flask-webapp
  version: 1.0.0
  description: Flask web application with PostgreSQL and Redis
  displayName: Python Flask Web App
  tags:
    - python
    - flask
    - postgresql
    - redis
  projectType: python
  language: python

# Define variables for reuse
variables:
  PYTHON_VERSION: "3.11"
  POSTGRES_VERSION: "15"
  REDIS_VERSION: "7"

# Container components
components:
  # Main Python development container
  - name: python-dev
    container:
      image: registry.access.redhat.com/ubi9/python-311:latest
      memoryLimit: 2Gi
      memoryRequest: 512Mi
      cpuLimit: 2000m
      cpuRequest: 500m
      mountSources: true
      sourceMapping: /projects

      # Environment variables
      env:
        - name: FLASK_APP
          value: app.py
        - name: FLASK_ENV
          value: development
        - name: DATABASE_URL
          value: postgresql://devuser:devpass@localhost:5432/webapp
        - name: REDIS_URL
          value: redis://localhost:6379/0
        - name: PYTHONPATH
          value: /projects

      # Volume mounts for persistence
      volumeMounts:
        - name: venv
          path: /home/user/.venv
        - name: pip-cache
          path: /home/user/.cache/pip

      # Endpoints exposed
      endpoints:
        - name: web
          targetPort: 5000
          exposure: public
          protocol: http
        - name: debug
          targetPort: 5678
          exposure: internal  # Debugger port

  # PostgreSQL database
  - name: postgresql
    container:
      image: registry.redhat.io/rhel9/postgresql-15:latest
      memoryLimit: 512Mi
      memoryRequest: 256Mi
      env:
        - name: POSTGRESQL_USER
          value: devuser
        - name: POSTGRESQL_PASSWORD
          value: devpass
        - name: POSTGRESQL_DATABASE
          value: webapp
        - name: POSTGRESQL_ADMIN_PASSWORD
          value: adminpass
      volumeMounts:
        - name: postgres-data
          path: /var/lib/pgsql/data
      endpoints:
        - name: postgresql
          targetPort: 5432
          exposure: internal

  # Redis cache
  - name: redis
    container:
      image: registry.redhat.io/rhel9/redis-7:latest
      memoryLimit: 256Mi
      memoryRequest: 128Mi
      env:
        - name: REDIS_PASSWORD
          value: ""  # No password for dev
      volumeMounts:
        - name: redis-data
          path: /data
      endpoints:
        - name: redis
          targetPort: 6379
          exposure: internal

  # Security scanning sidecar
  - name: security-scanner
    container:
      image: aquasec/trivy:latest
      memoryLimit: 512Mi
      mountSources: true
      command: ['/bin/sh']
      args: ['-c', 'tail -f /dev/null']  # Keep alive

  # Persistent volumes
  - name: venv
    volume:
      size: 2Gi
  - name: pip-cache
    volume:
      size: 1Gi
  - name: postgres-data
    volume:
      size: 5Gi
  - name: redis-data
    volume:
      size: 1Gi

# Development commands
commands:
  # Setup commands
  - id: install-dependencies
    exec:
      component: python-dev
      commandLine: |
        python -m venv /home/user/.venv
        source /home/user/.venv/bin/activate
        pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: build
        isDefault: true

  - id: init-database
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flask db upgrade
        flask seed  # Load sample data
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: build

  # Run commands
  - id: run-app
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flask run --host=0.0.0.0 --port=5000
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: run
        isDefault: true

  - id: run-with-reload
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flask run --host=0.0.0.0 --port=5000 --reload
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: run

  # Debug command
  - id: debug-app
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        python -m debugpy --listen 0.0.0.0:5678 --wait-for-client -m flask run --host=0.0.0.0 --port=5000 --no-debugger --no-reload
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: debug
        isDefault: true

  # Test commands
  - id: run-tests
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        pytest tests/ -v --cov=app --cov-report=html --cov-report=term
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: test
        isDefault: true

  - id: run-tests-watch
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        ptw tests/ -- -v
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: test

  - id: integration-tests
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        pytest tests/integration/ -v --maxfail=1
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: test

  # Code quality commands
  - id: lint
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flake8 app/ tests/
        pylint app/ tests/
        black --check app/ tests/
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: test

  - id: format
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        black app/ tests/
        isort app/ tests/
      workingDir: ${PROJECT_SOURCE}

  - id: type-check
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        mypy app/
      workingDir: ${PROJECT_SOURCE}
      group:
        kind: test

  # Security commands
  - id: security-scan-dependencies
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        safety check --json
        pip-audit
      workingDir: ${PROJECT_SOURCE}

  - id: security-scan-code
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        bandit -r app/ -f json
      workingDir: ${PROJECT_SOURCE}

  - id: scan-secrets
    exec:
      component: security-scanner
      commandLine: |
        trivy fs --scanners secret /projects
      workingDir: /projects

  - id: scan-image
    exec:
      component: security-scanner
      commandLine: |
        trivy image --severity HIGH,CRITICAL registry.access.redhat.com/ubi9/python-311:latest
      workingDir: /projects

  # Database commands
  - id: db-shell
    exec:
      component: postgresql
      commandLine: psql -U devuser -d webapp
      workingDir: /

  - id: db-migrate
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flask db migrate -m "${MIGRATION_MESSAGE:-Auto migration}"
      workingDir: ${PROJECT_SOURCE}

  - id: db-reset
    exec:
      component: python-dev
      commandLine: |
        source /home/user/.venv/bin/activate
        flask db downgrade base
        flask db upgrade
        flask seed
      workingDir: ${PROJECT_SOURCE}

  # Redis commands
  - id: redis-cli
    exec:
      component: redis
      commandLine: redis-cli
      workingDir: /

  - id: redis-flush
    exec:
      component: redis
      commandLine: redis-cli FLUSHALL
      workingDir: /

# Lifecycle events
events:
  preStart:
    - install-dependencies
  postStart:
    - init-database
```

## Project Structure

```
python-flask-webapp/
├── devfile.yaml                 # This file
├── requirements.txt             # Production dependencies
├── requirements-dev.txt         # Development dependencies
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── routes.py
│   ├── services.py
│   └── utils.py
├── tests/
│   ├── unit/
│   │   ├── test_models.py
│   │   └── test_services.py
│   └── integration/
│       └── test_api.py
├── migrations/                  # Database migrations
├── .flake8                     # Linter config
├── .pylintrc                   # Pylint config
├── pyproject.toml              # Black/isort config
└── README.md
```

## Using the Devfile

### With OpenShift Dev Spaces

```bash
# Create workspace from Git repository
# Navigate to Dev Spaces dashboard
# Click "Create Workspace"
# Enter Git URL: https://github.com/example/python-flask-webapp
# Dev Spaces automatically detects and uses devfile.yaml

# Or use CLI
oc login https://openshift.example.com
crwctl workspace:start --devfile-path=https://github.com/example/python-flask-webapp/raw/main/devfile.yaml
```

### With Local devcontainers

```bash
# VS Code DevContainers extension
# Open folder in VS Code
# Command Palette -> "Dev Containers: Reopen in Container"
# VS Code converts devfile to devcontainer.json automatically
```

### With odo CLI

```bash
# Install odo
curl -L https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/odo/latest/odo-linux-amd64 -o /usr/local/bin/odo
chmod +x /usr/local/bin/odo

# Create component from devfile
odo create python-flask-webapp --devfile devfile.yaml

# Push to cluster (creates dev environment)
odo push

# Watch for changes and auto-sync
odo watch

# Execute commands
odo run install-dependencies
odo run run-app

# Port forward to access app
odo url list
odo describe

# Delete when done
odo delete
```

## Running Commands

### From IDE (VS Code/IntelliJ with plugins)

```
View -> Command Palette -> "Tasks: Run Task"
Select from available commands:
- install-dependencies
- run-app
- run-tests
- lint
- security-scan-dependencies
```

### From Terminal

```bash
# Inside workspace container
source /home/user/.venv/bin/activate

# Run tests
pytest tests/ -v

# Start development server
flask run --host=0.0.0.0

# Run linter
flake8 app/ tests/

# Security scan
safety check
```

## Key Elements

**Reproducible Environments:**
- Identical development environment for all team members
- Environment defined in code (devfile.yaml), versioned with application
- No "works on my machine" problems
- New developers productive immediately

**Multi-Container Development:**
- Application container (Python) + Database (PostgreSQL) + Cache (Redis)
- Containers networked together automatically
- Mimics production architecture in development
- Enables testing with real services, not mocks

**Integrated Tooling:**
- Security scanning (Trivy, Safety, Bandit) built into workflow
- Code quality tools (Black, Flake8, Pylint) readily available
- Database migrations and CLI access
- Debugging support with remote debugger

**Resource Management:**
- CPU and memory limits prevent resource exhaustion
- Persistent volumes for dependencies and data
- Efficient caching (pip cache, venv) speeds up workspace startup

**Lifecycle Automation:**
- `preStart`: Install dependencies before workspace ready
- `postStart`: Initialize database after startup
- Developers immediately have working environment

## Production Considerations

**Enterprise Devfile Registry:**

```yaml
# Deploy devfile registry in cluster
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devfile-registry
  namespace: devtools
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devfile-registry
  template:
    metadata:
      labels:
        app: devfile-registry
    spec:
      containers:
      - name: registry
        image: quay.io/devfile/devfile-index:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: devfile-registry
  namespace: devtools
spec:
  selector:
    app: devfile-registry
  ports:
  - port: 80
    targetPort: 8080
```

**Custom Devfile Stack:**

```yaml
# devfile-stack.yaml
# Template for all Python projects in organization
schemaVersion: 2.2.0
metadata:
  name: company-python-stack
  displayName: Company Python Standard Stack
  version: 2.0.0
  tags:
    - python
    - company-standard
components:
  - name: python
    container:
      image: internal-registry.company.com/python:3.11-company
      # Includes company CA certs, internal package indexes, security tools
      memoryLimit: 2Gi
      env:
        - name: PIP_INDEX_URL
          value: https://pypi.company.com/simple
        - name: SSL_CERT_FILE
          value: /etc/pki/tls/certs/ca-bundle.crt
  # Include mandatory security scanning
  - name: security
    container:
      image: internal-registry.company.com/security-scanner:latest
commands:
  # Mandatory security check before commit
  - id: pre-commit-security
    exec:
      component: security
      commandLine: scan-code --fail-on-high
```

**Policy Enforcement:**

```yaml
# OPA/Gatekeeper policy: All devfiles must use approved images
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: devfile-allowed-repos
spec:
  match:
    kinds:
      - apiGroups: ["workspace.devfile.io"]
        kinds: ["DevWorkspace"]
  parameters:
    repos:
      - "registry.access.redhat.com/"
      - "registry.redhat.io/"
      - "internal-registry.company.com/"
```

**Resource Quotas:**

```yaml
# Limit resources per developer
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-workspace-quota
  namespace: user-workspaces
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
    requests.storage: "50Gi"
```

## When to Use This Pattern

- **Onboarding new developers** - eliminate environment setup friction
- **Ensuring environment parity** across team members
- **Standardizing development stacks** across organization
- **Enabling cloud-based development** with Dev Spaces
- **Reproducible builds** - same environment in dev, CI, and production

## Benefits for Digital Sovereignty

**Infrastructure Independence:**
- Devfiles work with any Kubernetes (AWS EKS, Azure AKS, on-prem OpenShift)
- Not locked into cloud provider development services (AWS Cloud9, GitHub Codespaces)
- Can migrate development infrastructure without disrupting developers

**Supply Chain Security:**
- Control base images used in development
- Integrate security scanning into every workspace
- Enforce approved package sources and registries
- Prevent dependency confusion and supply chain attacks

**Compliance:**
- Ensure all developers use compliant tooling
- Audit development environment configurations
- Enforce security policies in development, not just production

## Related Examples

- See `../02-base-images/dockerfile-ubi-python.md` for building production images
- See `../05-cicd/tekton-secure-build-pipeline.md` for CI/CD using same tools
- See Section 06 for Dev Spaces architecture and deployment
