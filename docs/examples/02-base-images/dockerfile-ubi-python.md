# Dockerfile: Python Application with UBI Minimal

## Overview

This example demonstrates best practices for building a Python application container using Red Hat Universal Base Image (UBI) Minimal as the foundation.

## Dockerfile

```dockerfile
# Example Dockerfile using UBI
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

# Application setup
RUN microdnf install -y python39 && \
    microdnf clean all

COPY requirements.txt /app/
RUN pip3 install -r /app/requirements.txt

COPY src/ /app/
WORKDIR /app

CMD ["python3", "main.py"]
```

## Key Elements

**Base Image Selection:**
- Uses `ubi9/ubi-minimal:latest` from `registry.access.redhat.com` (no authentication required)
- UBI Minimal includes `microdnf` package manager for installing runtime dependencies
- ~90MB base size, suitable for microservices

**Build Optimization:**
- `microdnf clean all` reduces final image size by removing package manager cache
- Layer ordering optimizes Docker build caching: system packages → dependencies → application code
- Changes to source code don't invalidate the dependency installation layer

**Production Considerations:**

For production deployments, consider these enhancements:

1. **Pin specific versions** instead of `:latest`:
   ```dockerfile
   FROM registry.access.redhat.com/ubi9/ubi-minimal:9.3-1361
   ```

2. **Use non-root user** for security:
   ```dockerfile
   USER 1001
   ```

3. **Multi-stage build** to exclude build tools from final image:
   ```dockerfile
   FROM registry.access.redhat.com/ubi9/ubi-minimal:9.3 AS builder
   # ... build steps ...

   FROM registry.access.redhat.com/ubi9/ubi-minimal:9.3
   COPY --from=builder /app /app
   USER 1001
   CMD ["python3", "main.py"]
   ```

4. **Consider Hummingbird** for stateless applications requiring minimal attack surface

## When to Use This Pattern

- **New Python microservices** requiring standard Linux utilities
- **Applications needing runtime package installation** for dependencies not available via pip
- **Migration from Debian/Alpine** where you need similar package manager capabilities
- **Development and testing** where flexibility is more important than minimal size

## Related Examples

- For ultra-minimal Python images, see the Hummingbird multi-stage build example
- For applications requiring full system utilities, see UBI Standard examples
