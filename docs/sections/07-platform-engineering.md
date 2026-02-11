# Platform Engineering with Backstage

[← Developer Experience](06-developer-experience.md) | [Table of Contents](../OUTLINE.md) | [Next: Confidential Containers →](08-confidential-containers.md)

---

## Executive Summary

Developers today navigate dozens of tools, services, and platforms just to deploy a simple application. GitHub for code, Jenkins or Tekton for builds, multiple Kubernetes clusters, separate monitoring tools, scattered documentation across wikis and repos—the cognitive load is staggering. Platform engineering emerged to solve this: dedicated teams building internal developer platforms (IDPs) that provide self-service infrastructure with "golden paths"—opinionated, secure defaults that make the right way the easy way.

Red Hat Developer Hub, powered by the open source Backstage project, delivers an enterprise-grade IDP that unifies service catalogs, software templates, documentation, and AI-assisted development in a single portal. Developers discover services, scaffold new projects with organizational best practices baked in, find documentation, and access infrastructure—all without leaving their workflow. Platform teams enforce standards through templates while preserving developer flexibility. Organizations reduce time-to-first-deployment for new developers from days to hours.

Unlike proprietary developer portals tied to specific clouds or vendors, Developer Hub is open source and self-hostable. It integrates with any toolchain, runs on any infrastructure, and follows the same upstream-first model that enables digital sovereignty across Red Hat's portfolio.

---

## The Rise of Platform Engineering

### From DevOps to Platform Engineering

DevOps promised collaboration between development and operations, breaking down silos. Reality proved messier: developers became responsible for infrastructure they didn't fully understand, while operators struggled with application-specific requirements. "You build it, you run it" worked at small scale but created bottlenecks as organizations grew. Platform engineering emerged as the next evolution—dedicated teams building internal platforms that enable developer self-service without requiring deep infrastructure expertise. Platform teams treat developers as customers, measuring success through adoption metrics and developer satisfaction rather than ticket volume.

**The Platform Engineering Promise:**
- Reduce cognitive load for developers
- Provide golden paths without removing flexibility
- Self-service infrastructure and tooling
- Metrics-driven improvement

### The Challenge: Tool Sprawl and Fragmentation

Enterprise developers interact with 15-30 different tools daily. Each tool has its own authentication, UI patterns, and conceptual model. Finding the right Kubernetes cluster for a project requires tribal knowledge. Discovering which team owns a service means searching Slack history. Starting a new project involves copying from existing repos, hoping to replicate the right patterns. This fragmentation taxes working memory and creates friction at every step. Gartner estimates developers spend only 35% of their time writing code, with the rest consumed by tooling navigation, environment setup, and searching for information.

**Typical Enterprise Developer Toolchain:**
- Source control (GitHub, GitLab, Bitbucket)
- CI/CD (Jenkins, Tekton, ArgoCD)
- Container registries
- Kubernetes clusters (multiple)
- Monitoring and logging tools
- Secret managers
- Service catalogs
- Documentation (scattered across wikis, repos)
- Security scanning tools
- And 20+ more...

---

## Internal Developer Platforms (IDPs)

### What is an Internal Developer Platform?

An Internal Developer Platform (IDP) is a curated layer of tools and workflows that simplifies application delivery for developers while maintaining flexibility and control for platform teams. IDPs provide self-service capabilities—developers create new services, deploy to environments, access infrastructure resources—through standardized interfaces rather than manual requests. Golden paths guide developers toward organizational best practices (approved frameworks, security scanning, CI/CD patterns) without prohibiting alternative approaches when needed. IDPs measure success through platform adoption metrics, developer velocity improvements, and reduced cognitive load.

**IDP Core Functions:**
1. **Service Catalog**: Inventory of all services, APIs, and resources
2. **Software Templates**: Scaffolding for new projects
3. **Documentation Hub**: Centralized, searchable docs
4. **Self-Service Actions**: Automated common tasks
5. **Observability**: Unified view of service health
6. **Developer Portal**: Single pane of glass

### Why Open Source IDPs Matter for Sovereignty

