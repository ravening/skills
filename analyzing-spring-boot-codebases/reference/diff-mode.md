# Diff Mode (PR-Scoped Review)

## Scope

This reference is loaded only when the skill is invoked against a pull request rather than a full codebase. It reframes the rest of the skill so that analysis stays bounded to the changes under review and their immediate blast radius. It does NOT replace any sibling reference; it narrows them. In Diff Mode only the subset of each sibling's checks that survives PR scoping is meaningful.

Cross-references: [reference/architecture.md](architecture.md), [reference/code-review.md](code-review.md), [reference/dependencies.md](dependencies.md), [reference/security.md](security.md), [reference/data-layer.md](data-layer.md), [reference/cloud-native.md](cloud-native.md), [reference/observability-and-resilience.md](observability-and-resilience.md), [reference/api-and-contracts.md](api-and-contracts.md), [reference/extended-checks.md](extended-checks.md), [reference/azure-resilience.md](azure-resilience.md), [reference/report-template.md](report-template.md).

## Review areas

- scoping the audit to the PR diff and its immediate blast radius
- deciding which sibling reference checks remain meaningful at PR scope and which do not
- summarizing impact at PR scope rather than at full-repository scope

## Scoping analysis to changed files only

Begin by reading the PR diff and building a concrete list of changed files, grouped by kind: production source, tests, build files, configuration (`application.yml`, `application.properties`), database migrations (Flyway, Liquibase), Dockerfiles, Kubernetes manifests, Helm charts, CI pipeline definitions, and documentation. The list of changed files is the only entry point into the codebase for the rest of the review; do not walk the repository tree as if performing a full audit.

From the changed-files list, expand outward to the **immediate blast radius** only. Immediate blast radius means direct callers and direct callees of changed code, the controllers that route to a changed service, the service that owns a changed repository, the entities reachable from a changed `@Entity`, the configuration keys consumed by a changed `@ConfigurationProperties` class, and the CI jobs that exercise a changed Dockerfile or manifest. One hop is the rule. Do not chase transitive callers through the whole repository; that is full-audit behavior, not Diff Mode.

Limit every sibling reference's checks — architecture, code review, security, data layer, cloud native, observability and resilience, API and contracts — to that scoped surface. If a check requires inputs that fall outside the changed files plus their immediate blast radius, the check does not run in Diff Mode. Flag the gap in the report instead of silently producing an under-scoped finding.

When the PR touches build files (`pom.xml`, `build.gradle`, `build.gradle.kts`, lockfiles), treat the changed dependency declarations themselves as in-scope and skip the broader dependency-tree review described in [reference/dependencies.md](dependencies.md).

When the PR touches a database migration, treat that migration plus the entities it affects as in-scope for the migration-safety guidance in [reference/data-layer.md](data-layer.md), and skip exhaustive ER-diagram regeneration described in [reference/architecture.md](architecture.md).

## Which checks remain meaningful in Diff Mode and which do not

For each sibling reference, the following describes what stays in scope at PR level and what is deferred to a full audit. Findings that depend on the deferred checks should be marked as out-of-scope-for-PR rather than reported speculatively.

**[reference/architecture.md](architecture.md).** In scope: per-file structural changes, layering violations introduced by the diff (controller calling repository directly, service skipped, package boundaries crossed), and small per-flow sequence diagrams limited to the touched flow. Out of scope: full architecture mapping, regenerating the full component diagram, exhaustive ER diagrams, deployment-diagram regeneration when no deployment artifact changed, and the full diagram set described for whole-repository audits.

**[reference/code-review.md](code-review.md).** In scope: maintainability findings on changed methods and classes — naming, cohesion, complexity, error-handling shape, test coverage of changed lines. Out of scope: repo-wide style harmonization, dead-code sweeps across unchanged modules, and refactoring proposals that would require touching files the PR does not touch.

**[reference/dependencies.md](dependencies.md).** In scope: changed dependency declarations in `pom.xml`, `build.gradle`, `build.gradle.kts`, and lockfiles — version bumps, additions, removals, scope changes, plugin changes. Out of scope: full dependency-tree review beyond the changed dependency declarations, and reviews of unchanged transitive dependencies.

**[reference/security.md](security.md).** In scope: every changed file that touches authentication, authorization, token handling, session management, secrets, deserialization, input binding, file upload, outbound HTTP, actuator surface, or template rendering — these are security-sensitive surfaces and changes to them are always meaningful at PR scope. Out of scope: a fresh repo-wide OWASP sweep over files the PR does not touch.

**[reference/data-layer.md](data-layer.md).** In scope: changed migration files (Flyway, Liquibase), changed `@Entity` classes, changed repository interfaces, changed fetch strategies and `EntityGraph` declarations, and changed HikariCP or `open-in-view` configuration. Out of scope: a repo-wide review of unchanged entities, unchanged caches, and unchanged dialect choices.

