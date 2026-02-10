# Foundation: Trusted Base Images

## Executive Summary

The foundation of any containerized application is its base image. Poor choices here create security vulnerabilities, licensing complications, and portability challenges. Red Hat Universal Base Images (UBI) and Project Hummingbird provide freely redistributable, enterprise-grade base images that enable true cloud independence while maintaining the highest security standards.

**Critically, UBI and Project Hummingbird images are completely free to use, redistribute, and run on any platform—including competitor clouds and non-Red Hat infrastructure.** There is no vendor lock-in: applications built on these images can run on AWS, Azure, Google Cloud, on-premises data centers, or any OCI-compliant container platform without requiring a Red Hat subscription for runtime. This freedom of choice is fundamental to digital sovereignty, allowing organizations to build portable applications while retaining the option to purchase enterprise support when needed, rather than being forced into proprietary ecosystems.

---

## The Base Image Problem

### Traditional Challenges

Most organizations default to popular community base images for their containers. While these serve the community well, they can present significant challenges in enterprise production environments. Common issues include library compatibility problems, unnecessary packages that expand the attack surface, and security update cadences that may not align with enterprise compliance requirements. More critically, many community images lack clear redistribution rights, enterprise support options, and long-term lifecycle guarantees—creating hidden risks for organizations building mission-critical applications.

- **Security and Patching**: Inconsistent security updates, unclear lifecycle
- **Licensing Concerns**: Redistribution restrictions, compliance complexities
- **Enterprise Support**: Lack of production-grade support and SLAs
- **Certification**: Limited ISV and hardware certification

### Why Base Images Matter for Digital Sovereignty

The base image decision has profound implications for digital sovereignty. An organization's entire application portfolio sits atop these foundations—if the base images create vendor dependencies, lack transparency, or have unclear licensing terms, every application built on them inherits these constraints. Organizations lose the ability to freely move workloads between clouds, redistribute their applications to partners and customers, or maintain control over their security posture.

Red Hat addresses these sovereignty challenges through UBI and Project Hummingbird by making enterprise-grade base images freely available to everyone—including organizations that are not Red Hat customers. These images can be pulled from public registries, used in any environment, and redistributed without restriction. The supply chain is fully transparent: every package is traceable to its source in RHEL, with public build provenance and cryptographic signing. Security updates follow a predictable, enterprise-grade lifecycle, eliminating the uncertainty that comes with community images.

**The key sovereignty advantage**: Organizations can build portable applications on Red Hat's enterprise foundation without committing to Red Hat infrastructure or subscriptions. Applications containerized with UBI run identically on AWS ECS, Azure Container Instances, Google Cloud Run, or any Kubernetes distribution. If an organization later needs enterprise support, they can purchase it. If they don't, they continue running freely. This separates the foundation from the infrastructure—true digital sovereignty means having choices at every layer.

---

## Red Hat Universal Base Images (UBI)

### What Makes UBI Unique

Red Hat Universal Base Images provide what enterprise applications require most: consistency, security, and predictability. Unlike community base images that may change unpredictably or lack clear support boundaries, UBI follows the well-established lifecycle of Red Hat Enterprise Linux. Each UBI version aligns with a specific RHEL major release (UBI 8 with RHEL 8, UBI 9 with RHEL 9), inheriting its 10-year lifecycle with clearly defined phases for full support, maintenance support, and extended life cycle support.

This lifecycle alignment means organizations can plan their application maintenance schedules with confidence. Security patches and critical bug fixes flow from RHEL to UBI on a predictable cadence, with the same enterprise-grade quality assurance and testing. The consistency extends across architectures—a container built on UBI 9 for x86_64 behaves identically when running on ARM64 or other supported platforms. This predictability is what separates enterprise-ready base images from community alternatives.

**Key Characteristics:**
- **Freely Redistributable**: Use in any environment, no subscription required for runtime
- **Enterprise-Grade**: Built from Red Hat Enterprise Linux (RHEL)
- **Lifecycle Managed**: Predictable security updates and long-term support
- **Multi-Architecture**: x86_64, ARM64, ppc64le, s390x support
- **OCI Compliant**: Industry standards for maximum portability