Proprietary developer portals from cloud vendors lock organizations into specific ecosystems. AWS Service Catalog integrates deeply with AWS services but becomes friction when adopting multi-cloud. Azure Developer Portal assumes Azure infrastructure. These platforms create gravitational pull toward vendor services, making cloud-agnostic architecture practically difficult even when technically possible. Migrating away requires rebuilding developer workflows, retraining teams, and often reverting to manual processes during transitions. Open source IDPs like Backstage run anywhere, integrate with any toolchain, and remain under organizational control. The platform becomes organizational IP that moves with you, not vendor property that constrains you.

**For hybrid infrastructure deployments**, open source IDPs provide critical flexibility. Canadian government organizations often operate applications across Shared Services Canada data centers (for Protected B workloads), departmental on-premises infrastructure (for specialized systems), and public cloud environments (for development, testing, or specific elastic workloads). Backstage's plugin architecture enables a unified service catalog spanning all these environments—developers discover and access services regardless of whether they run in AWS development clusters, SSC production OpenShift, or air-gapped departmental infrastructure. Software templates enforce consistent standards (security scanning, supply chain verification, compliance policies) across all deployment targets. Developers don't learn separate portals for each infrastructure—one Developer Hub instance provides self-service capabilities across the entire hybrid estate. When workloads migrate from cloud to on-premises, the service catalog updates automatically through Kubernetes discovery plugins, maintaining consistent developer experience without manual catalog maintenance.

---

## Backstage: The Open Source Developer Portal

### What is Backstage?

Backstage is an open-source framework for building internal developer portals, originally created by Spotify to manage their sprawling microservices ecosystem. As Spotify scaled beyond 1,000 microservices, engineers struggled to discover services, understand ownership, and find documentation. Backstage emerged as their solution: a centralized portal providing service catalog, software templates, and unified documentation. Spotify open-sourced Backstage in 2020, contributing it to the CNCF where it rapidly gained adoption across industries. Today Backstage powers developer platforms at American Airlines, Netflix, Expedia, and hundreds of other organizations managing complex service architectures.

**Backstage Origins:**
- Created by Spotify to manage their microservices ecosystem
- Open sourced in 2020
- Adopted by hundreds of organizations
- CNCF Incubating project

### Backstage Architecture

Backstage uses a plugin-based architecture where core functionality (catalog, templates, documentation) provides the foundation, and plugins extend capabilities for specific tools and use cases. The frontend is a React application, backend is Node.js with TypeScript, and the software catalog stores metadata in PostgreSQL or SQLite. Plugins can add UI components, backend APIs, and integrations with external systems. This modularity enables organizations to compose precisely the functionality they need—Kubernetes visibility, CI/CD integration, cloud resource management—without bloating the platform with unused features. The plugin ecosystem includes 100+ open-source plugins for common integrations and organizations develop custom plugins for proprietary systems.

**Core Components:**
- **Software Catalog**: Service ownership and metadata
- **Software Templates**: Project scaffolding with Cookiecutter-style templates
- **TechDocs**: Docs-as-code with Markdown
- **Kubernetes Plugin**: Multi-cluster visibility
- **Plugin Ecosystem**: Extensible with hundreds of plugins

```yaml
# Example: Service definition in Backstage catalog
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Handles payment processing
  annotations:
    github.com/project-slug: myorg/payment-service
    backstage.io/kubernetes-id: payment-service
    sonarqube.org/project-key: payment-service
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: ecommerce-platform
  dependsOn:
    - component:database
    - component:notification-service
  providesApis:
    - payment-api
```

---

## Red Hat Developer Hub: Enterprise Backstage

### From Upstream to Enterprise

Red Hat Developer Hub packages Backstage with enterprise hardening, support, and Red Hat ecosystem integrations. While upstream Backstage requires assembly of plugins, configuration management, and operational practices, Developer Hub provides a curated, tested distribution ready for production deployment. Security hardening includes vulnerability scanning, SBOM generation, and regular security updates through Red Hat's release process. Dynamic plugins enable plugin updates without rebuilding the entire platform—critical for air-gapped environments. Red Hat provides enterprise support with SLAs, professional services for implementation, and ongoing engineering investment ensuring compatibility with OpenShift platform evolution.

