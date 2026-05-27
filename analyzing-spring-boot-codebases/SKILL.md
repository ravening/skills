---
name: analyzing-spring-boot-codebases
version: 1.1.0
description: Audits Java Spring Boot codebases end-to-end as a prose-driven, Markdown-only guidance package, covering architecture mapping with a richer Mermaid diagram set, build dependencies and platform versions, Spring-specific security review (JWT, OAuth2/OIDC, sessions, deserialization, SpEL, path traversal, mass assignment, race conditions, template injection), data-layer depth for Hibernate and JPA (open-in-view, fetch strategies, second-level cache, batch and bulk operations, dialect issues, Flyway and Liquibase migration safety, HikariCP sizing), cloud-native posture (Dockerfile hardening, Kubernetes manifests, secrets management, optional service mesh), observability and resilience (Resilience4j, MDC, OpenTelemetry, Micrometer, SLO and SLI, async tracing), API and contracts review (OpenAPI, versioning, idempotency, throttling, GraphQL, optional gRPC), incremental PR-scope review via Diff Mode, and Azure-specific resilience review that produces a separate `azure-resilience-report.md` when Azure deployment signals are present; preserves the existing eight-step workflow and P0–P3 severity scheme. Use when inspecting a Spring Boot codebase, performing backend code review, running a security or architecture assessment, reviewing a pull request scoped to a Spring Boot service, or assessing Azure resilience for a Spring Boot deployment.
keywords:
  - Spring Boot
  - Java
  - Maven
  - Gradle
  - JPA
  - Hibernate
  - Spring Security
  - Kubernetes
  - Dockerfile
  - OpenAPI
  - audit
  - code review
  - security review
---

# Analyzing Spring Boot Codebases

Use this skill when the repository is a Java Spring Boot application and the goal is to understand how it works, review design and dependencies, identify security risks, and generate diagrams plus a structured audit report.

## Outputs

### Required outputs

Always produce all of the following, even when the user does not explicitly ask for diagrams. Diagrams are part of the audit, not an optional add-on. Do not skip them, do not summarize them in prose, and do not defer them to a later step.

- `spring-boot-audit-report.md`
- A Mermaid **component diagram** showing application structure with actual discovered controllers, services, repositories, data stores, message brokers, caches, and external systems
- A Mermaid **representative sequence diagram** for one end-to-end request, including auth, validation, service logic, persistence, integration calls, and error handling
- A Mermaid **package dependency diagram** (always-on per `reference/architecture.md`)
- A prioritized findings list using P0, P1, P2, and P3 severity

### Conditional outputs

Produce each of the following when its trigger is present in the target codebase. The triggers are defined in `reference/architecture.md` (per-diagram "When to generate" rules) and in the Conditional Loading Guidance table at the bottom of this document. If the trigger is present, the diagram is required, not optional.

- **Per-flow sequence diagrams** — one per top business flow discovered in Step 2 (target three to five), in addition to the representative sequence diagram above
- **ER / domain model diagram** — when JPA `@Entity` classes are present
- **Async / event flow diagram** — when `@KafkaListener`, `@RabbitListener`, or `@EventListener` are present
- **Deployment diagram** — when `Dockerfile`, `docker-compose.yml`, or Kubernetes manifests are present
- **State diagram per entity** — one per entity that has an explicit lifecycle or status enum
- **Auth / security flow diagram** — when OAuth2, OIDC, or JWT are in use
- `azure-resilience-report.md` — produced **only** when Azure detection signals are present in the target codebase (Azure SDK dependencies, `azure.*` or `spring.cloud.azure.*` configuration keys, ARM or Bicep templates, `azurerm_*` or `azapi_*` Terraform resources, container references to `*.azurecr.io` or `mcr.microsoft.com`, App Service or `*.azurewebsites.net` references, or CI steps invoking `azure/login`, `azure/cli`, `azure/webapps-deploy`, or `aks-set-context`). When Azure signals are present, this report includes one Azure-aware end-to-end Mermaid sequence diagram per top critical flow. When no Azure signals are present, do not produce this file.

## Workflow

Copy this checklist into the working notes and keep it updated:

```text
Spring Boot Audit Progress
- [ ] Step 1: Detect build system and project structure
- [ ] Step 2: Inventory modules, layers, and endpoints
- [ ] Step 3: Review dependencies and platform versions
- [ ] Step 4a: Always-on diagrams
  - [ ] Component diagram
  - [ ] Representative sequence diagram
  - [ ] Package dependency diagram
- [ ] Step 4b: Conditional diagrams (skip rows whose trigger is absent)
  - [ ] Per-flow sequence diagrams (3–5 top flows)
  - [ ] ER / domain model diagram (when @Entity present)
  - [ ] Async / event flow diagram (when @KafkaListener / @RabbitListener / @EventListener present)
  - [ ] Deployment diagram (when Dockerfile / docker-compose / k8s manifests present)
  - [ ] State diagram per entity (when status/lifecycle enum present)
  - [ ] Auth / security flow diagram (when OAuth2 / OIDC / JWT present)
  - [ ] Azure end-to-end flow diagram(s) (when Azure signals present; lives in azure-resilience-report.md)
- [ ] Step 5: Run security and vulnerability review
- [ ] Step 6: Run code quality and maintainability review
- [ ] Step 7: Review configuration and tests
- [ ] Step 8: Write the final report (with all diagrams above embedded)
```

## Step 1: Detect project shape

1. Detect whether the project uses Maven or Gradle.
2. Identify the Spring Boot entry point by locating `@SpringBootApplication`.
3. Inventory modules, package roots, Java sources, test sources, config files, migrations, Docker/Kubernetes manifests, and CI files.

## Step 2: Map application layers

Classify code into these groups where present:

- Controllers and request handlers
- Services and domain orchestration
- Repositories and persistence adapters
- Entities, DTOs, and mappers
- Security configuration and filters
- Messaging, scheduling, async, and integration clients
- Cross-cutting concerns such as exception handling, logging, metrics, and caching

For every major layer, capture real class names and file paths.

## Step 3: Review dependencies

Use [reference/dependencies.md](reference/dependencies.md) when reviewing:

- Spring Boot version and support status
- Java target version
- Direct and transitive dependencies
- Outdated libraries
- Known vulnerable components
- Build plugins that affect security, packaging, or testing

Do not label a dependency as vulnerable unless the version is identified and the issue plausibly applies to the project.

## Step 4: Build architecture and flow diagrams

Use [reference/architecture.md](reference/architecture.md). Diagrams are mandatory output, not narrative. Emit each diagram as a Mermaid fenced code block in the audit report. Do not replace any diagram with a prose summary.

### Step 4a: Always generate

Produce these diagrams every time, regardless of project size. Skipping them is a defect of the audit, not a scoping choice.

- One Mermaid **component diagram** with actual discovered controllers, services, repositories, data stores, message brokers, caches, and external systems
- One Mermaid **representative sequence diagram** for an end-to-end request, including auth, validation, service logic, persistence, integration calls, and error handling. Prefer a business-critical or structurally rich endpoint.
- One Mermaid **package dependency diagram** surfacing layering between top-level packages (controller, service, repository, domain, config). Show any layering violations as visible edges.

### Step 4b: Conditional diagrams

For each diagram below, check the trigger against the target codebase. If the trigger is present, generate the diagram — do not defer it, do not summarize it in prose, do not skip it because the codebase "looks small". If the trigger is absent, skip it without comment.

- **Per-flow sequence diagrams** — trigger: more than one significant business flow discovered in Step 2. Generate one diagram for each of the top three to five flows. These are in addition to the representative sequence diagram in Step 4a.
- **ER / domain model diagram** — trigger: JPA `@Entity` classes present.
- **Async / event flow diagram** — trigger: `@KafkaListener`, `@RabbitListener`, or `@EventListener` present.
- **Deployment diagram** — trigger: `Dockerfile`, `docker-compose.yml`, or Kubernetes manifests present.
- **State diagram per entity** — trigger: an entity has an explicit lifecycle or status enum. Generate one diagram per such entity.
- **Auth / security flow diagram** — trigger: OAuth2, OIDC, or JWT in use.
- **Azure end-to-end flow diagram(s)** — trigger: any Azure detection signal in the target codebase. Owned by [reference/azure-resilience.md](reference/azure-resilience.md), not by this step. Lives in `azure-resilience-report.md`, not in the main audit report.

Use the Mermaid templates in [reference/architecture.md](reference/architecture.md) for shape and syntax. Replace every placeholder label with a real discovered name from the target codebase — class names, package names, topic and queue names, deployment and ingress names, enum values. Do not emit a diagram with placeholder labels.

## Step 5: Run security review

Use [reference/security.md](reference/security.md).

Check for:

- Broken access control
- Authentication and session flaws
- Secrets in source or config
- Injection risks
- Insecure deserialization or unsafe binding
- Sensitive data exposure
- Security misconfiguration
- Vulnerable components
- SSRF, file upload, and actuator exposure risks
- Logging and monitoring gaps

