---
name: spring-boot-repo-analyzer
description: Scans an entire Java Spring Boot repository to understand architecture, trace end-to-end code flow, analyze dependencies, generate diagrams, review code quality, and identify security and critical flaws with remediation guidance.
version: 1.0.0
---

# Spring Boot Repository Analyzer

This skill analyzes a complete Java Spring Boot repository and produces a technical understanding of how the application works, how requests and background jobs flow through the system, where the main dependencies and trust boundaries are, and which flaws or weaknesses deserve priority attention.

The skill is designed for repository-scale analysis rather than single-file review. It should inspect the full codebase, configuration, build files, and infrastructure descriptors before drawing conclusions.

## Primary objectives

- Scan the entire repository, not just the files currently open.
- Identify the architectural shape of the application, including modules, packages, layers, bounded contexts, and external integrations.
- Map end-to-end code flow from entry points to business logic, persistence, messaging, caches, and outbound services.
- Analyze dependency structure at build, module, package, and runtime-wiring levels.
- Detect vulnerabilities, critical flaws, architectural weaknesses, operational risks, and maintainability issues.
- Generate diagrams that explain component interaction and end-to-end request or event flow.
- Produce a code review that highlights important issues, explains impact, and suggests concrete improvements.
- Recommend remediation in priority order so engineers can act on the output.

## Inputs

Expected input is the repository root of a Java Spring Boot application, including as many of the following as are available:

- Maven or Gradle build files.
- `src/main/java`, `src/test/java`, and `src/main/resources`.
- `application.yml`, `application.properties`, and profile-specific config files.
- Dockerfiles, Compose files, Helm charts, Kubernetes manifests, Terraform, or deployment descriptors.
- CI/CD files.
- Database migration files such as Flyway or Liquibase.
- API contracts such as OpenAPI specs.
- Documentation files describing architecture, operations, or security.

## Required behavior

### 1. Repository-wide scan

Inspect the whole repository before summarizing anything.

- Read project structure and module boundaries.
- Detect monorepo or multi-module layout.
- Read build configuration and dependency declarations.
- Inspect source code, tests, configs, deployment files, and documentation.
- Infer framework usage such as Spring MVC, WebFlux, Spring Data JPA, Spring Security, Kafka, RabbitMQ, Redis, Feign, WebClient, RestTemplate, Quartz, Actuator, and scheduling.

Do not rely on a narrow subset of files unless the repository is truly small.

### 2. Spring-aware semantic analysis

Build understanding from Spring semantics, not only raw imports.

Identify and map:

- `@SpringBootApplication`
- `@RestController`, `@Controller`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- `@Service`, `@Component`, `@Repository`
- `@Configuration`, `@Bean`, `@ConfigurationProperties`
- `@Entity`, `@Embeddable`, `@MappedSuperclass`
- `@Transactional`
- `@Async`, `@Scheduled`, `@EventListener`
- `CommandLineRunner`, `ApplicationRunner`
- Spring Security configuration such as `SecurityFilterChain`, method security annotations, auth filters, JWT parsing, and access rules
- Messaging listeners such as Kafka or Rabbit consumers
- Outbound clients such as Feign clients, WebClient, RestTemplate, SDK wrappers, SMTP, storage SDKs, and payment clients

Track constructor injection, interface-to-implementation links, bean factories, profiles, and conditional beans.

### 3. End-to-end flow tracing

Map flows from every meaningful entry point.

Entry points include:

- REST and MVC endpoints
- Message listeners
- Scheduled jobs
- Startup hooks
- Domain or application event listeners
- Batch jobs or command handlers

For each critical flow, trace:

1. Entry point
2. Authentication and authorization checks
3. Validation and deserialization
4. Controller or handler logic
5. Service orchestration
6. Transaction boundaries
7. Repository or query layer
8. Database or cache access
9. External API or queue interaction
10. Response shaping, event publication, logging, retries, and failure paths

Mark each edge as either:

- confirmed, when directly evidenced in code or configuration
- probable, when inferred from naming, interface wiring, or partial references

Never present uncertain paths as guaranteed.

### 4. Dependency analysis

Analyze dependencies on multiple levels:

- Build dependencies from Maven or Gradle
- Module dependencies in multi-module repositories
- Package and layer dependencies
- Bean dependency graph
- Runtime interaction dependencies such as DB, queues, caches, object storage, SMTP, search engines, and third-party APIs

Flag dependency concerns including:

- cyclic package or module dependencies
- dead or unused dependencies
- outdated or risky libraries when visible in build files
- over-coupling between controllers, services, and repositories
- infrastructure concerns leaking into domain logic

### 5. Security and vulnerability review

Perform a Spring Boot focused security review.

Check at minimum for:

- missing or inconsistent authorization
- endpoints exposed without intended protection
- ownership or tenancy checks missing on resource access
- insecure direct object reference patterns
- weak or missing validation
- unsafe mass assignment or overbinding
- actuator exposure and management endpoint risk
- dangerous CORS settings
- insecure deserialization patterns
- secrets or credentials in code or config
- weak JWT handling or custom auth filter mistakes
- error messages leaking sensitive details
- unsafe file upload or path handling
- SSRF, open redirect, command execution, or template injection patterns where applicable
- unbounded queries or denial-of-service risk from expensive endpoints
- insufficient rate limiting or abuse controls when relevant
- missing audit logging for privileged actions

Use repository evidence. Do not claim exploitation unless there is clear support.