**Enterprise Enhancements:**
- Supported and hardened distribution
- Pre-integrated with Red Hat ecosystem
- Enterprise authentication (SSO, RBAC)
- Air-gapped and disconnected support
- Dynamic plugins for easier customization
- Professional support and SLA

### Integration with OpenShift Ecosystem

Developer Hub integrates deeply with OpenShift platform services. The Kubernetes plugin visualizes workloads across multiple OpenShift clusters, showing pod status, resource consumption, and deployment history directly in service catalog entries. Software templates create Tekton pipelines automatically when scaffolding new projects. ArgoCD integration displays GitOps sync status and deployment history. Templates can provision Dev Spaces workspaces, enabling developers to start coding immediately after creating services. Authentication integrates with OpenShift OAuth, inheriting corporate SSO configurations. This tight integration provides unified developer experience across the entire OpenShift platform while maintaining Backstage's cloud-agnostic architecture for non-OpenShift integrations.

---

## Technical Deep Dive: Software Templates

### Golden Paths with Templates

Software templates encode organizational best practices as reusable scaffolding. Rather than copying existing projects and hoping to replicate the right patterns, developers use templates that embed security scanning, approved frameworks, standardized CI/CD, monitoring instrumentation, and compliance requirements from day one. Templates don't mandate a single approach—organizations create multiple templates for different use cases (microservices, data pipelines, frontend apps, batch jobs)—but each template represents a vetted golden path. Developers can deviate when needed, but the default experience delivers compliant, secure, observable services without requiring deep platform expertise.

