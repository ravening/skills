---
name: analyzing-spring-boot-codebases
description: Analyzes Java Spring Boot repositories to map architecture, trace end-to-end request flow, review dependencies, identify vulnerabilities, and produce a prioritized audit report. Use when inspecting a Spring Boot codebase, performing backend code review, or running a security and architecture assessment.
---

# Analyzing Spring Boot Codebases

Use this skill when the repository is a Java Spring Boot application and the goal is to understand how it works, review design and dependencies, identify security risks, and generate diagrams plus a structured audit report.

## Outputs

Produce these outputs unless the user asks for a narrower scope:

- `spring-boot-audit-report.md`
- A Mermaid component diagram showing application structure
- A Mermaid sequence diagram showing one representative end-to-end flow
- A prioritized findings list using P0, P1, P2, and P3 severity

## Workflow

Copy this checklist into the working notes and keep it updated:

```text
Spring Boot Audit Progress
- [ ] Step 1: Detect build system and project structure
- [ ] Step 2: Inventory modules, layers, and endpoints
- [ ] Step 3: Review dependencies and platform versions
- [ ] Step 4: Map architecture and representative request flow
- [ ] Step 5: Run security and vulnerability review
- [ ] Step 6: Run code quality and maintainability review
- [ ] Step 7: Review configuration and tests
- [ ] Step 8: Generate diagrams and final report
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

Use [reference/architecture.md](reference/architecture.md).

Generate:

- One Mermaid component diagram with actual discovered controllers, services, repositories, data stores, message brokers, caches, and external systems
- One Mermaid sequence diagram for a representative request from entry to response, including auth, validation, service logic, persistence, integration calls, and error handling

Prefer a business-critical or structurally rich endpoint for the sequence diagram.

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

Rules:

- Include file paths and line numbers for findings whenever possible
- Prefer concise evidence over vague statements
- Use severity levels P0 to P3
- Pair each important flaw with a specific remediation
- When useful, include before/after code snippets
- End with the most important architectural and security improvements first

## Guardrails

- Use forward-slash file paths.
- Keep terminology consistent: endpoint, controller, service, repository, entity, DTO, mapper, filter, config, vulnerability, finding.
- Prefer constructor injection over field injection in recommendations.
- Prefer DTOs at API boundaries over exposing entities directly.
- Prefer confirmed evidence over assumptions.
- If a check cannot be completed, mark it as unknown and say why.

## Additional references

- Dependency review: [reference/dependencies.md](reference/dependencies.md)
- Architecture mapping: [reference/architecture.md](reference/architecture.md)
- Security review: [reference/security.md](reference/security.md)
- Code review: [reference/code-review.md](reference/code-review.md)
- Report format: [reference/report-template.md](reference/report-template.md)
- Optional deep checks: [reference/extended-checks.md](reference/extended-checks.md)