### UBI Variants

**UBI Minimal** (~90MB) uses `microdnf` instead of full `dnf`, omitting documentation and unnecessary locales. Best for microservices where you need to install a few runtime dependencies but want to minimize image size. Includes Python, Node.js, and Go runtime variants pre-configured for common development stacks.

**UBI Standard** (~200MB) provides the complete RHEL userspace with `dnf`, standard utilities, and full documentation. Use this when applications expect traditional Linux tools or when ISV software explicitly requires UBI Standard certification. The additional size brings compatibility and troubleshooting capabilities.

**UBI Init** (~230MB) includes systemd for managing multiple processes within a single container. Reserved for legacy applications or specific architectural patterns that require process supervision. Not recommended for new cloud-native development—prefer orchestrator-level process management instead.

**UBI Micro** (~25MB) contains no package manager, no shell, and minimal libraries—similar in philosophy to Hummingbird but following the UBI 9 lifecycle. Use for extremely constrained environments or where security policy mandates absolute minimalism with RHEL 9 compatibility guarantees.

### Technical Deep Dive: UBI Architecture

UBI images are built directly from the same RPM packages that comprise Red Hat Enterprise Linux, following an identical build and quality assurance process. Every package in a UBI image is traceable to a specific RHEL release, signed with Red Hat's GPG keys, and maintained through the RHEL security response process. The build system selects packages based on dependency analysis and use case requirements—UBI Minimal includes only packages required for basic containerized applications plus `microdnf` for runtime package management, while UBI Standard mirrors a minimal RHEL server installation.

Security hardening happens at multiple layers: packages are compiled with modern security flags (PIE, RELRO, stack protectors), default permissions follow least-privilege principles, and unnecessary setuid binaries are removed. Each image includes CA certificates for TLS validation and timezone data, but excludes kernel modules, firmware, and hardware-specific packages irrelevant to containers. The resulting images are cryptographically signed and published to `registry.access.redhat.com` without authentication requirements, enabling unrestricted pulls while maintaining supply chain integrity through signature verification.

### Lifecycle and Security Updates

UBI follows the RHEL lifecycle: 10 years of support divided into Full Support (5 years), Maintenance Support (5 years), and optional Extended Life Cycle Support. Security updates are released as soon as fixes are validated—critical CVEs typically receive patches within days, with updates published simultaneously to RHEL and UBI. Organizations can track security advisories through the Red Hat CVE Database and Red Hat Security Data API.

The update process is predictable: Red Hat publishes new UBI image tags whenever security updates affect included packages. Images use both version-specific tags (`:9.3`, `:9.3-1234567890`) and rolling tags (`:latest`, `:9`) to support both pinned and automatic update workflows. For production deployments, pin to specific version tags and update deliberately through your CI/CD pipeline. For development, `:latest` provides automatic security updates without manual intervention.

---

## Project Hummingbird: Next-Generation Minimal Images

### The Evolution of Minimal

While UBI provides enterprise-grade foundations for containerized applications, modern cloud-native architectures increasingly demand even smaller attack surfaces and faster deployment times. Project Hummingbird represents Red Hat's response to this evolution—a next-generation family of ultra-minimal base images engineered specifically for stateless microservices, serverless functions, and ephemeral workloads. Built on the same enterprise RHEL foundation as UBI, Hummingbird strips away traditional system administration tools and reduces the image to only what modern applications require at runtime. The result is measurably smaller images (often 50-70% smaller than UBI Minimal), faster cold-start times in serverless environments, and a significantly reduced attack surface—all while maintaining the same free redistribution rights, security lifecycle, and sovereignty guarantees that make UBI valuable for enterprise adoption.

**Hummingbird's Innovation:**
- Ultra-minimal base images optimized for cloud-native workloads
- Hardened security posture with minimal attack surface
- Optimized for modern application patterns
- Enhanced supply chain transparency

### Technical Architecture