```yaml
# Example: Software template for microservice
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: spring-boot-microservice
  title: Spring Boot Microservice
  description: Create a new Spring Boot microservice with CI/CD
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service Information
      required:
        - name
        - owner
      properties:
        name:
          title: Service Name
          type: string
          description: Unique name for your service
        owner:
          title: Owner
          type: string
          description: Team responsible for this service
          ui:field: OwnerPicker

    - title: Database
      properties:
        database:
          title: Database Type
          type: string
          enum:
            - postgresql
            - mysql
            - mongodb
            - none

  steps:
    - id: fetch-template
      name: Fetch Application Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          database: ${{ parameters.database }}

    - id: create-repo
      name: Create GitHub Repository
      action: publish:github
      input:
        repoUrl: github.com?owner=myorg&repo=${{ parameters.name }}
        description: ${{ parameters.description }}

    - id: create-pipeline
      name: Create Tekton Pipeline
      action: kubernetes:create
      input:
        manifest: |
          apiVersion: tekton.dev/v1beta1
          kind: Pipeline
          metadata:
            name: ${{ parameters.name }}-pipeline

    - id: register
      name: Register Component
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.create-repo.output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  output:
    links:
      - title: Repository
        url: ${{ steps.create-repo.output.remoteUrl }}
      - title: Pipeline
        url: ${{ steps.create-pipeline.output.pipelineUrl }}
      - title: View in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

This template demonstrates Backstage's power: developers fill a simple form (service name, database choice) and the template creates a GitHub repository, scaffolds application code with selected database configuration, provisions Tekton CI/CD pipeline, and registers the service in the catalog—all in under 60 seconds. The scaffolded code includes security scanning in the pipeline, standardized logging configuration, health check endpoints, and Prometheus metrics collection. Templates use Jinja-style templating for code generation and execute actions (create repos, apply Kubernetes manifests, trigger workflows) through Backstage's action framework.

### Automating Best Practices

Templates enforce organizational standards automatically. Security teams define required scanning tools—templates include them in scaffolded pipelines. Compliance requires specific logging formats—templates generate code with correct log structured. Platform SLOs demand health checks—templates provide working implementations. This automation shifts policy enforcement left: compliance happens at creation time rather than through post-deployment audits. Developers don't experience this as restriction—templates simply generate working code that happens to meet all requirements. Platform teams update templates once and all future services inherit improvements.

**Government organizations use templates to codify regulatory compliance requirements.** Canadian federal departments create templates embedding TBS security controls, CCCS hardening guidance, and departmental standards—every new service automatically includes required audit logging, approved encryption algorithms, supply chain security scanning, and compliance documentation stubs. Templates can be environment-aware: services destined for Protected B on-premises deployment include stricter security controls and mandatory SPIFFE workload identity, while development environment templates optimize for rapid iteration. When policy changes (new TBS directives, updated CCCS guidance), platform teams update templates and developers automatically adopt new requirements in future services—ensuring compliance propagates organically rather than through manual policy enforcement campaigns. This approach scales: as departments grow from dozens to hundreds of services, templates ensure consistency without expanding compliance teams.

---

## Service Catalog and Ownership

### The Software Catalog

The software catalog is Backstage's core—a structured inventory of everything in your software ecosystem. Each entity (service, API, resource) has a YAML descriptor defining metadata, relationships, and annotations linking to external systems. Catalog ingestion happens automatically: Backstage discovers `catalog-info.yaml` files in Git repositories, imports definitions from external sources (ServiceNow, PagerDuty, cloud providers), and accepts manual registrations. The catalog maintains relationships: services depend on APIs, APIs are provided by components, components are owned by teams. This graph structure enables impact analysis (what breaks if we change this API?), cost attribution (which team owns expensive resources?), and organizational understanding (how many services does platform team support?).

**Catalog Entity Types:**
- **Components**: Services, libraries, websites
- **APIs**: REST, GraphQL, gRPC interfaces
- **Resources**: Databases, queues, caches
- **Systems**: Collections of components
- **Domains**: Business areas
- **Groups and Users**: Teams and individuals

### Establishing Ownership

Clear ownership is foundational for operational excellence. Every catalog entity declares an owner—typically a team, but can be an individual or organizational unit. Ownership drives accountability: when services fail, on-call rotations know who to page; when APIs need changes, consumers know who to contact; when costs spike, finance knows who to charge. Backstage enforces ownership through catalog validation—unowned entities fail catalog checks. This simple requirement transforms operational culture: services can't exist in production without someone responsible for their operation. Platform teams query ownership data for metrics: services per team, ownership coverage percentages, orphaned resources needing adoption.

### Dependency Mapping

Backstage visualizes service dependencies as directed graphs. Declare that payment-service depends on database and notification-service, and Backstage renders these relationships graphically while enabling impact analysis queries. Before changing an API, check its dependents—Backstage lists every consuming service. Planning database maintenance? Query all components depending on that resource. Dependency data comes from catalog definitions, but plugins can enhance it with runtime data: actual API calls observed in service mesh, database connections monitored by observability tools. This living dependency map reduces coordination overhead and prevents "we didn't know that service relied on this" incidents.

---

## TechDocs: Documentation as Code

### Docs-Like-Code Philosophy

TechDocs treats documentation as code: Markdown files live in service repositories alongside source code, maintained by the same engineers who write the services. Backstage builds documentation using MkDocs or similar static site generators, hosting rendered docs directly in the portal. When engineers update services, they update docs in the same pull request—documentation stays synchronized with code because they're versioned together. This approach solves the perennial problem of outdated docs: documentation drift becomes visible in code review, docs inherit the same quality practices as code (review, versioning, CI checks), and engineers own documentation like they own code.

```markdown
# Example: MkDocs documentation structure
# docs/index.md in your repository

# Payment Service Documentation

## Overview
This service handles all payment processing for our e-commerce platform.

## Architecture
See [architecture diagram](./architecture.md)

## API Reference
[OpenAPI Specification](./openapi.yaml)