**[reference/cloud-native.md](cloud-native.md).** In scope: changed Dockerfiles, changed Kubernetes manifests, changed Helm values, changed network policies, changed `securityContext`, changed probes, changed secrets-management wiring. Out of scope: re-reviewing an unchanged Dockerfile or unchanged manifests just because the PR touched application code.

**[reference/observability-and-resilience.md](observability-and-resilience.md).** In scope: changed Resilience4j configuration, changed retry / circuit-breaker / bulkhead / rate-limiter / time-limiter declarations, changed logging configuration, MDC handling on changed async paths, and metrics or tracing wiring on changed integrations. Out of scope: a full SLO posture review across unchanged services.

**[reference/api-and-contracts.md](api-and-contracts.md).** In scope: changed controllers, changed request and response DTOs, changed OpenAPI annotations, changed versioning, changed idempotency-key handling, changed throttling config, and any GraphQL or gRPC schema or service definition the PR touches. Out of scope: completeness review of OpenAPI documentation for endpoints the PR does not touch.

**[reference/extended-checks.md](extended-checks.md).** In scope: only the subset of deep-review topics that intersects the changed files. Out of scope: invoking extended checks as a whole-repository sweep from within Diff Mode.

**[reference/azure-resilience.md](azure-resilience.md).** In scope: changes that touch Azure SDK dependencies, `azure.*` or `spring.cloud.azure.*` configuration, ARM or Bicep templates, `azurerm_*` or `azapi_*` Terraform, AKS manifests, App Service configuration, or CI steps that invoke Azure tooling — and the Azure flow segments those changes affect. Out of scope: regenerating the full end-to-end Azure flow diagram when the PR only changes one Azure boundary, and rebuilding the entire breaking-point catalog when the PR's risk surface is a single service.

**[reference/report-template.md](report-template.md).** Stays in scope unchanged; the PR-scoped audit still uses the same report shape, just with narrower content.

## Summarizing impact at PR scope

The audit summary in Diff Mode describes the **change's** risk surface, not the repository's. Express findings as "this PR introduces", "this PR removes", "this PR widens", or "this PR narrows" rather than as repository-wide statements. State the blast radius reached during scoping (which controllers, services, repositories, manifests, or Azure boundaries the change touches) so reviewers can verify the scope was correct.

When a check could not run because its inputs sit outside the scoped surface, list it explicitly under an out-of-scope-for-PR note rather than silently skipping it. This keeps the PR review honest and tells reviewers what a follow-up full audit would still need to cover.

Carry forward severities from the rest of the skill without inflation: a P2 in a full audit is a P2 in Diff Mode unless the diff itself changes the trigger condition. Do not raise severity simply because a finding showed up in a PR.

## Severity guidance

### P0
- the PR introduces a confirmed injection, auth bypass, broken access control, or remote-code-execution path on a touched endpoint
- the PR removes an existing security control on a sensitive surface (authentication filter, CSRF protection on a session flow, deserialization safeguard, secret management) without an equivalent replacement
- the PR commits a production secret, private key, or credential into the repository

### P1
- the PR widens authorization (`permitAll` added, role check removed, CORS opened) on an endpoint that previously enforced it
- the PR weakens a database migration's safety properties (drops a column without backfill, renames a column inline, adds a non-concurrent index on a large table) and the migration ships in this PR
- the PR changes a Dockerfile or Kubernetes manifest in a way that drops `runAsNonRoot`, removes resource limits, or removes a probe that was previously enforced
- the PR changes Resilience4j configuration in a way that disables retry, circuit breaker, bulkhead, or time limiter on a critical integration

### P2
- the PR introduces a new endpoint without OpenAPI documentation, without input validation at the boundary, or without idempotency for a mutating operation
- the PR introduces a new outbound HTTP client without timeout, retry, or circuit-breaker configuration
- the PR changes a DTO or entity serialization shape in a way that exposes additional fields without a clear need
- the PR changes structured logging or MDC propagation on an async path in a way that breaks correlation IDs

### P3
- maintainability and clarity findings on changed methods and classes (naming, cohesion, complexity)
- missing or weakened tests for the changed lines
- documentation drift between changed code and changed OpenAPI annotations
- minor hardening opportunities on touched files that were already deferred at full-audit scope

## Evidence rules

Use PR-relative locations. For every finding include:

- file path as it appears in the PR
- line number in the PR's post-change file
- the diff line marker (`+`, `-`, or context line) when it clarifies whether the issue is introduced, removed, or merely surfaced by the change
- why the issue matters at PR scope, including the blast radius reached during scoping
- exact remediation expressed as a change the author can make inside this PR

When a finding cannot be tied to a specific changed line — for example, a missing check that should have been added but was not — cite the file path and the nearest anchor line in the PR, and state that the remediation requires adding code rather than modifying a specific line.