Project Hummingbird achieves its minimal footprint through aggressive package reduction and architectural constraints designed for immutable, single-process containers. Unlike traditional base images that include package managers, shell utilities, and system maintenance tools, Hummingbird images ship without `dnf`, `yum`, or even a full shell—the assumption is that containers are built once, validated through CI/CD pipelines, and run immutably in production. The image contains only the essential runtime libraries required by applications: core glibc dependencies, SSL/TLS libraries for secure communications, and critical system libraries like libcrypto. Security hardening is built into the image structure itself—the absence of package managers eliminates entire classes of runtime attacks, while the minimal library surface reduces exposure to CVEs. For package installation during builds, developers use multi-stage Dockerfiles: compile and install dependencies in a UBI-based builder stage, then copy only the application binary and runtime dependencies into the final Hummingbird layer. This separation ensures build-time tools never reach production while maintaining the enterprise security lifecycle for runtime components.

### When to Use Hummingbird vs. UBI

Choosing the right base image depends on your application architecture, operational requirements, and deployment patterns. Here's practical guidance for matching workloads to base images:

**Use Hummingbird when:**
- **Deploying stateless microservices** that follow strict immutability patterns—no runtime modifications, no debugging in production
- **Building serverless functions** where cold-start time and image size directly impact cost and performance (AWS Lambda, Azure Functions, Google Cloud Functions)
- **Optimizing for security-critical environments** where minimal attack surface is paramount and you can enforce immutable infrastructure
- **Running ephemeral workloads** that scale up and down frequently, where smaller images reduce registry bandwidth and node storage pressure
- **You have mature CI/CD pipelines** that include comprehensive testing, since you cannot install debugging tools at runtime

**Use UBI Minimal when:**
- **Migrating existing applications** to containers where you may need `microdnf` to install additional packages for compatibility
- **Building general-purpose microservices** that benefit from a small footprint but require occasional runtime package installation
- **You need debugging flexibility** with access to package managers and common utilities during troubleshooting
- **Teams are learning containerization** and need the safety net of traditional Linux tools

**Use UBI Standard when:**
- **Deploying monolithic applications** that expect a full Linux environment with standard utilities (`ls`, `ps`, `grep`, etc.)
- **Running applications with complex dependencies** requiring build tools and development packages
- **You need ISV-certified software** where vendors explicitly support UBI Standard
- **Legacy application compatibility** requires specific system packages and tools

**Use UBI Init when:**
- **Running multi-process containers** that require a proper init system (systemd-based)
- **Containerizing traditional applications** designed to run multiple daemons within a single container
- **Migration scenarios** where refactoring to single-process containers is not immediately feasible

**The practical decision path**: Start with UBI Minimal for most new cloud-native applications. Once your deployment patterns mature, your CI/CD pipeline includes comprehensive validation, and your team embraces immutability, migrate to Hummingbird for maximum security and efficiency. Reserve UBI Standard for applications that genuinely need the full environment, and use UBI Init only when architectural constraints require multi-process containers.

---

## Practical Implications for Digital Sovereignty

### Portability Across Clouds

Because UBI and Hummingbird are freely redistributable OCI-compliant images with no runtime licensing requirements, the same container image runs identically across any infrastructure:

- **AWS**: Deploy to ECS, EKS, Lambda (container images), or EC2-based container platforms
- **Azure**: Run on AKS, Azure Container Instances, Azure Container Apps, or Azure Functions
- **Google Cloud**: Use with GKE, Cloud Run, or Compute Engine container-optimized VMs
- **On-Premises**: Deploy to OpenShift, upstream Kubernetes, Podman, or any OCI-compatible runtime

An application containerized with UBI can be developed on a developer's laptop, tested in Azure, staged in an on-premises OpenShift cluster, and deployed to production on AWS—with zero base image changes. This portability extends to hybrid and edge deployments: the same image that runs in a datacenter runs on edge nodes, industrial gateways, or disconnected environments. Organizations maintain true cloud independence—switching cloud providers requires no application repackaging, just registry migration.

### Supply Chain Integration

UBI images integrate seamlessly with modern supply chain security tooling. Each image is cryptographically signed using Red Hat's container signing infrastructure, verifiable through `podman image trust` or Kubernetes admission controllers like Sigstore Policy Controller. Every package in UBI includes traceable provenance—organizations can generate Software Bill of Materials (SBOM) using tools like Syft or the Red Hat SBOM API, with each component linked back to specific RHEL RPM packages and upstream sources.