## Runbooks
- [Deployment](./runbooks/deployment.md)
- [Troubleshooting](./runbooks/troubleshooting.md)
```

### Searchable, Centralized Documentation

TechDocs aggregates documentation from hundreds of repositories into unified search. Rather than remembering which repo contains runbook documentation, developers search Backstage for "database failover procedure" and find relevant docs across all services. Full-text search indexes all documentation, with results weighted by relevance and recency. Catalog annotations link services to their documentation—viewing a service in the catalog provides one-click navigation to its docs. This centralization dramatically reduces information discovery time. Onboarding new engineers becomes: "search Backstage for X" rather than "ask someone who knows where docs might be."

---

## AI-Assisted Development in Developer Hub

### GenAI Integration Points

Red Hat integrates InstructLab-based AI capabilities into Developer Hub, leveraging the same open source AI framework discussed in the AI Platform section. AI assistance appears at natural friction points: when developers search documentation, AI summarizes findings and suggests related topics; when using templates, AI helps populate complex parameters with context-aware suggestions; when troubleshooting incidents, AI analyzes logs and metrics to suggest root causes; when exploring unfamiliar services, AI generates plain-language explanations of architecture and dependencies. These integrations use retrieval-augmented generation (RAG) with catalog metadata and documentation, grounding AI responses in organizational knowledge rather than generic training data.

**AI Capabilities:**
1. **Intelligent Documentation**: AI-generated summaries and explanations
2. **Code Generation**: Template expansion with AI assistance
3. **Troubleshooting Help**: AI-powered incident response
4. **Search Enhancement**: Natural language queries
5. **Recommendation Engine**: Suggesting relevant services and templates

### Responsible AI in the Enterprise

AI features in Developer Hub respect enterprise data governance. AI models run within organizational infrastructure—no data leaves the environment for external AI services. Organizations control which data sources AI can access: public documentation might be fully accessible, while proprietary code or customer data remains restricted. Audit logs track AI interactions: what questions were asked, what responses were provided, which data sources were consulted. This transparency enables compliance verification and helps identify potential data leakage. AI suggestions clearly indicate uncertainty levels, and critical actions (deployments, infrastructure changes) require human approval regardless of AI confidence. The goal is augmentation, not automation of judgment.

---

## Observability and Metrics

### Unified Service Visibility

Backstage plugins integrate with observability platforms to surface service health directly in the catalog. The Prometheus plugin displays key metrics (request rate, error rate, latency) on service pages. Grafana integration embeds relevant dashboards. Logging plugins (Elasticsearch, Loki) provide log search scoped to specific services. PagerDuty integration shows on-call schedules and recent incidents. This unified visibility eliminates context-switching: developers viewing a service in Backstage see its code repository, documentation, deployment status, runtime metrics, and operational status without leaving the portal. On-call engineers responding to alerts use Backstage as their starting point for investigation.

### Platform Metrics

Platform teams measure Developer Hub effectiveness through adoption and impact metrics. Track template usage to understand which golden paths resonate with developers and which need improvement. Measure service creation velocity—how long from template instantiation to first deployment? Monitor documentation coverage percentage—what fraction of services have up-to-date docs? Survey developers regularly for satisfaction scores. DORA metrics (deployment frequency, lead time, change failure rate, recovery time) often improve after IDP adoption as friction decreases. These metrics inform platform roadmaps: which integrations to prioritize, which templates need refinement, where developer experience has gaps.

**Key Platform Metrics:**
- Template usage and success rate
- Service creation velocity
- Documentation coverage
- Mean time to onboard new developers
- Developer satisfaction (DORA metrics)

---

## The Upstream Connection

### Red Hat's Backstage Contributions

Red Hat engineers serve as Backstage maintainers and plugin contributors. Red Hat contributed the dynamic plugins capability enabling runtime plugin loading without rebuilding Backstage—critical for air-gapped enterprise deployments. OpenShift-specific plugins (topology visualization, operator integration, build config management) originate from Red Hat's engineering teams. Red Hat employs ~15 engineers dedicated to upstream Backstage development and ecosystem plugin maintenance. Engineering investment includes security hardening contributions, performance optimizations for large-scale catalogs (100,000+ entities), and enterprise authentication integrations. Red Hat holds Backstage steering committee representation, influencing project direction toward enterprise requirements while maintaining community accessibility.

### CNCF Ecosystem Integration

Backstage serves as a unifying layer for the cloud-native ecosystem. Kubernetes plugins provide cluster visibility, Tekton plugins integrate CI/CD, ArgoCD plugins show deployment status, Prometheus provides observability, Harbor manages container images. Backstage doesn't replace these tools—it surfaces their functionality in a unified developer experience. This integration model enables organizations to adopt best-of-breed CNCF tools without forcing developers to master each tool's unique interface. The CNCF ecosystem provides capabilities, Backstage provides coherent developer workflow. As the ecosystem evolves, new plugins extend Backstage rather than requiring developers to learn entirely new platforms.

---

## Real-World Use Case: Enterprise Platform Adoption

A multinational financial services company with 2,000 developers across 15 countries faced severe fragmentation: 8 different CI/CD tools, services split between AWS and Azure, documentation scattered across Confluence, SharePoint, and GitHub wikis. New developers took 3-4 weeks to become productive, learning organizational patterns through informal mentorship rather than standardized onboarding. Platform team spent 60% of time on support tickets answering "how do I deploy?" questions. Leadership mandated developer experience improvement while maintaining security and compliance requirements.

**Organization Profile:**
- 2,000+ developers
- 500+ microservices
- Multiple clouds and on-prem
- Compliance requirements (SOC 2, HIPAA)

**Implementation:**
1. Deploy Red Hat Developer Hub on OpenShift
2. Integrate with GitHub and GitLab
3. Create 15 golden path templates
4. Import existing services to catalog
5. Establish ownership model
6. Enable TechDocs for all services
7. Deploy to multiple geographic regions

**Results After 6 Months:**
- 300+ new services created from templates
- 98% of services have documented owners
- 40% reduction in "how do I..." support tickets
- 25% faster time-to-first-deployment for new developers
- 100% of new services follow security best practices

---

## Advanced Patterns

### Custom Plugins Development

Organizations build custom Backstage plugins for proprietary systems and unique workflows. A custom plugin might integrate with internal service mesh dashboards, visualize data pipeline lineage from a proprietary orchestration platform, or provide cost attribution from an internal cloud billing system. Backstage's plugin SDK (React for frontend, Node.js for backend) enables rapid development. Custom plugins inherit Backstage's authentication, navigation, and styling—developers focus on integration logic rather than building complete UIs. Organizations maintain internal plugin registries, treating plugins as reusable platform components. This extensibility prevents Backstage from becoming a constraint—if functionality is missing, build it rather than working around platform limitations.

### Multi-Tenant Deployments

Large enterprises often deploy multiple Backstage instances for different business units, ensuring separation of catalogs, access controls, and customization. Alternatively, a single Backstage deployment can serve multiple tenants using namespace-based access controls—teams see only their services and can access only their permitted resources. Multi-tenancy decisions balance operational simplicity (one deployment to maintain) against isolation requirements (regulatory separation, acquisition integration). Red Hat Developer Hub supports both models through flexible RBAC configuration and catalog filtering. Organizations often start with single deployment and split to multi-tenant as scale or compliance requirements demand.

### GitOps Integration

Software templates integrate seamlessly with GitOps workflows. Templates generate application manifests (Kubernetes YAML, Kustomize overlays, Helm charts) and commit them to GitOps repositories. ArgoCD or OpenShift GitOps detects new manifests and synchronizes applications to clusters automatically. This pattern enables developers to scaffold a service through Backstage and see it deployed to development environment within minutes—all following GitOps principles where Git remains the source of truth. Template actions can trigger initial ArgoCD syncs, set up sync policies, and configure automated deployment progression through environments. The combination of Backstage templates and GitOps delivers self-service deployment without sacrificing audit trails or rollback capabilities.

---

## Key Benefits Summary

**For Developers:**
- Single portal for all development needs
- Quick project scaffolding with best practices
- Clear service ownership and documentation

**For Platform Teams:**
- Metrics on platform usage and effectiveness
- Enforced golden paths without removing flexibility
- Extensible plugin architecture

**For Organizations:**
- Reduced cognitive load and faster innovation
- Consistent security and compliance
- Improved developer experience and retention

**For Digital Sovereignty:**
- Open source, self-hostable platform
- No dependency on proprietary developer portals
- Extensible to integrate any tools

---

## References and Further Reading

- [Backstage](https://backstage.io/)
- [Red Hat Developer Hub Documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/)
- [CNCF Backstage Project](https://www.cncf.io/projects/backstage/)
- [Platform Engineering](https://platformengineering.org/)
- [Team Topologies](https://teamtopologies.com/)

---

**Next:** [8. Advanced Security: Confidential Containers →](08-confidential-containers.md)