Separate confirmed findings from suspected risks.

## Step 6: Run code review

Use [reference/code-review.md](reference/code-review.md).

Review for:

- Layering violations and tight coupling
- God classes and overlong methods
- Field injection and circular dependency smells
- Transaction boundary mistakes
- DTO/entity leakage across boundaries
- Pagination, caching, and N+1 query issues
- Blocking I/O or thread misuse
- Error handling and logging quality
- Testability and maintainability concerns

## Step 7: Review configuration and tests

Check:

- `application.yml`, `application.yaml`, and `application.properties`
- profile separation for local, dev, test, staging, and prod
- datasource, JPA, cache, messaging, and actuator settings
- security-relevant defaults
- test presence by layer, including controller, service, repository, and integration coverage

Use [reference/extended-checks.md](reference/extended-checks.md) for optional deeper validation.

## Step 8: Write the report

Use [reference/report-template.md](reference/report-template.md).

Embed every diagram produced in Step 4a and Step 4b directly in the report as Mermaid fenced code blocks. The report is not complete until every required and triggered diagram is present.

Rules:

- Include file paths and line numbers for findings whenever possible
- Prefer concise evidence over vague statements
- Use severity levels P0 to P3
- Pair each important flaw with a specific remediation
- When useful, include before/after code snippets
- End with the most important architectural and security improvements first

## Worked Example

The following walkthrough illustrates how the skill applies to one hypothetical Spring Boot project, `orders-service`, a Maven-built REST service with JPA, Kafka, and an AKS deployment. Artifact names only — no executable code.

- **Step 1 — Detect project shape.** Build system is Maven (`pom.xml` at repo root). Entry point `com.acme.orders.OrdersApplication` carries `@SpringBootApplication`. Inventory captures `src/main/java`, `src/test/java`, `src/main/resources/application.yml`, `db/migration/` Flyway scripts, `Dockerfile`, `k8s/` manifests, and `.github/workflows/ci.yml`.
- **Step 2 — Map application layers.** Controllers under `web/`, services under `service/`, JPA repositories under `repo/`, entities under `domain/`, DTOs under `web/dto/`, security config under `config/security/`, a `KafkaListener` under `messaging/`. Real class names recorded (e.g., `OrderController`, `OrderService`, `OrderRepository`, `Order`).
- **Step 3 — Review dependencies.** Spring Boot version, Java target, direct and transitive coordinates, and build plugins reviewed against [reference/dependencies.md](reference/dependencies.md). Findings filed under `spring-boot-audit-report.md`.
- **Step 4 — Architecture and flow diagrams.** Using [reference/architecture.md](reference/architecture.md), produce one component diagram (controllers → services → repositories → PostgreSQL, Kafka, external `payments-api`) and one representative sequence diagram for `POST /orders`. Because JPA entities, Kafka listeners, and Kubernetes manifests are present, additionally produce ER, async/event, and deployment diagrams as the file's per-diagram rules direct.
- **Step 5 — Security review.** Using [reference/security.md](reference/security.md), review JWT validation, OAuth2 flows, session handling, deserialization, mass-assignment binding on `OrderRequest`, and actuator exposure. Findings classified P0–P3.
- **Step 6 — Code review.** Using [reference/code-review.md](reference/code-review.md), flag layering violations, transaction boundaries, DTO/entity leakage, and N+1 query risks across `OrderService` and `OrderRepository`.
- **Step 7 — Configuration and tests.** Validate `application.yml` profile separation (`dev`, `staging`, `prod`), datasource settings, actuator defaults, and test coverage by layer. Use [reference/extended-checks.md](reference/extended-checks.md) for deeper validation.
- **Step 8 — Final report.** Using [reference/report-template.md](reference/report-template.md), write `spring-boot-audit-report.md` with file paths, line numbers, P0–P3 severity, and remediations. Because the codebase contains `azure-identity` and `azure-security-keyvault-secrets` dependencies plus `*.azurewebsites.net` references in `k8s/`, additionally load [reference/azure-resilience.md](reference/azure-resilience.md) and produce `azure-resilience-report.md` alongside the main report, with an Azure end-to-end flow diagram, a breaking-point catalog, and prioritized resilience recommendations.

## Guardrails

- Use forward-slash file paths.
- Keep terminology consistent: endpoint, controller, service, repository, entity, DTO, mapper, filter, config, vulnerability, finding.
- Prefer constructor injection over field injection in recommendations.
- Prefer DTOs at API boundaries over exposing entities directly.
- Prefer confirmed evidence over assumptions.
- If a check cannot be completed, mark it as unknown and say why.