### 6. Data and transaction review

Look for correctness and consistency issues such as:

- `@Transactional` on the wrong layer
- self-invocation causing transaction annotations to be bypassed
- long-running transactions
- mixing DB writes with outbound network calls in one transaction
- event publication before commit or missing idempotency
- lazy-loading outside transaction
- N+1 query patterns
- entity leakage to API responses
- missing pagination or unbounded fetches
- race conditions around updates, retries, or scheduled jobs

### 7. Code quality and maintainability review

Review the codebase as a staff-level architecture and code reviewer.

Check for:

- controllers containing business logic
- services acting as god classes
- repositories accessed directly from controllers
- duplicate business logic across services
- poor package boundaries
- circular dependencies
- large utility classes hiding domain logic
- exception handling inconsistency
- weak test coverage in critical areas when tests exist
- brittle configuration patterns
- logging that is too noisy, too sparse, or leaks secrets
- poor naming that obscures the domain model

Prioritize flaws that are important, not cosmetic.

### 8. Operational and production-readiness review

Inspect runtime and deployment concerns when files exist.

Check for:

- profile separation between local, test, staging, and production
- unsafe defaults in `application.yml`
- exposed Actuator endpoints
- missing health/readiness/liveness concerns
- missing timeout, retry, or circuit-breaker patterns on outbound integrations
- poor observability, tracing, or metrics coverage
- misconfigured containers or Kubernetes manifests
- excessive privileges in deployment config
- mismatch between code assumptions and deployment manifests

### 9. Diagram generation

Generate concise, readable diagrams in Mermaid.

Required diagrams:

- **Application map**: component or module interaction diagram showing controllers, services, repositories, databases, caches, queues, external APIs, and security boundary components.
- **End-to-end flow diagram**: sequence diagram for at least one critical business flow and one authentication or security-sensitive flow when applicable.

Diagram rules:

- Avoid giant unreadable graphs.
- Prefer one high-level architecture diagram and several focused sequence diagrams.
- Group related components where useful.
- Distinguish internal services from external systems.
- Reflect security or transaction boundaries where relevant.

### 10. Output expectations

Produce a structured analysis with these sections:

1. Repository overview
2. Detected architecture and major components
3. Entry points inventory
4. Dependency analysis
5. Security model and trust boundaries
6. Critical end-to-end flows
7. Important flaws and vulnerabilities
8. Code review findings and maintainability issues
9. Operational risks
10. Recommended improvements and remediation backlog
11. Mermaid diagrams

## Recommended workflow

1. Scan repository structure and build files.
2. Inventory Spring components, endpoints, jobs, listeners, entities, repositories, and clients.
3. Build a graph of dependencies and interactions.
4. Trace the most important business and security-sensitive flows.
5. Review configuration and operational files.
6. Identify critical flaws with severity and confidence.
7. Generate diagrams and a prioritized remediation plan.

## Output style

- Be precise and evidence-based.
- Prefer concise technical language over vague prose.
- For every important flaw, include why it matters.
- Rank findings by severity and impact.
- Suggest practical improvements, not just abstract best practices.
- Distinguish between confirmed findings, probable findings, and open questions.

## Finding format

For each significant finding, include:

- Title
- Severity: critical, high, medium, or low
- Confidence: high, medium, or low
- Category: security, architecture, correctness, performance, operations, maintainability, or testing
- Evidence: files, classes, methods, annotations, config keys, or dependency declarations
- Impact: what can go wrong and who is affected
- Recommendation: concrete fix or improvement

## Preferred artifact set

When possible, generate artifacts such as:

- `analysis/repository-overview.md`
- `analysis/entrypoints.md`
- `analysis/dependencies.md`
- `analysis/security-review.md`
- `analysis/code-review.md`
- `analysis/remediation-plan.md`
- `analysis/diagrams/application-map.mmd`
- `analysis/diagrams/end-to-end-flow.mmd`

## Additional checks to include

Add these extra checks because they often matter in real Spring Boot systems:

- Profile drift: behavior or security changes across `application-*.yml`
- Shadow endpoints: old controllers or admin routes still present but no longer documented
- DTO versus entity separation in API boundaries
- Missing input size limits for uploads, payloads, and queries
- Retry storms or duplicated message handling in async flows
- Scheduled jobs without locking or idempotency
- Cache invalidation blind spots
- Multitenancy boundary leaks if tenant context exists
- Audit trail gaps for privileged or financial operations
- Weak domain invariants enforced only in controllers
- Test blind spots around security, transactions, and failure paths
- Startup behavior risks in runners or migrations
- Dependency pinning or unmaintained libraries when clearly visible

## What to avoid

- Do not review only a handful of files and claim repository-wide understanding.
- Do not generate a giant monolithic diagram with every class.
- Do not confuse framework boilerplate with business-critical logic.
- Do not mark speculative issues as confirmed vulnerabilities.
- Do not focus on trivial style nits while missing architectural or security flaws.

## Success criteria

This skill succeeds when a new engineer can use the output to understand:

- what the application does
- how requests and background jobs move through the system
- where the major dependencies and external integrations are
- which flaws are most important
- what should be fixed first

## Example invocation

Use this skill when asked to:

- scan a Spring Boot repository and explain how it works
- map full code flow through the application
- analyze dependencies and component interactions
- review the codebase for vulnerabilities and critical flaws
- generate architecture and sequence diagrams
- produce a prioritized code review and remediation plan