Vulnerability scanning tools (Clair, Trivy, Snyk, Anchore) recognize UBI images and leverage Red Hat's CVE data for accurate security assessment. Because UBI packages receive timely security updates through the RHEL lifecycle, scan results remain actionable—detected CVEs have clear remediation paths through base image updates rather than requiring application changes. This integration enables automated policy enforcement: organizations can block deployments of images with known CVEs, require signed base images, or mandate SBOM generation as part of their CI/CD pipeline.

### Regulatory Compliance

UBI's enterprise foundation helps organizations meet compliance requirements across multiple regulatory frameworks. RHEL's FIPS 140-2 validated cryptography is available in UBI images, supporting FedRAMP and other government compliance mandates. The predictable security update lifecycle and documented CVE remediation process align with PCI-DSS requirements for timely patching and vulnerability management.

For data sovereignty regulations (GDPR, CCPA, national data protection laws), UBI's free redistribution rights enable organizations to maintain complete control over where container images are stored and executed—no requirement to pull from US-based registries or route through specific geographic regions. Organizations can mirror UBI images to private registries in compliant jurisdictions, build custom derivative images with additional hardening, and maintain full audit trails of base image provenance without vendor dependencies or licensing restrictions that could conflict with sovereignty requirements.

---

## Real-World Example

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

This example demonstrates UBI best practices: using the `:latest` tag pulls the most recent security updates (acceptable for development; use version-specific tags in production), `microdnf clean all` removes package metadata to reduce image size, and the registry path `registry.access.redhat.com` requires no authentication. The layering approach—install system packages first, copy dependency manifests, install application dependencies, then copy application code—optimizes Docker build caching so code changes don't trigger package reinstallation.

For production hardening, specify a non-root user with `USER 1001`, use multi-stage builds to separate build-time dependencies from runtime, and pin to specific UBI versions like `ubi9/ubi-minimal:9.3-1361` for reproducible builds. Consider switching to Hummingbird for stateless applications that don't require runtime package installation.

---

## The Upstream Connection

### Red Hat's Investment in RHEL

Red Hat employs over 1,000 engineers dedicated to RHEL development, security, and quality assurance—this engineering investment directly benefits every UBI user. The company maintains thousands of RPM packages, contributes to hundreds of upstream open source projects (Linux kernel, systemd, glibc, OpenSSL, and more), and operates a dedicated Product Security team that triages CVEs, develops patches, and coordinates security disclosures.

This investment extends beyond code: Red Hat engineers serve as maintainers and core contributors to critical infrastructure projects, participate in security response teams, and contribute to standards bodies that define container specifications and security practices. When organizations build on UBI, they inherit the results of this sustained engineering effort without requiring a subscription—the same packages, security patches, and quality assurance that enterprise customers receive.

### Community Benefits

UBI's free availability creates positive externalities across the container ecosystem. Independent software vendors can build and redistribute applications on enterprise-grade foundations without licensing barriers, making commercial software more accessible. Open source projects gain a stable, long-lifecycle base image option that doesn't compromise on security or redistributability. Educational institutions can teach containerization using production-quality images without procurement obstacles.

The broader impact extends to standardization: by providing freely available enterprise base images with clear security practices and transparent supply chains, Red Hat establishes patterns that raise the bar for the entire ecosystem. Organizations that start with UBI for cost and sovereignty reasons often discover that the predictable lifecycle and security update process improves their operational maturity—benefits that extend beyond Red Hat technologies to their entire infrastructure.

---

## Key Benefits Summary

**For Technical Teams:**
- Consistent, well-tested foundation
- Multi-architecture support
- Clear security update process

**For Organizations:**
- No licensing restrictions on redistribution
- Enterprise support available when needed
- Reduced supply chain risk

**For Digital Sovereignty:**
- Freedom from vendor lock-in
- Transparent build and update process
- Global availability without restrictions

---

## References and Further Reading

- [Red Hat UBI Documentation](https://developers.redhat.com/products/rhel/ubi)
- [Project Hummingbird Resources]
- [OCI Image Specification]
- [Container Security Best Practices]