## Additional references

- Architecture mapping: [reference/architecture.md](reference/architecture.md)
- Code review: [reference/code-review.md](reference/code-review.md)
- Dependency review: [reference/dependencies.md](reference/dependencies.md)
- Security review: [reference/security.md](reference/security.md)
- Report format: [reference/report-template.md](reference/report-template.md)
- Data layer (Hibernate, JPA, migrations, HikariCP): [reference/data-layer.md](reference/data-layer.md)
- Cloud-native (Docker, Kubernetes, secrets, service mesh): [reference/cloud-native.md](reference/cloud-native.md)
- Observability and resilience: [reference/observability-and-resilience.md](reference/observability-and-resilience.md)
- API and contracts (OpenAPI, versioning, GraphQL, gRPC): [reference/api-and-contracts.md](reference/api-and-contracts.md)
- Diff mode (PR-scoped review): [reference/diff-mode.md](reference/diff-mode.md)
- Optional deep checks: [reference/extended-checks.md](reference/extended-checks.md)
- Azure resilience (conditional): [reference/azure-resilience.md](reference/azure-resilience.md)

## Conditional Loading Guidance

Load each reference file only when its triggering signals are present in the target codebase. **If no listed signal is present, skip loading that reference file.**

| Reference File | Load Always? | Triggering Signals |
|---|---|---|
| [reference/architecture.md](reference/architecture.md) | Yes | Always; the skill always produces architecture mapping. |
| [reference/code-review.md](reference/code-review.md) | Yes | Always; the skill always performs maintainability review. |
| [reference/dependencies.md](reference/dependencies.md) | Yes | Always; the skill always reviews build dependencies. |
| [reference/security.md](reference/security.md) | Yes | Always; security review is a first-class output. |
| [reference/report-template.md](reference/report-template.md) | Yes | Always; defines the report shape. |
| [reference/data-layer.md](reference/data-layer.md) | No | Presence of `spring-boot-starter-data-jpa`, `spring-data-jpa`, `hibernate-core`, JPA `@Entity` classes, `JpaRepository`, Flyway or Liquibase migrations, or HikariCP configuration. |
| [reference/cloud-native.md](reference/cloud-native.md) | No | Presence of `Dockerfile`, `docker-compose.yml`, Kubernetes manifests under `k8s/`, `helm/`, `kustomization.yaml`, `Chart.yaml`, or service-mesh artifacts (Istio CRDs, Linkerd annotations). |
| [reference/observability-and-resilience.md](reference/observability-and-resilience.md) | No | Presence of `resilience4j-*`, `io.micrometer:*`, `opentelemetry-*`, `spring-boot-starter-actuator` with metrics or tracing exporters, MDC usage, or async executors needing trace propagation. |
| [reference/api-and-contracts.md](reference/api-and-contracts.md) | No | Presence of `@RestController`, `@RequestMapping`, OpenAPI or Swagger dependencies (`springdoc-openapi-*`), GraphQL dependencies (`spring-boot-starter-graphql`, `graphql-java`), gRPC dependencies (`grpc-spring-boot-starter`, `io.grpc:*`), or any REST endpoint surface. |
| [reference/diff-mode.md](reference/diff-mode.md) | No | The user invokes the skill against a PR diff or asks for a changed-files-only review. |
| [reference/extended-checks.md](reference/extended-checks.md) | No | The user requests a deeper review, or `cloud-native.md` / `observability-and-resilience.md` overlap conditions are present and extra depth is wanted. |
| [reference/azure-resilience.md](reference/azure-resilience.md) | No | Azure SDK dependencies (`com.azure:*`, `com.microsoft.azure:*`, `azure-spring-boot-starter-*`, `azure-identity`, `azure-security-keyvault-*`, `azure-storage-*`, `azure-messaging-servicebus`, `azure-messaging-eventhubs`, `azure-cosmos`, `azure-monitor-*`, `applicationinsights-*`); configuration keys under `azure.*` and `spring.cloud.azure.*` in `application.yml` or `application.properties`; ARM and Bicep templates (`*.bicep`, `azuredeploy.json`); Terraform resources of type `azurerm_*` or `azapi_*`; Docker or Kubernetes references to `*.azurecr.io`, `mcr.microsoft.com`, `mi-*` Managed Identity tokens, or AKS-specific annotations; App Service `web.config` or `*.azurewebsites.net` references; CI pipeline steps invoking `azure/login`, `azure/cli`, `azure/webapps-deploy`, or `aks-set-context`. |

If no triggering signal for a given reference file is present in the target codebase, skip loading that reference file.
