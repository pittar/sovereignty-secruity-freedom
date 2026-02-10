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

1. **UBI Minimal**: Smallest footprint for microservices
2. **UBI Standard**: Balanced size and tooling
3. **UBI Init**: For multi-process containers
4. **UBI Micro**: Minimal attack surface, ultra-small

[Technical details on each variant, use cases, and selection criteria]

### Technical Deep Dive: UBI Architecture

```
[Diagram or detailed explanation of UBI build process, package selection, and security hardening]
```

### Lifecycle and Security Updates

[Explanation of RHEL lifecycle alignment, CVE patching process, and update cadence]

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

[Examples of using UBI/Hummingbird images across AWS, Azure, GCP, on-prem]

### Supply Chain Integration

[How UBI integrates with signing, scanning, and SBOM generation]

### Regulatory Compliance

[How UBI helps meet various compliance requirements: FedRAMP, PCI-DSS, etc.]

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

[Explanation of best practices demonstrated in this example]

---

## The Upstream Connection

### Red Hat's Investment in RHEL

[Details on engineering effort, package maintainership, and security team]

### Community Benefits

[How UBI's free availability benefits the entire ecosystem]

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
